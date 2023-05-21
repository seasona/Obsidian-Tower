Workqueue的逻辑其实并不复杂，但网上的资料讲的并不清楚，主要是各个结构体的作用都叙述的很模糊，让人云里雾里，所以写这篇文章解析一下，主要分析的是Bound这种情况。

![[workqueue.drawio.svg]]

开局一张大图，Workqueue主要有以下的几个重要结构：
- Work：实际执行的任务，会把自己挂在worker_pool的worklist中
- Worker：就是一个内核线程，会从对应的worker_pool的worklist中取出任务并运行
- Worker_pool(Per_CPU)：顾名思义，就是管理Worker池，每个CPU会有两个
- pool_workqueue(Per_CPU)：负责连接workqueue和pool_workqueue，当workqueue创建时，会对每个CPU分配一个pool_workqueue
- workqueue：工作队列，其实主要是负责找到对应的pool

## 工作队列的创建

工作队列的创建由`alloc_workqueue`开始，它是`__alloc_workqueue_key`的封装

```c
struct workqueue_struct *__alloc_workqueue_key(const char *fmt,
					       unsigned int flags,
					       int max_active,
					       struct lock_class_key *key,
					       const char *lock_name, ...)
{
	size_t tbl_size = 0;
	va_list args;
	struct workqueue_struct *wq;
	struct pool_workqueue *pwq;

	/* init wq */
	...

	if (alloc_and_link_pwqs(wq) < 0)
		goto err_free_wq;

	if (wq_online && init_rescuer(wq) < 0)
		goto err_destroy;

	if ((wq->flags & WQ_SYSFS) && workqueue_sysfs_register(wq))
		goto err_destroy;

	mutex_lock(&wq_pool_mutex);

	mutex_lock(&wq->mutex);
	for_each_pwq(pwq, wq)
		pwq_adjust_max_active(pwq);
	mutex_unlock(&wq->mutex);

	list_add_tail_rcu(&wq->list, &workqueues);

	mutex_unlock(&wq_pool_mutex);

	return wq;

err_free_wq:
	free_workqueue_attrs(wq->unbound_attrs);
	kfree(wq);
	return NULL;
err_destroy:
	destroy_workqueue(wq);
	return NULL;
}
```

`__alloc_workqueue_key`中进行一些初始化分配工作，主要的重点在`alloc_and_link_pwqs`这个函数：

```c
static int alloc_and_link_pwqs(struct workqueue_struct *wq)
{
	bool highpri = wq->flags & WQ_HIGHPRI;
	int cpu, ret;

	if (!(wq->flags & WQ_UNBOUND)) {
		wq->cpu_pwqs = alloc_percpu(struct pool_workqueue);
		if (!wq->cpu_pwqs)
			return -ENOMEM;

		for_each_possible_cpu(cpu) {
			struct pool_workqueue *pwq =
				per_cpu_ptr(wq->cpu_pwqs, cpu);
			struct worker_pool *cpu_pools =
				per_cpu(cpu_worker_pools, cpu);

			init_pwq(pwq, wq, &cpu_pools[highpri]);

			mutex_lock(&wq->mutex);
			link_pwq(pwq);
			mutex_unlock(&wq->mutex);
		}
		return 0;
	} else if (wq->flags & __WQ_ORDERED) {
		ret = apply_workqueue_attrs(wq, ordered_wq_attrs[highpri]);
		/* there should only be single pwq for ordering guarantee */
		WARN(!ret && (wq->pwqs.next != &wq->dfl_pwq->pwqs_node ||
			      wq->pwqs.prev != &wq->dfl_pwq->pwqs_node),
		     "ordering guarantee broken for workqueue %s\n", wq->name);
		return ret;
	} else {
		return apply_workqueue_attrs(wq, unbound_std_wq_attrs[highpri]);
	}
}
```

可以看到，如果是非UNBOUND类型的workqueue，会对每个CPU都分配一个对应的pool_workqueue作为cpu_pwqs成员，然后对每个CPU，把work->cpu_pwqs和每个CPU对应的worker_pool连接在一起，这样通过workqueue就可以找到每个CPU的worker_pool了

```ad-note
这也是为什么内核建议驱动使用系统默认的workqueue而不是新建，每个workqueue都需要在每个CPU建立一个pool_workqueue，还是有一定的开销的
```

## 工作的分配与运行

当workqueue创建完毕，就可以进行工作的分配与运行，可以通过`queue_work_on`选择将work运行在特定的CPU上，

```c
static void __queue_work(int cpu, struct workqueue_struct *wq,
			 struct work_struct *work)
{
	struct pool_workqueue *pwq;
	struct worker_pool *last_pool;
	struct list_head *worklist;
	unsigned int work_flags;
	unsigned int req_cpu = cpu;

	/* if draining, only works from the same workqueue are allowed */
	if (unlikely(wq->flags & __WQ_DRAINING) &&
	    WARN_ON_ONCE(!is_chained_work(wq)))
		return;
retry:
	/* pwq which will be used unless @work is executing elsewhere */
	if (wq->flags & WQ_UNBOUND) {
		if (req_cpu == WORK_CPU_UNBOUND)
			cpu = wq_select_unbound_cpu(raw_smp_processor_id());
		pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
	} else {
		if (req_cpu == WORK_CPU_UNBOUND)
			cpu = raw_smp_processor_id();
		pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
	}

	...

	if (WARN_ON(!list_empty(&work->entry))) {
		spin_unlock(&pwq->pool->lock);
		return;
	}

	pwq->nr_in_flight[pwq->work_color]++;
	work_flags = work_color_to_flags(pwq->work_color);

	if (likely(pwq->nr_active < pwq->max_active)) {
		trace_workqueue_activate_work(work);
		pwq->nr_active++;
		worklist = &pwq->pool->worklist;
		if (list_empty(worklist))
			pwq->pool->watchdog_ts = jiffies;
	} else {
		work_flags |= WORK_STRUCT_DELAYED;
		worklist = &pwq->delayed_works;
	}

	debug_work_activate(work);
	insert_work(pwq, work, worklist, work_flags);

	spin_unlock(&pwq->pool->lock);
}
```

首先，通过wq->cpu_pwqs找到cpu对应的pool_worker，然后获得pool_worker的worklist工作集

```c
static void insert_work(struct pool_workqueue *pwq, struct work_struct *work,
			struct list_head *head, unsigned int extra_flags)
{
	struct worker_pool *pool = pwq->pool;

	/* we own @work, set data and link */
	set_work_pwq(work, pwq, extra_flags);
	list_add_tail(&work->entry, head);
	get_pwq(pwq);

	/*
	 * Ensure either wq_worker_sleeping() sees the above
	 * list_add_tail() or we see zero nr_running to avoid workers lying
	 * around lazily while there are works to be processed.
	 */
	smp_mb();

	if (__need_more_worker(pool))
		wake_up_worker(pool);
}
```

通过`insert_work`函数将work插入worklist工作集，并通过`wake_up_worker`唤醒worker

```c
static void wake_up_worker(struct worker_pool *pool)
{
	struct worker *worker = first_idle_worker(pool);

	if (likely(worker))
		wake_up_process(worker->task);
}
```

## 工作线程

```c
static int worker_thread(void *__worker)
{
	struct worker *worker = __worker;
	struct worker_pool *pool = worker->pool;

	/* tell the scheduler that this is a workqueue worker */
	set_pf_worker(true);
woke_up:
	spin_lock_irq(&pool->lock);

	/* am I supposed to die? */
	if (unlikely(worker->flags & WORKER_DIE)) {
		spin_unlock_irq(&pool->lock);
		WARN_ON_ONCE(!list_empty(&worker->entry));
		set_pf_worker(false);

		set_task_comm(worker->task, "kworker/dying");
		ida_simple_remove(&pool->worker_ida, worker->id);
		worker_detach_from_pool(worker);
		kfree(worker);
		return 0;
	}

	worker_leave_idle(worker);
recheck:
	/* no more worker necessary? */
	if (!need_more_worker(pool))
		goto sleep;

	/* do we need to manage? */
	if (unlikely(!may_start_working(pool)) && manage_workers(worker))
		goto recheck;

	do {
		struct work_struct *work =
			list_first_entry(&pool->worklist,
					 struct work_struct, entry);

		pool->watchdog_ts = jiffies;

		if (likely(!(*work_data_bits(work) & WORK_STRUCT_LINKED))) {
			/* optimization path, not strictly necessary */
			process_one_work(worker, work);
			if (unlikely(!list_empty(&worker->scheduled)))
				process_scheduled_works(worker);
		} else {
			move_linked_works(work, &worker->scheduled, NULL);
			process_scheduled_works(worker);
		}
	} while (keep_working(pool));

	worker_set_flags(worker, WORKER_PREP);
sleep:
	worker_enter_idle(worker);
	__set_current_state(TASK_IDLE);
	spin_unlock_irq(&pool->lock);
	schedule();
	goto woke_up;
}
```

Worker工作线程创建之后会通过schedule让出调度，当work插入worklist时将其唤醒，进入到`process_one_work`函数中执行work->func，也就是设定的工作函数。
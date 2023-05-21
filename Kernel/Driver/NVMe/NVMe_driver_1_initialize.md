![[nvme_queue.png]]

NVMe驱动主要分为两个模块：

- nvme-core：实现nvme内部命令，比如admin命令
- nvme：依赖于nvme-core，实现pci相关的初始化操作和IO传输

nvme-core模块实现的都是nvme的内部操作，最重要的IO传输路径相关操作是在nvme模块，所以我们主要关注nvme模块。

## 初始化

初始化从nvme_probe函数开始，在经过一系列常规的pci设备初始化操作后，将主要的nvme相关的初始化操作放入`nvme_reset_work`工作中，通过[[Workqueue]]机制，异步完成。

```c
nvme_reset_work
├── nvme_pci_enable (初始化pci相关)
│   ├── nvme_pci_configure_admin_queue (初始化admin queue)
│   └── nvme_alloc_admin_tags (向blk-mq注册admin queue对应tags)
├── nvme_setup_io_queues
│   ├── nvme_remap_bar (remap bar)
│   └── nvme_create_io_queues (每个cpu分配一个io queue) 
│       ├── nvme_alloc_queue 
│       └── nvme_create_queue
│           ├── adapter_alloc_cq (通过admin命令创建cq)
│           ├── adapter_alloc_sq (通过admin命令创建sq)
│           ├── nvme_init_queue (初始化doorbell)
│           └── queue_request_irq (每个cpu分配一个io中断)
└── nvme_dev_add
```

nvme中存在三种io queue：
- write queue：设置后，读写将分为两个queue完成，一读一写
- poll queue：通过poll来读写数据的queue
- default queue：常规硬件queue，读写双向

一般来说，我们只要关注defualt queue即可。

### 初始化admin queue

Nvme协议中，IO queue是通过给Admin queue发送admin命令创建的，颇有一种“一生二，二生三，三生万物”的思想。

因此，对于queue的初始化，首先是初始化admin queue。admin queue的初始化对于驱动层来说没有什么特别的，跟io queue不同的地方在于会直接将DMA物理地址写入Nvme设备的Controller register。

![[register.png]]

接下来介绍queue初始化中的一些基本操作。

### 获取io queue个数

理论上来说，一个CPU分配一个IO queue的性能肯定是最好的，在初始化的时候，nvme驱动会通过admin命令`set_features`设置硬件的feature为CPU个数。然而，可能存在NVMe硬件io queue的个数比CPU个数少的情况，此时会设置失败，返回实际支持的硬件io queue个数，此时nvme驱动只会分配对应个数的io queue。

```c
int nvme_set_queue_count(struct nvme_ctrl *ctrl, int *count)
{
	u32 q_count = (*count - 1) | ((*count - 1) << 16);
	u32 result;
	int status, nr_io_queues;

	/*
	 * Set nvme hardware to setup q_count io queue, however if hardware 
	 * can't support so many queue, it will return maximum queue number.
	 */
	status = nvme_set_features(ctrl, NVME_FEAT_NUM_QUEUES, q_count, NULL, 0,
			&result);
	if (status < 0)
		return status;

	if (status > 0) {
		dev_err(ctrl->device, "Could not set queue count (%d)\n", status);
		*count = 0;
	} else {
		nr_io_queues = min(result & 0xffff, result >> 16) + 1;
		*count = min(*count, nr_io_queues);
	}

	return 0;
}
```

![[feature.png]]

### 初始化bar

```c
static int nvme_remap_bar(struct nvme_dev *dev, unsigned long size)
{
	struct pci_dev *pdev = to_pci_dev(dev->dev);

	if (size <= dev->bar_mapped_size)
		return 0;
	if (size > pci_resource_len(pdev, 0))
		return -ENOMEM;
	if (dev->bar)
		iounmap(dev->bar);
	dev->bar = ioremap(pci_resource_start(pdev, 0), size);
	if (!dev->bar) {
		dev->bar_mapped_size = 0;
		return -ENOMEM;
	}
	dev->bar_mapped_size = size;
	dev->dbs = dev->bar + NVME_REG_DBS;

	return 0;
}
```

通过ioremap映射nvme设备bar空间，此时访问dev->bar等于通过mmio访问nvme相关寄存器。特别的，一个queue对应两个doorbell寄存器，sq->tail和cq->head。

![[doorbell.png]]

### 分配queue使用的内存

先看一下nvme_queue结构体，非常简单，初始化也就是申请queue深度的内存用于dma即可。
- sq_cmds: sq的虚拟地址，CPU使用
- cqes：cq的虚拟地址，CPU使用
- sq_dma_addr：sq的DMA物理地址，设备使用
- cq_dma_addr：cq的DMA物理地址，设备使用

```c
struct nvme_queue {
	struct nvme_dev *dev;
	spinlock_t sq_lock;
	void *sq_cmds;		/* va of sq, size: queue depth */
	 /* only used for poll queues: */
	spinlock_t cq_poll_lock ____cacheline_aligned_in_smp;
	struct nvme_completion *cqes;	/* va of cq, size: queue depth */
	dma_addr_t sq_dma_addr; /* physical dma address for sq */
	dma_addr_t cq_dma_addr; /* physical dma address for cq */
	u32 __iomem *q_db;		/* doorbell for queue */
	u32 q_depth;
	u16 cq_vector;
	u16 sq_tail;
	u16 last_sq_tail;
	u16 cq_head;
	u16 qid;
	u8 cq_phase;
	u8 sqes;
	unsigned long flags;
	u32 *dbbuf_sq_db;
	u32 *dbbuf_cq_db;
	u32 *dbbuf_sq_ei;
	u32 *dbbuf_cq_ei;
	struct completion delete_done;
};
```

### 通知后端queue的DMA地址

对于io queue，会通过admin命令`nvme_admin_create_cq`创建。

![[admin order.png]]

可以看到`adapter_alloc_cq`函数中，将cq对应的dma地址cq_dma_addr存入cmd->prp1，构建了一下admin cmd，通过`nvme_submit_sync_cmd`投递到admin sq中，这里是sync同步的，所以io queue创建完毕后才能执行接下来的操作。

```c
static int adapter_alloc_cq(struct nvme_dev *dev, u16 qid,
		struct nvme_queue *nvmeq, s16 vector)
{
	struct nvme_command c;
	int flags = NVME_QUEUE_PHYS_CONTIG;

	if (!test_bit(NVMEQ_POLLED, &nvmeq->flags))
		flags |= NVME_CQ_IRQ_ENABLED;

	/*
	 * Note: we (ab)use the fact that the prp fields survive if no data
	 * is attached to the request.
	 */
	memset(&c, 0, sizeof(c));
	c.create_cq.opcode = nvme_admin_create_cq;
	c.create_cq.prp1 = cpu_to_le64(nvmeq->cq_dma_addr);
	c.create_cq.cqid = cpu_to_le16(qid);
	c.create_cq.qsize = cpu_to_le16(nvmeq->q_depth - 1);
	c.create_cq.cq_flags = cpu_to_le16(flags);
	c.create_cq.irq_vector = cpu_to_le16(vector);

	return nvme_submit_sync_cmd(dev->ctrl.admin_q, &c, NULL, 0);
}
```

### 分配中断

nvme使用msix中断，初始化时`nvme_setup_irqs`函数会通过`pci_alloc_irq_vectors_affinity`分配(nr_io_queues + 1)个msix中断资源，再通过`pci_request_irq`将中断激活。

```c
static int queue_request_irq(struct nvme_queue *nvmeq)
{
	struct pci_dev *pdev = to_pci_dev(nvmeq->dev->dev);
	int nr = nvmeq->dev->ctrl.instance;

	if (use_threaded_interrupts) {
		return pci_request_irq(pdev, nvmeq->cq_vector, nvme_irq_check,
				nvme_irq, nvmeq, "nvme%dq%d", nr, nvmeq->qid);
	} else {
		return pci_request_irq(pdev, nvmeq->cq_vector, nvme_irq,
				NULL, nvmeq, "nvme%dq%d", nr, nvmeq->qid);
	}
}
```

可以看到，如果硬件资源比较富裕，会为每个CPU绑定一个硬件IO queue中断，另有一个admin queue中断。

![[irq.png]]

## 小结

本章主要介绍了NVMe驱动如何进行初始化的，NVMe比较特别的一点在于它很多操作是通过admin queue完成的，这让NVMe可以比较优雅的支持很多高级特性。

## IO路径

![[io_path.png]]

NVMe协议的IO路径如上图所示，sq和cq都是环形队列，位于host memory中，doorbell位于ssd内部寄存器，一次完整的IO路径如下：

1. bio往nvme驱动传递request，nvme驱动组装cmd
2. nvme驱动增加sq的tail指针，写入sq doorbell tail寄存器
3. nvme controller从[head, tail]拉取cmd进行处理
4. nvme controller写cq，填入completion entry
5. nvme controller发出nvme_irq中断
6. nvme驱动获取completion entry进行处理
7. nvme驱动更新head，写入cq doorbell head寄存器

### 写操作

NVMe驱动对接的上层blk-mq，我们以queue_rq出发，整个路径如下：

```c
nvme_mq_ops->queue_rq
└── nvme_queue_rq
    ├── nvme_setup_cmd (初始化cmd)
    ├── nvme_map_data (组装数据)
    │   ├── nvme_pci_setup_prps (admin queue使用prp)
    │   └── nvme_pci_setup_sgls (io queue使用sgl)
    └── nvme_submit_cmd
        ├── nvmeq->sq_tail++
        └── nvme_write_sq_db (kick doorbell)
```

其中关键在于`nvme_map_data`，这一步会提取request中的bio对应的DMA地址，并将其组织成nvme cmd中所需的数据格式，nvme支持两种数据传输格式：

 - prp：以页为粒度，admin queue均采用这种格式进行传输
 - sgl：可以链接任意长度，更加灵活，io queue采用这种方式进行传输

SGL和PRP本质的区别在于，一段数据空间，对PRP来说，它只能映射到一个个物理页，而对SGL来说，它可以映射到任意大小的连续物理空间，具有更大的灵活性，也能够描述更大的数据空间。

通过cmd.psdt表示采用哪种方式传输，具体细节如下图：

![[dptr.png]]

#### PRP格式

![[nvme-prp.drawio.svg]]

prp会通过两个prp指针指示，根据数据大小来进行组织：

1. datalen <= 4KB，prp1直接指向数据，prp2为空
2. 4KB < datalen <= 8KB，prp1和prp2分别指向数据
3. 8KB < datalen，prp1指向4KB数据，prp2以列表的形式指向剩下的数据

#### SGL格式

![[nvme-sgl.drawio.svg]]

sgl的链接方式如上图，其实和linux中的scatterlist很相似，DMA操作的地址都是4字节对齐的，所以内核中的scatterlist通过最后1个字节来表示是Chain还是Last，nvme类似这种方式

```c
static blk_status_t nvme_pci_setup_sgls(struct nvme_dev *dev,
		struct request *req, struct nvme_rw_command *cmd, int entries)
{
	struct nvme_iod *iod = blk_mq_rq_to_pdu(req);
	struct dma_pool *pool;
	struct nvme_sgl_desc *sg_list;
	struct scatterlist *sg = iod->sg;
	dma_addr_t sgl_dma;
	int i = 0;

	/* Set the transfer type as SGL */
	cmd->flags = NVME_CMD_SGL_METABUF;

    /* 如果只有一个entry, 直接填充data block descriptor */
	if (entries == 1) {
		nvme_pci_sgl_set_data(&cmd->dptr.sgl, sg);
		return BLK_STS_OK;
	}

	/* if entries < 16, use 256B memory pool */
	if (entries <= (256 / sizeof(struct nvme_sgl_desc))) {
		pool = dev->prp_small_pool;
		iod->npages = 0;
	} else {
		pool = dev->prp_page_pool;
		iod->npages = 1;
	}

	sg_list = dma_pool_alloc(pool, GFP_ATOMIC, &sgl_dma);
	if (!sg_list) {
		iod->npages = -1;
		return BLK_STS_RESOURCE;
	}

	nvme_pci_iod_list(req)[0] = sg_list;
	iod->first_dma = sgl_dma;

	/* 第一个desc是segment descriptor，用于指向数据页 */
	nvme_pci_sgl_set_seg(&cmd->dptr.sgl, sgl_dma, entries);

	do {
		/* 需要链接新一页，将最后一个设为segment descriptor */
		if (i == SGES_PER_PAGE) {
			struct nvme_sgl_desc *old_sg_desc = sg_list;
			struct nvme_sgl_desc *link = &old_sg_desc[i - 1];

			sg_list = dma_pool_alloc(pool, GFP_ATOMIC, &sgl_dma);
			if (!sg_list)
				goto free_sgls;

			i = 0;
			nvme_pci_iod_list(req)[iod->npages++] = sg_list;
			sg_list[i++] = *link;
			nvme_pci_sgl_set_seg(link, sgl_dma, entries);
		}

		nvme_pci_sgl_set_data(&sg_list[i++], sg);
		sg = sg_next(sg);
	} while (--entries > 0);

	return BLK_STS_OK;
free_sgls:
	nvme_free_sgls(dev, req);
	return BLK_STS_RESOURCE;
}
```

#### kick doorbell

当填充完毕cmd后，就会将cmd的64B填充到nvmeq->sq_cmds中，之后往对应的tail doorbell写入sq_tail即可

```c
static inline void nvme_write_sq_db(struct nvme_queue *nvmeq, bool write_sq)
{
	if (!write_sq) {
		u16 next_tail = nvmeq->sq_tail + 1;

		if (next_tail == nvmeq->q_depth)
			next_tail = 0;
		if (next_tail != nvmeq->last_sq_tail)
			return;
	}

	if (nvme_dbbuf_update_and_check_event(nvmeq->sq_tail,
			nvmeq->dbbuf_sq_db, nvmeq->dbbuf_sq_ei))
		writel(nvmeq->sq_tail, nvmeq->q_db);
	nvmeq->last_sq_tail = nvmeq->sq_tail;
}

static void nvme_submit_cmd(struct nvme_queue *nvmeq, struct nvme_command *cmd,
			    bool write_sq)
{
	spin_lock(&nvmeq->sq_lock);
	memcpy(nvmeq->sq_cmds + (nvmeq->sq_tail << nvmeq->sqes),
	       cmd, sizeof(*cmd));
	if (++nvmeq->sq_tail == nvmeq->q_depth)
		nvmeq->sq_tail = 0;
	nvme_write_sq_db(nvmeq, write_sq);
	spin_unlock(&nvmeq->sq_lock);
}
```

### 读操作

```c
nvme_irq（msix中断）
└── nvme_process_cq
    └── nvme_handle_cqe
        ├── nvme_find_rq
        ├── nvme_try_complete_req	
        │   └── blk_mq_complete_request_remote？
        │       ├── blk_done_softirq	
        │       │   └── nvme_pci_complete_rq
        │       │       └── nvme_end_req
        │       │           └── blk_mq_end_request
        │       └── smp_call_function_single_async 
        │           └── __ipi_send_mask
        │               └── generic_smp_call_function_single_interrupt
        │                   └── flush_smp_call_function_queue
        │                       └── __blk_mq_complete_request_remote
        │                           └── nvme_pci_complete_rq
        ├── nvme_update_cq_head
        └── nvme_ring_cq_doorbell
```

处理nvme response的流程如上，收到msix中断后，获取到对应的request->result，返回给bio处理即可。

#### phase tag

cq的处理有一个比较难理解的地方，不同于sq，tail可以通过doorbell告知后端，cq的tail不是直接由后端告知nvme driver的，而是通过completion entry中的phase tag指示该entry是否有效来间接完成。

![[cqe.png]]

![[phase tag.png]]

nvme driver从cq中取出cqe，如果cte的phase tag和nvme driver中维护的cq_phase一致，那么说明该cqe有效。

```c
/* We read the CQE phase first to check if the rest of the entry is valid */
static inline bool nvme_cqe_pending(struct nvme_queue *nvmeq)
{
	struct nvme_completion *hcqe = &nvmeq->cqes[nvmeq->cq_head];

	/* If phase tag equal cq_phase, means this entry is valid, 
	 * cq_phase will be reversed if head reach queue depth.
	 */
	return (le16_to_cpu(READ_ONCE(hcqe->status)) & 1) == nvmeq->cq_phase;
}
```

当cq_head到达队列最大深度时，nvme驱动会将cq_phase反转。后端nvme controller内部也会采取相同的操作。通过phase tag判断**剩余的cqe**是否有效。

```c
static inline void nvme_update_cq_head(struct nvme_queue *nvmeq)
{
	u32 tmp = nvmeq->cq_head + 1;

	if (tmp == nvmeq->q_depth) {
		nvmeq->cq_head = 0;
		nvmeq->cq_phase ^= 1;
	} else {
		nvmeq->cq_head = tmp;
	}
}
```

这一块很不好理解，举个例子：

1. 已知队列深度4096，nvme driver的cq_phase初始值为1，nvme controller初始化cq时会将所有的cqe全置0，新的cqe的phase tag会被设置为1，其余旧cqe的phase tag均为0，此时通过phase tag就可以判断cqe是否有效
3. 假设目前head和tail指向4094，此时nvme controller填充了3个cqe，那么这些cqe将为：
	1. index = 4094: phase tag = 1
	2. index = 4095: phase tag = 1
	3. index = 0: 已经超过了队列深度从头开始，phase tag = 0
4. nvme driver类似，当处理4094和4095时，cq_phase为1，说明这些cqe有效，而当处理0时，此时cq_phase也变为0，那么index=0的cqe可以验证为有效的
5. nvme driver往后遍历时，此时index=1的cqe的phase tag是1，则说明是无效的，那么nvme driver处理cq就到此为止了

#### remote CPU处理

```ad-question
如果硬件queue少于CPU个数，那么仅有部分CPU会挂上中断，如果发起request是其他的CPU，那么需要回到发起CPU的进程上下文，那么如何通知到该CPU进行complete处理？
```

[[NVMe_driver_1_initialize]]初始化获取queue个数就可能出现这种情况，事实上，中断处理流程中`blk_mq_complete_request_remote`会进行以下操作：

1.  如果中断对应的CPU和bio上下文一致，那么为local cpu，即发起bio的cpu和收到中断回应的是同一个CPU，那么直接软中断处理即可
2.  如果中断对应的CPU和bio上下文不一致，那么为remote cpu，会发一个ipi中断，将request交给发起bio的CPU进行处理，性能肯定是会相对差一些

```c
bool blk_mq_complete_request_remote(struct request *rq)
{
	WRITE_ONCE(rq->state, MQ_RQ_COMPLETE);

	/*
	 * For a polled request, always complete locallly, it's pointless
	 * to redirect the completion.
	 */
	if (rq->cmd_flags & REQ_HIPRI)
		return false;

	if (blk_mq_complete_need_ipi(rq)) {
		rq->csd.func = __blk_mq_complete_request_remote;
		rq->csd.info = rq;
		rq->csd.flags = 0;
		smp_call_function_single_async(rq->mq_ctx->cpu, &rq->csd);
	} else {
		if (rq->q->nr_hw_queues > 1)
			return false;
		blk_mq_trigger_softirq(rq);
	}

	return true;
}
```

## 小结

本章主要介绍了NVMe驱动的IO传输路径，其中比较特别的是phase tag这套机制，以一个比较新颖的方式告知前端，有机会可以和virtio-blk对比一下。
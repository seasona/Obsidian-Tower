Virtio is a para-virtual technology which transmit data between Linux virtio driver and virtio devices emulated by hypervisors like QEMU. 

At the start of story, we should do some initial job, Let's go!

## Get number of descriptors

At first, Linux driver should set up queue of virtio and alloc memory for them. But how do we know how many memory should virtio driver apply? it mainly depend on how many descriptors virtio device has. So driver must get number of descriptors first, we focus on virtio-pci device for simplify.

Typically, virtio driver read/write information from registers of emulated virtio devices. So we agreed with `VIRTIO_PCI_QUEUE_NUM` as descriptor number register.

```c
/* A 16-bit r/o queue size for the currently selected queue */
#define VIRTIO_PCI_QUEUE_NUM            12
```

When virtio device setup, it will call `virtio_add_queue` function to add queue, which set vring.num as number of descriptors. For example, virtio-serial device will add two queue for control which have 32 descriptors. 

```c
VirtQueue *virtio_add_queue(VirtIODevice *vdev, int queue_size,
                            void (*handle_output)(VirtIODevice *, VirtQueue *))
{
    int i;

    for (i = 0; i < VIRTIO_QUEUE_MAX; i++) {
        if (vdev->vq[i].vring.num == 0)
            break;
    }

    if (i == VIRTIO_QUEUE_MAX || queue_size > VIRTQUEUE_MAX_SIZE)
        abort();

    vdev->vq[i].vring.num = queue_size;
    vdev->vq[i].vring.align = VIRTIO_PCI_VRING_ALIGN;
    vdev->vq[i].handle_output = handle_output;
    vdev->vq[i].handle_aio_output = NULL;

    if (vdev->vq[i].vring.num > 256)
        fprintf(stdout, "queue size %d\n", vdev->vq[i].vring.num);

    return &vdev->vq[i];
}

/* virtio-serial */
/* control queue: host to guest */
vser->c_ivq = virtio_add_queue(vdev, 32, control_in);
/* control queue: guest to host */
vser->c_ovq = virtio_add_queue(vdev, 32, control_out);
```

Since virtio device has settled descriptor number, virtio driver can get it from register. In Linux virtio driver `setup_vq` function which will setup virtqueue:

```c
static struct virtqueue *setup_vq(struct virtio_pci_device *vp_dev,
				  struct virtio_pci_vq_info *info,
				  unsigned index,
				  void (*callback)(struct virtqueue *vq),
				  const char *name,
				  bool ctx,
				  u16 msix_vec)
{
	struct virtqueue *vq;
	u16 num;
	int err;
	u64 q_pfn;

	/* Select the queue we're interested in */
	iowrite16(index, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_SEL);

	/* Check if queue is either not available or already active. */
	num = ioread16(vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NUM);
	if (!num || ioread32(vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN))
		return ERR_PTR(-ENOENT);

	info->msix_vector = msix_vec;

	/* create the vring */
	vq = vring_create_virtqueue(index, num,
				    VIRTIO_PCI_VRING_ALIGN, &vp_dev->vdev,
				    true, false, ctx,
				    vp_notify, callback, name);
	if (!vq)
		return ERR_PTR(-ENOMEM);

	q_pfn = virtqueue_get_desc_addr(vq) >> VIRTIO_PCI_QUEUE_ADDR_SHIFT;

	/* activate the queue */
	iowrite32(q_pfn, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN);

	vq->priv = (void __force *)vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NOTIFY;

	...

	return vq;

out_deactivate:
	iowrite32(0, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN);
out_del_vq:
	vring_del_virtqueue(vq);
	return ERR_PTR(err);
}
```

Virtio driver write interest vq index to `VIRTIO_PCI_QUEUE_SEL`，this register is selector for which queue driver will operate.

```c
/* A 16-bit r/w queue selector */
#define VIRTIO_PCI_QUEUE_SEL            14
```

Then driver can get number of descriptor. Implementation in QEMU is:

```c
int virtio_queue_get_num(VirtIODevice *vdev, int n)
{
    return vdev->vq[n].vring.num;
}

/* virtio_ioport_read */
ret = virtio_queue_get_num(vdev, vdev->queue_sel);
```

## Alloc memory page for vring

Now that driver can perpare shared memory page for vring, specific size can caculated by `vring_size` function.

```c
static inline unsigned vring_size(unsigned int num, unsigned long align)
{
	return ((sizeof(struct vring_desc) * num + sizeof(__virtio16) * (3 + num)
		 + align - 1) & ~(align - 1))
		+ sizeof(__virtio16) * 3 + sizeof(struct vring_used_elem) * num;
}
```

Normally virtio driver will alloc two 4k pages, memory layout as following:

![[virtio.drawio.svg]]

This 8k memory is shared between guest and host, now driver should notify this memory address(GPA) to virtio device by writing register `VIRTIO_PCI_QUEUE_PFN`.

```c
/* A 32-bit r/w PFN for the currently selected queue */
#define VIRTIO_PCI_QUEUE_PFN            8
```

And virtio device can init virtqueue and conform memory layout.

```c
static void virtqueue_init(VirtQueue *vq)
{
    hwaddr pa = vq->pa;

    vq->vring.desc = pa;
    vq->vring.avail = pa + vq->vring.num * sizeof(VRingDesc);
    vq->vring.used = vring_align(vq->vring.avail +
                                 offsetof(VRingAvail, ring[vq->vring.num]),
                                 vq->vring.align);
}
```

Finally, guest and host can both modify the same memory page and transmit data by it.

```ad-note
Next section will be soon...
```

## Reference

- https://www.cnblogs.com/LoyenWang/p/14589296.html


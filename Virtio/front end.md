![[virtio-frontend.drawio.svg]]

## 前端通知后端

当前端填充并映射好数据到virtqueue时，会通过`virtqueue_kick`通知后端是哪个queue有新数据了

```
virtqueue_add_split
└── virtqueue_kick
    └── virtqueue_kick_prepare
        └── virtqueue_notify
            └── vp_notify
```

最终会调用到`vq_notify`，这个函数主要的功能就是往vq->priv写入标记queue的index

```c
/* Linux virtio driver */
/* the notify function used when creating a virt queue */
bool vp_notify(struct virtqueue *vq)
{
	/* we write the queue's selector into the notification register to
	 * signal the other end */
	iowrite16(vq->index, (void __iomem *)vq->priv);
	return true;
}

vq->priv = (void __force *)vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NOTIFY;
```

vq->priv会在初始化`setup_vq`时被设置为`VIRTIO_PCI_QUEUE_NOTIFY`对应的寄存器地址(PCI bar0)，往该地址写数据，会触发后端设备处理数据：

```c
/* QEMU */
void virtio_queue_notify_vq(VirtQueue *vq)
{
    if (vq->vring.desc && vq->handle_output) {
        VirtIODevice *vdev = vq->vdev;

        trace_virtio_queue_notify(vdev, vq - vdev->vq, vq);
        virtio_cache_flush(&vdev->cache);
        vq->handle_output(vdev, vq);
    }
}

void virtio_queue_notify(VirtIODevice *vdev, int n)
{
    virtio_queue_notify_vq(&vdev->vq[n]);
}
```
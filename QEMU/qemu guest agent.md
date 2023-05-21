## qemu命令行参数解析

![[qemu_guest_agent.0.png]]

可以看到QEMU启动时，参数中与qemu guest agent相关的分别是：

- chardev charchannel10：以监听模式打开一个socket作为server，socket文件路径为/dev/aliport-xxx
- virtserialport即virtio-console设备org.qemu.guest_agent.0，绑定了chardev charchannel10

## qemu guest agent如何与qemu进行通信

qemu通过event loop监听socket，当一些client向socket(/dev/aliport-xxx)写入数据时，该socket变成可读，此时会调用到`chr_read`将socket中的数据读出来，并通过virtqueue发给guest中的qemu guest agent。

这整个过程牵涉到[[qemu char]]设备的模拟

## virtio-console设备分析

下面分析一下virtio-console设备的相关函数：

### chr_can_read

`chr_can_read`函数用于判断virtio-console设备是否处于可读的状态

```c
/* Readiness of the guest to accept data on a port */
static int chr_can_read(void *opaque)
{
    VirtConsole *vcon = opaque;

    return virtio_serial_guest_ready(VIRTIO_SERIAL_PORT(vcon));
}
```

它的主要判断逻辑是通过`virtio_serial_guest_ready`完成的，该函数中主要判断一下virtqueue是否ready，通过`port->guest_connected`判断guest是否存活，如果判断都通过，会返回virtqueue能够传输的byte数(一般是4096)

```c
/*
 * Readiness of the guest to accept data on a port.
 * Returns max. data the guest can receive
 */
size_t virtio_serial_guest_ready(VirtIOSerialPort *port)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(port->vser);
    VirtQueue *vq = port->ivq;
    unsigned int bytes;

    if (!virtio_queue_ready(vq) ||
        !(vdev->status & VIRTIO_CONFIG_S_DRIVER_OK) ||
        virtio_queue_empty(vq)) {
        return 0;
    }
    if (use_multiport(port->vser) && !port->guest_connected) {
        return 0;
    }
    virtqueue_get_avail_bytes(vq, &bytes, NULL, 4096, 0);
    return bytes;
}
```

### set_guest_connected

当guest agent发生状态变动，如启动或停止时，会通过`set_guest_connected`设置`port->guest_connected`

```c
/* Callback function that's called when the guest opens/closes the port */
static void set_guest_connected(VirtIOSerialPort *port, int guest_connected)
{
    VirtConsole *vcon = VIRTIO_CONSOLE(port);
    DeviceState *dev = DEVICE(port);

    if (vcon->chr) {
        qemu_chr_fe_set_open(vcon->chr, guest_connected);
    }

    if (dev->id) {
        qapi_event_send_vserport_change(dev->id, guest_connected,
                                        &error_abort);
    }
}
```

比如，guest agent停止后，会通过control virtqueue，发送一段控制命令

```c
/* control queue: host to guest */
vser->c_ivq = virtio_add_queue(vdev, 32, control_in);
/* control queue: guest to host */
vser->c_ovq = virtio_add_queue(vdev, 32, control_out);
```

通过以下的路径，最终调用virtio-console的`set_guest_connected`，将guest的连接状态设为false

```shell
control_out
->handle_control_message
-->set_guest_connected
```

此时，自然的，virtio-console处于can't read的状态，往socket写数据是不会有反馈的

### chr_read

当满足以下两个条件：

- virtio-console处于可读状态
- socket有可读事件

此时会调用到`chr_read`函数，该函数所做的就很简单，就是将socket中读出数据，通过virtqueue传递给qemu guest agent

```c
/* Send data from a char device over to the guest */
static void chr_read(void *opaque, const uint8_t *buf, int size)
{
    VirtConsole *vcon = opaque;
    VirtIOSerialPort *port = VIRTIO_SERIAL_PORT(vcon);

    trace_virtio_console_chr_read(port->id, size);
    virtio_serial_write(port, buf, size);
}
```

我们通过`virsh gshellcheckready $vm_name`命令，可以看到此时qemu收到一串json命令，qemu会将其传递给qemu guest agent

![[chr_read.png]]

### write_to_port

最终会调用到write_to_port函数中

```c
static size_t write_to_port(VirtIOSerialPort *port,
                            const uint8_t *buf, size_t size)
{
    VirtQueueElement elem;
    VirtQueue *vq;
    size_t offset;

    /* input vq for host to write */
    vq = port->ivq;
    if (!virtio_queue_ready(vq)) {
        return 0;
    }

    offset = 0;
    while (offset < size) {
        size_t len;

        if (!virtqueue_pop(vq, &elem)) {
            break;
        }

        len = virtiov_from_buf(elem.in_sg, elem.in_num, 0,
                           buf + offset, size - offset);

        offset += len;
        virtqueue_push(vq, &elem, len);
    }

    virtio_notify(VIRTIO_DEVICE(port->vser), vq);
    return offset;
}
```

注意virtio-console会有两个vq

```c
/* Add a queue for host to guest transfers for port 0 (backward compat) */
vser->ivqs[0] = virtio_add_queue(vdev, 128, handle_input);
/* Add a queue for guest to host transfers for port 0 (backward compat) */
vser->ovqs[0] = virtio_add_queue(vdev, 128, handle_output);
```

ivq用于host向guest传入数据，需要host先预分配desc

## Guest侧的驱动处理

在linux的virtio-console驱动中，会经过以下的调用：

```c
port_fops_read
  ->port_has_data
  ->fill_readbuf
    ->add_inbuf
      ->virtqueue_add_inbuf
        ->virtqueue_add
```

guest中驱动会为host分配一个out_sg作为写入空间
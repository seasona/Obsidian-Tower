qemu中字符设备用于对char相关的socket等进行处理

## 打开socket

对于socket设备，入口函数是`qemu_chr_open_socket`，该函数将从qemu启动参数中判断是否为server端、是否为unix等，从而执行相应的socket处理函数。比如对于[[qemu guest agent]]打开的socket，参数中是server，就会调用`unix_listen_opts`打开socket fd，并对其listen

```c
static CharDriverState *qemu_chr_open_socket(QemuOpts *opts)
{
    CharDriverState *chr = NULL;
    Error *local_err = NULL;
    int fd = -1;
    bool saved_on_update = false;
    char *eci_port_name = NULL;

    bool is_listen      = qemu_opt_get_bool(opts, "server", false);
    bool is_waitconnect = is_listen && qemu_opt_get_bool(opts, "wait", true);
    bool is_telnet      = qemu_opt_get_bool(opts, "telnet", false);
    bool do_nodelay     = !qemu_opt_get_bool(opts, "delay", true);
    const char *path    = qemu_opt_get(opts, "path");
    bool is_unix        = path != NULL;
    const char *port    = qemu_opt_get(opts, "port");

    if (is_waitconnect && is_unix && is_listen) {
        if (!strncmp(path, "/var/run/", 9))
            saved_on_update = true;
    }

    if (is_unix) {
        if (is_listen) {
            fd = unix_listen_opts(opts, saved_on_update, is_updating_me(), &local_err);
        } else {
            fd = unix_connect_opts(opts, saved_on_update, is_updating_me(), &local_err, NULL, NULL);
        }
    } else {
        if (is_listen) {
            fd = inet_listen_opts(opts, 0, &local_err);
        } else {
            fd = inet_connect_opts(opts, &local_err, NULL, NULL);
        }
    }
    if (fd < 0) {
        goto fail;
    }

    if (!is_waitconnect)
        qemu_set_nonblock(fd);

    chr = qemu_chr_open_socket_fd(fd, do_nodelay, is_listen, is_telnet,
                                  is_waitconnect, saved_on_update, &local_err);

    if (local_err) {
        goto fail;
    }
    return chr;

```

最终会调用`qemu_chr_open_socket_fd`创建一个`CharDriverState`来设定如何对该socket进行各种处理

## socket监听

`qemu_chr_open_socket_fd`函数中最主要的功能是在主循环中添加了一个`GIOChannel`，调用了glib对listen fd进行watch，当满足`G_IO_IN`即有进来的IO数据时，就调用`tcp_chr_accept`

```c
 if (is_listen) {
        s->listen_fd = fd;
        s->listen_chan = io_channel_from_socket(s->listen_fd);
        s->listen_tag = g_io_add_watch(s->listen_chan, G_IO_IN, tcp_chr_accept, chr);
        if (is_telnet) {
            s->do_telnetopt = 1;
        }
```

`tcp_chr_accept`跟普通的socket通信一样，新生成一个fd用于网络传输，这就是为什么[[qemu guest agent]]此时会打开两个fd的原因，当`tcp_chr_add_client`将数据处理完，才会将fd关闭

```c
static gboolean tcp_chr_accept(GIOChannel *channel, GIOCondition cond, void *opaque)
{
    CharDriverState *chr = opaque;
    TCPCharDriver *s = chr->opaque;
    struct sockaddr_in saddr;
#ifndef _WIN32
    struct sockaddr_un uaddr;
#endif
    struct sockaddr *addr;
    socklen_t len;
    int fd;

    for(;;) {
#ifndef _WIN32
	    if (s->is_unix) {
	        len = sizeof(uaddr);
	        addr = (struct sockaddr *)&uaddr;
	    } else
#endif
	    {
	        len = sizeof(saddr);
	        addr = (struct sockaddr *)&saddr;
	    }
        fd = qemu_accept(s->listen_fd, addr, &len);
        if (fd < 0 && errno != EINTR) {
            s->listen_tag = 0;
            return FALSE;
        } else if (fd >= 0) {
            if (s->do_telnetopt)
                tcp_chr_telnet_init(fd);
            break;
        }
    }
    if (tcp_chr_add_client(chr, fd) < 0)
        close(fd);

    return TRUE;
}
```

## 数据处理

此时，server处于listen的socket，收到了来自client的connect，因此会将其加入到主事件循环中

```c
static void tcp_chr_connect(void *opaque)
{
    CharDriverState *chr = opaque;
    TCPCharDriver *s = chr->opaque;

    s->connected = 1;
    if (s->chan) {
        chr->fd_in_tag = io_add_watch_poll(s->chan, tcp_chr_read_poll,
                                           tcp_chr_read, chr);
    }
    qemu_chr_be_generic_open(chr);
}
```

当`tcp_chr_read_poll`函数返回1时，调用`tcp_chr_read`读取数据

```c
static int tcp_chr_read_poll(void *opaque)
{
    CharDriverState *chr = opaque;
    TCPCharDriver *s = chr->opaque;
    if (!s->connected)
        return 0;
    s->max_size = qemu_chr_be_can_write(chr);
    return s->max_size;
}

int qemu_chr_be_can_write(CharDriverState *s)
{
    if (!s->chr_can_read)
        return 0;
    return s->chr_can_read(s->handler_opaque);
}
```

最终是会调用到具体的chr设备的can_read函数，如[[qemu guest agent#chr_can_read]]

## io_add_watch_poll

Qemu在这里没有直接使用glib提供的poll，而是自定义了一套io_watch_poll，首先我们看`io_add_watch_poll`

```c
/* Can only be used for read */
static guint io_add_watch_poll(GIOChannel *channel,
                               IOCanReadHandler *fd_can_read,
                               GIOFunc fd_read,
                               gpointer user_data)
{
    IOWatchPoll *iwp;
    int tag;

    iwp = (IOWatchPoll *) g_source_new(&io_watch_poll_funcs, sizeof(IOWatchPoll));
    iwp->fd_can_read = fd_can_read;
    iwp->opaque = user_data;
    iwp->channel = channel;
    iwp->fd_read = (GSourceFunc) fd_read;
    iwp->src = NULL;

    tag = g_source_attach(&iwp->parent, NULL);
    g_source_unref(&iwp->parent);
    return tag;
}
```

它首先通过`g_source_new`创建了一个g_source，`g_source_new`允许申请一块内存并返回一个g_source，可以看到这里`io_add_watch_poll`申请了一块`IOWatchPoll`大小的内存，此时返回的g_source就是`IOWatchPoll->parent`

```c
typedef struct IOWatchPoll
{
    GSource parent;

    GIOChannel *channel;
    GSource *src;

    IOCanReadHandler *fd_can_read;
    GSourceFunc fd_read;
    void *opaque;
} IOWatchPoll;
```

此时可以通过结构体`IOWatchPoll`的方式，设置g_source对应的`fd_can_read`等，并且指定了g_source对应的行为`io_watch_poll_funcs`，并通过`g_source_attach`将其注册到主事件循环中

## Event Loop

这时候我们就接入了glib的主事件循环，事件循环中会调用我们自己设定的poll function

```c
static GSourceFuncs io_watch_poll_funcs = {
    .prepare = io_watch_poll_prepare,
    .check = io_watch_poll_check,
    .dispatch = io_watch_poll_dispatch,
    .finalize = io_watch_poll_finalize,
};
```

事件循环的判定是：

- prepare：
	- `gboolean (*prepare) (GSource *source, gint *timeout_);`
	- Glib初始化完成后会调用此接口，此接口返回TRUE表示事件源都已准备好，告诉Glib跳过poll直接检查判断是否执行对应回调。返回FALSE表示需要poll机制监听事件源是否准备好，如果有事件源没准备好，通过参数timeout指定poll最长的阻塞时间，超时后直接返回，超时接口可以防止一个fd或者其它事件源阻塞整个应用
- query：
	- `gint (g_main_context_query) (GMainContext *context, gint max_priority, gint *timeout_, GPollFD *fds, gint n_fds);`
	- Glib在prepare完成之后，可以通过query查询一个上下文将要poll的所有事件，这个接口需要用户主动调用
- check：
	- `gboolean (*check) (GSource *source);`
	- Glib在poll返回后会调用此接口，用户通过注册此接口判断哪些事件源需要被处理，此接口返回TRUE表示对应事件源的回调函数需要被执行，返回FALSE表示不需要被执行
- dispatch：
	- `gboolean (*dispatch) (GSource *source, GSourceFunc callback, gpointer user_data);`
	- Glib根据check的结果调用此接口，参数callback和user_data是用户通过g_source_set_callback注册的事件源回调和对应的参数，用户可以在dispatch中选择直接执行callback，也可以不执行

qemu没有使用glib原生的事件监听机制，而是通过prepare完成了大部分工作，check等函数直接返回False

```c
static gboolean io_watch_poll_prepare(GSource *source, gint *timeout_)
{
    IOWatchPoll *iwp = io_watch_poll_from_source(source);
    bool now_active = iwp->fd_can_read(iwp->opaque) > 0; // true if socket can read
    bool was_active = iwp->src != NULL; // false
    if (was_active == now_active) {
        return FALSE;
    }

    if (now_active) {
        /**
         * when channel aka socket has event: G_IO_IN | G_IO_ERR | G_IO_HUP,
         * we add another source to event main loop to receive data in socket.
         * reference: https://docs.gtk.org/glib/func.io_create_watch.html#description
         */
        iwp->src = g_io_create_watch(iwp->channel, G_IO_IN | G_IO_ERR | G_IO_HUP);
        g_source_set_callback(iwp->src, iwp->fd_read, iwp->opaque, NULL);
        g_source_attach(iwp->src, NULL);
    } else {
        /**
         * if last time we add source to event main loop but now socket can't read,
         * we will remove source from event main loop.
         */
        g_source_destroy(iwp->src);
        g_source_unref(iwp->src);
        iwp->src = NULL;
    }
    return FALSE;
}
```

`io_watch_poll_prepare`函数在每次事件循环时被调用，它先调用`iwp->fd_can_read`检查客户端fd是否处于可读状态，例如gpa关闭[[qemu guest agent#set_guest_connected]]，就是属于不可读的状态

如果fd此时可读，并且`iwp->src`不为空，那么就会再通过`g_io_create_watch`生成新的g_source并设置该source的判定条件：是否有数据进来等，并且设置回调函数为`iwp->fd_read`，最后将其注册到主事件循环；如果fd此时不可读，就会从主事件循环移除之前添加的g_source

## 参考链接

- https://blog.csdn.net/huang987246510/article/details/90738137
- https://docs.gtk.org/glib/func.io_add_watch.html
- https://docs.gtk.org/glib/ctor.Source.new.html
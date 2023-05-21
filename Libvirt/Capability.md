## Libvirt获取Hypervisor的Caps

Libvirt在正式启动VM前，会获取对应Hypervisor的Caps，从而进行版本协商，根据Caps来判断Hypervisor是否支持某些特性。

在启动VM之前，Libvirt会通过xml中指定的Hypervisor的binary路径启动一个进程：

>/usr/bin/qemu-kvm -S -no-user-config -nodefaults -nographic -M none -qmp unix:/var/lib/libvirt/qemu/capabilities.monitor.sock,server,nowait -pidfile /var/lib/libvirt/qemu/capabilities.pidfile -daemonize

该进程不会载入任何配置文件，但qmp功能会打开，Libvirt通过以下的操作获取到Hypervisor支持的Caps：

1. libvirt发送qmp命令"qmp_capabilities"，使qemu端退出Capabilities Negotiation模式进入command模式；
2. libvirt发送qmp命令"query-version"获取qemu版本信息；
3. libvirt直接默认所有qemu版本支持 basic cap，参见virQEMUCapsInitQMPBasic()。其中包括QEMU_CAPS_DRIVE、QEMU_CAPS_DEVICE、QEMU_CAPS_NETDEV等；
4. libvirt发送qmp命令"query-target"获取qemu支持模拟的架构类型，比如VIR_ARCH_X86_64、VIR_ARCH_I686等；
5. libvirt发送qmp命令"query-commands"获取qemu支持哪些命令；
6. libvirt发送qmp命令"qom-list-types"获取qemu支持模拟哪些qemu object model对象模型；
7. 对于一些对象模型libvirt还关心这些对象模型支持哪些属性Property，因此libvirt针对这些对象模型逐个构造qmp命令"device-list-properties s:typename type"向qemu端查询相关的属性。
8. 对于一些特定的属性，比如"disable-legacy"，libvirt非常关心，如果发现某些对象模型支持这种属性，那么libvirt也会将其记录到virQEMUCaps中。
9. libvirt发送qmp命令"query-machines"获取qemu支持哪些机器类型以及该机器上最大的cpu数量；
10. libvirt发送qmp命令"query-cpu-definitions"获取qemu支持模拟哪些cpu类型；

## Hypervisor和Caps之间的映射

Libvirt中记录了一个全局的哈希表，存储了Hypervisor的binary地址与对应的qemucaps，如</usr/bin/qemu-kvm, 0xffff3eeee>

```c
struct _virQEMUCapsCache {
    virMutex lock;
    virHashTablePtr binaries;
    char *libDir;
    char *runDir;
    uid_t runUid;
    gid_t runGid;
};
```

当Libvirt通过xml创建VM时，首先会通过`virQEMUCapsCacheLookup` 查看`driver->qemuCapsCache` 是否存在Hypervisor对应的qemucaps

```c
virQEMUCapsPtr
virQEMUCapsCacheLookup(virQEMUCapsCachePtr cache, const char *binary)
{
    virQEMUCapsPtr ret = NULL;
    virMutexLock(&cache->lock);
    ret = virHashLookup(cache->binaries, binary);
    if (ret &&
        !virQEMUCapsIsValid(ret)) {
        VIR_WARN("Cached capabilities %p no longer valid for %s",
                  ret, binary);
        virHashRemoveEntry(cache->binaries, binary);
        ret = NULL;
    }
    if (!ret) {
        VIR_DEBUG("Creating capabilities for %s", binary);
        ret = virQEMUCapsNewForBinary(binary, cache->libDir,
                                      cache->runUid, cache->runGid);
        if (ret) {
            VIR_DEBUG("Caching capabilities %p for %s", ret, binary);
            if (virHashAddEntry(cache->binaries, binary, ret) < 0) {
                virObjectUnref(ret);
                ret = NULL;
            }
        }
    }
    VIR_DEBUG("Returning caps %p for %s", ret, binary);
    virObjectRef(ret);
    virMutexUnlock(&cache->lock);
    return ret;
}
```

如果查询到存在对应的qemucaps，首先会通过stat查看目前的binary的时间是否与qemuCaps对应的binary时间存在差别

```c
bool virQEMUCapsIsValid(virQEMUCapsPtr qemuCaps)
{
    struct stat sb;

    if (!qemuCaps->binary)
        return true;

    if (stat(qemuCaps->binary, &sb) < 0)
        return false;

    return sb.st_mtime == qemuCaps->mtime;
}

```

如果时间不同，说明Hypervisor被更新，此时我们需要通过`virQEMUCapsNewForBinary` 进行Capability的重新创建

```c
virQEMUCapsPtr virQEMUCapsNewForBinary(const char *binary,
                                       const char *libDir,
                                       uid_t runUid,
                                       gid_t runGid)
{
    virQEMUCapsPtr qemuCaps = virQEMUCapsNew();
    struct stat sb;
    int rv;

    if (VIR_STRDUP(qemuCaps->binary, binary) < 0)
        goto error;

    /* We would also want to check faccessat if we cared about ACLs,
     * but we don't.  */
    if (stat(binary, &sb) < 0) {
        virReportSystemError(errno, _("Cannot check QEMU binary %s"),
                             binary);
        goto error;
    }
    qemuCaps->mtime = sb.st_mtime;

    /* Make sure the binary we are about to try exec'ing exists.
     * Technically we could catch the exec() failure, but that's
     * in a sub-process so it's hard to feed back a useful error.
     */
    if (!virFileIsExecutable(binary)) {
        virReportSystemError(errno, _("QEMU binary %s is not executable"),
                             binary);
        goto error;
    }

    if ((rv = virQEMUCapsInitQMP(qemuCaps, libDir, runUid, runGid)) < 0)
        goto error;
    VIR_WARN("startting libvirtd,cpu_live ptr:%p",qemuCaps->cpu_live);

    if (!qemuCaps->usedQMP &&
        virQEMUCapsInitHelp(qemuCaps, runUid, runGid) < 0)
        goto error;

    return qemuCaps;

error:
    virObjectUnref(qemuCaps);
    qemuCaps = NULL;
    return NULL;
}
```

## Caps如何存储

`virQEMUCapsNewForBinary` 通过`virQEMUCapsInitQMP` 函数建立Caps，这里面就是[[#Libvirt获取Hypervisor的Caps]]的过程，主要记录到Hypervisor对应的qemucaps中，该结构体如下：

```c
struct _virQEMUCaps {
    virObject object;

    bool usedQMP;

    char *binary;
    time_t mtime;

    virBitmapPtr flags;

    unsigned int version;
    unsigned int kvmVersion;

    virArch arch;

    size_t ncpuDefinitions;
    char **cpuDefinitions;

    size_t nmachineTypes;
    char **machineTypes;
    char **machineAliases;
    unsigned int *machineMaxCpus;
    virCPUDefPtr cpu_live;
};
```

该结构体中主要关注`virBitmapPtr`，这就是一个位图，记录了全局`enum virQEMUCapsFlags`

```c
struct _virBitmap {
    size_t max_bit;
    size_t map_len;
    size_t map_alloc;
    unsigned long *map;
};
```

当Libvirt认为Hypervisor支持某些功能时，会把`enum virQEMUCapsFlags` 对应的位图置为1

```c
int virBitmapSetBit(virBitmapPtr bitmap, size_t b)
{
    if (bitmap->max_bit <= b)
        return -1;

    bitmap->map[VIR_BITMAP_UNIT_OFFSET(b)] |= VIR_BITMAP_BIT(b);
    return 0;
}

void
virQEMUCapsSet(virQEMUCapsPtr qemuCaps,
               enum virQEMUCapsFlags flag)
{
    ignore_value(virBitmapSetBit(qemuCaps->flags, flag));
}
```

## 如何添加新的Caps

以qmp为例，可以看到Libvirt想知道Hypervisor是否支持以下的qmp

```c
struct virQEMUCapsStringFlags virQEMUCapsCommands[] = {
    { "system_wakeup", QEMU_CAPS_WAKEUP },
    { "transaction", QEMU_CAPS_TRANSACTION },
    { "block_job_cancel", QEMU_CAPS_BLOCKJOB_SYNC },
    { "block-job-cancel", QEMU_CAPS_BLOCKJOB_ASYNC },
    { "dump-guest-memory", QEMU_CAPS_DUMP_GUEST_MEMORY },
    { "query-spice", QEMU_CAPS_SPICE },
    { "query-kvm", QEMU_CAPS_KVM },
    { "block-commit", QEMU_CAPS_BLOCK_COMMIT },
    { "query-vnc", QEMU_CAPS_VNC },
    { "drive-mirror", QEMU_CAPS_DRIVE_MIRROR },
    { "blockdev-snapshot-sync", QEMU_CAPS_DISK_SNAPSHOT },
    { "add-fd", QEMU_CAPS_ADD_FD },
    { "nbd-server-start", QEMU_CAPS_NBD_SERVER },
};
```

那么它会通过`virQEMUCapsProbeQMPCommands`，先通过`qemuMonitorGetCommands`函数，建立与QEMU的monitor，并发送一条`query-commands`命令给QEMU，QEMU回复它支持的所有qmp命令

```c
static int
virQEMUCapsProbeQMPCommands(virQEMUCapsPtr qemuCaps,
                            qemuMonitorPtr mon)
{
    char **commands = NULL;
    int ncommands;

    if ((ncommands = qemuMonitorGetCommands(mon, &commands)) < 0)
        return -1;

    virQEMUCapsProcessStringFlags(qemuCaps,
                                  ARRAY_CARDINALITY(virQEMUCapsCommands),
                                  virQEMUCapsCommands,
                                  ncommands, commands);
    virQEMUCapsFreeStringList(ncommands, commands);

    /* QMP add-fd was introduced in 1.2, but did not support
     * management control of set numbering, and did not have a
     * counterpart -add-fd command line option.  We require the
     * add-fd features from 1.3 or later.  */
    if (virQEMUCapsGet(qemuCaps, QEMU_CAPS_ADD_FD)) {
        int fd = open("/dev/null", O_RDONLY);
        if (fd < 0) {
            virReportError(VIR_ERR_INTERNAL_ERROR, "%s",
                           _("unable to probe for add-fd"));
            return -1;
        }
        if (qemuMonitorAddFd(mon, 0, fd, "/dev/null") < 0)
            virQEMUCapsClear(qemuCaps, QEMU_CAPS_ADD_FD);
        VIR_FORCE_CLOSE(fd);
    }

    return 0;
}
```

Libvirt将QEMU支持的命令转为字符串，并与`virQEMUCapsCommands`进行匹配，如果相同，就将`enum virQEMUCapsFlags` 对应的位图置为1

```c
static void
virQEMUCapsProcessStringFlags(virQEMUCapsPtr qemuCaps,
                              size_t nflags,
                              struct virQEMUCapsStringFlags *flags,
                              size_t nvalues,
                              char *const*values)
{
    size_t i, j;
    for (i = 0; i < nflags; i++) {
        for (j = 0; j < nvalues; j++) {
            if (STREQ(values[j], flags[i].value)) {
                virQEMUCapsSet(qemuCaps, flags[i].flag);
                break;
            }
        }
    }
}
```

最后，`virQEMUCapsProbeQMPCommands`在`virQEMUCapsInitQMP`被调用，而后者在Libvirt创建VM时通过`virQEMUCapsNewForBinary`被调用

因此，想加入新的Caps，只需要完成以下步骤即可：

1. 在全局`enum virQEMUCapsFlags`新增标志
2. 在`virQEMUCapsInitQMP`中加入判断逻辑，比如判断QEMU是否支持一条qmp，或者通过monitor发一条qmp，判断返回值等等
3. 通过`virQEMUCapsSet`将新增标志对应的位图置1
Libvirt中可以连接到QEMU的monitor，通过QMP向QEMU获取一些信息。在连接之前，需要先Enter Monitor，分为同步和异步两种方式：

```c
void qemuDomainObjEnterMonitor(virQEMUDriverPtr driver,
                               virDomainObjPtr obj)
{
    ignore_value(qemuDomainObjEnterMonitorInternal(driver, obj,
                                                   QEMU_ASYNC_JOB_NONE));
}
```

```c
int
qemuDomainObjEnterMonitorAsync(virQEMUDriverPtr driver,
                               virDomainObjPtr obj,
                               enum qemuDomainAsyncJob asyncJob)
{
    return qemuDomainObjEnterMonitorInternal(driver, obj, asyncJob);
}

```

可以看到，两者的区别在于传入的`asyncJob`参数，同步直接传入`QEMU_ASYNC_JOB_NONE`，而异步则会传入实际的异步任务

```c
static int
qemuDomainObjEnterMonitorInternal(virQEMUDriverPtr driver,
                                  virDomainObjPtr obj,
                                  enum qemuDomainAsyncJob asyncJob)
{
    qemuDomainObjPrivatePtr priv = obj->privateData;

    if (asyncJob != QEMU_ASYNC_JOB_NONE) {
        if (qemuDomainObjBeginNestedJob(driver, obj, asyncJob) < 0)
            return -1;
        if (!virDomainObjIsActive(obj)) {
            virReportError(VIR_ERR_OPERATION_FAILED, "%s",
                           _("domain is no longer running"));
            qemuDomainObjEndJob(driver, obj);
            return -1;
        }
    } else if (priv->job.asyncOwner == virThreadSelfID()) {
        VIR_WARN("This thread seems to be the async job owner; entering"
                 " monitor without asking for a nested job is dangerous");
    }

    VIR_DEBUG("Entering monitor (mon=%p vm=%p name=%s)",
              priv->mon, obj, obj->def->name);

    virObjectLock(priv->mon);
    virObjectRef(priv->mon);
    ignore_value(virTimeMillisNow(&priv->monStart));
    virObjectUnlock(obj);

    return 0;
}
```

在实际实现函数内部，首先会对`asyncJob`进行判断，如果是`QEMU_ASYNC_JOB_NONE`即同步调用，直接获取mon锁，获得该QEMU的Monitor控制权。如果是异步进入，如热迁移过程中，此时上层调入的`asyncJob`是`QEMU_ASYNC_JOB_MIGRATION_IN`

```c
if (qemuProcessStartRequestId(dconn, driver, vm, QEMU_ASYNC_JOB_MIGRATION_IN,
                         migrateFrom, dataFD[0], NULL, NULL,
                         VIR_NETDEV_VPORT_PROFILE_OP_MIGRATE_IN_START,
                         processStartFlags, requestID) < 0) {
        VIR_ERROR("vm %s qemu precess start failed.", vm->def->name);
        virDomainAuditStart(vm, "migrated", false);
        goto endjob;
    }
```

所以，会进入异步的处理逻辑内进行判断，如果此时连接的不是QEMU自己而是Iohub，那么就会出现验证不通过的问题
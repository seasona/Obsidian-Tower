## VM状态机

VM内部其实在执行mainloop循环

```c
static void main_loop(void)
{
    bool nonblocking;
    int last_io = 0;
#ifdef CONFIG_PROFILER
    int64_t ti;
#endif

    do {
        nonblocking = !kvm_enabled() && !xen_enabled() && last_io > 0;

        if (unlikely(get_update_stage() == ADVANCE_UPDATE_STAGE_FATHER_PREPARE)) {
            if (update_get_clone_state()) {
                qemu_update_log(stdout, "ready to send init complete event.\n");
                set_update_state(ADVANCE_UPDATE_STAGE_FATHER_CHILD_READY);
                qapi_event_send_init_complete(&error_abort);
            }
        }

#ifdef CONFIG_PROFILER
        ti = profile_getclock();
#endif
        if (main_loop_should_nonblock) {
            printf("qemu-log: set main loop non-block\n");
            nonblocking = 1;
            main_loop_should_nonblock = false;
        }
        last_io = main_loop_wait(nonblocking);
#ifdef CONFIG_PROFILER
        dev_time += profile_getclock() - ti;
#endif
    } while (!main_loop_should_exit());
}
```

而判断mainloop是否退出是通过`main_loop_should_exit`函数进行判断，该函数中对VM的运行状态进行判断，根据request来使VM进入关机、重启或者暂停状态

```c
static bool main_loop_should_exit(void)
{
    RunState r;
    if (qemu_debug_requested()) {
        vm_stop(RUN_STATE_DEBUG);
    }
    if (qemu_suspend_requested()) {
        qemu_system_suspend();
    }
    if (qemu_shutdown_requested()) {
        qemu_kill_report();
        if (in_reboot_reset) {
            qapi_event_send_reset(&error_abort);
            in_reboot_reset = 0;
        } else
            qapi_event_send_shutdown(&error_abort);
        if (no_shutdown) {
            main_loop_should_nonblock = true;
            vm_stop(RUN_STATE_SHUTDOWN);
        } else {
            return true;
        }
    }
    if (qemu_reset_requested()) {
        pause_all_vcpus();
        cpu_synchronize_all_states();
        qemu_system_reset(VMRESET_REPORT);
        resume_all_vcpus();
        if (runstate_needs_reset()) {
            runstate_set(RUN_STATE_PAUSED);
        }
    }
    if (qemu_wakeup_requested()) {
        pause_all_vcpus();
        cpu_synchronize_all_states();
        qemu_system_reset(VMRESET_SILENT);
        notifier_list_notify(&wakeup_notifiers, &wakeup_reason);
        wakeup_reason = QEMU_WAKEUP_REASON_NONE;
        resume_all_vcpus();
        qapi_event_send_wakeup(&error_abort);
    }

    if (qemu_powerdown_requested()) {
        qemu_system_powerdown();
    }

    if (qemu_sleep_requested()) {
        qemu_system_sleep();
    }

    if (qemu_vmstop_requested(&r)) {
        vm_stop(r);
    }
    return false;
}
```

## 内部重启

内部重启通过`qemu_system_reset_requeset`将static变量`reset_requested`置为1

```c
void qemu_system_reset_request(void)
{
    qemu_reset_time = qemu_clock_get_ns(QEMU_CLOCK_REALTIME);
    if (no_reboot || reboot_reset) {
        shutdown_requested = 1;
        in_reboot_reset = reboot_reset;
    } else {
        reset_requested = 1;
    }
    lu_post_reset_request |= BIT(LU_PR_GUEST_RESET);
    cpu_stop_current();
    qemu_notify_event();
}
```

此时会进入到`qemu_system_reset`函数进行系统reset操作，对设备reset等，VM内部重启会经历两次reset操作

![[pin-dragonfly.drawio.svg]]
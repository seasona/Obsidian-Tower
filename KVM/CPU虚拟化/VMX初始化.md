首先会分配一个percpu变量vmxaree

`static DEFINE_PER_CPU(struct vmcs *, vmxarea);` 

在`alloc_kvm_area`函数中，会分配一个物理页做为vmcs，并将vmxarea指向vmcs指针

```c
hardware_setup
  setup_vmcs_config
```


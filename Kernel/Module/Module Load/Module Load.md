## 模块插入

接下来我们具体分析模块是如何插入内核的

```c
sys_init_module	                /* 系统调用init_module */
├── copy_module_from_user
└── load_module
    ├── setup_load_info		/* info指向__this_module */
    ├── layout_and_allocate 	/* 分配module的真实内存 *
    ├── add_unformed_module 	/* 插入modules列表 */
    ├── find_module_sections 	/* 查找其他section(ksymtab等) */
    ├── setup_modinfo		/* 设置modinfo */
    ├── simplify_symbols        /* 找到未定义的符号 */
    ├── apply_relocations 	/* 重定向 */
    ├── post_relocation
    │   └── add_kallsyms	/* 将模块的所有符号都放入module->kallsyms中 */
    ├── complete_formation	/* 设置module->state为coming */
    ├── prepare_coming_module   /* 通知notifier chain */
    ├── parse_args		/* 模块传入参数处理 */
    └── do_init_module		/* 调用module_init */
```

从用户态将模块文件插入内核的入口肯定是系统调用，Linux内核为insmod等工具提供了`init_module`系统调用，通过该系统调用进入内核态，内核使用`copy_from_user`将模块文件拷入预先分配的一块内存中，并将`load_info`的hdr指针指向这块内存地址

`struct load_info` 是管理模块插入的重要结构体

```c
struct load_info {
	const char *name;
	/* pointer to module in temporary copy, freed at end of load_module() */
	struct module *mod;
	Elf_Ehdr *hdr;
	unsigned long len;
	Elf_Shdr *sechdrs;
	char *secstrings, *strtab;
	unsigned long symoffs, stroffs, init_typeoffs, core_typeoffs;
	struct _ddebug *debug;
	unsigned int num_debug;
	bool sig_ok;
#ifdef CONFIG_KALLSYMS
	unsigned long mod_kallsyms_init_off;
#endif
	struct {
		unsigned int sym, str, mod, vers, info, pcpu;
	} index;
};
```

进入

## 模块拔除

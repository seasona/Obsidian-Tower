## 符号导出

在看内核代码时，我们经常可以看到`EXPORT_SYMBOL`这个宏，顾名思义，这个宏的作用是将符号导出，but why?

编写用户态程序时，在链接这个步骤，有时会遇到undefined reference的问题，这往往是由于缺少了某些静态库。内核其实同理，对于模块来说，Linux内核本身就像一个静态库，内核本身或者模块定义的函数只有通过`EXPORT_SYMBOL`才能被其他模块调用。

那么`EXPORT_SYMBOL`在Linux内核内部是如何实现的呢？

```c
#define ___EXPORT_SYMBOL(sym, sec, ns)						\
	extern typeof(sym) sym;							\
	extern const char __kstrtab_##sym[];					\
	extern const char __kstrtabns_##sym[];					\
	__CRC_SYMBOL(sym, sec);							\
	asm("	.section \"__ksymtab_strings\",\"aMS\",%progbits,1	\n"	\
	    "__kstrtab_" #sym ":					\n"	\
	    "	.asciz 	\"" #sym "\"					\n"	\
	    "__kstrtabns_" #sym ":					\n"	\
	    "	.asciz 	\"" ns "\"					\n"	\
	    "	.previous						\n");	\
	__KSYMTAB_ENTRY(sym, sec)
#endif

#define EXPORT_SYMBOL(sym) _EXPORT_SYMBOL(sym, "")
```

从代码中，我们可以直观的看出，`EXPORT_SYMBOL`是一个宏，它完成了以下三步操作：

1. 将符号的CRC校验放入ELF文件的__kcrctab section，通过CRC来检验函数ABI是否兼容，如果函数的参数类型或者数量改变，符号的CRC就会变动，导致无法兼容
2. 将符号的名字和命名空间(默认为空)放入__ksymtab_strings
3. 将一个kernel_symbol entry放入ELF文件的__ksymtab section

```c
struct kernel_symbol {
	unsigned long value;
	const char *name;
	const char *namespace;
};

#define __KSYMTAB_ENTRY(sym, sec)					\
	static const struct kernel_symbol __ksymtab_##sym		\
	__attribute__((section("___ksymtab" sec "+" #sym), used))	\
	__aligned(sizeof(void *))					\
	= { (unsigned long)&sym, __kstrtab_##sym, __kstrtabns_##sym }
```

再跟一下`__KSYMTAB_ENTRY`这个宏，可以看出来就是声明了一个静态的struct kernel\_symbol，并将value初始化为函数的地址，name初始化为\_\_kstrtab_函数名

使用`readelf -t`查看模块编译的.o文件，可以看到多出了四个section

![](readelf_nvme_stop_ctrl.png)

在链接时，会将所有__ksymtab子段聚合到最终的ko文件中

![](readelf_symtab.png)

模块插入时，通过解析__ksymtab section，将module->syms指向第一个导出符号的地址，并得到导出符号的个数num_syms，内核就获得了所有导出符号的地址信息。

```c
static int find_module_sections(struct module *mod, struct load_info *info)
{
	mod->kp = section_objs(info, "__param",
			       sizeof(*mod->kp), &mod->num_kp);
	mod->syms = section_objs(info, "__ksymtab",
				 sizeof(*mod->syms), &mod->num_syms);
	mod->crcs = section_addr(info, "__kcrctab");
	mod->gpl_syms = section_objs(info, "__ksymtab_gpl",
				     sizeof(*mod->gpl_syms),
				     &mod->num_gpl_syms);
	mod->gpl_crcs = section_addr(info, "__kcrctab_gpl");
	...
}
```

而对于模块中非导出的符号，在模块插入流程的`setup_load_info`函数中，会遍历找出type为`SHT_SYMTAB`的section，也就是`.symtab`section，设置index.sym，并在模块插入的后续步骤中，通过`add_kallsyms`将所有符号放进`module->kallsyms`中。

## 符号解析

```ad-question
模块如何调用Linux内核或者其他模块提供的函数？如果模块A需要依赖模块B的某个函数，而模块A插入时模块B还未插入会发生什么？
```

如果是用户态程序，链接时会查找到需要的函数地址，对于内核模块，怎么处理这个问题？

实际上，在模块插入时，会通过`simplify_symbols`解决所有未定义的符号，未定义的符号在ELF中的类型为`SHN_UNDEF`

![](module_undefined_symbol.png)

`simplify_symbols`函数会遍历内核自身和所有已插入模块已导出的符号[查找已导出的符号](#查找已导出的符号)，并且比较crc确定是否二进制兼容，之后直接将内存中ELF对应sym的value修改为符号地址，此后模块就可以正常调用该符号对应的函数了。

```c
static int simplify_symbols(struct module *mod, const struct load_info *info)
{
	Elf_Shdr *symsec = &info->sechdrs[info->index.sym];
	Elf_Sym *sym = (void *)symsec->sh_addr;
	unsigned long secbase;
	unsigned int i;
	int ret = 0;
	const struct kernel_symbol *ksym;

	for (i = 1; i < symsec->sh_size / sizeof(Elf_Sym); i++) {
		const char *name = info->strtab + sym[i].st_name;
		switch (sym[i].st_shndx) {
		...
		case SHN_ABS:
			/* Don't need to do anything */
			pr_debug("Absolute symbol: 0x%08lx\n",
			       (long)sym[i].st_value);
			break;
		case SHN_UNDEF:
			ksym = resolve_symbol_wait(mod, info, name);
			/* Ok if resolved.  */
			if (ksym && !IS_ERR(ksym)) {
				/* Fill in symbol address of module */
				sym[i].st_value = kernel_symbol_value(ksym);
				break;
			}
			/* Ok if weak or ignored.  */
			if (!ksym &&
			    (ELF_ST_BIND(sym[i].st_info) == STB_WEAK ||
			     ignore_undef_symbol(info->hdr->e_machine, name)))
				break;

			ret = PTR_ERR(ksym) ?: -ENOENT;
			pr_warn("%s: Unknown symbol %s (err %d)\n",
				mod->name, name, ret);
			break;
		default:
			/* Divert to percpu allocation if a percpu var. */
			if (sym[i].st_shndx == info->index.pcpu)
				secbase = (unsigned long)mod_percpu(mod);
			else
				secbase = info->sechdrs[sym[i].st_shndx].sh_addr;
			sym[i].st_value += secbase;
			break;
		}
	}

	return ret;
}
```

回答之前的问题：如果模块插入前依赖的模块未插入，会找不到对应符号的地址，模块插入会直接失败。

## 符号查找

插入模块时，已经将模块中未定义的符号和内核已导出函数的地址关联了起来，因此模块自身可以正常运行，为了查找方便，module结构体中会将符号区别存放在：

- module->syms：存放导出的符号
- module->kallsyms：存放模块所有的符号，包括未导出的符号

当模块需要做一些特殊操作，比如：

- A模块依赖模块B，而模块B进行了热升级或者重插入，需要重更新模块A的函数地址
- 模块中需要调用内核未导出的函数等

此时，就需要查找符号的地址。

### 查找已导出的符号

如果仅仅想找到已导出的符号地址，可以使用`__symbol_get`函数，最终调用到`find_symbol`函数：

```c
static const struct kernel_symbol *find_symbol(const char *name,
					struct module **owner,
					const s32 **crc,
					enum mod_license *license,
					bool gplok,
					bool warn)
{
	struct find_symbol_arg fsa;

	fsa.name = name;
	fsa.gplok = gplok;
	fsa.warn = warn;

	if (each_symbol_section(find_exported_symbol_in_section, &fsa)) {
		if (owner)
			*owner = fsa.owner;
		if (crc)
			*crc = fsa.crc;
		if (license)
			*license = fsa.license;
		return fsa.sym;
	}

	pr_debug("Failed to find symbol %s\n", name);
	return NULL;
}
```

该函数首先会在内核自身的导出符号集`__start___ksymtab`根据名字查找，如果找不到就会在所有模块中查找[符号导出](#符号导出)的符号，对于模块来说，就是遍历一遍module->syms和module->syms_gpl。

```c
static bool each_symbol_section(bool (*fn)(const struct symsearch *arr,
				    struct module *owner,
				    void *data),
			void *data)
{
	struct module *mod;

	/* Search symbol from kernel */
	static const struct symsearch arr[] = {
		{ __start___ksymtab, __stop___ksymtab, __start___kcrctab,
		  NOT_GPL_ONLY, false },
		{ __start___ksymtab_gpl, __stop___ksymtab_gpl,
		  __start___kcrctab_gpl,
		  GPL_ONLY, false },
		{ __start___ksymtab_gpl_future, __stop___ksymtab_gpl_future,
		  __start___kcrctab_gpl_future,
		  WILL_BE_GPL_ONLY, false },
	};

	module_assert_mutex_or_preempt();
	if (each_symbol_in_section(arr, ARRAY_SIZE(arr), NULL, fn, data))
		return true;

	/* Search symbol from loaded modules */
	list_for_each_entry_rcu(mod, &modules, list,
				lockdep_is_held(&module_mutex)) {
		struct symsearch arr[] = {
			{ mod->syms, mod->syms + mod->num_syms, mod->crcs,
			  NOT_GPL_ONLY, false },
			{ mod->gpl_syms, mod->gpl_syms + mod->num_gpl_syms,
			  mod->gpl_crcs,
			  GPL_ONLY, false },
			{ mod->gpl_future_syms,
			  mod->gpl_future_syms + mod->num_gpl_future_syms,
			  mod->gpl_future_crcs,
			  WILL_BE_GPL_ONLY, false },
		};

		if (mod->state == MODULE_STATE_UNFORMED)
			continue;

		if (each_symbol_in_section(arr, ARRAY_SIZE(arr), mod, fn, data))
			return true;
	}
	return false;
}
```

### 查找未导出的符号

我们的模块如果需要做一些比较hack的操作，比如调用Linux内核中未导出的函数，可以通过`kallsyms_lookup_name`直接找到符号对应的地址

```c
/* Lookup the address for this symbol. Returns 0 if not found. */
unsigned long kallsyms_lookup_name(const char *name)
{
	char namebuf[KSYM_NAME_LEN];
	unsigned long i;
	unsigned int off;

	for (i = 0, off = 0; i < kallsyms_num_syms; i++) {
		off = kallsyms_expand_symbol(off, namebuf, ARRAY_SIZE(namebuf));

		if (strcmp(namebuf, name) == 0)
			return kallsyms_sym_address(i);
	}
	return module_kallsyms_lookup_name(name);
}
```

`kallsyms_lookup_name`函数分为两步：
- 从Linux内核自身的符号集中查找
- 如果找不到，则遍历所有模块查找模块的符号集

查找内核自身的kallsyms符号集比较复杂，并不是简单的字符串比较，内核用了一种比较巧妙的数据组织形式，我们暂时略过不表。

```c
unsigned long module_kallsyms_lookup_name(const char *name)
{
	struct module *mod;
	char *colon;
	unsigned long ret = 0;

	/* Don't lock: we're in enough trouble already. */
	preempt_disable();
	if ((colon = strnchr(name, MODULE_NAME_LEN, ':')) != NULL) {
		if ((mod = find_module_all(name, colon - name, false)) != NULL)
			ret = find_kallsyms_symbol_value(mod, colon+1);
	} else {
		list_for_each_entry_rcu(mod, &modules, list) {
			if (mod->state == MODULE_STATE_UNFORMED)
				continue;
			if ((ret = find_kallsyms_symbol_value(mod, name)) != 0)
				break;
		}
	}
	preempt_enable();
	return ret;
}
```

从模块查找所有符号就比较简单，模块中的符号会以`module_name:symbol_name`的形式存储，所以先根据前半部从全局`modules`链表找到模块，然后遍历`module->kallsyms`找到后半部对应的符号地址即可。

## 总结

本篇文章主要介绍了模块如何导出、查找符号，其实内核模块本身也就是运行在内存中的一个elf文件，只要在插入模块时，完善所有未定义的符号对应的地址，内核模块自然可以正常运行。下一篇文章将介绍Linux内核如何加载模块，以及模块的入口。



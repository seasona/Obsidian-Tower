## 序言

最近研究了一下Cache模型，关于Cache一致性的问题前人讲的已经很多了，去读Paul E. McKenney大神的论文就好。这篇文章主要是想研究Caching type(aka memory type)，也就是对于memory来说，如何选择和控制它使用的cache方式。

## 相关术语

首先介绍一下相关术语：

- **cache line fill**：CPU以cache line(一般64B)为单位从内存读取数据并填充对应的cache硬件
- **cache hit**：CPU在cache中找到了对应的内存地址(VA)，可以直接从cache读取数据，而不用去读内存
- **write hit**：CPU发生写操作时，在cache中找到了对应的内存地址(VA)，那么它自己改写cache中存储的值即可，不用直接写入内存
- **write miss**：CPU发生写操作时，在cache中没有找到对应地址，那么它也会先新生成一条cache line，通过cache写入内存

在现代的多核CPU中，CPU可以通过**snoop**去查看别的CPU访问的内存以及它们内部的cache，比如当前CPU发现(by snoop)其他CPU正在访问某块内存，并且该cache line处于shared state，它把这个cache line invalidate掉，这样下次它访问这块内存时，就必须做一次cache line fill把最新的内存数据从内存读回cache。

![](MESI.png)

## Cache模式

![](Memory%20types%20and%20their%20properies.png)

现代CPU支持多种cache模式：

- **Strong Uncacheable (UC)**：读写内存操作都不会经过cache，就当没cache这个硬件。并且指令流水线的投机访问和prefetch等乱序指令都不会进行，Strong ording，所有读写操作都不会乱序，可以说这种模式下你就不需要蛋疼的memory barrier了。当然，这种模式下访问系统内存的性能非常差，一般是给MMIO使用的
- **Uncacheable (UC-)**：跟UC基本没差别，唯一差别在于可以被通过MTRR被改变成WC类型
- **Write Combining (WC)**：读写内存操作都不会经过cache，允许投机读，投机写会被delay并通过combining buffer聚合后写入。这种cache模式比较适合游戏或视频帧缓冲
- **Write-through (WT)**：读写内存操作都会经过cache，允许投机读，如果write miss，不会填充cache line再写入内存，而是直接写入内存；如果write hit，写入cache line，不会直接写入内存。write combining也会开启。这种模式比较适合帧缓冲以及可以访问系统内存但不会snoop的设备
- **Write-back (WB)**：性能最高的模式，在WT模式的基础上，write miss也不再直写内存，而是fill一个新的cache line，尽可能的减少内存写入，只有cache满了需要重新分配才会将cache中的值写回内存
- **Write protected (WP)**：尽可能的从cache读，写操作会传递给系统总线并把所有其他CPU的相关cache line给invalidate掉，该模式比WT模式更严格，无论是否write hit，都直接写入内存

这么多模式中，最常用的就是UC和WB模式，WT模式对图像缓冲比较有用。

我们都知道对于系统内存，一般cache都会设置为WB模式，可以极大的提高性能，但是对于bus memory比如mmio来说，往往会使用UC模式cache，这是因为mmio一般映射的是pcie device bar，这些地址对应的是设备的register，如果使用WB模式，会有两个问题：

1. cache不会实时写入memory，对于驱动来说，往设备register写入操作就希望立即获得设备反馈，开启cache就意味着不可控，这对于一般的设备来说都是不可接受的
2. cache line一般是64B大小，驱动写入值可能只有8B或其他大小，flush cache line会造成boost write 64B，需要设备对写入值进行处理，一般设备没有这个能力

因此，对于MMIO映射的pcie device bar内存，不能选择WB模式的cache。

软件也可以通过细粒度的cache control来为某块具体的内存选择更适合的cache模式，例如，对于一个较大的数据结构，如果软件只会在另一个agent对该结构体进行修改后才会读它，那么我们可以把它设置成UC模式，因为每次有修改才会读，每次读都需要cache line fill，之后不仅不会访问还会把其他的cache line给刷掉，如果设置成WB模式会是负收益。

## Cache指令

Intel提供了一系列cache相关的指令：

- **INVD, WBINVD**：用于invalidate cache内容，区别是，WBINVD会先把所有被标记为modify的cache内存写回内存，再清空所有cache；INVD直接invalidate不会写回内存，所以INVD使用起来需要特别小心
- **PREFETCHh**：在比较新的CPU中提供，该指令给CPU一个hint去尽快的把指定地址的数据从内存提前读取到cache里，有利于减少读取的时延（高级用法，一般人玩不转）
- **CLFLUSH, CLFLUSHOPT**：用于把指定的某些cache lines刷回内存并清空，适用于该cache line对应的数据后续不会被访问到的情况
- **MOVNTI, MOVNTQ, MOVNTDQ, MOVNTPS, MOVNTPD**：这些指令可以把数据从CPU的resigter直接存入内存，不经过cache，比较适合那些只会被修改一次的数据，避免污染cache

## Cache control

对于Intel的CPU，提供了两种机制来控制cache模式，分别是memory type range registers (MTRRs)和Page Attribute Table (PAT)，这两者是正交的关系。

### MTRR

mtrr能够对指定的物理地址范围设置cache type，其原理是通过Intel的MSR寄存器来进行控制，分为1个全局控制寄存器和128对范围设置寄存器。

![](mtrrcap%20register.png)

控制寄存器没什么好说的，只是一些全局的设置。

![](smrr_physbase%20pair.png)

范围设置寄存器是成对的，IA32_SMRR_PHYSBASE寄存器负责存储PhyBase(4K对齐)和cache type，IA32_SMRR_PHYSMASK寄存器负责存储PhysMask，两者配合使用，物理地址满足：

$$
Address & PhysMask = PhysBase & PhysMask
$$

则该物理地址使用IA32_SMRR_PHYSBASE寄存器中设置的cache type，举个例子：

```c
IA32_MTRR_PHYSBASE0 = 0x0000 0000 0000 0006H
IA32_MTRR_PHYSMASK0 = 0x0000 000F FC00 0800H
```

那么\[0-64MB)的物理地址经过mask之后和physbase相等，均会采用WB cache type。

### PAT

MTRR只能支持物理地址访问的cache type设置，而且由于MTRR寄存器存在个数上限，所以使用起来没有那么灵活。Intel还提出了PAT来提供对页表级别的cache type设置，在页表entry中选择了三个bit位：PAT，PCD和PWT用于确定cache类型：

```c
#define _PAGE_BIT_PWT 3        /* page write through */
#define _PAGE_BIT_PCD 4        /* page cache disabled */
#define _PAGE_BIT_PAT 7        /* on 4KB pages */
#define _PAGE_BIT_PAT_LARGE 12 /* On 2MB or 1GB pages */
```

Intel的CPU中会提供一个MSR寄存器：IA32_PAT。

![](pat%20msr.png)

IA32_PAT寄存器分为8个PAT entry，每个PAT entry 3bit，可以编码为8种cache type。注意，每个PAT
entry代表的cache type基本是固定的，每次机器上电都会复位回下图的固定值，注意下图的cache type是最基本的四种类型，Linux内核中`pat_init`函数会根据CPU type来启用Full type suppot。

![](pat%20entry.png)

在页表walk的过程中，通过PAT，PCD和PWT这三个bit位，会编码指向对应的PAT entry，从而获得设定的cache type。

![](pat%20bit%20to%20entry.png)

举个例子，我把一个页表的entry设置为PAT=0, PCD=PWT=1，那么对应的PAT entry即为PAT3，从上图可以看到PAT3对应的cache type是UC模式，那么我访问这个页时将使用UC模式即不经过cache。

## 实践分析

以ioremap为例，ioremap最广泛的应用是设备驱动将pcie设备bar空间的物理地址映射为MMIO空间，从而通过驱动控制设备。

```c
void __iomem *ioremap(resource_size_t phys_addr, unsigned long size)
{
	/*
	 * Ideally, this should be:
	 *	pat_enabled() ? _PAGE_CACHE_MODE_UC : _PAGE_CACHE_MODE_UC_MINUS;
	 *
	 * Till we fix all X drivers to use ioremap_wc(), we will use
	 * UC MINUS. Drivers that are certain they need or can already
	 * be converted over to strong UC can use ioremap_uc().
	 */
	enum page_cache_mode pcm = _PAGE_CACHE_MODE_UC_MINUS;

	return __ioremap_caller(phys_addr, size, pcm,
				__builtin_return_address(0), false);
}

void __iomem *ioremap_cache(resource_size_t phys_addr, unsigned long size)
{
	return __ioremap_caller(phys_addr, size, _PAGE_CACHE_MODE_WB,
				__builtin_return_address(0), false);
}
```

可以看到，其实ioremap默认设置cache type为UC-模式，并提供了一些变种比如ioremap_cache会传递WB模式参数，而这块内存能否设置为指定的cache type还得看`__ioremap_caller`的实现：

```c
ioremap
└── __ioremap_caller
    ├── memtype_reserve             (检查是否已设置memtype)
    │   ├── pat_x_mtrr_type
    │   │   └── mtrr_type_lookup    (遍历所有mtrr寄存器)
    │   └── memtype_check_insert    (新memtype段插入rbtree)
    ├── cachemode2protval           (memtype转换为prot)
    ├── get_vm_area_caller          (获取vm_area)
    ├── memtype_kernel_map_sync     (设置va对应的PAT bit)
    └── ioremap_page_range          (remap建立内核页表)
```

根据ioremap的实现，我们可以看出，cache type的设置分为以下几步：

1. 先检查mtrr是否已经已经对该地址设置过cache type，mtrr优先度最高
2. 构建新的memtype插入全局rbtree
3. 将cache type转换为prot，设置va对应的PAT等bit，建立页表

实际上，只有mtrr设置为WB模式后，ioremap才能设置PAT为WB模式

## 参考链接

[1] Intel® 64 and IA-32 Architectures Software Developer Manuals, chapter 12 - Memory Cache   Control
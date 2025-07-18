---
title: Linux的内存
categories:
  - 操作系统
  - Linux
tags:
  - 页表
  - 内存管理
  - 内存寻址
excerpt: 本文综合了我对本科操作系统课程的回顾已经对Linux下内存管理以及内存寻址的理解
---
<!-- more -->

写在前面，本文所讨论的是基于`x86_64`架构上的`Linux`。

#### 内存管理

##### 物理内存划分

- 什么是`UMA`？什么是`NUMA`？

  - UMA（Uniform Memory Access，统一内存访问）是一种内存体系结构，指所有CPU访问主存的速度是一致的。在计算机发展的早期，主板上通常设有北桥（Northbridge）芯片，负责连接CPU与内存、显卡等高速设备，实现了CPU对内存的统一高效访问。随着DMA技术和存储设备IO速度的发展，北桥的功能逐步被集成进CPU，现代主板上已经没有独立的北桥芯片。绝大多数个人台式机和笔记本仍然采用UMA的内存访问方式。
  - NUMA（Non-Uniform Memory Access，非统一内存访问）是一种内存体系结构，值所有的CPU访问主存的速度是不一致的，常见于大型服务器，有多个CPU，总线和巨大的主存。

- 什么是内存节点？

  在 NUMA（非统一内存访问）架构下，每个CPU（或CPU组）都被分配一块本地内存（NUMA节点），CPU访问本地节点内存时速度最快，延迟最低；而访问其他节点的内存时，需要通过跨节点总线（如QPI、Infinity Fabric等）进行数据传输，速度会有所降低。为了优化性能，操作系统通常会尽量将进程和其使用的内存分配在同一个节点上。相比之下，UMA（统一内存访问）架构下，所有CPU共享同一个内存节点，访问速度和延迟完全一致，没有本地和远程之分。

- 什么是内存区域？

  这里取64位为例：截取部分Linux内核源代码如下

  ```c
  enum zone_type {
  #ifdef CONFIG_ZONE_DMA
   ZONE_DMA,
  #endif
  #ifdef CONFIG_ZONE_DMA32
   ZONE_DMA32,
  #endif
   ZONE_NORMAL,
  #ifdef CONFIG_HIGHMEM
   ZONE_HIGHMEM,
  #endif
   ZONE_MOVABLE,
  #ifdef CONFIG_ZONE_DEVICE
   ZONE_DEVICE,
  #endif
   __MAX_NR_ZONES
  };
  ```

  从代码可知，内存区域至少有`ZONE_DMA32`，`ZONE_NORMAL`和`ZONE_MOVABLE`

  - `ZONE_DMA32`:用于外部IO设备的高速读写内存，通常在前4G内存
  - `ZONE_NORMAL`:用于常规的内存使用
  - `ZONE_MOVABLE`:主要用于存放可迁移页，方便内存碎片整理、支持大页分配和内存热插拔

- 什么是内存页面

  物理内存页面也叫做页帧。物理内存从开始起每4K、4K的，构成一个个页帧，这些页帧的编号依次是0、1、2、3......。每个页帧的真实物理地址是页帧号乘以页帧的大小。页帧是物理内存分配的最小单元，大小为固定的4KB

##### 物理内存分配

<img src="/home/wangwenhai/下载/memory_allocate.png" alt="memory_allocate" style="zoom:40%;" />

首先是Buddy System，Buddy System既是直接的内存分配接口，也是所有其它内存分配器的底层分配器。伙伴系统的基本管理单位是区域，最小分配粒度是页面。因为伙伴系统是建立在物理内存的三级区划上的，所以最小分配粒度是页面，不能比页面再小了。基本管理单位是区域，是因为每个区域的内存都有特殊的用途或者用法，不能随便混用，所以不能用节点作为基本管理单位。伙伴系统并不是直接管理一个个页帧的，而是把页帧组成页块(pageblock)来管理，页块是由连续的2^n^个页帧组成，n叫做这个页块的阶，n的范围是0到10。而且2^n^个页帧还有对齐的要求，首页帧的页帧号(pfn)必须能除尽2^n^，比如3阶页块的首页帧(pfn)必须除以8(2^3^)能除尽，10阶页块的首页帧必须除以1024(2^10^)能除尽。0阶页块只包含一个页帧，任意一个页帧都可以构成一个0阶页块，而且符合对齐要求。

- `Slab Allocator`

  `slab allocator`负责管理`slab cache`中的内存。在`slab cache` 中有多个`slab`来存储不同的对象。每个slab对应着从主存中分割来的内存块。其中，内存块按照slab要存储的对象大小提前切割好。这样设计将内存分层和对象池相切割开，为内核小对象的**高效分配**（开销小，且不存在内存碎片）和**回收**提供了强大支持

  ```css
  [slab cache]（对象类型专属池-> inode, dentry, mm_struct这些使用很频繁的数据结构）
        ├─ [slab 1]（一大块内存，分割成若干对象块）
        │       ├─ object 1
        │       ├─ object 2
        │       └─ ...
        ├─ [slab 2]
        │       ├─ object 1
        │       └─ ...
        └─ [slab 3]
                ├─ object 1
                └─ ...
  ```

  - `kmalloc` 是 Linux 内核为内核代码提供的高效小对象内存分配API，其内部实现基于 slab allocator。内核为不同大小的常用对象维护专用 slab cache，`kmalloc` 会自动选择合适的 slab cache，并从中快速分配对象。这使得内核在进行频繁的小块内存分配和释放时，能获得极高的性能和较低的碎片率。

- `vmalloc`：vmalloc则是在内核需要使用大块内存的时候，但是物理内存碎片较多无法提供，这里就会使用vmalloc来将这些物理内存碎片整合起来，这样内核看起来是连续的虚拟地址，但是实际的物理地址不是连续的。使用vmalloc可以灵活的应对碎片，充分利用内存资源

#### 内存地址

​	众所周知，在计算机实际运行的时候，物理地址(PA)是唯一。CPU等资源的数量是固定的，而进程却可以有很多，这就导致了直接访问物理地址这一方式，不仅很复杂，而且安全性差，且难以对内存进行管理。因此引入了逻辑地址即虚拟地址(VA),所有的进程访问的地址都是`VA`，在通过`MMU`（内存管理单元，负责地址转化的实际物理电路）进行从`VA`到`PA`的映射。



#### 逻辑地址的组成

​	由于技术的发展，在比较早的时期，Linux是通过逻辑地址到线性地址再到物理地址的转化。其中结合了段寻址和页寻址，而现在段寻址的方式已经被基本遗弃，更多的是直接使用页寻址的方式，通过四级页表的方式从逻辑地址到物理地址的转化。并且保存了对`LDT(local description table)`和`GDT(global description table)`的支持，当对应的字段有效时，才使用段寻址的方式去查询。因此本文着重介绍使用页寻址的方式。

​	进程的所使用的地址空间就是用虚拟地址来表示的，同样的进程的地址空间被划分为了用户态虚拟地址和内核态虚拟地址。这部分的内容会在Linux的进程中有更详细的介绍。

#### 硬件中的分页

​	现代分页的最小页表是`4KB`的大小，当然也会有一些大页（`2MB`），超大页（`1GB`）的出现，不过这些大页也都是大页和超大页是通过页表的上层（`PD、PDPT`）**直接映射大块连续物理内存**来实现的，并且管理方式有些不同。

​	同时，必须强调的一点是，现代CPU是无法直接访问内存的，而是通过利用局部性原理，使用多级缓存的设计来保证高速访问，因此根据最终转换的PA，去高速缓存中读取数据（即使发生缓存不命中，也会从内存中读入数据到高速缓存中）。为了叙述简便，后续我们仍使用“访问内存”这一表达，代表的是**通过物理地址最终完成的数据访问过程**，不论数据是否实际命中 cache。

#### Linux中的分页

​	现在的Linux采用的是4级分页模型，分别是页全局目录，页上级目录，页中间目录，页表。缩写分别为：`PGD`，`PUD`，`PMD`，`PT`。通过这四级别的映射之后，得到最终的`PTE`，最后用`PTE`的帧地址加上虚拟地址的偏移得到最终的`PA`。每次从VA到PA的转化，首先要依据`CR3`寄存器中的页全局地址，和VA前面的字段，按照上述规则去计算得到最终的PA。

#### TLB（Translation Lookaside Buffer）

​	`TLB`，即快表，是一块保存在`MMU`中的高速缓存区，用来直接保存VA到PA的映射，从而不用去多级查找，以此来提高效率。（这里也是利用了局部性原理），不过要注意区分的是，`TLB`不是保存在`Cache`中的，它是保存在`MMU`中独立的高速缓存区。因此实际的内存访问，是首先在`TLB`中访问，如果没有找到对应项，再去使用多级页表查询的方式去访问，同时更新`TLB`。

#### 最终的内存访问方式

当然可以，以下是你文档中“最终的内存访问方式”这一节的补充内容，风格与上文保持一致：

------

#### 最终的内存访问方式

​	结合上述机制，可以梳理出 Linux 在 `x86_64` 架构下完整的一次内存访问过程。假设当前 CPU 执行一个访问内存的指令（如读取变量、访问数组等），它首先会提供一个 **虚拟地址 VA**。这段地址会依次经历以下步骤，最终转换为物理地址并完成访问：

1. **TLB 查询**：CPU 首先将 VA 提交给 MMU，MMU 会在 `TLB` 中查找是否存在该虚拟页的映射（即页帧地址）。如果查找命中（称为 **TLB hit**），则立即得到物理页帧号，再与 VA 的页内偏移合并，得到最终的物理地址。
2. **页表查找（TLB miss）**：若 TLB 中未命中（**TLB miss**），则 MMU 会从 `CR3` 寄存器中获取当前活动页表的基地址，按虚拟地址的高位字段，逐级遍历页表结构（PGD → PUD → PMD → PT），最终得到 `PTE` 中的物理页帧地址。
3. **更新 TLB**：一旦找到有效的页表项，MMU 会将该 VA → PA 的映射（准确说是PA所在的页表的帧地址）插入到 `TLB` 中，以便未来的访问能更快命中。
4. **访问 Cache / 内存**：
   - 使用得到的物理地址，CPU 向 Cache 系统发起数据访问请求；
   - 若 Cache 命中，则数据直接来自 L1/L2/L3 缓存；
   - 若 Cache 不命中（**Cache miss**），则从主存（DRAM）中加载数据到缓存行，再由 CPU 使用。
5. **完成数据访问**：数据被加载到寄存器或被用于指令执行，整个访问过程完成。

​	这个过程中，**TLB 和 Cache 分别负责加速“地址转换”与“数据访问”**，共同构成现代处理器内存访问延迟优化的核心。


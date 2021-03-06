---
title: Virtual Memory and TLB
date: 2019-03-31 14:05:11
tags: Tech, Operating Susyem, Virtual Memory, TLB
author: 王克
---

# Virtual Memory and TLB

## 虚拟地址空间

x86 CPU 的地址总选宽度为32位，理论寻址上限为4GB。而虚拟地址空间的大小就是4GB，占满总线，且空间中的每一个字节分配一个虚拟地址

* 其中高2G`0x80000000 ~ 0xFFFFFFFF`为内和空间，由操作系统调用；
* 低2G`0x00000000 ~ 0x7FFFFFFF`为用户空间，由用户使用。

在系统中运行的每一个进程都独自拥有一个虚拟空间，进城之间的虚拟空间不共用。

虚拟地址空间是一种通过机制映射出来的空间，与实际物理空间大小无必然联系，在x86保护模式下，无论计算及实际主存是512MB还是8GB，虚拟地址空间总是4GB，**这是由CPU和操作系统的宽度决定的**，即：

> CPU地址总线宽度 → 物理地址范围
> CPU的ALU宽度 → 操作系统位数 → 虚拟地址范围

### 虚拟内存

虚拟地址空间 = 主存 + 虚拟内存(交换空间 Swap Space)

虚拟内存：将硬盘的一部分作为存储器使用，来扩充物理内存。

利用了自动覆盖、交换技术。内存中存不下、暂时不用的内容会存在硬盘中。

> Assume: 32位操作系统，32位寻址总线宽度 → 4G线性空间

## 保护模式下的进程运行

虚拟地址空间是硬件行为，CPU自动完成(同时与操作系统协作)虚拟地址到物理地址(可能差熬过实际内存，这样会产生一个异常中断，揭晓来有操作系统处理(如从虚拟内存中调出对应的页框内容))。

所以，一个程序若运行在保护模式下，其汇编级、机器语言级的寻址都是用的虚拟地址，即在一般的编程中不会接触到物理一层。

在进程被加载时，系统为进程建立唯一的数据结构`进程控制块(PCB = Process Control Block)`，直至进程结束。

PCB中描述了该进程的现状以及控制运行的全部信息，有了PCB，一个进程才可以在保护模式下和其他进程一起被并发地运行起来，操作系统通过PCB对进程进行控制。

PCB中的程序ID(PID(unix、linux)、句柄(windows))是进程的唯一标识；PCB中的一个指针指向 **页表** ，这些都与地址转化有关。

## 地址转化

地址转化的全过程可以用以下这张图来概括：

![OG](OG.png)

以下是具体步骤介绍。

### 1. 逻辑地址 → 线性地址 (段式内存管理，Intel早期策略的保留)

* 段内偏移地址(32位)

* 段选择符：16位长的序列，是索引值，定位段描述符；结构：
  ![#2](%232.png)
  * 高13位为表内索引号 —— 但注意由于GDT第一项留空，所以索引要先加1；
  * 而2位为TI表指示器，0是指GDT，1是指LDT；
  * 0、1位是RPL请求者特权级，00最高，11最低 —— 在x86保护模式下修改寄存器是系统之灵，必须有对应的权限才能修改(当前执行权限和段寄存器中(被修改的)的RPL均不低于目标段的RPL)

* 段描述符：8x8=64位长的结构，用来描述一个段的各种属性。结构：
   ![#1](%231.png)
  * 0、1字节+6字节低4位(20位) 段边界/段长度：最大1MB或者4G(看粒度位的单位)
  * 2、3、4、7字节(32位) 段基址：4G线性地址的任意位置(不一定非要被16整除)
  * 6、7字节的奇怪设计是为了兼容80286(24位地址总线)
  * 剩下的那些是段属性，详见`20180819143434`

* 段描述表：多任务操作系统中，含有多个任务，而每个人物都有多个段，其段描述符存于段描述表中。
  IA-32处理器有3个段描述表：GDT、LDT和IDT。
  * GDT(Global Descripter Table) 全局段描述符表：一个系统一般只有一个GDT，含有每一个任务都可以访问的段；通常包含操作系统所使用的代码段、数据段和堆栈段，GDT同时包含各进程LDT数据段项，以及进程间通讯所需要的段。
    GDTR是CPU提供的寄存器，存储GDT的位置和边界；在32位模式下RGDT有48位长(高32位基地址+低16位边界)，在32e模式下有80位长(高64位基地址+低16位边界)。
    GDT的第一个表项留空不用，是空描述符，所以索引号要加1。
    GDT最多128项。
  * LDT(Local Descripter Table) 局部段描述符表：16位长，属于某个进程。一个进程一个LDT，对应有RLDT寄存器，进程切换时RLDT改变。
    RLDT和RGDT不一样，RLDT是一个索引值而不是实际指向，指向GDT中某一个LDT描述项。所以如果要获取LDT中的某一项，先要访问GDT找到对应LDT，再找到LDT中的一项。
    编译程序时，程序内赋予了虚拟页号。在程序运行时，通过对应LDT转译成物理地址。故虚拟页号是局部性的、不同进程的页号会有冲突。
    LDT没有空选择子。
  * IDT(Interrupt Descripter Table) 中断段描述符表；一个系统一般也只有一个。
  * 以下这个图能做一点解释：
    ![#7](%237.png)

### 2. 线性地址 → 物理地址 (页式内存管理)

这一步由CPU的页式管理单元来负责转换。——MMU(内存管理单元)。

* 线性地址可以拆分为三部分(或者两部分)：
  ![#3](%233.png)

* 页(Page)：线性地址被划分为大小一致的若干内存区域，其对应映射到大小相同的与物理空间区域页框(Frame)上。这个映射不一定是连贯而有序的。

* CR3：页目录基址寄存器。对于每一个进程，CR3的内容不同(有点像RLDT)，页目录基址也不同，线性地址-物理地址的映射也不同。

* 页目录：占用一个4kb的内存页，最多存储1024个页目录表项(PDE)，一个PDE有4字节。在没启用PAE时，有两种PDE，规格不同。

* 页目录表项(PDE)：每个程序有多个页表，即拥有多个PDE。PDE的结构如下：
  ![#4](%234.png)
  12~31位(20位)表示页表起始物理地址的高20位(页表基址低12位为0，即一定以4kb对齐)。

* 页表：一个页表占4kb的内存页，最多存储1024个页表项(PTE)，一个PTE是4字节。页表的基址是4kb对齐的，低12位是0。

采用对页表项的二级管理模式(也目录→页表→页)能够节约空间。因为不存在的页表就可以不分配空间，并且对于Windows来说只有一级页表才会存在主存中，二级可以存在辅存中——不过Linux中它们都常驻主存。

一些CPU会提供更多级的架构，如三级、四级。Linux中，有对应的高层次抽象，提供了一个四层页管理架构：
![#6](%236.png)
把中间的某几个定为长度为0，就可以调整架构级数。如“四化二”：某地址0x08147258，对应的PUD、PMD里只有一个表项为PUD→PMD，PMD→PT；划分的时候，PGD=0000100000，PUD=PMD=0，PT=0101000111.

### 3. TLB (转换检测缓冲区、快表、转译后被缓冲区)

处理器中，一个具有并行朝赵能力的特殊高速缓存器，存储最近访问过的一些页表项(时空局部性原理，减少页映射的内存访问次数)。

TLB较贵，通常能够存放16~512个页表项。

* TLB命中：直接取出对应的页表项
* TLB缺失：先淘汰TLB中的某一项(TLB替换策略，一些算法，可以由硬件或软件来实现)
  * 硬件处理TLB Miss：CPU会遍历页表，找到正确的PTE；如果没有找到，CPU就会发起一个页错误并将控制权交给操作系统。
  * 软件处理TLB Miss：CPU直接发出未命中错误，让操作系统来处理。

* 脏记录：当TLB中某个PTE项失效(如切换进程、进程退出、虚拟页换出到磁盘)，PTE标记为不存在，此时映射已经不成立了。
  操作系统要保证即时刷新掉这些脏记录，不同的CPU有不同的刷新TLB方法，但每次都完全刷新TLB会很慢，所以现在有一些策略，扩展对一个PTE的描述(如针对某个进程、空间的标识，如果目前进程与PTE相关，就会忽略掉)，这样可以让多个进程同时共存TLB

## Linux 段式管理

Linux似乎没有理会Intel的那一套段的机制，而是做了一个高级的抽象。
Linux对所有的进程使用了相同的段来对指令和数据寻址，让每个段寄存器都指向同一个段描述符，让这个段描述符的基址为0，长度为4G。即用这种方式略去了段式内存管理。
对应多有用户代码段、用户数据段、内核代码段和内核数据段。可以在`segment.h`中看到，四种段对应的段基址都是0，这就是“平坦内存模型”，这样就有`段内偏移地址=逻辑地址`

且，四种段对应的都为GDT。即Linux大多数情况都不使用LDT，除非使用wine等Windows防真程序。

Linux 0.11中每个进程划分64MB的虚拟内存空间。故逻辑地址范围为0~0x4000000

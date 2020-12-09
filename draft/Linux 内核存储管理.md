---
0xFFtypora-copy-images-to: ./
---

# Linux 内核存储管理

[toc]

## 内存管理的基本框架

​	Linux内核的映射机制分三层：页面目录：`PGD`, 中间目录：`PMD`， 页表`PT`。`PT`的表项称为`PTE (Page Table Entry)`.  `PGD`  `PMD`以及`PT`三者都是数组。三者的关系如图所示。

![image-20201209084314672](D:\kevin-y-ma.github.io\draft\image-20201209084314672.png)

CPU给出的线性地址在逻辑上可以分为4个位段，各占若干位。分别用作在页面目录表`PGD`中的下标志，页中间目录表`PMD`中的下标，页表`PT`中的下标，以及物理页内的地址偏移。线性地址映射为物理地址的大致过程如下。

1. CPU发出线性地址；
2. 用1中线性地址的最高位段中的值，在`PGD`中找到相应的表项，该表项的内容指向`PMD`的地址；
3. 用1中线性地址的第二个位段中的值，在`PMD`中找到相应的表项，该表项的内容指向`PT`的地址；
4. 用1中线性地址的第三个位段中的值，在`PT`中找到相应的表项，该表项是`PTE`,即存储物理页面的地址。
5. 在4中找到的物理页中，以1中线性地址的最后一个位段的内容作为地址偏移量，找到对应的物理地址。

i386的MMU只支持两级页表，对于Pentium Pro开始Intel引入了物理地址扩充功能`PAE`，允许将32位的地址线扩展为36位，并在硬件上支持了三层映射。两层映射、三层映射甚至64位机器的四层映射，原理都是相似的，力图简单，先搞懂两层映射。

`arch/x86/include/asm/pgtable_32_types.h`中的宏`CONFIG_X86_PAE`在内核配置的过程中(`arch/x86/Kconfig`中)生成，决定是否开启`PAE`。

```c
/* arch/x86/include/asm/pgtable_32_types.h */
/*
 * The Linux x86 paging architecture is 'compile-time dual-mode', it
 * implements both the traditional 2-level x86 page tables and the
 * newer 3-level PAE-mode page tables.
 */
#ifdef CONFIG_X86_PAE
# include <asm/pgtable-3level_types.h>
# define PMD_SIZE       (1UL << PMD_SHIFT)
# define PMD_MASK       (~(PMD_SIZE - 1))
#else
# include <asm/pgtable-2level_types.h>
#endif
```

不开启`PAE`时即包含`<asm/pgtable-2level_types.h>`头文件，即采用两级页表。

```c
/* arch/x86/include/asm/pgtable-2level_types.h */
/*
 * traditional i386 two-level paging structure:
 */
#define PGDIR_SHIFT     22   /* 表示线性地址中PGD下标位段在线性地址中的起始位置，即从bit22开始到bit31共10位 */
#define PTRS_PER_PGD    1024 /* 代表每个PGD表中指针个数 */

/* arch/x86/include/asm/pgtable_32_types.h */
#define PGDIR_SIZE      (1UL << PGDIR_SHIFT) /* 表示PGD中每一个表代表1x2^22的地址空间 */
```

默认状态下，32位的地址空间分为 `0x00000000 - 0xBFFFFFFF` 3G的用户态空间，和`0xC0000000 - 0xFFFFFFFF` 1G的内核态空间。下边的配置项决定内核空间的开始位置，默认是从`0xC0000000`开始的1G空间。

```c
/* arch/x86/Kconfig */
config PAGE_OFFSET
        hex
        default 0xB0000000 if VMSPLIT_3G_OPT
        default 0x80000000 if VMSPLIT_2G
        default 0x78000000 if VMSPLIT_2G_OPT
        default 0x40000000 if VMSPLIT_1G
        default 0xC0000000
        depends on X86_32
```

关于`PAGE_OFFSET`宏的定义如下过程。

```c
/* arch/x86/include/asm/page_types.h */
#define PAGE_OFFSET             ((unsigned long)__PAGE_OFFSET)

/* 其中__PAGE_OFFSET定义在 arch/x86/include/asm/page_32_types.h*/
/*
 * This handles the memory map.
 *
 * A __PAGE_OFFSET of 0xC0000000 means that the kernel has
 * a virtual address space of one gigabyte, which limits the
 * amount of physical memory you can use to about 950MB.
 *
 * If you want more physical memory than this then see the CONFIG_HIGHMEM4G
 * and CONFIG_HIGHMEM64G options in the kernel configuration.
 */
#define __PAGE_OFFSET           _AC(CONFIG_PAGE_OFFSET, UL)

/* CONFIG_PAGE_OFFSET 由前边arch/x86/Kconfig中的配置项PAGE_OFFSET决定 */
```

由此原理内核定义了一套用于内核空间中简单的地址转换方式。

```c
/* arch/x86/include/asm/page.h */
#define __pa(x)         __phys_addr((unsigned long)(x))
#define __va(x)                 ((void *)((unsigned long)(x)+PAGE_OFFSET))

/*arch/x86/include/asm/page_32.h*/
#define __phys_addr_nodebug(x)  ((x) - PAGE_OFFSET)
#define __phys_addr(x)          __phys_addr_nodebug(x)
```



## 地址映射全过程

假设hello-world程序编译链接加载完的程序入口地址是`0x08048568`，CPU要执行此程序，首先此地址经过`MMU`得到对应的物理地址，将`CS:EIP`的地址指向该物理地址，之后CPU就可以运行此程序的代码。

首先看一下段式映射机制，即得到物理地址后CPU怎么执行。

### 段式映射机制



![image-20201209091927275](D:\kevin-y-ma.github.io\draft\image-20201209091927275.png)



```c
/* arch/x86/include/asm/processor.h */
extern void start_thread(struct pt_regs *regs, unsigned long new_ip,
                                               unsigned long new_sp);
/* /arch/x86/kernel/process_32.c */
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
        set_user_gs(regs, 0);
        regs->fs                = 0;
        set_fs(USER_DS);
        regs->ds                = __USER_DS;
        regs->es                = __USER_DS;
        regs->ss                = __USER_DS;
        regs->cs                = __USER_CS;
        regs->ip                = new_ip;
        regs->sp                = new_sp;
        /*
         * Free the old FP and other extended state
         */
        free_thread_xstate(current);
}
EXPORT_SYMBOL_GPL(start_thread);

/* /arch/x86/include/asm/segment.h */
#define __KERNEL_CS     (GDT_ENTRY_KERNEL_CS * 8) 
								/*(12+0)*8=96 : 60H :	0000 0000 0110 0000*/
#define __KERNEL_DS     (GDT_ENTRY_KERNEL_DS * 8) 
								/*(12+1)*8=104 : 68H :	0000 0000 0110 1000*/
#define __USER_DS     (GDT_ENTRY_DEFAULT_USER_DS* 8 + 3)
								/* 15*8+3=123 : 7BH :	0000 0000 0111 1011*/
#define __USER_CS     (GDT_ENTRY_DEFAULT_USER_CS* 8 + 3)
								/* 14*8+3=115 : 73H :	0000 0000 0111 0011*/


#define GDT_ENTRY_KERNEL_BASE   12
#define GDT_ENTRY_KERNEL_CS             (GDT_ENTRY_KERNEL_BASE + 0)
#define GDT_ENTRY_KERNEL_DS             (GDT_ENTRY_KERNEL_BASE + 1)

#define GDT_ENTRY_DEFAULT_USER_CS       14
#define GDT_ENTRY_DEFAULT_USER_DS       15

```

对应到段寄存器格式定义可得下边的表，即四个段全在`GDT`中。

|             | index | TI   | RPL  |
| ----------- | ----- | ---- | ---- |
| __KERNEL_CS | 12    | 0    | 0    |
| __KERNEL_DS | 13    | 0    | 0    |
| __USER_DS   | 15    | 0    | 3    |
| __USER_CS   | 14    | 0    | 3    |



内核`GDT`表的映像在`arch/x86/include/asm/segment.h`的注释可以看出来。

```c
/*  
 * The layout of the per-CPU GDT under Linux:
 *
 *   0 - null
 *   1 - reserved
 *   2 - reserved
 *   3 - reserved
 *
 *   4 - unused                 <==== new cacheline
 *   5 - unused
 *
 *  ------- start of TLS (Thread-Local Storage) segments:
 *

 *   6 - TLS segment #1                 [ glibc's TLS segment ]
 *   7 - TLS segment #2                 [ Wine's %fs Win32 segment ]
 *   8 - TLS segment #3
 *   9 - reserved
 *  10 - reserved
 *  11 - reserved
 *
 *  ------- start of kernel segments:
 *
 *  12 - kernel code segment            <==== new cacheline
 *  13 - kernel data segment
 *  14 - default user CS
 *  15 - default user DS
 *  16 - TSS
 *  17 - LDT
 *  18 - PNPBIOS support (16->32 gate)
 *  19 - PNPBIOS support
 *  20 - PNPBIOS support
 *  21 - PNPBIOS support
 *  22 - PNPBIOS support
 *  23 - APM BIOS support
 *  24 - APM BIOS support
 *  25 - APM BIOS support
 *
 *  26 - ESPFIX small SS
 *  27 - per-cpu                        [ offset to per-cpu data area ]
 *  28 - stack_canary-20                [ for stack protector ]
 *  29 - unused
 *  30 - unused
 *  31 - TSS for double fault handler
 */
/* Constructor for a conventional segment GDT (or LDT) entry */
/* This is a macro so it can be used in initializers */
#define GDT_ENTRY(flags, base, limit)                   \
        ((((base)  & 0xff000000ULL) << (56-24)) |       \
         (((flags) & 0x0000f0ffULL) << 40) |            \
         (((limit) & 0x000f0000ULL) << (48-16)) |       \
         (((base)  & 0x00ffffffULL) << 16) |            \
         (((limit) & 0x0000ffffULL)))

```

关于`gdt`的建立过程，在内核的初始化阶段完成，搜索内核代码发现在`arch/x86/kernel/cpu/common.c`定义。

```c
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
   /* 省略其他 */
 [GDT_ENTRY_KERNEL_CS]           = { { { 0x0000ffff, 0x00cf9a00 } } },
 [GDT_ENTRY_KERNEL_DS]           = { { { 0x0000ffff, 0x00cf9200 } } },
 [GDT_ENTRY_DEFAULT_USER_CS]     = { { { 0x0000ffff, 0x00cffa00 } } },
 [GDT_ENTRY_DEFAULT_USER_DS]     = { { { 0x0000ffff, 0x00cff200 } } },
    /*省略其他*/
} };
EXPORT_PER_CPU_SYMBOL_GPL(gdt_page);
/* 
	GDT_ENTRY_KERNEL_CS = 12
	GDT_ENTRY_KERNEL_DS = 13
	GDT_ENTRY_DEFAULT_USER_CS = 14
	GDT_ENTRY_DEFAULT_USER_DS = 15
	这里还有一点需要注意
	{ 0x0000ffff, 0x00cf9a00 }在内存中应该是0x00cf9a000000ffff
*/

/*include/linux/percpu-defs.h" */
#define DEFINE_PER_CPU_PAGE_ALIGNED(type, name)                         \
        DEFINE_PER_CPU_SECTION(type, name, ".page_aligned")

/*include/linux/percpu-defs.h */
#define DEFINE_PER_CPU_SECTION(type, name, section)                     \
        __attribute__((__section__(PER_CPU_BASE_SECTION section)))      \
        PER_CPU_ATTRIBUTES PER_CPU_DEF_ATTRIBUTES                       \
        __typeof__(type) per_cpu__##name

```

![image-20201209100428575](D:\kevin-y-ma.github.io\draft\image-20201209100428575.png)

参照`gdt`表项定义可以得出以下结论

|             | B31-B24 |      | L19-L16 |      | B23-B16 | B15-B0 | L15-L0 |
| ----------- | ------- | ---- | ------- | ---- | ------- | ------ | ------ |
| __KERNEL_CS | 0x00    | 0xC  | 0xF     | 0x9A | 0x00    | 0x0000 | 0xFFFF |
| __KERNEL_DS | 0x00    | 0xC  | 0xF     | 0x92 | 0xFF    | 0x00CF | 0xFFFF |
| __USER_CS   | 0x00    | 0xC  | 0xF     | 0xFA | 0xFF    | 0x00CF | 0xFFFF |
| __USER_DS   | 0x00    | 0xC  | 0xF     | 0xF2 | 0xFF    | 0x00CF | 0xFFFF |

即

- B0-B15, B16-B24 全是0， 段的基地址是0；
- L0-L15， L16-L19 全是1， 段的上限是0xFFFFF;
- G位都是1， 段长单位都为4KB；
- D/B位都为1，对四个段访问的指令都是32位的；
- P位都为1，四个段都在内存中；
- bit40-bit46字段有区别
  - __KERNEL_CS ： DPL = 0； S = 1代码或数据段；101 代码段，可读；未被访问过；
  - __KERNEL_DS ： DPL = 0； S = 1代码或数据段；001 数据段，可读，可写；未被访问过；
  - __USER_CS ： DPL = 3； S = 1代码或数据段；101 代码段，可读；未被访问过；
  - __USER_DS ： DPL = 3； S = 1代码或数据段；001 数据段，可读，可写；未被访问过；

小结

​	分段机制中，所有进程公用一个`GDT`，主要分了四个段：内核代码段、内核数据段、用户代码段、用户数据段，所有进程共享这四个段。这四个段的映射整个内存空间。

### 页式映射机制

分页机制主要完成虚拟地址（线性地址）到物理地址的映射。

分页机制中，每个进程都有其自己的页目录`PGD`，指向这个目录的指针保存在每个进程的`mm_struct`结构中，每当调度一个进程进入运行时，内核都据此设置`CR3`寄存器，`MMU`的硬件又从`CR3`中取得`PGD`的地址。

CPU给出线性地址`0x08048568`，写成二进制形式`0000 1000 0000 0100 1000 0101 0110 1000`。

- 查`PGD`： `0000 1000 00`十进制为32，以32位为下标，在`PGD`中找到相应表项，取此表项的高20位后边补12个0组成`PT`的地址。
- 查`PT` : `00 0100 1000` 十进制为72，以72位下标，在`PT`中找到相应的表项，取此表项的高20位后边补12个0，组成页表的地址。
- 查页表：线性地址中省下的12位` 0101 0110 1000`为地址偏移，上一步查到的页表地址加上此偏移，得到线性地址`0x08048568`相应的物理地址。

此过程中共访存了3次，第一次取`PGD[32]`，第二次取`PT[72]`，第三次取`PT[72]+0101 0110 1000`得到物理地址。 

时间优化： 快表 `TLB`;

空间优化： 多级页表。

---

内核启动过程和启动完成后`gdt`有所不同，以下为内核启动时`gdt`的初始化函数调用过程。

参考[这篇文章](https://blog.csdn.net/baidu_31504167/article/details/100525791)

```c
arch/x86/boot/main.c -> main()
    -> arch/x86/boot/pm.c : go_to_protected_mode()
        -> arch/x86/boot/pm.c : setup_gdt();
static void setup_gdt(void)
{
        /* There are machines which are known to not boot with the GDT
           being 8-byte unaligned.  Intel recommends 16 byte alignment. */
        static const u64 boot_gdt[] __attribute__((aligned(16))) = {
                /* CS: code, read/execute, 4 GB, base 0 */
                [GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
                /* DS: data, read/write, 4 GB, base 0 */
                [GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
                /* TSS: 32-bit tss, 104 bytes, base 4096 */
                /* We only have a TSS here to keep Intel VT happy;
                   we don't actually use it for anything. */
                [GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
        };
        /* Xen HVM incorrectly stores a pointer to the gdt_ptr, instead
           of the gdt_ptr contents.  Thus, make it static so it will
           stay in memory, at least long enough that we switch to the
           proper kernel GDT. */
        static struct gdt_ptr gdt;

        gdt.len = sizeof(boot_gdt)-1;
        gdt.ptr = (u32)&boot_gdt + (ds() << 4);

        asm volatile("lgdtl %0" : : "m" (gdt));
}

#define GDT_ENTRY_BOOT_CS       2
#define GDT_ENTRY_BOOT_DS       (GDT_ENTRY_BOOT_CS + 1)

```



## 几个重要的数据结构

![image-20201209182949066](D:\kevin-y-ma.github.io\draft\image-20201209182949066.png)

### pgd_t / pmd_t / pte_t



```c
/* arch/x86/include/asm/pgtable_types.h" */
typedef struct { pgdval_t pgd; } pgd_t;
#if PAGETABLE_LEVELS > 2
typedef struct { pmdval_t pmd; } pmd_t; /* 这里为了简单只看 */
#esle
typedef struct { pud_t pud; } pmd_t; //include/asm-generic/pgtable-nopmd.h
#endif


/* arch/x86/include/asm/pgtable-2level_types.h */

typedef unsigned long   pteval_t;
typedef unsigned long   pmdval_t;
typedef unsigned long   pudval_t;
typedef unsigned long   pgdval_t;
typedef unsigned long   pgprotval_t;

typedef union {
        pteval_t pte;
        pteval_t pte_low;
} pte_t;

```

页面目录`PGD`  中间目录`PMD` 页表`PT` 分别是由表项`pgd_t` `pmd_t` `pte_t`组成的数组。

```c
/* arch/x86/include/asm/pgtable_types.h  */
typedef struct pgprot { pgprotval_t pgprot; } pgprot_t; //页面保护的结构
```



### struct page 

内核中有一个全局量`mem_map`（定义在mm/memory.c的70行`struct page *mem_map;`）是一个指针，指向一个`page`数据结构的数组，每个`page`代表着一个物理页面，整个数组就代表着系统中的全部物理页面。系统在初始化时根据物理内存的大小建立`mem_map`。

```c
/* include/linux/mm_types.h +40 */
/*
 * Each physical page in the system has a struct page associated with
 * it to keep track of whatever it is we are using the page for at the
 * moment. Note that we have no way to track which tasks are using
 * a page, though if it is a pagecache page, rmap structures can tell us
 * who is mapping it.
 */
struct page {
       /* 内容暂时不看 */
};

```

如果把物理内存看做一个物理页面的“数组”，高20位就是数组的下标，也就是物理页面的序号，用这个下标，可以在上述`page`结构数组中找到代表目标物理页面的数据结构，实现如下边代码。

```c
/* arch/x86/include/asm/pgtable.h */
#define pte_page(pte)   pfn_to_page(pte_pfn(pte))
130 static inline unsigned long pte_pfn(pte_t pte)
131 {
132         return (pte_val(pte) & PTE_PFN_MASK) >> PAGE_SHIFT;
    /*
    	176 /* PTE_PFN_MASK extracts the PFN from a (pte|pmd|pud|pgd)val_t */
		177 #define PTE_PFN_MASK            ((pteval_t)PHYSICAL_PAGE_MASK)

    */
133 }
/*include/asm-generic/memory_model.h*/
70 #define pfn_to_page __pfn_to_page
30 #define __pfn_to_page(pfn)      (mem_map + ((pfn) - ARCH_PFN_OFFSET))
    
  8 #ifndef ARCH_PFN_OFFSET
  9 #define ARCH_PFN_OFFSET         (0UL)
 10 #endif

/*
即
    pte_page(pte) 展开为 (mem_map + ((pte) - 0))
    其中pte = (pte_val(pte) & PTE_PFN_MASK) >> PAGE_SHIFT
*/
```



### zone_t



### pg_data_t



---

之前的几个结构都是用于物理空间管理的，下面是虚拟空间管理的。  

虚拟空间的管理是以进程为基础的，每个进程都有各自的虚拟内存空间，所有进程的内核空间是共享的。

### struct vm_area_struct

一个进程所需要使用的虚拟空间中的各个部位未必连续，通常形成若干离散的虚存**区间**，对虚存区间的抽象就是数据结构`struct vm_area_struct`.

```c
/* include/linux/mm_types.h */
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
        struct mm_struct * vm_mm;       /* The address space we belong to. */
        unsigned long vm_start;         /* Our start address within vm_mm. */
        unsigned long vm_end;           /* The first byte after our end address
                                           within vm_mm. */

        /* linked list of VM areas per task, sorted by address */
        struct vm_area_struct *vm_next;

        pgprot_t vm_page_prot;          /* Access permissions of this VMA. */
        unsigned long vm_flags;         /* Flags, see mm.h. */

        struct rb_node vm_rb;

        /*
         * For areas with an address space and backing store,
         * linkage into the address_space->i_mmap prio tree, or
         * linkage to the list of like vmas hanging off its node, or
         * linkage of vma in the address_space->i_mmap_nonlinear list.
         */
        union {
                struct {
                        struct list_head list;
                        void *parent;   /* aligns with prio_tree_node parent */
                        struct vm_area_struct *head;
                } vm_set;

                struct raw_prio_tree_node prio_tree_node;
        } shared;

        /*
         * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
         * list, after a COW of one of the file pages.  A MAP_SHARED vma
         * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
         * or brk vma (with NULL file) can only be in an anon_vma list.
         */
        struct list_head anon_vma_node; /* Serialized by anon_vma->lock */
        struct anon_vma *anon_vma;      /* Serialized by page_table_lock */

        /* Function pointers to deal with this struct. */
        struct vm_operations_struct * vm_ops;

        /* Information about our backing store: */
        unsigned long vm_pgoff;         /* Offset (within vm_file) in PAGE_SIZE
                                           units, *not* PAGE_CACHE_SIZE */
        struct file * vm_file;          /* File we map to (can be NULL). */
        void * vm_private_data;         /* was vm_pte (shared mem) */
        unsigned long vm_truncate_count;/* truncate_count or restart_addr */
#ifndef CONFIG_MMU
        struct vm_region *vm_region;    /* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
        struct mempolicy *vm_policy;    /* NUMA policy for the VMA */
#endif
};
```

- `vm_mm` : 指向一个`mm_struct`结构，是比`vm_area_struct`更高层次的结构，隶属于`task_struct`。
- `vm_start`和`vm_end` : 决定了虚存区间，前闭后开即start在区间内，end不包含在区间内；
- `vm_next` :  隶属于同一个进程的所有区间的结构链表指针；
- `vm_page_prot` 和`vm_flags` : 表示同一个区间所有页面相同的访问权限以及其他一些属性；
- `vm_rb` :   红黑树节点，当区间数量较大时只通过`vm_next`链式搜索速度太慢，加速搜索；
- `vm_ops` : 指向一个`vm_operations_struct`指针，此结构中保存了一些函数指针，用于操作此区间，比如打开，关闭，建立映射以及处理缺页异常等操作。
- 在两种情况下虚存页面（或区间）会和磁盘文件发生联系，`mapping`  `vm_file`等几个结构保存这种联系。
  - swap，内存页面不够分配时，将一些久未使用的页面交换到磁盘上去；
  - 通过`mmap()`将一个磁盘文件映射到进程的用户空间。

```c
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs.
 */
struct vm_operations_struct {
        void (*open)(struct vm_area_struct * area);
        void (*close)(struct vm_area_struct * area);
        int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf);

        /* notification that a previously read-only page is about to become
         * writable, if an error is returned it will cause a SIGBUS */
        int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);

        /* called by access_process_vm when get_user_pages() fails, typically
         * for use by special VMAs that can switch between memory and hardware
         */
        int (*access)(struct vm_area_struct *vma, unsigned long addr,
                      void *buf, int len, int write);
#ifdef CONFIG_NUMA
        /*
         * set_policy() op must add a reference to any non-NULL @new mempolicy
         * to hold the policy upon return.  Caller should pass NULL @new to
         * remove a policy and fall back to surrounding context--i.e. do not
         * install a MPOL_DEFAULT policy, nor the task or system default
         * mempolicy.
         */
        int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);
        /*
         * get_policy() op must add reference [mpol_get()] to any policy at
         * (vma,addr) marked as MPOL_SHARED.  The shared policy infrastructure
         * in mm/mempolicy.c will do this automatically.
         * get_policy() must NOT add a ref if the policy at (vma,addr) is not
         * marked as MPOL_SHARED. vma policies are protected by the mmap_sem.
         * If no [shared/vma] mempolicy exists at the addr, get_policy() op
         * must return NULL--i.e., do not "fallback" to task or system default
         * policy.
         */
        struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
                                        unsigned long addr);
        int (*migrate)(struct vm_area_struct *vma, const nodemask_t *from,
                const nodemask_t *to, unsigned long flags);
#endif
};

```





### struct mm_struct

每个进程只有一个`mm_struct`结构，`mm_struct`是进程整个用户空间的抽象。

```c
struct mm_struct {
        struct vm_area_struct * mmap;           /* list of VMAs */
        struct rb_root mm_rb;
        struct vm_area_struct * mmap_cache;     /* last find_vma result */
        unsigned long (*get_unmapped_area) (struct file *filp,
                                unsigned long addr, unsigned long len,
                                unsigned long pgoff, unsigned long flags);
        void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
        unsigned long mmap_base;                /* base of mmap area */
        unsigned long task_size;                /* size of task vm space */
        unsigned long cached_hole_size;         /* if non-zero, the largest hole below free_area_cache */
        unsigned long free_area_cache;          /* first hole of size cached_hole_size or larger */
        pgd_t * pgd;
        atomic_t mm_users;                      /* How many users with user space? */
        atomic_t mm_count;                      /* How many references to "struct mm_struct" (users count as 1) */
        int map_count;                          /* number of VMAs */
        struct rw_semaphore mmap_sem;
        spinlock_t page_table_lock;             /* Protects page tables and some counters */

        struct list_head mmlist;                /* List of maybe swapped mm's.  These are globally strung
                                                 * together off init_mm.mmlist, and are protected
                                                 * by mmlist_lock
                                                 */

        /* Special counters, in some configurations protected by the
         * page_table_lock, in other configurations by being atomic.
         */
        mm_counter_t _file_rss;
        mm_counter_t _anon_rss;

        unsigned long hiwater_rss;      /* High-watermark of RSS usage */
       unsigned long hiwater_vm;       /* High-water virtual memory usage */

        unsigned long total_vm, locked_vm, shared_vm, exec_vm;
        unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
        unsigned long start_code, end_code, start_data, end_data;
        unsigned long start_brk, brk, start_stack;
        unsigned long arg_start, arg_end, env_start, env_end;

        unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

        cpumask_t cpu_vm_mask;

        /* Architecture-specific MM context */
        mm_context_t context;

        /* Swap token stuff */
        /*
         * Last value of global fault stamp as seen by this process.
         * In other words, this value gives an indication of how long
         * it has been since this task got the token.
         * Look at mm/thrash.c
         */
        unsigned int faultstamp;
        unsigned int token_priority;
        unsigned int last_interval;

        unsigned long flags; /* Must use atomic bitops to access the bits */

        struct core_state *core_state; /* coredumping support */

        /* aio bits */
        spinlock_t              ioctx_lock;
        struct hlist_head       ioctx_list;

#ifdef CONFIG_MM_OWNER
        /*
         * "owner" points to a task that is regarded as the canonical
         * user/owner of this mm. All of the following must be true in
         * order for it to be changed:
         *
         * current == mm->owner
         * current->mm != mm
         * new_owner->mm == mm
         * new_owner->alloc_lock is held
         */
        struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
        /* store ref to file /proc/<pid>/exe symlink points to */
        struct file *exe_file;
        unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
        struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
```

- `* mmap` ： 用来建立一个虚存区间`vm_area_struct`的单链线性队列；
- `mm_rb` : 用来建立`vm_area_struct`的红黑树结构；
- `* mmap_cache` : 用来指向最近一次用到的`vm_area_struct`结构；
- `* pgd` : 指向该进程的页表目录，当内核调度一个进程进入运行时，就将这个地址转换成物理地址，并写入`CR3`寄存器；
- `map_count` : 说明此进程有多少个`vm_area_struct`结构；
- `mm_users`和`mm_count` : `atomic_t`（原子）类型，一个进程只能有一个`mm_struct`，但是一个`mm_struct`可能同时为多个进程服务，此两个变量用以计数。
- `mmap_sem`和`page_table_lock` : 用于此结构的信号量和用于页表的锁，用于进程间的互斥和并发；
- `start_code, end_code, start_data, end_data` : 进程的代码段、数据段，此外还有堆栈段等的起点和终点。
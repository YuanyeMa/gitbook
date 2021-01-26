# Linux 内核临时页表初始化

> 基于 linux 2.6.30.4分析x86平台相关代码



- **64位系统：**使用四级分页或三级分页，跟硬件有关。
- **未开启PAE(物理地址扩展)的32位系统：**只使用二级分页，页上级目录和页中间目录里的值全为0。
- **开启PAE的32位系统：**使用三级分页，这种情况下被排除在外的是页上级目录，也就是页上级目录中所有值都为0。

## 表项

实际上页全局目录、页上级目录、页中间目录、页表都是保存在一个一个页框中，我们知道常规情况下页框大小为4K(特殊情况有2MB、1GB)，也就是页框的布局都是以4K倍数的地址进行排列的，要寻址一个页框，只需要20位地址就足够了。这些目录和页表中保存的都是表项，页全局目录保存的是页全局目录项，页中间目录保存的是页中间目录项，在32位系统中这些项都是32位(20位是所指页框的基地址，12位是标志位)的，在开启PAE后会变成64位，这些项保存着很多标志，我们罗列几个重要的：

- Present标志：为1，所指的页在内存中，为0，不在。
- 所指的页框基地址：占20位。
- Accessed标志：每当分页单元对相应页框进行寻址时设置。
- Dirty标志：只用于页表项，每次对一个页框进行写操作时设置。
- Read/Write标志：读写权限标志。
- User/Supervisor标志：所指的页的特权级(进程能否访问)。
- Page size标志：为1表示指的是2MB或4MB的页框。也就是页表是2MB或者4MB。

　　在这些里面，最重要的或许就是所指页框基地址了，一个页中间目录项保存的页框基地址就是对应的页表的基地址，而页表项中保存的页框基地址，就是页(用于保存数据)的地址。而Present标志是用于判断是否发生缺页异常处理的标志。由于这些标志加上所指的页框基地址一共32位，一个4K的页框就能够保存1024个表项。



## 物理地址扩展(PAE)

　　这个技术是用于X86_32位体系下的，因为32位线性地址最多能表示4GB大小的空间，而PAE技术将物理地址线扩大到36条，也就是CPU能够寻址64GB大小的物理内存。但是物理地址线扩大到36条，但是线性地址还是使用32位，这时候没办法用32位的线性地址去表示64GB大小的物理内存。实际上PAE做的就是让内核有多个“主内核页全局目录”，第一个主内核页全局目录寻址0~4GB的地址，第二个寻址5~8GB的地址，所以当寻址不同区域的地址时，只需要将不同的“主内核页全局目录”基地址存入cr3中。这些多个主内核页全局目录被称为**页目录指针表(PDPT)**。

　　开启PAE后，32位系统寻址方式将大大改变：

- 二级分页会变成三级分页
- 表项的大小也由原来的32位变成了64位(原来是12位标志+20位页框基地址，变成12位标志+24位页框基地址)。
- 页框大小将可选择4K或者2MB，通过修改表项中的Page size标志即可指定所指页框大小。
- 线性地址表示也变成如下：
  - 当把线性地址映射为4KB的页时（页目录项中的PS标志清零），32位线性地址按以下方式解释：
    - cr3 : 指向一个PDPT；
    - 位31-30 ： 指向PDPT中4项中的一个；
    - 位29-21 ： 指向页目录中512个项中的一个；
    - 位20-12 ： 指向页表中512个项中的一个；
    - 位 11 - 0： 4KB页中的偏移量；
  - 当把线性地址映射为2MB的页（页目录项中的PS标志置1）时，32位线性地址按下列方式解释：
    - cr3 : 指向一个PDPT；
    - 位31-30 ： 指向PDPT中4项中的一个；
    - 位29-21 ： 指向页目录中512个项中的一个；
    - 位 20 - 0： 2MB页中的偏移量；



## 临时内核页表的构造

x86系统刚刚启动的时候，运行在实模式下，这个时候线性地址就是物理地址。为了进入32位保护模式，首先就要启用分页。这就要求我们构建一个页表：这张页表把线性地址映射转换为物理地址。

为了解决构造页表时鸡生蛋 蛋生鸡的问题，Linux使用了一个临时的内核页表。它只有两个页表项（这里指用来索引页框的最后一级页表）。在不启用PAE(Page Addression Extension) 和 PSE (Page Size Extension)的情况下，一个页表可以指向 `2^10 = 1024`个内存页，一个内存页 4k ， 所以两个页表允许索引8M的内存。

### `swapper_pg_dir`

顶层的页目录 (page directory) 使用全局变量 `swapper_pg_dir`定义。

```c
/* 在 arch/x86/include/asm/pgtable_32.h 中声明为外部全局变量 */
extern pgd_t swapper_pg_dir[1024];

/* 其中pgd_t的定义如下 */
typedef struct { pgdval_t pgd; } pgd_t;
typedef unsigned long   pgdval_t;
/* 即，swapper_pg_dir 是unsigned long类型的数组，数组大小为1024 */


/* 在 arch/x86/kernel/head_32.S 中定义此变量 */
/*
 * BSS section
 */
.section ".bss.page_aligned","wa"
        .align PAGE_SIZE_asm
#ifdef CONFIG_X86_PAE
swapper_pg_pmd:
        .fill 1024*KPMDS,4,0
#else
ENTRY(swapper_pg_dir)
        .fill 1024,4,0
#endif
swapper_pg_fixmap:
        .fill 1024,4,0
ENTRY(empty_zero_page)
        .fill 4096,1,0

```

`.fill 1024, 4, 0` 表示用0填充1024个 4 byte 长度的内存（一个页目录项 page table entry 的大小是32bits 即 4Byte）。

### pg0

ULK3 P74 原文 ： 临时页全局目录放在 swapper_pg_dir地址处，临时页表在pg0地址处，紧接在内核bss段之后。在linux 2.6.30.4中没有找到pg0变量。在分析代码时发现，应该是替换为了变量 `__brk_base`

```assembly
/* __brk_base 定义在 arch/x86/kernel/vmlinux_32.lds.S 中*/
  .bss : AT(ADDR(.bss) - LOAD_OFFSET) {
        __init_end = .;
        __bss_start = .;                /* BSS */
        *(.bss.page_aligned)
        *(.bss)
        . = ALIGN(4);
        __bss_stop = .;
  }

  .brk : AT(ADDR(.brk) - LOAD_OFFSET) {
        . = ALIGN(PAGE_SIZE);
        __brk_base = . ;
        . += 64 * 1024 ;        /* 64k alignment slop space */
        *(.brk_reservation)     /* areas brk users have reserved */
        __brk_limit = . ;
  }
    _end = . ;
```



### 初始化过程

```c
/*
 * Initialize page tables.  This creates a PDE and a set of page
 * tables, which are located immediately beyond __brk_base.  The variable
 * _brk_end is set up to point to the first "safe" location.
 * Mappings are created both at virtual address 0 (identity mapping)
 * and PAGE_OFFSET for up to _end.
 *
 * Note that the stack is not yet set up!
 */
default_entry:
#ifdef CONFIG_X86_PAE
	
	/* 暂时不看 */
	
#else   /* Not PAE */

page_pde_offset = (__PAGE_OFFSET >> 20);
		/* 其中 __PAGE_OFFSET = 0xC000 0000 所以 page_pde_offset = 0xC00 */

        movl $pa(__brk_base), %edi
        movl $pa(swapper_pg_dir), %edx
            /*
           		/* Physical address */
				#define pa(X) ((X) - __PAGE_OFFSET)
            */
        movl $PTE_IDENT_ATTR, %eax
10:
        leal PDE_IDENT_ATTR(%edi),%ecx          /* Create PDE entry */
        movl %ecx,(%edx)                        /* Store identity PDE entry */
        movl %ecx,page_pde_offset(%edx)         /* Store kernel PDE entry */
        addl $4,%edx
        movl $1024, %ecx
11:
        stosl
        addl $0x1000,%eax
        loop 11b
        /*
         * End condition: we must map up to the end + MAPPING_BEYOND_END.
         */
        movl $pa(_end) + MAPPING_BEYOND_END + PTE_IDENT_ATTR, %ebp
        cmpl %ebp,%eax
        jb 10b
        addl $__PAGE_OFFSET, %edi
        movl %edi, pa(_brk_end)
        shrl $12, %eax
        movl %eax, pa(max_pfn_mapped)

        /* Do early initialization of the fixmap area */
        movl $pa(swapper_pg_fixmap)+PDE_IDENT_ATTR,%eax
        movl %eax,pa(swapper_pg_dir+0xffc)
#endif
             jmp 3f
```

> stosl 指令 ： 相当于将eax中的值保存到 ES:EDI 所指向的地址中，若设置了EFLAGS中的方向位，则EDI自减4否则自增4。  
>
> loop 指令 : 借助ECX寄存器作为计数实现循环，每轮循环先将ECX减一，再判断ECX的值，循环直到ECX为0时止。
>
> leal 指令 ：load effective address 功能是取偏移地址  
>
> ​	mov 是将数据从源传到目的； lea是将源目的地址传到目的  
>
> ​	例如   
>
> ​		movl 18(%eax), %ebx # 将内存中 %eax+18内存处的内容，传递到 %ebx中；  
>
> ​		leal 18(%eax), %ebx  # 将（18+%eax中内容）的值，传入%ebx;

```assembly

/* PTE_IDENT_ATTR和PDE_IDENT_ATTR 定义在arch/x86/include/asm/pgtable_types.h中 */
/*
 * early identity mapping  pte attrib macros.
 */
#ifdef CONFIG_X86_64
#define __PAGE_KERNEL_IDENT_LARGE_EXEC  __PAGE_KERNEL_LARGE_EXEC
#else
/*
 * For PDE_IDENT_ATTR include USER bit. As the PDE and PTE protection
 * bits are combined, this will alow user to access the high address mapped
 * VDSO in the presence of CONFIG_COMPAT_VDSO
 */
#define PTE_IDENT_ATTR   0x003          /* PRESENT+RW */
#define PDE_IDENT_ATTR   0x067          /* PRESENT+RW+USER+DIRTY+ACCESSED */
#define PGD_IDENT_ATTR   0x001          /* PRESENT (no other attributes) */
#endif
```

分步骤来看

1.  初始化变量

    ```assembly
    page_pde_offset = (__PAGE_OFFSET >> 20);
            /* 其中 __PAGE_OFFSET = 0xC000 0000 所以 page_pde_offset = 0xC00 */

            movl $pa(__brk_base), %edi		# 将pg0的物理地址存储到 %edi寄存器中
            movl $pa(swapper_pg_dir), %edx	# 将页全局目录swapper_pg_dir的地址存到寄存器%edx中
            movl $PTE_IDENT_ATTR, %eax		# 存储PTE的属性 PRESENT+RW 到%eax中
    ```

2. 设置页全局目录

   ```assembly
   10:
           leal PDE_IDENT_ATTR(%edi),%ecx          /* Create PDE entry */
           movl %ecx,(%edx)                        /* Store identity PDE entry */
           movl %ecx,page_pde_offset(%edx)         /* Store kernel PDE entry */
           addl $4,%edx
           movl $1024, %ecx
   ```

   > pde ： page directory entry 页目录表

   - 先将pg0的物理地址+PDE_IDENT_ATTR -> %ecx； 即将页表的地址加上页表的属性写入到%ECX;

   - %ecx -> swapper_pg_dir的物理地址；即将页表的地址写入到页目录中；

   - %ecx -> swapper_pg_dir的物理地址+page_pde_offset；将页表地址写入到内核页表所在位置；

   - 4+%edx ; 即将指针往后移动四个字节，指向第二个页目录项，为初始化第二个页表做准备；

   - 最后将1024写入%ecx， 作为初始化页表时的循环变量。

3.   初始化页表

    ```assembly
    11:
            stosl ;将%eax的内容(0+PRESENT+RW)复制到 %es:%edi ，即pg0第一个表项处，并%edi+4 ;
            addl $0x1000,%eax ; %eax+0x1000, 即 %eax+4K, 
            loop 11b ; %ecx-=1, 若%ecx!=0, 跳转到标号11处，继续执行，即循环1024次。
    ```

4.  初始化剩下的页表

    ```assembly
            /*
             * End condition: we must map up to the end + MAPPING_BEYOND_END.
             */
            movl $pa(_end) + MAPPING_BEYOND_END + PTE_IDENT_ATTR, %ebp
            cmpl %ebp,%eax
            jb 10b
    ```

    其中 `_end`在前边连接器脚本中定义，`MAPPING_BEYOND_END`定义如下，`PTE_IDENT_ATTR`为页表的属性。

    ```assembly
    /*
     * This is how much memory in addition to the memory covered up to
     * and including _end we need mapped initially.
     * We need:
     *     (KERNEL_IMAGE_SIZE/4096) / 1024 pages (worst case, non PAE)
     *     (KERNEL_IMAGE_SIZE/4096) / 512 + 4 pages (worst case for PAE)
     *
     * Modulo rounding, each megabyte assigned here requires a kilobyte of
     * memory, which is currently unreclaimed.
     *
     * This should be a multiple of a page.
     *
     * KERNEL_IMAGE_SIZE should be greater than pa(_end)
     * and small than max_low_pfn, otherwise will waste some page table entries
     */
    
    #if PTRS_PER_PMD > 1
    #define PAGE_TABLE_SIZE(pages) (((pages) / PTRS_PER_PMD) + PTRS_PER_PGD)
    #else
    #define PAGE_TABLE_SIZE(pages) ((pages) / PTRS_PER_PGD)
    #endif
    
    /* Enough space to fit pagetables for the low memory linear map */
    MAPPING_BEYOND_END = \
            PAGE_TABLE_SIZE(((1<<32) - __PAGE_OFFSET) >> PAGE_SHIFT) << PAGE_SHIFT
    
    ```

    

### 启用分页

初始化过程的最后执行了` jmp 3f` , 标号3处执行PAE相关的判断和操作，最后跳转到了标号6处执行。

```assembly
6:

/*
 * Enable paging
 */
        movl $pa(swapper_pg_dir),%eax
        movl %eax,%cr3          /* set the page table pointer.. */
        movl %cr0,%eax
        orl  $X86_CR0_PG,%eax
        movl %eax,%cr0          /* ..and set paging (PG) bit */
        ljmp $__BOOT_CS,$1f     /* Clear prefetch and normalize %eip */
1:
        /* Set up the stack pointer */
        lss stack_start,%esp

```





## 参考

[][linux内存源码分析 - 页表的初始化][linux内存源码分析 - 页表的初始化](https://www.cnblogs.com/tolimit/p/4585803.html)


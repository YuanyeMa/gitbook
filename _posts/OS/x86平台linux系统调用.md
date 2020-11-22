# x86平台linux系统调用分析



## 系统调用原理简介





## x86 中断处理流程





## 中断向量表IDT的初始化

基于`linux 2.6.30.4`

```c
/* "arch/x86/kernel/traps.c" */
void __init trap_init(void)
{	
        set_intr_gate(0, &divide_error);
        set_intr_gate_ist(1, &debug, DEBUG_STACK);
        set_intr_gate_ist(2, &nmi, NMI_STACK);
        /* int3 can be called from all */
        set_system_intr_gate_ist(3, &int3, DEBUG_STACK);
        /* int4 can be called from all */
        set_system_intr_gate(4, &overflow);
        set_intr_gate(5, &bounds);
        set_intr_gate(6, &invalid_op);
        set_intr_gate(7, &device_not_available);
    
        set_task_gate(8, GDT_ENTRY_DOUBLEFAULT_TSS);
        set_intr_gate(9, &coprocessor_segment_overrun);
        set_intr_gate(10, &invalid_TSS);
        set_intr_gate(11, &segment_not_present);
        set_intr_gate_ist(12, &stack_segment, STACKFAULT_STACK);
        set_intr_gate(13, &general_protection);
        set_intr_gate(14, &page_fault);
        set_intr_gate(15, &spurious_interrupt_bug);
        set_intr_gate(16, &coprocessor_error);
        set_intr_gate(17, &alignment_check);
    
        set_system_trap_gate(SYSCALL_VECTOR, &system_call);
    /*
    SYSCALL_VECTOR
    arch/x86/include/asm/irq_vectors.h:36:# define SYSCALL_VECTOR                   0x80
    asmlinkage int system_call(void);
    */

		cpu_init();		
}
```

`set_intr_gate()` 等函数都调用了`_set_gate()`函数设置门描述符，本文重点分析系统调用相关的内容，即 `set_system_trap_gate()`函数。

```c
/*"arch/x86/include/asm/desc.h" */

/* 
	set_system_trap_gate(SYSCALL_VECTOR, &system_call); 
	参数
    		n = SYSCALL_VECTOR = 0x80;
    		addr = &system_call
*/
static inline void set_system_trap_gate(unsigned int n, void *addr)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_TRAP, addr, 0x3, 0, __KERNEL_CS);
}
```

`_set_gate()`函数完成设置门描述符的工作。

```c
/* 
	_set_gate(n, GATE_TRAP, addr, 0x3, 0, __KERNEL_CS); 
	"arch/x86/include/asm/desc_defs.h"	
        enum {
                GATE_INTERRUPT = 0xE,
                GATE_TRAP = 0xF,
                GATE_CALL = 0xC,
                GATE_TASK = 0x5,
        };
	arch/x86/include/asm/segment.h 
			#define __KERNEL_CS     (GDT_ENTRY_KERNEL_CS * 8)
			#define GDT_ENTRY_KERNEL_CS 2
*/

static inline void _set_gate(int gate, unsigned type, void *addr,
                             unsigned dpl, unsigned ist, unsigned seg)
{
    /*
    参数如下：
    	gate = n = SYSCALL_VECTOR = 0x80;
        type = GATE_TRAP = 0xF;
        addr = &system_call;
        dpl = 0x3;
        ist = 0;
        seg = __KERNEL_CS = 2*8;
    */
        gate_desc s;
        pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
        /*
         * does not need to be atomic because it is only done once at
         * setup time
         */
        write_idt_entry(idt_table, gate, &s);
}
```
![trap_gate](/images/20201123/trap_gate.png)

先看看第一行`gate_desc s;`，主要分配了门描述符的结构。

```c
/*gate_desc 结构 "arch/x86/include/asm/desc_defs.h" */
typedef struct desc_struct gate_desc;
struct desc_struct {
        union {
                struct {
                        unsigned int a; // 4字节
                        unsigned int b; // 4字节
                };
                struct {
                        u16 limit0; // 2字节 位移的低16位
                        u16 base0;  // 2字节 段选择码
                        unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1; // 2字节
                        unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8; //2字节
                };
        };
} __attribute__((packed));
```

> __attribute__ ((packed)) 的作用就是告诉编译器取消结构在编译过程中的优化对齐,按照实际占用字节数进行对齐，是GCC特有的语法。这个功能是跟操作系统没关系，跟编译器有关，gcc编译器不是紧凑模式的，在windows下，用vc的编译器也不是紧凑的，用tc的编译器就是紧凑的。
>
> 例如：
>
> 在TC下：struct my{ char ch; int a;} sizeof(int)=2;sizeof(my)=3;（紧凑模式）
> 在GCC下：struct my{ char ch; int a;} sizeof(int)=4;sizeof(my)=8;（非紧凑模式）
> 在GCC下：struct my{ char ch; int a;}__attrubte__ ((packed)) sizeof(int)=4;sizeof(my)=5
>

`struct desc_struct`包含一个联合体，可以用其中任何一种类型，后边可以看到`pack_gate()`函数使用联合体中的第一种，即`struct {unsigned int a; unsigned  int b};`。

```c
/* 
参数
	gate = n = SYSCALL_VECTOR = 0x80;
	type = GATE_TRAP = 0xF;
    addr = &system_call;
    dpl = 0x3;
    ist = 0;
    seg = __KERNEL_CS = 2*8;
*/
static inline void pack_gate(gate_desc *gate, unsigned char type,
                             unsigned long base, unsigned dpl, unsigned flags,
                             unsigned short seg)
{
        gate->a = (seg << 16) | (base & 0xffff);
        gate->b = (base & 0xffff0000) |
                  (((0x80 | type | (dpl << 5)) & 0xff) << 8);
    /*
    gate->a = (seg << 16) | (base & 0xffff);
    gate->a = ((2*8)<<16 | (&system_call & 0xffff);
    
    gate->b = (base & 0xffff0000) | (((0x80 | type | (dpl << 5)) & 0xff) << 8);
    gate->b = (&system_call & 0xffff0000) | (((0x80 | 0xF  | (0x3 << 5)) & 0xff) << 8)
    */
}
```
`pack_gate()`此函数包装一个陷阱门描述符。

```c
#define write_idt_entry(dt, entry, g)           \
        native_write_idt_entry(dt, entry, g)

/*
参数
    idt = idt_table;
    entry = gate = n = SYSCALL_VECTOR = 0x80;;
    gate = &s；
其中
	gate_desc idt_table[256] 
			__attribute__((__section__(".data.idt"))) = { { { { 0, 0 } } }, };
			
	typedef struct desc_struct gate_desc;
*/
static inline void native_write_idt_entry(gate_desc *idt, int entry,
                                          const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

`native_write_idt_entry()`此函数将`pack_gate()`函数封装的陷阱门描述符拷贝进`idt`的0x80位置。
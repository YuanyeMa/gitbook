# kzalloc & kmalloc





```c
/* include/linux/slab.h */
/**
 * kzalloc - allocate memory. The memory is set to zero.
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate (see kmalloc).
 */
static inline void *kzalloc(size_t size, gfp_t flags)
{
        return kmalloc(size, flags | __GFP_ZERO);
}

/* 调用了 kmalloc此函数在include/linux/slab_def.h中定义 */

static __always_inline void *kmalloc(size_t size, gfp_t flags)
{
        struct kmem_cache *cachep;
        void *ret;

        if (__builtin_constant_p(size)) {
            /*
            	__builtin_constant_p 是编译器gcc内置函数，用于判断一个值是否为编译时常量，  
            	如果是常数，函数返回1 ，否则返回0。
            	此内置函数的典型用法是在宏中用于手动编译时优化。
            */
                int i = 0;

                if (!size)
                        return ZERO_SIZE_PTR;

#define CACHE(x) \
                if (size <= x) \
                        goto found; \
                else \
                        i++;
#include <linux/kmalloc_sizes.h>
#undef CACHE
                return NULL;
found:
#ifdef CONFIG_ZONE_DMA
                if (flags & GFP_DMA)
                        cachep = malloc_sizes[i].cs_dmacachep;
                else
#endif
                        cachep = malloc_sizes[i].cs_cachep;

                ret = kmem_cache_alloc_notrace(cachep, flags);

                trace_kmalloc(_THIS_IP_, ret,
                              size, slab_buffer_size(cachep), flags);

                return ret;
        }
        return __kmalloc(size, flags);
}
```
## `if(__builtin_constant_p(size))` 为真时

`__builtin_constant_p(size)`如果`size`在编译时是一个常数则执行`if`中间的代码，中间插入了一个头文件`kmalloc_sizes.h`其内容如下

```c
/* include/linux/kmalloc_sizes.h */
#if (PAGE_SIZE == 4096)
        CACHE(32)
#endif
        CACHE(64)
#if L1_CACHE_BYTES < 64
           	/*
                L1 cache line size 
                #define L1_CACHE_SHIFT  (CONFIG_X86_L1_CACHE_SHIFT)
                #define L1_CACHE_BYTES  (1 << L1_CACHE_SHIFT)

            config X86_L1_CACHE_SHIFT
        			int
        			default "7" if MPENTIUM4 || MPSC
        			default "4" if X86_ELAN || M486 || M386 || MGEODEGX1
        			default "5" if MWINCHIP3D || MWINCHIPC6 || MCRUSOE || MEFFICEON || MCYRIXIII || MK6 || MPENTIUMIII || MPENTIUMII || M686 || M586MMX || M586TSC || M586 || MVIAC3_2 || MGEODE_LX
        			default "6" if MK7 || MK8 || MPENTIUMM || MCORE2 || MVIAC7 || X86_GENERIC || GENERIC_CPU
            */
        CACHE(96)
#endif
        CACHE(128)
#if L1_CACHE_BYTES < 128
        CACHE(192)
#endif
        CACHE(256)
        CACHE(512)
        CACHE(1024)
        CACHE(2048)
        CACHE(4096)
        CACHE(8192)
        CACHE(16384)
        CACHE(32768)
        CACHE(65536)
        CACHE(131072)
#if KMALLOC_MAX_SIZE >= 262144
        CACHE(262144)
#endif
#if KMALLOC_MAX_SIZE >= 524288
        CACHE(524288)
#endif
#if KMALLOC_MAX_SIZE >= 1048576
        CACHE(1048576)
#endif
#if KMALLOC_MAX_SIZE >= 2097152
        CACHE(2097152)
#endif
#if KMALLOC_MAX_SIZE >= 4194304
        CACHE(4194304)
#endif
#if KMALLOC_MAX_SIZE >= 8388608
        CACHE(8388608)
#endif
#if KMALLOC_MAX_SIZE >= 16777216
        CACHE(16777216)
#endif
#if KMALLOC_MAX_SIZE >= 33554432
        CACHE(33554432)
#endif
            
/* 其中KMALLOC_MAX_SIZE定义在include/linux/slab.h */
/*
 * The largest kmalloc size supported by the slab allocators is
 * 32 megabyte (2^25) or the maximum allocatable page order if that is
 * less than 32 MB.
 *
 * WARNING: Its not easy to increase this value since the allocators have
 * to do various tricks to work around compiler limitations in order to
 * ensure proper constant folding.
 */
#define KMALLOC_SHIFT_HIGH      ((MAX_ORDER + PAGE_SHIFT - 1) <= 25 ? \
                                (MAX_ORDER + PAGE_SHIFT - 1) : 25)

#define KMALLOC_MAX_SIZE        (1UL << KMALLOC_SHIFT_HIGH)
            
/* 其中MAX_ORDER定义在include/linux/mmzone.h */
/* Free memory management - zoned buddy allocator.  */
#ifndef CONFIG_FORCE_MAX_ZONEORDER
#define MAX_ORDER 11
#else
#define MAX_ORDER CONFIG_FORCE_MAX_ZONEORDER
#endif
            
/* PAGE_SHIFT定义在arch/x86/include/asm/page_types.h */
/* PAGE_SHIFT determines the page size */
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT) /*_AC定义在include/linux/const.h中用以定义常量， 即PAGE_SIZE = 1<<PAGE_SHIFT = 1<<12 = 4096*/
           
/*
	总结： 即KMALLOC_MAX_SIZE = 1<<22 = 4194304;
*/
```
`malloc_sizes[]`定义在`mm/slab.c`

```c
/*
 * These are the default caches for kmalloc. Custom caches can have other sizes.
 */
struct  cache_sizes malloc_sizes[] = {
#define CACHE(x) { .cs_size = (x) },
#include <linux/kmalloc_sizes.h>
        CACHE(ULONG_MAX) /* #define ULONG_MAX       (~0UL) 定义在include/linux/kernel.h */
#undef CACHE
};
EXPORT_SYMBOL(malloc_sizes);

/* struct cache_sizes类型定义在include/linux/slab_def.h */
/* Size description struct for general caches. */
struct cache_sizes {
        size_t                  cs_size;
        struct kmem_cache       *cs_cachep;
#ifdef CONFIG_ZONE_DMA
        struct kmem_cache       *cs_dmacachep;
#endif
};
extern struct cache_sizes malloc_sizes[];

/* 展开写 */
struct  cache_sizes malloc_sizes[] = {
    { .cs_size = (32) },
    { .cs_size = (64) },
    { .cs_size = (128) },
    { .cs_size = (192) },
    { .cs_size = (256) },
    { .cs_size = (512) },
    { .cs_size = (1024) },
    { .cs_size = (2048) },
    { .cs_size = (4096) },
    { .cs_size = (8192) },
    { .cs_size = (16384) },
    { .cs_size = (32768) },
    { .cs_size = (65536) },
    { .cs_size = (131072) },
    { .cs_size = (262144) },
    { .cs_size = (524288) },
    { .cs_size = (1048576) },
    { .cs_size = (2097152) },
    { .cs_size = (4194304) },
    { .cs_size = (0xFFFFFFFF) },
};
EXPORT_SYMBOL(malloc_sizes);

```

总结 ： 根据`size`的大小在`malloc_sizes`数组中找到对应的`struct kmem_cache`结构，之后调用`kmem_cache_alloc_notrace()`函数分配内存，此函数在`include/linux/slab_def.h`中定义。

```c
static __always_inline void *
kmem_cache_alloc_notrace(struct kmem_cache *cachep, gfp_t flags)
{
        return kmem_cache_alloc(cachep, flags);
}
/** mm/slab.c
 * kmem_cache_alloc - Allocate an object
 * @cachep: The cache to allocate from.
 * @flags: See kmalloc().
 *
 * Allocate an object from this cache.  The flags are only relevant
 * if the cache has no available objects.
 */
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
        void *ret = __cache_alloc(cachep, flags, __builtin_return_address(0));

        trace_kmem_cache_alloc(_RET_IP_, ret,
                               obj_size(cachep), cachep->buffer_size, flags);

        return ret;
}
EXPORT_SYMBOL(kmem_cache_alloc);

```

到此为止，可以看出`kmalloc()`函数分配内存的时候，并不是严格按照给定的`size`参数进行分配的，而是找到大小最接近的高速缓存块（只能多分不能少分），然后在相应的高速缓存中进行分配。实际分配内存的操作是函数`__cache_alloc()`完成的。

## `if(__builtin_constant_p(size))` 为假时

如果`__builtin_constant_p(size)`为0，则执行`__kmalloc(size, flags)`；

```c
/* 
	调用了__kmalloc()函数 此函数有两个定义都在mm/slab.c中 
	第一个用于slab的调试或者kmemtrace暂时不看 
*/
void *__kmalloc(size_t size, gfp_t flags)
{
        return __do_kmalloc(size, flags, NULL);
}
EXPORT_SYMBOL(__kmalloc);

/* 调用同一个文件中的__do_kmalloc() 函数 */
/**
 * __do_kmalloc - allocate memory
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate (see kmalloc).
 * @caller: function caller for debug tracking of the caller
 */
static __always_inline void *__do_kmalloc(size_t size, gfp_t flags,
                                          void *caller)
{
        struct kmem_cache *cachep;
        void *ret;

        /* If you want to save a few bytes .text space: replace
         * __ with kmem_.
         * Then kmalloc uses the uninlined functions instead of the inline
         * functions.
         */
        cachep = __find_general_cachep(size, flags);
        if (unlikely(ZERO_OR_NULL_PTR(cachep)))
                return cachep;
        ret = __cache_alloc(cachep, flags, caller);

        trace_kmalloc((unsigned long) caller, ret,
                      size, cachep->buffer_size, flags);

        return ret;
}
```

`__do_kmalloc()`函数，先调用`__find_general_cachep()`找到合适的高速缓存，然后和前边函数`kmem_cache_alloc()`一样, 都调用到了`__cache_alloc()`函数进行内存分配。

```c
static __always_inline void *
__cache_alloc(struct kmem_cache *cachep, gfp_t flags, void *caller)	
{
    /* 简略写如下 */
    objp = __do_cache_alloc(cachep, flags);
  
	if (unlikely((flags & __GFP_ZERO) && objp)) 
                memset(objp, 0, obj_size(cachep));
}
/*
	如果设置了__GFP_ZERO标志，则清零所分配的内存，对应kzalloc调用kmalloc时设置的__GFP_ZERO标志。
       *kzalloc(size_t size, gfp_t flags)
		{
        	return kmalloc(size, flags | __GFP_ZERO);
		}
*/
```

`__find_general_cachep()`函数实现如下

```c
static inline struct kmem_cache *__find_general_cachep(size_t size,
                                                        gfp_t gfpflags)
{
        struct cache_sizes *csizep = malloc_sizes;

        if (!size)
                return ZERO_SIZE_PTR;

        while (size > csizep->cs_size)
                csizep++;
        /*
         * Really subtle: The last entry with cs->cs_size==ULONG_MAX
         * has cs_{dma,}cachep==NULL. Thus no special case
         * for large kmalloc calls required.
         */
#ifdef CONFIG_ZONE_DMA
        if (unlikely(gfpflags & GFP_DMA))
                return csizep->cs_dmacachep;
#endif
        return csizep->cs_cachep;
}
```



## kree

释放由`kmalloc`或者`kzalloc`分配的内存。

```c
/* mm/slab.c */
/**
 * kfree - free previously allocated memory
 * @objp: pointer returned by kmalloc.
 *
 * If @objp is NULL, no operation is performed.
 *
 * Don't free memory not originally allocated by kmalloc()
 * or you will run into trouble.
 */
void kfree(const void *objp)
{
        struct kmem_cache *c;
        unsigned long flags;

        trace_kfree(_RET_IP_, objp);

        if (unlikely(ZERO_OR_NULL_PTR(objp)))
                return;
        local_irq_save(flags);
        kfree_debugcheck(objp);
        c = virt_to_cache(objp);
        debug_check_no_locks_freed(objp, obj_size(c));
        debug_check_no_obj_freed(objp, obj_size(c));
        __cache_free(c, (void *)objp);
        local_irq_restore(flags);
}
EXPORT_SYMBOL(kfree);
```

此函数首先调用`virt_to_cache(objp)`得到对应的高速缓存，然后调用`__cache_free(c, (void *)objp)`释放相应内存。尝试看看是怎么找到 `objp`所在的高速缓存的。

```c
/*mm/slab.c*/
static inline struct kmem_cache *virt_to_cache(const void *obj)
{
        struct page *page = virt_to_head_page(obj); /* 得到obj所在的page */
        return page_get_cache(page); /* 通过page得到高速缓存的指针 */
}

/* mm/slab.h */
static inline struct page *virt_to_head_page(const void *x)
{
        struct page *page = virt_to_page(x);
        return compound_head(page);
}
```

先看看`virt_to_page`的实现

```c
/* arch/x86/include/asm/page.h */
/*
 * virt_to_page(kaddr) returns a valid pointer if and only if
 * virt_addr_valid(kaddr) returns true.
 */
#define virt_to_page(kaddr)     pfn_to_page(__pa(kaddr) >> PAGE_SHIFT)

/*其中*/
/* PAGE_SHIFT determines the page size */
#define PAGE_SHIFT      12

#define __pa(x)         __phys_addr((unsigned long)(x))
#define __phys_addr(x)          __phys_addr_nodebug(x)
#define __phys_addr_nodebug(x)  ((x) - PAGE_OFFSET)
/*
	即 virt_to_page(kaddr)
	1. 将kaddr经__pa(kaddr)转换为物理地址，此过程直接减去PAGE_OFFSET也即0xC0000000
	2. 将kaddr的物理地址右移12位，即除以4k，得到页号。
*/
```

再看`compound_head`做了什么操作

```c
static inline struct page *compound_head(struct page *page)
{
        if (unlikely(PageTail(page)))
                return page->first_page;
        return page;
}
/*
 * PG_reclaim is used in combination with PG_compound to mark the
 * head and tail of a compound page. This saves one page flag
 * but makes it impossible to use compound pages for the page cache.
 * The PG_reclaim bit would have to be used for reclaim or readahead
 * if compound pages enter the page cache.
 *
 * PG_compound & PG_reclaim     => Tail page
 * PG_compound & ~PG_reclaim    => Head page
 */
#define PG_head_tail_mask ((1L << PG_compound) | (1L << PG_reclaim))

static inline int PageTail(struct page *page)
{
        return ((page->flags & PG_head_tail_mask) == PG_head_tail_mask);
}
/*
	PG_compound 和 PG_reclaim定义在include/linux/page-flags.h中
	函数compound_head()用户找到第一个page
*/
```

找到要释放的地址所在的页后，就是找到页所在的高速缓存, 主要通过`page_get_cache(page)`函数调用实现。

```c
static inline struct kmem_cache *page_get_cache(struct page *page)
{
        page = compound_head(page);
        BUG_ON(!PageSlab(page));
        return (struct kmem_cache *)page->lru.next;
}
/*
lru ：链表头，主要有3个用途：
    a：page处于伙伴系统中时，用于链接相同阶的伙伴（只使用伙伴中的第一个page的lru即可达到目的）。
    b：page属于slab时，page->lru.next指向page驻留的的缓存的管理结构，page->lru.prec指向保存该page的slab的管理结构。
    c：page被用户态使用或被当做页缓存使用时，用于将该page连入zone中相应的lru链表，供内存回收时使用
*/
```

lru链表作用参考[Linux中页的一些知识](https://blog.csdn.net/weixin_48101150/article/details/108940623)和[Linux 内存管理窥探（5）：page 数据结构](https://blog.csdn.net/zhoutaopower/article/details/87090982)



最后`__cache_free(c, (void *)objp)`是实际的释放函数，`kmem_cache_free()`也是调用此函数进行释放操作。

```c
/**
 * kmem_cache_free - Deallocate an object
 * @cachep: The cache the allocation was from.
 * @objp: The previously allocated object.
 *
 * Free an object which was previously allocated from this
 * cache.
 */
void kmem_cache_free(struct kmem_cache *cachep, void *objp)
{
        unsigned long flags;

        local_irq_save(flags);
        debug_check_no_locks_freed(objp, obj_size(cachep));
        if (!(cachep->flags & SLAB_DEBUG_OBJECTS))
                debug_check_no_obj_freed(objp, obj_size(cachep));
        __cache_free(cachep, objp);
        local_irq_restore(flags);

        trace_kmem_cache_free(_RET_IP_, objp);
}
EXPORT_SYMBOL(kmem_cache_free);
```



## 关于__builtin_constant_p的一个测试



```c
#include <stdio.h>

int main()
{
        if (__builtin_constant_p(sizeof(int)))
                printf("yes\n");
        else
                printf("no\n");
        return 0;
}
/*
	以上代码用gcc编译后输出为"yes"，即sizeof(int)在由gcc进行编译时自动转换为了一个常量。
	那__builtin_constant_p(size)什么时候为假呢？ 看下边的程序。
	a是一个变量，虽然已经赋值为4，但是代码在运行时a的值可能会变化，编译时a的值无法确定下来。
	因此__builtin_constant_p(a)就返回0，所以代码输出"no"。
*/
#include <stdio.h>

int a;

int main()
{
        a = 4;
        if (__builtin_constant_p(a))
                printf("yes\n");
        else
                printf("no\n");
        return 0;
}

```


# linux kernel slab系统

基于linux kernel v2.6.30.4

## 基本原理
​		以字节为单位的内存管理
​			SLAB 为了提高内核中一些十分频繁进行分配释放的“对象”的分配效率， SLAB 的做法是：每次释放掉某个对象之后，不是立即将其返回给伙伴系统，而是存放在一个叫 array_cache 的结构中，下次要分配的时候就可以直接从这里分配，从而加快了速度。

slab系统的内存区域主要分为专用和通用，专用指的是用于存放数据结构的，通用主要指`kmalloc`用于通用目的而分配的内存。

![image-20201227201452112](/images/mm/image-20201227201452112.png)

## 概念以及数据结构

### `struct kmem_cache`
  “高速缓存” 其实是一个管理slab的结构

```c
/*
 * DEBUG        - 1 for kmem_cache_create() to honour; SLAB_RED_ZONE & SLAB_POISON.
 *                0 for faster, smaller code (especially in the critical paths).
 *
 * STATS        - 1 to collect stats for /proc/slabinfo.
 *                0 for faster, smaller code (especially in the critical paths).
 *
 * FORCED_DEBUG - 1 enables SLAB_RED_ZONE and SLAB_POISON (if possible)
 */

/*
 * struct kmem_cache
 *
 * manages a cache.
 */

struct kmem_cache {
/* 1) per-cpu data, touched during every alloc/free */
        struct array_cache *array[NR_CPUS];
/* 2) Cache tunables. Protected by cache_chain_mutex */
        unsigned int batchcount;
        unsigned int limit;
        unsigned int shared;

        unsigned int buffer_size;
        u32 reciprocal_buffer_size;
/* 3) touched by every alloc & free from the backend */

        unsigned int flags;             /* constant flags */
        unsigned int num;               /* # of objs per slab */

/* 4) cache_grow/shrink */
        /* order of pgs per slab (2^n) */
        unsigned int gfporder;

        /* force GFP flags, e.g. GFP_DMA */
        gfp_t gfpflags;

        size_t colour;                  /* cache colouring range */
        unsigned int colour_off;        /* colour offset */
        struct kmem_cache *slabp_cache;
        unsigned int slab_size;
        unsigned int dflags;            /* dynamic flags */

        /* constructor func */
        void (*ctor)(void *obj);

/* 5) cache creation/removal */
        const char *name;
        struct list_head next;

/* 6) statistics */
#if STATS
        unsigned long num_active;
        unsigned long num_allocations;
        unsigned long high_mark;
        unsigned long grown;
        unsigned long reaped;
        unsigned long errors;
        unsigned long max_freeable;
        unsigned long node_allocs;
        unsigned long node_frees;
        unsigned long node_overflow;
        atomic_t allochit;
        atomic_t allocmiss;
        atomic_t freehit;
        atomic_t freemiss;
#endif
#if DEBUG
        /*
         * If debugging is enabled, then the allocator can add additional
         * fields and/or padding to every object. buffer_size contains the total
         * object size including these internal fields, the following two
         * variables contain the offset to the user object and its size.
         */
        int obj_offset;
        int obj_size;
#endif
        /*
         * We put nodelists[] at the end of kmem_cache, because we want to size
         * this array to nr_node_ids slots instead of MAX_NUMNODES
         * (see kmem_cache_init())
         * We still use [MAX_NUMNODES] and not [1] or [0] because cache_cache
         * is statically defined, so we reserve the max number of nodes.
         */
        struct kmem_list3 *nodelists[MAX_NUMNODES];
        /*
         * Do not add fields after nodelists[]
         */
};
```



### 本地CPU空闲对象链表
  `struct kmem_cache`中定义了一个 `struct array_cache`结构体指针数组，即成员`struct array_cache *array[NR_CPUS];`，数组的元素个数对应了系统的CPU数。和伙伴系统的每CPU页框高速缓存类似，该结构用来描述每个CPU的本地高速缓存，在每个`array_cache`的末端，都用一个指针数组记录了slab中的空闲对象。分配对象的时候采用LIFO方式，也就是将该数组中的最后一个索引对应的对象分配出去。实际上，每次分配内存都是直接与本地CPU高速缓存进行交互，只有当其空闲内存不足时，才会从`kmem_list3`中的slab中取一些加入到本地高速缓存中。


>本地CPU空闲对象链表，存在的意义：
>
>每个CPU都有它自己的硬件缓存（L1 L2 Cache），当此CPU上释放对象时，这个对象很可能还在CPU Cache中，所以内核为每个CPU维护这样一个链表，当需要新的对象时，会优先尝试从当前CPU的“pre-cpu”缓存中空闲对象列表获取相应大小的对象。
>
>减少所的竞争，假设多个CPU同时申请一个大小的slab，这时如果没有"pre-cpu"缓存空闲对象链表，就会导致分配的流程互斥，需要上锁，这就导致分配效率低下。而如果分配对象时从各自的CPU缓存中分配就不会出现竞争问题。

```c
/*
 * struct array_cache
 *
 * Purpose:
 * - LIFO ordering, to hand out cache-warm objects from _alloc
 * - reduce the number of linked list operations
 * - reduce spinlock operations
 *
 * The limit is stored in the per-cpu structure to reduce the data cache
 * footprint.
 *
 */
struct array_cache {
        unsigned int avail; /* 可用对象数目 */
        unsigned int limit; /* 可拥有的最大对象数目，和kmem_cache中一样 */
        unsigned int batchcount; /* 同kmem_cache，要转移进本地高速缓存或从本地高速缓存中转移出去的对象的个数 */
        unsigned int touched; /* 是否在收缩后被访问过 */
        spinlock_t lock;
        void *entry[];  /*
                         * Must have this definition in here for the proper
                         * alignment of array_cache. Also simplifies accessing
                         * the entries.
                         */
    /* entry是一个伪数组，初始没有任何数据项，之后会增加并保存释放的对象指针 */
};
```

这个本地CPU空闲对象链表在初始化完成后是一个空的链表，只有释放对象时才会将对象加入这个链表。链表中对象的个数受`limit`字段限制，其最大值就是`limit`。链表中对象个数超过这个值时，会将`batchcount`个对象返回到所有CPU共享的空闲对象链表（也是这样一个结构）中。

`array_cache`中的`entry`数组保存的是指向空闲对象的首地址的指针，此链表是在`kmem_cache`结构中的，**即每一种类型的`kmem_cache`都有它自己的本地CPU空闲对象链表**。

### 所有CPU共享的空闲对象链表

原理和本地CPU空闲对象列表是一样的，唯一的区别就是所有CPU都可以从这个链表中获取对象。所有CPU共享的空闲对象链表不是一直都有，而是根据`kmem_cache`的`shared`字段确定，如果`shared`为1，则此`kmem_cache`中存在此链表，否则不存在。

一个常规的对象申请流程：系统首先会从本地CPU空闲对象链表中尝试获取一个对象用于分配；如果失败，则尝试从所有CPU共享的空闲对象列表中尝试获取；如果还是失败，就会从SLAB中分配一个；这时如果还失败`kmem_cache`会尝试从页框分配器中获取一组连续页框，建立一个新的slab，然后将从新的slab中获取一个对象。

对象释放流程也类似：首先将对象释放到本地CPU空闲对象链表，如果本地CPU空闲对象链表中的对象过多（大于limit），`kmem_cache`会将本地CPU空闲对象链表中的`batchcount`个对象移动到所有CPU共享的空闲对象链表中，如果所有CPU共享的空闲对象链表中的对象数量也超过了limit，`kmem_cache`会把所有CPU共享的空闲对象链表中的`batchcount`个对象移回他们自己所属的SLAB中，这是如果SLAB中空闲对象也太多了，`kmem_cache`会整理出一些空闲的slab，将这些slab占用的页框释放回页框分配器中。

### `struct kmem_list3`

每个内存结点都有自己的`kmem_list3`，每个CPU都有`array_cache`。

```c
/*
 * The slab lists for all objects.
 */
struct kmem_list3 {
        struct list_head slabs_partial; /* partial list first, better asm code */
        struct list_head slabs_full;
        struct list_head slabs_free;
        unsigned long free_objects;
        unsigned int free_limit;
        unsigned int colour_next;       /* Per-node cache coloring */
        spinlock_t list_lock;
        struct array_cache *shared;     /* shared per node */
        struct array_cache **alien;     /* on other nodes */
        unsigned long next_reap;        /* updated without locking */
        int free_touched;               /* updated without locking */
};
```

### `sturct slab`

  负责管理从buddy系统分配的一个或多个物理页框，slab主要包含两大部分，管理性数据和obj对象，其中管理性数据包括`struct slab`和`kmem_bufctl_t`。

slab有两种形式的结构：管理数据外挂或者内嵌。如果obj较小，那么`struct slab`和`kmem_bufctl_t`可以和obj分配在同一个物理页中，称为内嵌。

```c
/*
 * kmem_bufctl_t:
 *
 * Bufctl's are used for linking objs within a slab
 * linked offsets.
 *
 * This implementation relies on "struct page" for locating the cache &
 * slab an object belongs to.
 * This allows the bufctl structure to be small (one int), but limits
 * the number of objects a slab (not a cache) can contain when off-slab
 * bufctls are used. The limit is the size of the largest general cache
 * that does not use off-slab slabs.
 * For 32bit archs with 4 kB pages, is this 56.
 * This is not serious, as it is only for large objects, when it is unwise
 * to have too many per slab.
 * Note: This limit can be raised by introducing a general cache whose size
 * is less than 512 (PAGE_SIZE<<3), but greater than 256.
 */

typedef unsigned int kmem_bufctl_t;
#define BUFCTL_END      (((kmem_bufctl_t)(~0U))-0)
#define BUFCTL_FREE     (((kmem_bufctl_t)(~0U))-1)
#define BUFCTL_ACTIVE   (((kmem_bufctl_t)(~0U))-2)
#define SLAB_LIMIT      (((kmem_bufctl_t)(~0U))-3)

/*
 * struct slab
 *
 * Manages the objs in a slab. Placed either at the beginning of mem allocated
 * for a slab, or allocated from an general cache.
 * Slabs are chained into three list: fully used, partial, fully free slabs.
 */
struct slab {
        struct list_head list;
        unsigned long colouroff;
        void *s_mem;            /* including colour offset 指向第一个对象的地址 */ 
        unsigned int inuse;     /* num of objs active in slab */
        kmem_bufctl_t free;
        unsigned short nodeid;
};
```

![kmem_bufctl_t](https://img-blog.csdn.net/2018041716304834?watermark/2/text/aHR0cHM6Ly9ibG9nLmN)

### object 

  每个slab只负责管理一种类型的“对象”， 将该大小进行内存对齐后的大小就是该“对象”的大小，依此大小对slab进行划分，从而实现细粒度的内存管理

### 结构之间的关系

![Memery_Layout_14](http://chenweixiang.github.io/assets/Memery_Layout_14.jpg)

`cache_chain`的定义：`static struct list_head cache_chain;`是一个`struct kmem_cache`的链表，将所有`struct kemem_cache`连接起来。

`cache_cache`是第一个`kemem_cache_struct`结构，通过`struct kmem_list3`结构维护的三个`struct slab`链表构成了内核的第一个`slab`系统，用于存储`slab`系统所有的`struct kmem_cache_struct`结构。

`struct slab`结构可以通过参数选择1.存储在自己`slab`管理的页面上，2.存储到`cache_cache`中?

`struct kmem_list3`结构存在哪里？



## 参考资料
[Linux内存管理之slab分配器分析(一)](http://blog.chinaunix.net/uid-31562863-id-5793195.html)

[Linux内存管理之SLAB原理浅析](https://blog.csdn.net/rockrockwu/article/details/79976833)

[linux内存源码分析 - SLAB分配器概述](https://www.cnblogs.com/tolimit/p/4566189.html)

## kmem_cache_init

此函数在`init/main.c`中被`start_kernel()`函数调用，其定义在`mm/slab.c`中定义。先看看源码中的注释： 引导程序很棘手，因为需要从尚不存在的caches中分配几个objects。

```c
/*
 * Initialisation.  Called after the page allocator have been initialised and
 * before smp_init().
 */
void __init kmem_cache_init(void)
{

        /* Bootstrap is tricky, because several objects are allocated
         * from caches that do not exist yet:
         * 1) initialize the cache_cache cache: it contains the struct
         *    kmem_cache structures of all caches, except cache_cache itself:
         *    cache_cache is statically allocated.
         *    Initially an __init data area is used for the head array and the
         *    kmem_list3 structures, it's replaced with a kmalloc allocated
         *    array at the end of the bootstrap.
         * 2) Create the first kmalloc cache.
         *    The struct kmem_cache for the new cache is allocated normally.
         *    An __init data area is used for the head array.
         * 3) Create the remaining kmalloc caches, with minimally sized
         *    head arrays.
         * 4) Replace the __init data head arrays for cache_cache and the first
         *    kmalloc cache with kmalloc allocated arrays.
         * 5) Replace the __init data for kmem_list3 for cache_cache and
         *    the other cache's with kmalloc allocated memory.
         * 6) Resize the head arrays of the kmalloc caches to their final sizes.
         */
}
```

从注释可以看出，`kmem_cache_init()`函数主要完成六项工作：

1. 初始化`cache_cache`高速缓存：静态分配`cache_cache`结构，它用于保存所有高速缓存的`struct kmem_cache`描述结构体。初始化一个`__init`数据区域，用于保存`head array`和`kmem_list3`结构体，在引导程序末尾将其替换为`kmalloc`分配的数组。

2. 创建第一个`kmalloc`用的`cache`:

3. 创建剩余的`kmalloc`缓存，并使用最小大小的头数组。

4. 将`cache_cache`的`__init`数据头数组和第一个`kmalloc`缓存替换为`kmalloc`分配的数组。

5. 将`kmem_list3`的`__init`数据替换为`cache_cache`，将其他缓存替换为`kmalloc`分配的内存。

6. 将`kmalloc`缓存的头数组调整为最终大小。


```
(1)构建好了kmem_cache实例cache_cache(静态分配)，且构建好了kmem_cache的slab分配器,并由initkmem_list3[0]组织, 相应的array为initarray_cache；
(2)构建好了kmem_cache实例（管理arraycache_init），且构建好了arraycache_init的slab分配器,并由initkmem_list3[1]组织,相应的array为initarray_generic；
(3)构建好了kmem_cache实例（管理kmem_list3）,此时还未构建好kmem_list3的slab分配器，但是一旦申请sizeof(kmem_list3)空间，将构建kmem_list3分配器,并由initkmem_list[2]组织,其array将通过kmalloc进行申请；
(4)为malloc_sizes的相应数组元素构建kmem_cache实例，并分配kmem_list3,用于组织slab链表，分配arraycache_init用于组织每CPU的同一个kmem_cache下的slab分配;
(5)替换kmem_cache、malloc_sizes[INDEX_AC].cs_cachep下的arraycache_init实例；
(6)替换kmem_cache、malloc_sizes[INDEX_AC].cs_cachep、malloc_sizes[INDEX_L3].cs_cachep下的kmem_list3实例;
(7)g_cpucachep_up = EARLY;
```

在slab初始化好之前，无法通过kmalloc分配初始化过程中必要的一些对象，只能使用静态的全局变量，待slab初始化后期，再使用kmalloc动态分配的对象替换全局变量 

### 初始化前的准备工作

静态全局变量

```c
/*
 * bootstrap: The caches do not work without cpuarrays anymore, but the
 * cpuarrays are allocated from the generic caches...
 */
#define BOOT_CPUCACHE_ENTRIES   1
struct arraycache_init {
        struct array_cache cache;
        void *entries[BOOT_CPUCACHE_ENTRIES];
};

static struct arraycache_init initarray_cache __initdata =
    { {0, BOOT_CPUCACHE_ENTRIES, 1, 0} };
static struct arraycache_init initarray_generic =
    { {0, BOOT_CPUCACHE_ENTRIES, 1, 0} };

即
struct arraycache_init initarray_generic = initarray_cache.cache = 
struct array_cache {
        unsigned int avail = 0; /* 可用对象数目 */
        unsigned int limit = BOOT_CPUCACHE_ENTRIES; /* 可拥有的最大对象数目，和kmem_cache中一样 */
        unsigned int batchcount = 1; /* 同kmem_cache，要转移进本地高速缓存或从本地高速缓存中转移出去的对象的个数 */
        unsigned int touched = 0; /* 是否在收缩后被访问过 */
};
```



slab系统三个最重要的结构体，`kmem_cache`     `arary_cache`   以及 `kmem_list3`，初始化slab系统，其实就是初始化这三个结构体， 在slab系统初始化的初期，slab系统还没建立起来，只能通过静态全局变量初始化这三个结构体，用以搭建一个初始化阶段用的临时slab系统，等slab系统建立起来后，再将这三个结构体加入到slab系统管理的内存中。

```c
void __init kmem_cache_init(void)
{
    /* 
        如前所述，先借用全局变量initkmem_list3表示的slab三链 ，每个内存节点对应一组slab三链。
        initkmem_list3是个slab三链数组，包含所有内存节点的slab三链。
        对于每个内存节点，包含三个三链： 
            struct kmem_cache的slab三链、
            struct arraycache_init的slab 三链、
            struct kmem_list3的slab三链 
        这里循环初始化所有内存节点的所有slab三链 
    */
        for (i = 0; i < NUM_INIT_LISTS; i++) { /* 每个内存结点都有kmem_list3 */
              	/*初始化所有node的所有slab中的三个链表*/  
                kmem_list3_init(&initkmem_list3[i]); /* 1 2 */
                /* 这里初始化所有内存节点的struct kmem_cache的slab三链为空。*/  
                if (i < MAX_NUMNODES)
                        cache_cache.nodelists[i] = NULL;
        }
        set_up_list3s(&cache_cache, CACHE_CACHE); /* 3 4 */
          /*
         * Fragmentation resistance on low memory - only use bigger
         * page orders on machines with more than 32MB of memory.
         */
        if (num_physpages > (32 << 20) >> PAGE_SHIFT)
                slab_break_gfp_order = BREAK_GFP_ORDER_HI;


/* 1.关于全局变量initkmem_list3的定义，以及宏NUM_INIT_LISTS的定义 */
/*
 * Need this for bootstrapping a per node allocator.
 */
#define NUM_INIT_LISTS (3 * MAX_NUMNODES)
struct kmem_list3 __initdata initkmem_list3[NUM_INIT_LISTS];   

/* include/linux/numa.h */    
#ifdef CONFIG_NODES_SHIFT 
#define NODES_SHIFT     CONFIG_NODES_SHIFT
#else
#define NODES_SHIFT     0
#endif

#define MAX_NUMNODES    (1 << NODES_SHIFT)
/*
	总结 ： 一个UMA系统中只有一个Node，而在NUMA中则可以存在多个Node。它由CONFIG_NODES_SHIFT配置选项决定，它是CONFIG_NUMA的子选项，所以只有配置了CONFIG_NUMA，该选项才起作用。
	UMA情况下，NODES_SHIFT被定义为0，MAX_NUMNODES也即为1。
*/
            
/* 2. kmem_list3_init()所做的初始化 */
static void kmem_list3_init(struct kmem_list3 *parent)
{
        INIT_LIST_HEAD(&parent->slabs_full);
        INIT_LIST_HEAD(&parent->slabs_partial);
        INIT_LIST_HEAD(&parent->slabs_free);
        parent->shared = NULL;
        parent->alien = NULL;
        parent->colour_next = 0;
        spin_lock_init(&parent->list_lock);
        parent->free_objects = 0;
        parent->free_touched = 0;
}

/* 3. 全局变量cache_cache的初始化 */
/* internal cache of cache description objs */
static struct kmem_cache cache_cache = {
        .batchcount = 1,
        .limit = BOOT_CPUCACHE_ENTRIES,
        .shared = 1, /* 所有CPU共享的空闲对象链表 */
        .buffer_size = sizeof(struct kmem_cache), /* kmem_cache中对象的大小 */
        .name = "kmem_cache", /* kmem_cache的名字 */
};
/* 4. set_up_list3s(&cache_cache, CACHE_CACHE); */
#define CACHE_CACHE 0
/*
 * For setting up all the kmem_list3s for cache whose buffer_size is same as
 * size of kmem_list3.
 */
static void __init set_up_list3s(struct kmem_cache *cachep, int index)
{
        int node;

        for_each_online_node(node) {
                cachep->nodelists[node] = &initkmem_list3[index + node];
                cachep->nodelists[node]->next_reap = jiffies +
                    REAPTIMEOUT_LIST3 +
                    ((unsigned long)cachep) % REAPTIMEOUT_LIST3;
        }
}
/*
	设置struct kmem_cache的slab三链指向initkmem_list3中的一组slab三链， 
    CACHE_CACHE为cache在内核cache链表中的索引， 
    struct kmem_cache对应的cache是内核中创建的第一个cache 
    ，故CACHE_CACHE为0  
*/

/* 5.     
	全局变量slab_break_gfp_order为每个slab最多占用几个页面 ，用来抑制碎片。
	如大小为3360的对象，如果其slab只占一个页面，碎片为736，slab占用两个页面，则碎片大小也翻倍。
     只有当对象很大以至于slab中连一个对象都放不下时才可以超过这个值。
     有两个可能的取值 ：
     	当可用内存大于32MB时，BREAK_GFP_ORDER_HI为1 ，即每个slab最多占用2个页面 ，只有当对象大小大于8192时 ，才可以突破slab_break_gfp_order的限制。
     	小于等于32MB时BREAK_GFP_ORDER_LO为0。
*/
```

### 1） 初始化`cache_chain`

```c
node = numa_node_id();
/* 1) create the cache_cache */
INIT_LIST_HEAD(&cache_chain);
list_add(&cache_cache.next, &cache_chain);
cache_cache.colour_off = cache_line_size();
cache_cache.array[smp_processor_id()] = &initarray_cache.cache;
cache_cache.nodelists[node] = &initkmem_list3[CACHE_CACHE + node];

/*
 * struct kmem_cache size depends on nr_node_ids, which
 * can be less than MAX_NUMNODES.
 */
cache_cache.buffer_size = offsetof(struct kmem_cache, nodelists) + nr_node_ids * sizeof(struct kmem_list3 *);
/*offsetof会生成一个类型为 size_t 的整型常量，它是一个结构成员相对于结构开头的字节偏移量*/

cache_cache.buffer_size = ALIGN(cache_cache.buffer_size, cache_line_size());
cache_cache.reciprocal_buffer_size = reciprocal_value(cache_cache.buffer_size);

for (order = 0; order < MAX_ORDER; order++) {
    /*计算每个slab中对象的数目。*/  
	cache_estimate(order, cache_cache.buffer_size,
                   cache_line_size(), 0, &left_over, &cache_cache.num);
    if (cache_cache.num)
       break;
}
BUG_ON(!cache_cache.num);
cache_cache.gfporder = order;
cache_cache.colour = left_over / cache_cache.colour_off;
cache_cache.slab_size = ALIGN(cache_cache.num * sizeof(kmem_bufctl_t) +
                              sizeof(struct slab), cache_line_size());

```



### 2）&3） 创建`kmalloc`用的高速缓存



```c
/* 2+3) create the kmalloc caches */
sizes = malloc_sizes; /* 1 */
names = cache_names;

/* begin of 2  */
/*
 * Initialize the caches that provide memory for the array cache and the
 * kmem_list3 structures first.  Without this, further allocations will
 * bug.
 */
sizes[INDEX_AC].cs_cachep = kmem_cache_create(names[INDEX_AC].name,
                                              sizes[INDEX_AC].cs_size,
                                              ARCH_KMALLOC_MINALIGN,
                                              ARCH_KMALLOC_FLAGS|SLAB_PANIC,
                                              NULL);

if (INDEX_AC != INDEX_L3) {
    sizes[INDEX_L3].cs_cachep =
        kmem_cache_create(names[INDEX_L3].name,
                          sizes[INDEX_L3].cs_size,
                          ARCH_KMALLOC_MINALIGN,
                          ARCH_KMALLOC_FLAGS|SLAB_PANIC,
                          NULL);
}
/* end of 2  */
slab_early_init = 0;

/* begin of 3*/
while (sizes->cs_size != ULONG_MAX) { /* ULONG_MAX = (~0UL)即最大值 */
                /*
                 * For performance, all the general caches are L1 aligned.
                 * This should be particularly beneficial on SMP boxes, as it
                 * eliminates "false sharing".
                 * Note for systems short on memory removing the alignment will
                 * allow tighter packing of the smaller caches.
                 */
    if (!sizes->cs_cachep) {
        sizes->cs_cachep = kmem_cache_create(names->name,
                                             sizes->cs_size,
                                             ARCH_KMALLOC_MINALIGN,
                                             ARCH_KMALLOC_FLAGS|SLAB_PANIC,
                                             NULL);
#ifdef CONFIG_ZONE_DMA
        sizes->cs_dmacachep = kmem_cache_create(names->name_dma,
                                            sizes->cs_size,
                                            ARCH_KMALLOC_MINALIGN,
                                            ARCH_KMALLOC_FLAGS|SLAB_CACHE_DMA|
                                            SLAB_PANIC,
                                            NULL);
#endif
        sizes++;
        names++;
    }
/* end of 3 */
```

分步骤看看

```c
/* 1.  */
struct cache_sizes *sizes;
struct cache_names *names;

sizes = malloc_sizes; 
names = cache_names;

/* 先看看这两个变量的类型 */
/* Size description struct for general caches. */
struct cache_sizes {
        size_t                  cs_size;
        struct kmem_cache       *cs_cachep;
#ifdef CONFIG_ZONE_DMA
        struct kmem_cache       *cs_dmacachep;
#endif
};
/* Must match cache_sizes above. Out of line to keep cache footprint low. */
struct cache_names {
        char *name;
        char *name_dma;
};

/* 然后看看全局变量malloc_sizes和cache_names的定义 */
/*
 * These are the default caches for kmalloc. Custom caches can have other sizes.
 */
#define ULONG_MAX       (~0UL)
struct cache_sizes malloc_sizes[] = {
#define CACHE(x) { .cs_size = (x) },
#include <linux/kmalloc_sizes.h>
        CACHE(ULONG_MAX)
#undef CACHE
};
EXPORT_SYMBOL(malloc_sizes);

static struct cache_names __initdata cache_names[] = {
#define CACHE(x) { .name = "size-" #x, .name_dma = "size-" #x "(DMA)" },
#include <linux/kmalloc_sizes.h>
        {NULL,}
#undef CACHE
};

/* 其中include/linux/kmalloc_sizes.h的内容如下 */
#if (PAGE_SIZE == 4096)
        CACHE(32)
#endif
        CACHE(64)
#if L1_CACHE_BYTES < 64
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
/*
展开后的定义如下
struct cache_sizes malloc_sizes[] = {
 { .cs_size = (32) },
 { .cs_size = (64) },
 { .cs_size = (128) },
 { .cs_size = (256) },
 { .cs_size = (512) },
	...
 { .cs_size = (~0UL) },
};

static struct cache_names __initdata cache_names[] = {
        {.name = "size-32", .name_dma = "size-32(DMA)"},
        {.name = "size-64", .name_dma = "size-64(DMA)"},
        {.name = "size-128", .name_dma = "size-128(DMA)"},
        {.name = "size-256", .name_dma = "size-256(DMA)"},
        .....
        {NULL,}
};
	即，定义了struct cache_sizes malloc_sizes[]和struct cache_names cache_names[]。
	并将这两个数组的地址赋值给了sizes和namas两个变量。
*/

/* 2 创建专用的高速缓存*/
#define INDEX_AC index_of(sizeof(struct arraycache_init))
#define INDEX_L3 index_of(sizeof(struct kmem_list3))
/*           
	INDEX_AC就是与sizeof(struct arraycache_init :28==>0)适配的kmalloc可分配缓存的索引.
	INDEX_L3就是与sizeof(struct kmem_list3 :56==>1)适配的kmalloc可分配缓存的索引.
*/          
其中
/*
 * bootstrap: The caches do not work without cpuarrays anymore, but the
 * cpuarrays are allocated from the generic caches...
 */
#define BOOT_CPUCACHE_ENTRIES   1
struct arraycache_init {
        struct array_cache cache;
        void *entries[BOOT_CPUCACHE_ENTRIES];
};

/* 
	即2这里创建了两个调用kmem_cache_create()函数创建了两个高速缓存。 
		即struct array_cache和struct kmem_list3所用的general cache。
    INDEX_AC是计算local cache所用的struct arraycache_init对象在kmalloc size中的索引 
    ，即属于哪一级别大小的general cache，创建此大小级别的cache为local cache所用。  
*/

/* 3 这里是一个循环，循环调用kmem_cache_create()函数，创建剩下的给kmalloc用的高速缓存 */
while (sizes->cs_size != ULONG_MAX) { /* ULONG_MAX = (~0UL)即最大值 */
    sizes->cs_cachep = kmem_cache_create(names->name, sizes->cs_size, ...);
    sizes->cs_dmacachep = kmem_cache_create(names->name_dma, sizes->cs_size, ...);
    sizes++;
    names++;
}
```





### 4） 用`kmalloc`对象替换静态分配的全局高速缓存变量



```c
/* 4) Replace the bootstrap head arrays */
/*
	用kmalloc对象替换静态分配的全局变量。
	到目前为止一共使用了两个全局local cache，
		一个是cache_cache的local cache指向initarray_cache.cache; 
    	另一个是malloc_sizes[INDEX_AC].cs_cachep的local cache指向initarray_generic.cache,参见setup_cpu_cache函数。
*/

{
    struct array_cache *ptr;

    ptr = kmalloc(sizeof(struct arraycache_init), GFP_KERNEL);
    /* 第2)和第3)步已经初始化了kmalloc用的高速缓存，这里可以使用kmalloc函数进行内存分配了 */

    local_irq_disable();
    BUG_ON(cpu_cache_get(&cache_cache) != &initarray_cache.cache);
    memcpy(ptr, cpu_cache_get(&cache_cache),
           sizeof(struct arraycache_init));
    /*
     * Do not assume that spinlocks can be initialized via memcpy:
     */
    spin_lock_init(&ptr->lock);

    cache_cache.array[smp_processor_id()] = ptr;
    local_irq_enable();

    ptr = kmalloc(sizeof(struct arraycache_init), GFP_KERNEL);

    local_irq_disable();
    BUG_ON(cpu_cache_get(malloc_sizes[INDEX_AC].cs_cachep)
           != &initarray_generic.cache);
    memcpy(ptr, cpu_cache_get(malloc_sizes[INDEX_AC].cs_cachep),
           sizeof(struct arraycache_init));
    /*
     * Do not assume that spinlocks can be initialized via memcpy:
     */
    spin_lock_init(&ptr->lock);

    malloc_sizes[INDEX_AC].cs_cachep->array[smp_processor_id()] = ptr; 
    local_irq_enable();
}
```

其中`cpu_cache_get()`定义如下。

```c
static inline struct array_cache *cpu_cache_get(struct kmem_cache *cachep)
{
        return cachep->array[smp_processor_id()];
}
```

`smp_processor_id()`函数用户获得当前的`CPU ID`的定义，详细定义暂时不展开。




### 5） 用`kmalloc`的空间替换静态分配的slab三链



```c
/* 5) Replace the bootstrap kmem_list3's */
/*
* 与第4步类似，用kmalloc的空间替换静态分配的slab三链 
*/
{
    int nid;

    for_each_online_node(nid) {
		/* 复制struct kmem_cache的slab三链 */  
        init_list(&cache_cache, &initkmem_list3[CACHE_CACHE + nid], nid);
        /* 复制struct arraycache_init的slab三链 */  
        init_list(malloc_sizes[INDEX_AC].cs_cachep,
                  &initkmem_list3[SIZE_AC + nid], nid);
		/* 复制struct kmem_list3的slab三链 */
        if (INDEX_AC != INDEX_L3) {
            init_list(malloc_sizes[INDEX_L3].cs_cachep,
                      &initkmem_list3[SIZE_L3 + nid], nid);
        }
    }
}

/* 其中 init_list()函数的定义如下 */
/*
 * swap the static kmem_list3 with kmalloced memory
 */
static void init_list(struct kmem_cache *cachep, struct kmem_list3 *list, int nodeid)
{
        struct kmem_list3 *ptr;

        ptr = kmalloc_node(sizeof(struct kmem_list3), GFP_KERNEL, nodeid); /* 1 */
        BUG_ON(!ptr);

        local_irq_disable();
        memcpy(ptr, list, sizeof(struct kmem_list3));
        /*
         * Do not assume that spinlocks can be initialized via memcpy:
         */
        spin_lock_init(&ptr->list_lock);

        MAKE_ALL_LISTS(cachep, ptr, nodeid); /* 2 */
        cachep->nodelists[nodeid] = ptr;
        local_irq_enable();
}
/* 1 */
/**
 * kmalloc_node - allocate memory from a specific node
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate (see kcalloc).
 * @node: node to allocate from.
 *
 * kmalloc() for non-local nodes, used to allocate from a specific node
 * if available. Equivalent to kmalloc() in the non-NUMA single-node
 * case.
 */
static inline void *kmalloc_node(size_t size, gfp_t flags, int node)
{
        return kmalloc(size, flags);
}
/* 2 */
#define MAKE_ALL_LISTS(cachep, ptr, nodeid)                             \
        do {                                                            \
        MAKE_LIST((cachep), (&(ptr)->slabs_full), slabs_full, nodeid);  \
        MAKE_LIST((cachep), (&(ptr)->slabs_partial), slabs_partial, nodeid); \
        MAKE_LIST((cachep), (&(ptr)->slabs_free), slabs_free, nodeid);  \
        } while (0)

#define MAKE_LIST(cachep, listp, slab, nodeid)                          \
        do {                                                            \
                INIT_LIST_HEAD(listp);                                  \
                list_splice(&(cachep->nodelists[nodeid]->slab), listp); \
        } while (0)

list_splice - join two lists, this is designed for stacks
```





### 6） `enable_cpucache`

```c
/* 6) resize the head arrays to their final sizes */
{
    struct kmem_cache *cachep;
    mutex_lock(&cache_chain_mutex);
    list_for_each_entry(cachep, &cache_chain, next)
        if (enable_cpucache(cachep))
            BUG();
    mutex_unlock(&cache_chain_mutex);
}
/* enable_cpucache()函数的定义如下 */
/* Called with cache_chain_mutex held always */
static int enable_cpucache(struct kmem_cache *cachep)
{
        int err;
        int limit, shared;

        /*
         * The head array serves three purposes:
         * - create a LIFO ordering, i.e. return objects that are cache-warm
         * - reduce the number of spinlock operations.
         * - reduce the number of linked list operations on the slab and
         *   bufctl chains: array operations are cheaper.
         * The numbers are guessed, we should auto-tune as described by
         * Bonwick.
         */
        if (cachep->buffer_size > 131072)
                limit = 1;
        else if (cachep->buffer_size > PAGE_SIZE)
                limit = 8;
        else if (cachep->buffer_size > 1024)
                limit = 24;
        else if (cachep->buffer_size > 256)
                limit = 54;
        else
                limit = 120;
        /*
         * CPU bound tasks (e.g. network routing) can exhibit cpu bound
         * allocation behaviour: Most allocs on one cpu, most free operations
         * on another cpu. For these cases, an efficient object passing between
         * cpus is necessary. This is provided by a shared array. The array
         * replaces Bonwick's magazine layer.
         * On uniprocessor, it's functionally equivalent (but less efficient)
         * to a larger limit. Thus disabled by default.
         */
        shared = 0;
        if (cachep->buffer_size <= PAGE_SIZE && num_possible_cpus() > 1)
                shared = 8;

#if DEBUG
        /*
         * With debugging enabled, large batchcount lead to excessively long
         * periods with disabled local interrupts. Limit the batchcount
         */
        if (limit > 32)
                limit = 32;
#endif
        err = do_tune_cpucache(cachep, limit, (limit + 1) / 2, shared);
        if (err)
                printk(KERN_ERR "enable_cpucache failed for %s, error %d.\n",
                       cachep->name, -err);
        return err;
}
```
### 收尾工作

```c
        /* Annotate slab for lockdep -- annotate the malloc caches */
        init_lock_keys();


        /* Done! */
        g_cpucache_up = FULL;

        /*
         * Register a cpu startup notifier callback that initializes
         * cpu_cache_get for all new cpus
         */
        register_cpu_notifier(&cpucache_notifier);

        /*
         * The reap timers are started later, with a module init call: That part
         * of the kernel is not yet operational.
         */
}

/* init_lock_keys()函数的定义 */
#ifdef CONFIG_LOCKDEP

/*
 * Slab sometimes uses the kmalloc slabs to store the slab headers
 * for other slabs "off slab".
 * The locking for this is tricky in that it nests within the locks
 * of all other slabs in a few places; to deal with this special
 * locking we put on-slab caches into a separate lock-class.
 *
 * We set lock class for alien array caches which are up during init.
 * The lock annotation will be lost if all cpus of a node goes down and
 * then comes back up during hotplug
 */
static struct lock_class_key on_slab_l3_key;
static struct lock_class_key on_slab_alc_key;

static inline void init_lock_keys(void)
{
        int q;
        struct cache_sizes *s = malloc_sizes;

        while (s->cs_size != ULONG_MAX) {
                for_each_node(q) {
                        struct array_cache **alc;
                        int r;
                        struct kmem_list3 *l3 = s->cs_cachep->nodelists[q];
                        if (!l3 || OFF_SLAB(s->cs_cachep))
                                continue;
                        lockdep_set_class(&l3->list_lock, &on_slab_l3_key);
                        alc = l3->alien;
                        /*
                         * FIXME: This check for BAD_ALIEN_MAGIC
                         * should go away when common slab code is taught to
                         * work even without alien caches.
                         * Currently, non NUMA code returns BAD_ALIEN_MAGIC
                         * for alloc_alien_cache,
                         */
                        if (!alc || (unsigned long)alc == BAD_ALIEN_MAGIC)
                                continue;
                        for_each_node(r) {
                                if (alc[r])
                                        lockdep_set_class(&alc[r]->lock,
                                             &on_slab_alc_key);
                        }
                }
                s++;
        }
}
#else
static inline void init_lock_keys(void)
{
}
#endif
                                                                         
```



### 总结

`kmem_cache_init()`函数主要，

- 初始化全局链表`cache_chain`

- 创建全局静态变量`cache_cache` , 将静态创建的变量`initarray_cache.array`赋值给`cache_cache.array`;将静态创建的`kmem_list3`按照内存结点编号赋值给`cache_cache.nodelists[nodeid]`;
- 创建通用内存分配用的`kmem_cache` ：`size-32`等；
- 将初始化`cache_cache`时用的`struct arraycache_init`类型的`initarray_cache.array`以及`struct kmem_list3`结构放入通用内存（`kmalloc`分配）的`kmem_cache`中。



### 参考资料

[Linux内存管理之slab分配器分析(二 初始化 kmem_cache_init)](http://blog.chinaunix.net/uid-31562863-id-5793196.html)

[Linux内存管理之slab机制（初始化）](https://www.linuxidc.com/Linux/2012-01/51239.htm)

[内存管理 初始化（五）kmem_cache_init 初始化slab分配器(上)](https://www.cnblogs.com/openix/p/3351656.html)



## kmem_cache_create



## kmem_cache_alloc



## kmem_cache_free



## kmem_cache_destroy



## 通用内存对象管理举例 ： zalloc/kmalloc和kfree
		malloc_sizes 数组



## 专用内存对象管理举例： task_struct




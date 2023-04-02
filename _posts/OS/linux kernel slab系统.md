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

![kmem_bufctl_t](https://gitee.com/BiscuitOS_team/PictureSet/raw/Gitee/RPI/TB000008.png)

在每个 slab 中，起始处是为着色预留的区间，接下来是` struct slab` 结构，该结构就是用于管理 slab 的数据，紧接着就是一个 `kmem_bufctl`数组，该数组是由 `kmem_bufctl_t` 类型的成员组成的数组。接下来是对象区，这就是用于分配的对象。 在 `kmem_bufctl` 数组中，每个成员按顺序与每个对象进行绑定，用于表示该对象的下 一个对象的索引，并且最后一个对象的值为 `BUFCTL_END`, 这样将 slab 中的所有对象通过一个单链表进行维护。通过上面的分析，`kmem_bufctl_t` 数据结构构造的数据用于指向当前对象指向的下一个对象的索引.

![slab](https://gitee.com/BiscuitOS_team/PictureSet/raw/Gitee/RPI/TB000007.png)

在 SLAB 分配器中，struct slab 可以位于 slab 内部，也可以位于 slab 外部，但 不论位于 slab 内部或外部，其成员的含义都是一致的。colouroff 成员用于指明该 slab 着色总长度，其包含了着色、struct slab 以及 kmem_bufctl array 的总长度， 这样可用对象加载到 CACHE 之后就会被插入到不同的 CACHE 行; s_mem 用于指向 slab 中对象的起始地址; inuse 成员用于指定 slab 中可用对象的数量; free 成员 用于指向 slab 当前可用对象在 kmem_bufctl 的索引; nodeid 成员用于指向该 slab 位于哪个 NODE; list 成员用于将 slab 加入到 struct kmem_cache 的 kmem_list3 中三个链表中的其中一个链表。

### object 

  每个slab只负责管理一种类型的“对象”， 将该大小进行内存对齐后的大小就是该“对象”的大小，依此大小对slab进行划分，从而实现细粒度的内存管理



### 结构之间的关系

![Memery_Layout_14](http://chenweixiang.github.io/assets/Memery_Layout_14.jpg)

`cache_chain`的定义：`static struct list_head cache_chain;`是一个`struct kmem_cache`的链表，将所有`struct kemem_cache`连接起来。

`cache_cache`是第一个`kemem_cache_struct`结构，通过`struct kmem_list3`结构维护的三个`struct slab`链表构成了内核的第一个`slab`系统，用于存储`slab`系统所有的`struct kmem_cache_struct`结构。

`struct slab`结构可以通过参数选择1.存储在自己`slab`管理的页面上，2.存储到`cache_cache`中?

`struct kmem_list3`结构存在哪里？

![结构](https://gitee.com/BiscuitOS_team/PictureSet/raw/Gitee/RPI/TB000001.png)

### 参考资料
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
```
主要两个函数起作用：`kmem_list3_init()`和`set_up_list3s`;
```c
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

/* begin of 3 */
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
/* enable_cpucache()函数  在分析kmem_cache_create函数的 “初始化本地以及共享的高速缓存”小结分析 */

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



### 关于`kmem_cache`的初始化阶段

由于 SLAB 分配 器的初始化是一个 “鸡和蛋” 的问题，因此 SLAB 分配器将初始化分作几个阶段，通过`g_cpucache_up` 进行描述，`g_cpucache_up` 可以取值:

```c
/*
 * chicken and egg problem: delay the per-cpu array allocation
 * until the general caches are up.
 */
static enum {
        NONE,
        PARTIAL_AC,
        PARTIAL_L3,
        FULL
} g_cpucache_up;


```

- `NONE` 阶段 SLAB 分配器只能分配 `struct kmem_cache` 对应的高速缓存，且功能不是很完整;   

- `PARTIAL_AC` 阶段 SLAB 分配器已经增加了` struct array_cache` 对应的高速 缓存，因此高速函数可以为本地高速缓存分配内存了;

- `PARTIAL_L3` 阶段，SLAB 分配器 增加了` struct kmem_list3` 的缓存，因此该阶段 `kmalloc` 等函数已经可以使用;

- ~~`EARLY` 阶段，SLAB 分配器已经将分配器相关的所有数据都使用 SLAB 分配器分配内存，不再使用静态数据~~; 

- `FULL`阶段 SLAB 分配器已经可以像一个正常的分配器为内核提供接口 进行内存分配和回收. 

  ~~因此 SLAB 如果处于 EARLY 及其之后都表示 SLAB 已经准备好， 可以开始使用了.~~



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

SLAB 分配器使用 struct kmem_cache 表示一个高速缓存，其包含了高速缓存的基础信息， 其中也包含了高速缓存的本地缓存、共享缓存，以及 slab 链表等。高速缓存的创建过程 就是用于填充和创建高速缓存所需的数据。

这函数主要创建一个`kmem_cache`，但是并不实际分配内存，创建好`kmem_cache`后调用`kmem_cache_alloc()`请求分配对象的时候才调用`cache_grow()`函数向伙伴系统申请内存页框。

### 函数参数说明



```c
/**
 * kmem_cache_create - Create a cache.
 * @name: A string which is used in /proc/slabinfo to identify this cache.
 * @size: The size of objects to be created in this cache.
 * @align: The required alignment for the objects.
 * @flags: SLAB flags
 * @ctor: A constructor for the objects.
 *
 * Returns a ptr to the cache on success, NULL on failure.
 * Cannot be called within a int, but can be interrupted.
 * The @ctor is run when new pages are allocated by the cache.
 *
 * @name must be valid until the cache is destroyed. This implies that
 * the module calling this has to destroy the cache before getting unloaded.
 * Note that kmem_cache_name() is not guaranteed to return the same pointer,
 * therefore applications must manage it themselves.
 *
 * The flags are
 *
 * %SLAB_POISON - Poison the slab with a known test pattern (a5a5a5a5)
 * to catch references to uninitialised memory.
 *
 * %SLAB_RED_ZONE - Insert `Red' zones around the allocated memory to check
 * for buffer overruns.
 *
 * %SLAB_HWCACHE_ALIGN - Align the objects in this cache to a hardware
 * cacheline.  This can be beneficial if you're counting cycles as closely
 * as davem.
 */
```
- name : `kmem_cache`的名字；
- size: `kmem_cache`中对象的大小；
- align ：对象所需的对齐方式
- flags : `SLAB flags`
  - SLAB_POISON
  - SLAB_RED_ZONE
  - SLAB_HWCACHE_ALIGN
- ctor :  一个构造函数，当向`kmem_cache`中添加新的物理页时被调用。

### 检查参数以及运行环境

```c
struct kmem_cache *
kmem_cache_create (const char *name, size_t size, size_t align,
        unsigned long flags, void (*ctor)(void *))
{
        size_t left_over, slab_size, ralign;
        struct kmem_cache *cachep = NULL, *pc;

        /*
         * Sanity checks... these are all serious usage bugs.
         */
        if (!name || in_interrupt() || (size < BYTES_PER_WORD) ||
            size > KMALLOC_MAX_SIZE) {
                printk(KERN_ERR "%s: Early error in slab %s\n", __func__,
                                name);
                BUG();
        }
```
这一段代码主要判断传入的参数或者CPU运行的环境是否满足。如果1.调用函数时参数中给的name为空; 2. 此时CPU在执行中断服务程序 （in_interrupt()此函数通过thread_info的一下信息去判断，以后分析进程管理的时候再分析; 3. 要创建的kmem_cache的size比一个字的字节数还少或比kmalloc所允许分配的最大的内存还大， 就报oops错。

`BYTES_PER_WORD`和`KMALLOC_MAX_SIZE`的定义如下：

```c
#define BYTES_PER_WORD          sizeof(void *)
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

```



继续分析`kmem_cache_create()`函数的流程。

```c

        /*
         * We use cache_chain_mutex to ensure a consistent view of
         * cpu_online_mask as well.  Please see cpuup_callback
         */
        get_online_cpus(); /* 如果此时系统支持CPU热插拔，那么此函数获得可用的CPU，否则什么都不干 */
        mutex_lock(&cache_chain_mutex);
		
		/* 遍历所有的高速缓存 */
        list_for_each_entry(pc, &cache_chain, next) {
                char tmp;
                int res;

                /*
                 * This happens when the module gets unloaded and doesn't
                 * destroy its slab cache and no-one else reuses the vmalloc
                 * area of the module.  Print a warning.
                 */
                res = probe_kernel_address(pc->name, tmp);
                if (res) {
                        printk(KERN_ERR
                               "SLAB: cache with size %d has lost its name\n",
                               pc->buffer_size);
                        continue;
                }

                if (!strcmp(pc->name, name)) { /* 如果发现已经存在相同名字的高速缓存就dump_stack()并报oops */
                        printk(KERN_ERR
                               "kmem_cache_create: duplicate cache %s\n", name);
                        dump_stack();
                        goto oops;
                }
        }

#if DEBUG
        WARN_ON(strchr(name, ' '));     /* It confuses parsers */
#if FORCED_DEBUG
        /*
         * Enable redzoning and last user accounting, except for caches with
         * large objects, if the increased size would increase the object size
         * above the next power of two: caches with object sizes just above a
         * power of two have a significant amount of internal fragmentation.
         */
        if (size < 4096 || fls(size - 1) == fls(size-1 + REDZONE_ALIGN +
                                                2 * sizeof(unsigned long long)))
                flags |= SLAB_RED_ZONE | SLAB_STORE_USER;
        if (!(flags & SLAB_DESTROY_BY_RCU))
                flags |= SLAB_POISON;
#endif
        if (flags & SLAB_DESTROY_BY_RCU)
                BUG_ON(flags & SLAB_POISON);
#endif
        /*
         * Always checks flags, a caller might be expecting debug support which
         * isn't available.
         */
        BUG_ON(flags & ~CREATE_MASK);
```
### 计算对齐

根据高速缓存创建时提供的标志进行对齐相关的操作， 最终会将对齐的结果存储在 align 变量里.

```c
        /*
         * Check that size is in terms of words.  This is needed to avoid
         * unaligned accesses for some archs when redzoning is used, and makes
         * sure any on-slab bufctl's are also correctly aligned.
         */
        if (size & (BYTES_PER_WORD - 1)) {
                size += (BYTES_PER_WORD - 1);
                size &= ~(BYTES_PER_WORD - 1);
        }     

		/* calculate the final buffer alignment: */

        /* 1) arch recommendation: can be overridden for debug */
        if (flags & SLAB_HWCACHE_ALIGN) {
                /*
                 * Default alignment: as specified by the arch code.  Except if
                 * an object is really small, then squeeze multiple objects into
                 * one cacheline.
                 */
                ralign = cache_line_size();
                while (size <= ralign / 2)
                        ralign /= 2;
        } else {
                ralign = BYTES_PER_WORD;
        }

        /*
         * Redzoning and user store require word alignment or possibly larger.
         * Note this will be overridden by architecture or caller mandated
         * alignment if either is greater than BYTES_PER_WORD.
         */
        if (flags & SLAB_STORE_USER)
                ralign = BYTES_PER_WORD;

        if (flags & SLAB_RED_ZONE) {
                ralign = REDZONE_ALIGN;
                /* If redzoning, ensure that the second redzone is suitably
                 * aligned, by adjusting the object size accordingly. */
                size += REDZONE_ALIGN - 1;
                size &= ~(REDZONE_ALIGN - 1);
        }

        /* 2) arch mandated alignment */
        if (ralign < ARCH_SLAB_MINALIGN) {
                ralign = ARCH_SLAB_MINALIGN;
        }
        /* 3) caller mandated alignment */
        if (ralign < align) {
                ralign = align;
        }
        /* disable debug if necessary */
        if (ralign > __alignof__(unsigned long long))
                flags &= ~(SLAB_RED_ZONE | SLAB_STORE_USER);
        /*
         * 4) Store it.
         */
        align = ralign;
```
### 分配`kmem_cache`描述符
```c
        /* Get cache's description obj. */
        cachep = kmem_cache_zalloc(&cache_cache, GFP_KERNEL);
        if (!cachep)
                goto oops;

#if DEBUG
        cachep->obj_size = size;

        /*
         * Both debugging options require word-alignment which is calculated
         * into align above.
         */
        if (flags & SLAB_RED_ZONE) {
                /* add space for red zone words */
                cachep->obj_offset += sizeof(unsigned long long);
                size += 2 * sizeof(unsigned long long);
        }
        if (flags & SLAB_STORE_USER) {
                /* user store requires one word storage behind the end of
                 * the real object. But if the second red zone needs to be
                 * aligned to 64 bits, we must allow that much space.
                 */
                if (flags & SLAB_RED_ZONE)
                        size += REDZONE_ALIGN;
                else
                        size += BYTES_PER_WORD;
        }
#if FORCED_DEBUG && defined(CONFIG_DEBUG_PAGEALLOC)
        if (size >= malloc_sizes[INDEX_L3 + 1].cs_size
            && cachep->obj_size > cache_line_size() && size < PAGE_SIZE) {
                cachep->obj_offset += PAGE_SIZE - size;
                size = PAGE_SIZE;
        }
#endif
#endif
```
### 判断slab管理数据是放在slab内部还是外部

判断依据是`kmem_cache`中对象的大小，即传递给`kmem_cache_create()`函数的`size`参数。

```c

        /*
         * Determine if the slab management is 'on' or 'off' slab.
         * (bootstrapping cannot cope with offslab caches so don't do
         * it too early on.)
         */
        if ((size >= (PAGE_SIZE >> 3)) && !slab_early_init) 
                /*
                 * Size is large, assume best to place the slab management obj
                 * off-slab (should allow better packing of objs).
                 */
                flags |= CFLGS_OFF_SLAB;  
```
这个`if`判断 slab 默认的管理数据是位于 slab 内部还是外部，判断的条件之一是 `slab_early_init` 是否为零（`kmem_cache_init()`函数）在2+3）步骤时赋值为0），这也说明了 SLAB 初始化完毕之前， slab 的管理数据默认只能待在 slab 内部。 

函数判断如果 slab 管理数据位于 slab 外部，必须满足这几个条件: 其一高速缓存对象的长度不小于 “PAGE_SIZE » 3”, 其二 SLAB 分配器已经过了基础初始化阶段， 只有同时满足上面条件，那么 slab 的管理数据才能位于 slab 外部. 如果条件都满足了，那么函数将 CFLAGS_OFF_SLAB 标志添加到 flags 参数 里面. （在后来版本的内核中还加入了第三个条件： flags 标志不能包含 SLAB_NOLEAKTRACE.）

### 计算slab的容量



```c
        size = ALIGN(size, align); /* 获得高速缓存对象对齐之后的长度 */

        left_over = calculate_slab_order(cachep, size, align, flags); /* 计算出高速缓存每个 slab 维护缓存对象的数量， */

        if (!cachep->num) {
                printk(KERN_ERR
                       "kmem_cache_create: couldn't create cache %s.\n", name);
                kmem_cache_free(&cache_cache, cachep);
                cachep = NULL;
                goto oops;
        }
        slab_size = ALIGN(cachep->num * sizeof(kmem_bufctl_t)
                          + sizeof(struct slab), align); /* 计算除了 slab 管理数据的长度，并存储在 slab_size 变量中. */
```
首先，获得高速缓存对象对齐之后的长度，然后，调用 `calculate_slab_order()` 函数计算出高速缓存每个 slab 维护缓存对象的数量，然后，计算每个 slab 占用物理页的数量，并计算出每个 slab 剩余的内存数量. 函数在 2306 行计算除了 slab 管理数据的长度，并存储在 slab_size 变量中.

### 最终确定slab管理数据的存储位置



```c
        /*
         * If the slab has been placed off-slab, and we have enough space then
         * move it on-slab. This is at the expense of any extra colouring.
         */
        if (flags & CFLGS_OFF_SLAB && left_over >= slab_size) {
                flags &= ~CFLGS_OFF_SLAB;
                left_over -= slab_size;
        }

        if (flags & CFLGS_OFF_SLAB) {
                /* really off slab. No need for manual alignment */
                slab_size =
                    cachep->num * sizeof(kmem_bufctl_t) + sizeof(struct slab);
        }
```
检测 slab 的管理数据是否位于 slab 外面，同时也检测 slab 剩余 的内存是否比 slab 管理数据大。 

如果高速缓存的 flags 参数包含了 CFLAGS_OFF_SLAB 标志，即表示想让 slab 管理数据位于 slab 外部，当如果此时 slab 管理数据的长度 小于等于 slab 剩余的内存，那么 SLAB 分配器不会让该高速缓存的 slab 管理数据 位于 slab 外部，因此将 CFLAGS_OFF_SLAB 标志从 flags 中移除，并将 slab 中剩余 的内存减去了 slab 管理数据的长度.

函数接着检测 slab 管理数据是否位于 slab 外部，如果此时 slab 管理数据 位于 slab 外部，那么 SLAB 分配器需要独立计算 slab 管理数据占用的内存数量，从上图可以看出 slab 管理数据包括了 `struct slab` 以及 `kmem_bufctl` 数组，`kmem_bufctl` 数组的长度与 slab 中位于缓存对象数量有关.

### 计算slab着色



```c
        cachep->colour_off = cache_line_size();
        /* Offset must be a multiple of the alignment. */
        if (cachep->colour_off < align)
                cachep->colour_off = align;
/*2337*/cachep->colour = left_over / cachep->colour_off;
        cachep->slab_size = slab_size;
        cachep->flags = flags;
        cachep->gfpflags = 0;
        if (CONFIG_ZONE_DMA_FLAG && (flags & SLAB_CACHE_DMA))
                cachep->gfpflags |= GFP_DMA;
        cachep->buffer_size = size;
        cachep->reciprocal_buffer_size = reciprocal_value(size);
```
调用`cache_line_size() `计算出当前 `cache line` 的长度，以此作为高速 缓存着色的长度，存储在高速缓存的 `colour_off` 成员里，如果 `cache line` 的长度 小于之前计算出来的对齐长度，那么函数还是将高速缓存的着色长度设置为` align`。  

函数在 2337 行将之前获得的 slab 剩余的内存除以着色长度，以此获得高速缓存着色范围，并存储在高速缓存的 colour 成员里.   

接着在 2338 行将 slab 管理数据的长度 存储在高速缓存的 slab_size 成员里，  

2339 行将高速缓存的创建标志存在在 flags 成员里，  

并在 2341 行到 2342 行计算高速缓存从 Buddy 分配器中获得物理页的标志.   

2343 行将缓存对象的长度存储在高速缓存的 buffer_size 成员里.

### 初始化slab构造函数

```c
        if (flags & CFLGS_OFF_SLAB) {
                cachep->slabp_cache = kmem_find_general_cachep(slab_size, 0u);
                /*
                 * This is a possibility for one of the malloc_sizes caches.
                 * But since we go off slab only for object size greater than
                 * PAGE_SIZE/8, and malloc_sizes gets created in ascending order,
                 * this should not happen at all.
                 * But leave a BUG_ON for some lucky dude.
                 */
                BUG_ON(ZERO_OR_NULL_PTR(cachep->slabp_cache));
        } 
        cachep->ctor = ctor; /* 设置了高速缓存的构造函数 */
        cachep->name = name; /* 设置了高速缓存的名字*/
```
检测 slab 管理数据是否位于 slab 外部，如果位于，那么函数调用`kmem_find_general_cachep()` 函数根据 slab 管理数据的长度，从 SLAB 中获得一个 合适的通用高速缓存，并使用高速缓存的 `slabp_cache` 指向该通用高速缓存.  

之后函数 设置了高速缓存的构造函数和高速缓存的名字.

### 初始化本地以及共享的高速缓存



```c

        if (setup_cpu_cache(cachep)) {
                __kmem_cache_destroy(cachep);
                cachep = NULL;
                goto oops;
        }

        /* cache setup completed, link it into the list */
        list_add(&cachep->next, &cache_chain);
```
调用`setup_cpu_cache()` 函数用于设置高速缓存的本地高速缓存、 共享高速缓存以及 slab 链表. 如果设置失败，那么函数调用 `__kmem_cache_destroy()` 函数摧毁该高速缓存，并将高速缓存指针 cachep 设置为 NULL, 最后跳转到 oops. 如果分配成功，那么将高速缓存插入到系统高速缓存链表 cache_chain 里.

#### setup_cpu_cache()

在每个高速缓存中，SLAB 分配器为加速高速缓存的分配速度，为每个 CPU 创建了一个 本地高速缓存，其使用一个缓存栈维护一定数量的可用缓存对象.  

由于 SLAB 分配器的初始化分作多个阶段，每个阶段都涉及到为高速缓存创建本地高速缓存。 

SLAB 分配器 使用 `g_cpucache_up` 变量来标示 SLAB 分配器的初始化进度。  

 SLAB 分配器初始化的第一阶段，这个阶段`struct kmem_list3` 和`struct cache_array` 对应的高速缓存还没有创建，因此为该阶段的高速缓存分配本地缓存只能使用静态变量`initarray_generic`对应的本地高速缓存，由于这个阶段只有 一个 CPU 在运行，因此函数在 首先设置了本地高速缓存. 

然后调用`set_up_list3s()`函数设置这个阶段的`kmem_list3` 链表. 如果此时`INDEX_AC` 的值与` INDEX_L3 `的值 相等，那么函数将 SLAB 初始化进度设置为` PARTIAL_L3`; 反之设置为` PARTIAL_AC`。通过 源码分析可以知道，此时` INDEX_AC` 与` INDEX_L3`不相等，因此将 SLAB 分配器初始化 进度设置为` PARTIAL_AC`. 

SLAB 分配器初始化进度达到 第二阶段之后，首先为高速缓存分配本地高速缓存, 此时 `kmalloc()` 函数已经可以使用，2047 行到 2048 行，函数调用`kmalloc()`函数为高速缓存分配内存， 此时系统还是只有一个 CPU，由于此时 SLAB 分配器初始化进度是 `PARTIAL_AC`，因此 函数在 2051 行将 SLAB 分配器初始化进度设置为 `PARTIAL_L3`;

 当 SLAB 分配器初始化 进入第三阶段，此时系统中多个 CPU 已经可以使用，因此函数在 2055 行遍历所有的 在线 CPU，并调用 `kmalloc_node()` 函数为每个 CPU 分配本地高速缓存，并调用` kmem_list3_init()`函数初始化高速缓存的` kmem_list3 `链表. 对于 SLAB 分配器 初始化进度前三个阶段，SLAB 分配器将这些阶段的本地缓存维护的缓存对象控制在 `BOOT_CPUCACHE_ENTRIES` 个.

函数 2068 行到 2073 行初始化了基础的成员. 

SLAB 分配器初始化进度到第四阶段，即 g_cpucache_up 等于 FULL, 那么函数在 2026 行 调用 `enable_cpucache()` 函数初始化高速缓存的本地高速缓存、共享高速缓存，以及`kmem_list3 slab` 链表.

`setup_cpu_cache()`函数用于为高速缓存设置本地高速缓存. 参数 cachep 指向高速缓存。

```c

static int __init_refok setup_cpu_cache(struct kmem_cache *cachep)
{
        if (g_cpucache_up == FULL)
                return enable_cpucache(cachep); /* 2026 行 */

        if (g_cpucache_up == NONE) {  /* slab分配器处于初始化第一阶段 */
                /*
                 * Note: the first kmem_cache_create must create the cache
                 * that's used by kmalloc(24), otherwise the creation of
                 * further caches will BUG().
                 */
                cachep->array[smp_processor_id()] = &initarray_generic.cache; /*设置了本地高速缓存*/

                /*
                 * If the cache that's used by kmalloc(sizeof(kmem_list3)) is
                 * the first cache, then we need to set up all its list3s,
                 * otherwise the creation of further caches will BUG().
                 */
                set_up_list3s(cachep, SIZE_AC); /* 设置初始化阶段的kmem_list3链表 */
                if (INDEX_AC == INDEX_L3)
                        g_cpucache_up = PARTIAL_L3;
                else
                        g_cpucache_up = PARTIAL_AC; 
        } else { /*第二阶段*/
/*2047*/        cachep->array[smp_processor_id()] =
                        kmalloc(sizeof(struct arraycache_init), GFP_KERNEL);

                if (g_cpucache_up == PARTIAL_AC) {
                        set_up_list3s(cachep, SIZE_L3);
                        g_cpucache_up = PARTIAL_L3;   /*2051 */
                } else { /*第三阶段*/
                        int node;
                        for_each_online_node(node) { 
                                cachep->nodelists[node] =
                                    kmalloc_node(sizeof(struct kmem_list3),
                                                GFP_KERNEL, node);
                                BUG_ON(!cachep->nodelists[node]);
                                kmem_list3_init(cachep->nodelists[node]);
                        }
                }
        } 
        cachep->nodelists[numa_node_id()]->next_reap =
                        jiffies + REAPTIMEOUT_LIST3 +
                        ((unsigned long)cachep) % REAPTIMEOUT_LIST3;
		/*初始化了基础的成员*/
        cpu_cache_get(cachep)->avail = 0;  /*2068*/
        cpu_cache_get(cachep)->limit = BOOT_CPUCACHE_ENTRIES;
        cpu_cache_get(cachep)->batchcount = 1;
        cpu_cache_get(cachep)->touched = 0;
        cachep->batchcount = 1;
        cachep->limit = BOOT_CPUCACHE_ENTRIES;
        return 0;
}
```



#### enable_cpucache()

`enable_cpucache()` 函数用于计算一个高速缓存的本地缓存维护缓存对象的数量，以及高速缓存对应的共享高速缓存维护缓存对象的数量，并为高速缓存分配本地缓存和 `kmem_list3` 链表, 最后更新本地缓存.

```c
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
        if (cachep->buffer_size > 131072) /* 3971  */
                limit = 1;
        else if (cachep->buffer_size > PAGE_SIZE)
                limit = 8;
        else if (cachep->buffer_size > 1024)
                limit = 24;
        else if (cachep->buffer_size > 256)
                limit = 54;
        else
                limit = 120; /* 3980 */

        /*
         * CPU bound tasks (e.g. network routing) can exhibit cpu bound
         * allocation behaviour: Most allocs on one cpu, most free operations
         * on another cpu. For these cases, an efficient object passing between
         * cpus is necessary. This is provided by a shared array. The array
         * replaces Bonwick's magazine layer.
         * On uniprocessor, it's functionally equivalent (but less efficient)
         * to a larger limit. Thus disabled by default.
         */
        shared = 0;  /* 3991 */
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
        err = do_tune_cpucache(cachep, limit, (limit + 1) / 2, shared); /* 4003  */
        if (err)
                printk(KERN_ERR "enable_cpucache failed for %s, error %d.\n",
                       cachep->name, -err);
        return err;
}

```

SLAB 分配器为加快高速缓存的分配速度，为每个 CPU 创建了一个本地高速缓存，其使用 一个缓存栈维护一定数量的可用缓存对象，每个 CPU 都可以快速从缓存栈上分配可用 的缓存对象.   

SLAB 分配器还为高速缓存创建了一个共享高速缓存，所有 CPU 都可以从共享高速缓存中获得可用的缓存对象。  

SLAB 分配器使用` kmem_list3` 链表为高速缓存在指定 NODE 上维护了 3 类链表，以此管理指定的 slab。  

`enable_cpucache()` 函数就是用于为一个高速缓存规划本地高速缓存和共享高速缓存中维护缓存对象的数量，以及创建并更新本地缓存，最后为高速缓存创建 `kmem_list3 `链表。

函数 3971 行到 3980 行用于计算一个高速缓存对应的本地高速缓存维护可用缓存对象的数量，从代码逻辑可以推断高速缓存对象越大，本地高速缓存能够维护可用缓存对象 数量越少。  

3991 行到 3993 行，如果高速缓存对象的大小不大于` PAGE_SIZE`, 且 `num_possible_cpus()` 数量大于 1，即 CPU 数量大于 1，那么将共享高速缓存维护的缓存对象设置为 8.  

 最后函数在 4003 行处调用 `do_tune_cpucache() `函数为高速缓存 创建了本地高速缓存和共享高速缓存，以及` kmem_list3` 链表.

#### do_tune_cpucache()

`do_tune_cpucache() `函数用于为缓存分配本地缓存、共享缓存和 l3 链表.   

参数 `cachep` 指向缓存，`limit` 用于指明本地缓存最大缓存对象的数量。

`shared` 参数用于 指明共享缓存中缓存对象的数量;

函数 3920 行调用 `kzalloc() `函数为 `struct ccupdate_struct `分配新的内存。  

3924 行到 3933 行，函数调用 `for_each_online_cpu()` 函数遍历所有的 CPU，并基于 `struct ccupdate_struct` 结构为 CPU 分配本地缓存结构 `struct cache_array`, 通过 `alloc_arraycache()`函数进行分配.   

函数 3936 行调用 `on_each_cpu()` 函数与` do_ccupdate_local()` 函数更新当前高速缓存的本地缓存.   

接着函数设置了高速缓存的 `batchcount`,` limit` 以及 `shared` 成员.  

 函数 3943 行到 3951 行遍历所有在线的 CPU，如果发生了 CPU 热插拔事件之后，离线的 CPU 没有释放本地缓存，那么函数并检测高速缓存对应的本地缓存是否存在，如果存在，那么释放本地缓存. 函数最后释放 new 变量，并调用 `alloc_kmemlist() `函数为高速缓存分配` kmem_list3 `链表数据.

```c
static int do_tune_cpucache(struct kmem_cache *cachep, int limit,
                                int batchcount, int shared)
{
        struct ccupdate_struct *new;
        int i;

        new = kzalloc(sizeof(*new), GFP_KERNEL); /* 3920  */
        if (!new)
                return -ENOMEM;

        for_each_online_cpu(i) { /* 3924 */
                new->new[i] = alloc_arraycache(cpu_to_node(i), limit,
                                                batchcount);
                if (!new->new[i]) {
                        for (i--; i >= 0; i--)
                                kfree(new->new[i]);
                        kfree(new);
                        return -ENOMEM;
                }
        } /* 3933 */
        new->cachep = cachep;

        on_each_cpu(do_ccupdate_local, (void *)new, 1); /* 3936 */

        check_irq_on();
        cachep->batchcount = batchcount;
        cachep->limit = limit;
        cachep->shared = shared;

        for_each_online_cpu(i) { /* 3943  */
                struct array_cache *ccold = new->new[i];
                if (!ccold)
                        continue;
                spin_lock_irq(&cachep->nodelists[cpu_to_node(i)]->list_lock);
                free_block(cachep, ccold->entry, ccold->avail, cpu_to_node(i));
                spin_unlock_irq(&cachep->nodelists[cpu_to_node(i)]->list_lock);
                kfree(ccold);
        } /* 3951  */
        kfree(new);
        return alloc_kmemlist(cachep);
}
```

alloc_arraycache()

alloc_kmemlist()

 struct ccupdate_struct

### oops的处理流程

```c
oops:
        if (!cachep && (flags & SLAB_PANIC))
                panic("kmem_cache_create(): failed to create slab `%s'\n",
                      name);
        mutex_unlock(&cache_chain_mutex);
        put_online_cpus();
        return cachep;
}
EXPORT_SYMBOL(kmem_cache_create);
                                                           
```

`oops`的处理流程：首先判断高速缓存是否已经成功分配了，如果没有成功分配且`flags` 中包含了` SLAB_PANIC `标志，那么调用` panic()` 函数.  否则，函数解除 `cache_chain_mutex` 互斥锁，并调用 `put_online_cpus()` 函数。函数最后返回指向高速缓存的指针 `cachep`，并将该函数通过 `EXPORT_SYMBOL()` 导出给内核其他部分使用.





## kmem_cache_alloc

`kmem_cache_alloc()`函数用于从缓存对象中分配一个可用的缓存对象. 参数 `cachep` 指向缓存对象; 参数`flags`用于指明分配物理内存标志。

```c

/**
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



__cache_alloc()定义如下

```c
static __always_inline void *
__cache_alloc(struct kmem_cache *cachep, gfp_t flags, void *caller)
{
        unsigned long save_flags;
        void *objp;

        lockdep_trace_alloc(flags);

        if (slab_should_failslab(cachep, flags))
                return NULL;

        cache_alloc_debugcheck_before(cachep, flags);
        local_irq_save(save_flags);
        objp = __do_cache_alloc(cachep, flags); /* 分配可用的缓存对象 */
        local_irq_restore(save_flags);
        objp = cache_alloc_debugcheck_after(cachep, flags, objp, caller);
        prefetchw(objp); /* 将可用缓存对象预加载到 CACHE */

        if (unlikely((flags & __GFP_ZERO) && objp)) /* 如果检测到 flags 参数中包含了 __GFP_ZERO 标志， */
                memset(objp, 0, obj_size(cachep)); /* 调用 memset() 函数将缓存对象指向的内存清零. */

        return objp; /*  最后返回缓存对象. */
}

```

__cache_alloc() 函数用于从缓存对象中分配可用的缓存对象. 参数 cachep 指向 缓存对象; 参数 flags 指向物理内存分配标志; 参数 caller 用于指向申请者.



UMA平台

```c
static __always_inline void *
__do_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
        return ____cache_alloc(cachep, flags);
}


```

____cache_alloc()函数

```c
static inline void *____cache_alloc(struct kmem_cache *cachep, gfp_t flags)
{
        void *objp;
        struct array_cache *ac;

        check_irq_off();

        ac = cpu_cache_get(cachep); /* 获得缓存对应的缓存栈 */
        if (likely(ac->avail)) { /* 检测缓存栈的栈顶, 以此判断缓存栈里是否有可用的缓存对象，如果栈顶不为零，即 avail 成员不为 0 */
                STATS_INC_ALLOCHIT(cachep);
                ac->touched = 1;
                objp = ac->entry[--ac->avail]; /* 从缓存栈上获得可用的缓存对象, 并更新缓存栈的栈顶*/ 
        } else { /* 如果栈顶为 0，那么表示缓存栈上没有可用的缓存对象 */
                STATS_INC_ALLOCMISS(cachep);
                objp = cache_alloc_refill(cachep, flags); /*调用 cache_alloc_refill() 函数将一定数量的可用缓存对象添加到缓存栈上，并再次使用 ac 变量指向缓存栈. */
        }
        return objp;
}
```

![array_cache](https://gitee.com/BiscuitOS_team/PictureSet/raw/Gitee/RPI/TB000001.png)

在 SLAB 分配器里，为了加速缓存对象的分配，SLAB 分配器为缓存对象针对每个 CPU 构建了一个缓存栈，CPU 可以快速从缓存栈上获得可用的缓存对象。其实现原理如上图， 缓存对象使用 struct kmem_cache 进行维护，其成员 array 是一个 `struct cache_array` 数组，数组中的每个成员对应一个缓存栈，因此 `struct cache_array` 维护着一个缓存栈.

#### cpu_cache_get()

```c
static inline struct array_cache *cpu_cache_get(struct kmem_cache *cachep)
{
        return cachep->array[smp_processor_id()];
}

```



#### cache_alloc_refill()

每当 SLAB 分配缓存对象的时候，如果发现缓存栈上没有可用的缓存对象时候，  

那么 SLAB 分配器就会检测缓存对象的 slabs_partial 上是否有可用的 slab，  

​	如果有，那么 SLAB 从 slabs_partial 的 slab 上获取 batchcount 数量的缓存对象到缓存栈里; 

​	如果 slabs_partial 上没有可用的 slab，那么 SLAB 分配器检测 slabs_free 上时候有可用 的 slab， 

​		如果有，那么就从对应的 slab 上获得 batchcount 数量的缓存对象到缓存栈 上;

​		 如果此时 slabs_free 上也找不到 slab，

​			那么 SLAB 分配器需要为缓存对象扩充新 的 slab，那么 SLAB 分配器从 Buddy 分配指定数量的物理内存，并将这些内存组织成 新的 slab，并插入到 slabs_free 链表上，并重新执行之前的查找过程. 

重新查找的过程中，slabs_free 链表上有可用的 slab，那么缓存栈获得 batchcount 数量的缓存对象 之后，将 slab 从 slabs_free 上移除并插入到 slabs_partial 链表上。

```c
static void *cache_alloc_refill(struct kmem_cache *cachep, gfp_t flags)
{
		int batchcount;
        struct kmem_list3 *l3;
        struct array_cache *ac;
        int node;

retry:
        check_irq_off();
        node = numa_node_id();
        ac = cpu_cache_get(cachep); /* 获得缓存对象对应的缓存栈，通过 ac 进行指定 */
        
		batchcount = ac->batchcount;
        if (!ac->touched && batchcount > BATCHREFILL_LIMIT) {
                /*
                 * If there was little recent activity on this cache, then
                 * perform only a partial refill.  Otherwise we could generate
                 * refill bouncing.
                 */
                batchcount = BATCHREFILL_LIMIT;
        }
        l3 = cachep->nodelists[node]; /* 获得缓存对象的 kmem_list3 链表，通过l3进行指定 */

        BUG_ON(ac->avail > 0 || !l3);
        spin_lock(&l3->list_lock);

        /* See if we can refill from the shared array */
        if (l3->shared && transfer_objects(ac, l3->shared, batchcount))
                goto alloc_done;

        while (batchcount > 0) {  /* 检测缓存栈的 batchount 是否大于 0，以此确认缓存栈是否在使用，如果小于 1，那么缓存栈不再使用*/
                struct list_head *entry;
                struct slab *slabp;
                /* Get slab alloc is to come from. */
                entry = l3->slabs_partial.next;
                if (entry == &l3->slabs_partial) {
                        l3->free_touched = 1;
                        entry = l3->slabs_free.next;
                        if (entry == &l3->slabs_free) /* 如果最终slabs_free上都没有找到可用的 slab */
                                goto must_grow; /*那么函数跳转到 must_grow分配新的 slab*/
                }
				/* 找到从 slabs_partial/slabs_free 里找到一个可用的 slab */
                slabp = list_entry(entry, struct slab, list);
                check_slabp(cachep, slabp);
                check_spinlock_acquired(cachep);

                /*
                 * The slab was either on partial or free list so
                 * there must be at least one object available for
                 * allocation.
                 */
                BUG_ON(slabp->inuse >= cachep->num);

            	/*
					 while 循环，只要确认缓存对象正在使用的缓存对象数量小于缓存对象总数，并且 batchcount 不为0，
					 那么进入循环，在循环中，函数不断从 slab 中将可用缓存对象加入到缓存栈里，并更新缓存栈的栈顶
				*/
                while (slabp->inuse < cachep->num && batchcount--) {
                        STATS_INC_ALLOCED(cachep);
                        STATS_INC_ACTIVE(cachep);
                        STATS_SET_HIGH(cachep);

                        ac->entry[ac->avail++] = slab_get_obj(cachep, slabp,
                                                            node);
                }
                check_slabp(cachep, slabp);
				/* 
					将 slab 从原先的链表中移除，
					如果此时slab 中没有可用的缓存对象，那么将 slab 插到slabs_full 链表中，
					如果此时 slab 中还有部分可用缓存对象，那么将 slab 插入到 slabs_partial 链表中. 
				*/
                /* move slabp to correct slabp list: */
                list_del(&slabp->list);
                if (slabp->free == BUFCTL_END)
                        list_add(&slabp->list, &l3->slabs_full);
                else
                        list_add(&slabp->list, &l3->slabs_partial);
        }

must_grow:
        l3->free_objects -= ac->avail;
alloc_done:
        spin_unlock(&l3->list_lock);

        if (unlikely(!ac->avail)) {
                int x;
                x = cache_grow(cachep, flags | GFP_THISNODE, node, NULL); /* 调用 cache_grow() 函数进行分配 新的 slab */

                /* cache_grow can reenable interrupts, then ac could change. */
                ac = cpu_cache_get(cachep);
                if (!x && ac->avail == 0)       /* no objects in sight? abort */
                        return NULL;
				/* 函数再次检测缓存栈是否等于 0，如果为真，那么函数重新从缓存栈上分配新的可用缓存栈 */
                if (!ac->avail)         /* objects refilled by interrupt? */ 
                        goto retry;
        }
        ac->touched = 1; /*当缓存栈被访问，那么缓存栈 的 touch 标记为 1*/ 
        return ac->entry[--ac->avail]; /* 从缓存栈中返回一个可用缓存对象.*/
}

```

#### cache_grow()

cache_grow() 函数用于扩充缓存对象的 slab。cachep 参数指向缓存对象; flags 参数 用于从 Buddy 分配器中分配内存的标志; nodeid 参数用于指明 NODE 信息; objp 用于 指向 slab.  



SLAB 分配器在为缓存对象分配新的 slab 时，首先从 Buddy 分配器中分配指定数量的 物理页，然后将这些物理内存进行组织，将其转换成 slab，转换之后在将 slab 对应的 物理页与缓存对象和 slab 进行绑定，最后初始化 slab 的 kmem_bufctl 数组。新的 slab 创建完毕之后，将其插入缓存对象的 slabs_free 链表。

```c
/*
 * Grow (by 1) the number of slabs within a cache.  This is called by
 * kmem_cache_alloc() when there are no active objs left in a cache.
 */
static int cache_grow(struct kmem_cache *cachep,
                gfp_t flags, int nodeid, void *objp)
{
        struct slab *slabp;
        size_t offset;
        gfp_t local_flags;
        struct kmem_list3 *l3;

        /*
         * Be lazy and only check for valid flags here,  keeping it out of the
         * critical path in kmem_cache_alloc().
         */
    	/* 对分配物理内存的标志进行检查，以此确认分配的物理页是否可以回收或者不可以回收 */
        BUG_ON(flags & GFP_SLAB_BUG_MASK);
        local_flags = flags & (GFP_CONSTRAINT_MASK|GFP_RECLAIM_MASK); 

        /* Take the l3 list lock to change the colour_next on this node */
        check_irq_off();
        l3 = cachep->nodelists[nodeid];     /* 获得缓存对象在指定 NODE 的 kmem_list3 数据结构*/
        spin_lock(&l3->list_lock);

        /* Get colour for the slab, and cal the next value. */
        offset = l3->colour_next; /* 获得了当前着色信息 */
        l3->colour_next++;
    /* 更新缓存对象的下一个着色信息 */
        if (l3->colour_next >= cachep->colour)
                l3->colour_next = 0;   /* 如果下一个着色信息超过了缓存支持的着色范围，那么将下一个着色信息设置为 0 */
        spin_unlock(&l3->list_lock); 

        offset *= cachep->colour_off; /* 计算着色预留的内存长度 */

        if (local_flags & __GFP_WAIT)
                local_irq_enable();

        /*
         * The test for missing atomic flag is performed here, rather than
         * the more obvious place, simply to reduce the critical path length
         * in kmem_cache_alloc(). If a caller is seriously mis-behaving they
         * will eventually be caught here (where it matters).
         */
        kmem_flagcheck(cachep, flags);

        /*
         * Get mem for the objs.  Attempt to allocate a physical page from
         * 'nodeid'.
         */
        if (!objp)
                objp = kmem_getpages(cachep, local_flags, nodeid); /* 从 Buddy 分配器中分配指定数量的物理内存，并获得物理内存对应的虚拟地址，存储中objp 变量里 */
        if (!objp)
                goto failed;

        /* Get slab management. */
    	/*
    		把新分配的内存组织成一个 slab,
    		无论 slab 采用 外部管理数据还是内部管理数据，函数都返回了 slab 管理数据对应的 struct slab 指针
    	*/
        slabp = alloc_slabmgmt(cachep, objp, offset,
                        local_flags & ~GFP_CONSTRAINT_MASK, nodeid); 
        if (!slabp)
                goto opps1;

        slab_map_pages(cachep, slabp, objp); /* 将从 Buddy 分配器分配的物理页 与缓存对象和 slab 进行绑定 */

        cache_init_objs(cachep, slabp); /* 初始化 slab 的 kmem_bufctl 数组 */

        if (local_flags & __GFP_WAIT)
                local_irq_disable();
        check_irq_off();
        spin_lock(&l3->list_lock);

        /* Make slab active. */
	    /*
	    	将 slab 加入到缓存对象的 slabs_free 链表里，
	    	并添加缓存对象的 kmem_list3 的 free_objects 可用对象数量.
	    */
        list_add_tail(&slabp->list, &(l3->slabs_free));
        STATS_INC_GROWN(cachep);
        l3->free_objects += cachep->num;
        spin_unlock(&l3->list_lock);
        return 1;
opps1:
        kmem_freepages(cachep, objp);
failed:
        if (local_flags & __GFP_WAIT)
                local_irq_disable();
        return 0;
}

```

##### kmem_getpages()

从 Buddy 分配器中分配指定数量的物理内存，并获得物理内存对应的虚拟地址，存储中objp 变量里。

```c
/*
 * Interface to system's page allocator. No need to hold the cache-lock.
 *
 * If we requested dmaable memory, we will get it. Even if we
 * did not request dmaable memory, we might get it, but that
 * would be relatively rare and ignorable.
 */
static void *kmem_getpages(struct kmem_cache *cachep, gfp_t flags, int nodeid)
{
        struct page *page;
        int nr_pages;
        int i;

#ifndef CONFIG_MMU
        /*
         * Nommu uses slab's for process anonymous memory allocations, and thus
         * requires __GFP_COMP to properly refcount higher order allocations
         */
        flags |= __GFP_COMP;
#endif

        flags |= cachep->gfpflags;
        if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
                flags |= __GFP_RECLAIMABLE;

        page = alloc_pages_node(nodeid, flags, cachep->gfporder);
        if (!page)
                return NULL;

        nr_pages = (1 << cachep->gfporder);
        if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
                add_zone_page_state(page_zone(page),
                        NR_SLAB_RECLAIMABLE, nr_pages);
        else
                add_zone_page_state(page_zone(page),
                        NR_SLAB_UNRECLAIMABLE, nr_pages);
        for (i = 0; i < nr_pages; i++)
                __SetPageSlab(page + i);
        return page_address(page);
}

```



##### alloc_slabmgmt()

把新分配的内存组织成一个 slab, 无论 slab 采用 外部管理数据还是内部管理数据，函数都返回了 slab 管理数据对应的 struct slab 指针。

```c
/*
 * Get the memory for a slab management obj.
 * For a slab cache when the slab descriptor is off-slab, slab descriptors
 * always come from malloc_sizes caches.  The slab descriptor cannot
 * come from the same cache which is getting created because,
 * when we are searching for an appropriate cache for these
 * descriptors in kmem_cache_create, we search through the malloc_sizes array.
 * If we are creating a malloc_sizes cache here it would not be visible to
 * kmem_find_general_cachep till the initialization is complete.
 * Hence we cannot have slabp_cache same as the original cache.
 */
static struct slab *alloc_slabmgmt(struct kmem_cache *cachep, void *objp,
                                   int colour_off, gfp_t local_flags,
                                   int nodeid)
{
        struct slab *slabp;

        if (OFF_SLAB(cachep)) {
                /* Slab management obj is off-slab. */
                slabp = kmem_cache_alloc_node(cachep->slabp_cache,
                                              local_flags, nodeid);
                if (!slabp)
                        return NULL;
        } else {
                slabp = objp + colour_off;
                colour_off += cachep->slab_size;
        }
        slabp->inuse = 0;
        slabp->colouroff = colour_off;
        slabp->s_mem = objp + colour_off;
        slabp->nodeid = nodeid;
        slabp->free = 0;
        return slabp;
}

```





##### slab_map_pages()

将从 Buddy 分配器分配的物理页 与缓存对象和 slab 进行绑定。

```c
/*
 * Grow (by 1) the number of slabs within a cache.  This is called by
 * kmem_cache_alloc() when there are no active objs left in a cache.
 */
static int cache_grow(struct kmem_cache *cachep,
                gfp_t flags, int nodeid, void *objp)
{
        struct slab *slabp;
        size_t offset;
        gfp_t local_flags;
        struct kmem_list3 *l3;

        /*
         * Be lazy and only check for valid flags here,  keeping it out of the
         * critical path in kmem_cache_alloc().
         */
        BUG_ON(flags & GFP_SLAB_BUG_MASK);
        local_flags = flags & (GFP_CONSTRAINT_MASK|GFP_RECLAIM_MASK);

        /* Take the l3 list lock to change the colour_next on this node */
        check_irq_off();
        l3 = cachep->nodelists[nodeid];
        spin_lock(&l3->list_lock);

        /* Get colour for the slab, and cal the next value. */
        offset = l3->colour_next;
        l3->colour_next++;
        if (l3->colour_next >= cachep->colour)
                l3->colour_next = 0;
        spin_unlock(&l3->list_lock);

        offset *= cachep->colour_off;

        if (local_flags & __GFP_WAIT)
                local_irq_enable();

        /*
         * The test for missing atomic flag is performed here, rather than
         * the more obvious place, simply to reduce the critical path length
         * in kmem_cache_alloc(). If a caller is seriously mis-behaving they
         * will eventually be caught here (where it matters).
         */
        kmem_flagcheck(cachep, flags);

        /*
         * Get mem for the objs.  Attempt to allocate a physical page from
         * 'nodeid'.
         */
        if (!objp)
                objp = kmem_getpages(cachep, local_flags, nodeid);
        if (!objp)
                goto failed;

        /* Get slab management. */
        slabp = alloc_slabmgmt(cachep, objp, offset,
                        local_flags & ~GFP_CONSTRAINT_MASK, nodeid);
        if (!slabp)
                goto opps1;

        slab_map_pages(cachep, slabp, objp);

        cache_init_objs(cachep, slabp);

        if (local_flags & __GFP_WAIT)
                local_irq_disable();
        check_irq_off();
        spin_lock(&l3->list_lock);

        /* Make slab active. */
        list_add_tail(&slabp->list, &(l3->slabs_free));
        STATS_INC_GROWN(cachep);
        l3->free_objects += cachep->num;
        spin_unlock(&l3->list_lock);
        return 1;
opps1:
        kmem_freepages(cachep, objp);
failed:
        if (local_flags & __GFP_WAIT)
                local_irq_disable();
        return 0;
}

```





##### cache_init_objs()

初始化 slab 的 kmem_bufctl 数组。

```c
static void cache_init_objs(struct kmem_cache *cachep,
                            struct slab *slabp)
{
        int i;

        for (i = 0; i < cachep->num; i++) {
                void *objp = index_to_obj(cachep, slabp, i);
#if DEBUG
                /* need to poison the objs? */
                if (cachep->flags & SLAB_POISON)
                        poison_obj(cachep, objp, POISON_FREE);
                if (cachep->flags & SLAB_STORE_USER)
                        *dbg_userword(cachep, objp) = NULL;

                if (cachep->flags & SLAB_RED_ZONE) {
                        *dbg_redzone1(cachep, objp) = RED_INACTIVE;
                        *dbg_redzone2(cachep, objp) = RED_INACTIVE;
                }
                /*
                 * Constructors are not allowed to allocate memory from the same
                 * cache which they are a constructor for.  Otherwise, deadlock.
                 * They must also be threaded.
                 */
                if (cachep->ctor && !(cachep->flags & SLAB_POISON))
                        cachep->ctor(objp + obj_offset(cachep));

                if (cachep->flags & SLAB_RED_ZONE) {
                        if (*dbg_redzone2(cachep, objp) != RED_INACTIVE)
                                slab_error(cachep, "constructor overwrote the"
                                           " end of an object");
                        if (*dbg_redzone1(cachep, objp) != RED_INACTIVE)
                                slab_error(cachep, "constructor overwrote the"
                                           " start of an object");
                }
                if ((cachep->buffer_size % PAGE_SIZE) == 0 &&
                            OFF_SLAB(cachep) && cachep->flags & SLAB_POISON)
                        kernel_map_pages(virt_to_page(objp),
                                         cachep->buffer_size / PAGE_SIZE, 0);
#else
                if (cachep->ctor)
                        cachep->ctor(objp);
#endif
                slab_bufctl(slabp)[i] = i + 1;
        }
        slab_bufctl(slabp)[i - 1] = BUFCTL_END;
}

```





## kmem_cache_free



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





## kmem_cache_destroy



```c

/**
 * kmem_cache_destroy - delete a cache
 * @cachep: the cache to destroy
 *
 * Remove a &struct kmem_cache object from the slab cache.
 *
 * It is expected this function will be called by a module when it is
 * unloaded.  This will remove the cache completely, and avoid a duplicate
 * cache being allocated each time a module is loaded and unloaded, if the
 * module doesn't have persistent in-kernel storage across loads and unloads.
 *
 * The cache must be empty before calling this function.
 *
 * The caller must guarantee that noone will allocate memory from the cache
 * during the kmem_cache_destroy().
 */

void kmem_cache_destroy(struct kmem_cache *cachep)
{
        BUG_ON(!cachep || in_interrupt());

        /* Find the cache in the chain of caches. */
        get_online_cpus();
        mutex_lock(&cache_chain_mutex);
        /*
         * the chain is never empty, cache_cache is never destroyed
         */
        list_del(&cachep->next);
        if (__cache_shrink(cachep)) {
                slab_error(cachep, "Can't free all objects");
                list_add(&cachep->next, &cache_chain);
                mutex_unlock(&cache_chain_mutex);
                put_online_cpus();
                return;
        }

        if (unlikely(cachep->flags & SLAB_DESTROY_BY_RCU))
                rcu_barrier();

        __kmem_cache_destroy(cachep);
        mutex_unlock(&cache_chain_mutex);
        put_online_cpus();
}
EXPORT_SYMBOL(kmem_cache_destroy);

```







## 通用内存对象管理举例 ： zalloc/kmalloc和kfree
		malloc_sizes 数组



## 专用内存对象管理举例： task_struct





## 参考资料

[SLAB Allocator](https://biscuitos.github.io/blog/HISTORY-SLAB/#D030063)


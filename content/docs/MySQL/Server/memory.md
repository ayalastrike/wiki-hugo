在MySQL中，为了高效，Server层也有自己的内存管理。

首先先介绍最简单的单链表内存管理，然后再介绍基于此优化的MEM_ROOT内存管理。

# 单链表内存管理

在server层中，对于字符集处理，使用单链表的内存管理来处理内存资源。

正如Monty在comments中写的：

{{< hint info >}}

*Wed Jun 21 01:34:04 1989  Monty  (monty at monty)*

** Added two new malloc-functions: my_once_alloc() and*
 *my_once_free(). These give easyer and quicker startup.*

{{</hint>}}

分配内存、释放全部内存由my_once_alloc、my_once_free负责，my_once_strdup和my_once_memdup则是wrapper，内部封装了alloc+memcpy。

alloc策略如下：

1. 采用单向链表的方式将多个malloc的内存块管理起来
2. 每个内存块采用malloc_chunk + raw的方式，即内存控制块+实际数据使用块两部分
3. raw部分如果剩余可用，则直接从其中划分，减少malloc开销
4. 如果申请量小且现有的left小于4k，则对齐到4k

从这里可以看出，#3，#4一方面可以减少malloc的系统调用次数和碎片，同时，尽量按照4k对齐申请。

另外，可以通过my_flags设定分配属性：

- MY_FAE /* Fatal if any error */ 内存分配失败就退出整个进程
- MY_WME /* Write message on error */ 记录到日志中
- MY_ZEROFILL /* Fill array with zero */ 分配内存后初始化为0

相关代码和示例如下：

**my_global.h**

````
#define MY_ALIGN(A,L)   (((A) + (L) - 1) & ~((L) - 1))
#define ALIGN_SIZE(A)   MY_ALIGN((A),sizeof(double))
````

**my_static.c**

````
/* from my_malloc */
USED_MEM* my_once_root_block=0;         /* pointer to first block */
uint      my_once_extra=ONCE_ALLOC_INIT;    /* Memory to alloc / block */
````

**my_alloc.h**

````
typedef struct st_used_mem
{                  /* struct for once_alloc (block) */
  struct st_used_mem *next;    /* Next block in use */
  unsigned int  left;          /* memory left in block  */
  unsigned int  size;          /* size of block */
} USED_MEM;
````

**示例**

````
accquire size = 5000
malloc from new chunk size = 5016
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fcb3a801000 next = 0x0
 
accquire size = 500
malloc from new chunk size = 4088
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fcb3a801000 next = 0x7fcb3b000000
USED_MEM size =     4088 left =     3568 cur = 0x7fcb3b000000 next = 0x0
 
accquire size = 100
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fcb3a801000 next = 0x7fcb3b000000
USED_MEM size =     4088 left =     3464 cur = 0x7fcb3b000000 next = 0x0
 
accquire size = 8500
malloc from new chunk size = 8520
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fcb3a801000 next = 0x7fcb3b000000
USED_MEM size =     4088 left =     3464 cur = 0x7fcb3b000000 next = 0x7fcb3a001000
USED_MEM size =     8520 left =        0 cur = 0x7fcb3a001000 next = 0x0
 
accquire size = 400
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fb5d1000000 next = 0x7fb5d1800000
USED_MEM size =     4088 left =     3064 cur = 0x7fb5d1800000 next = 0x7fb5d1801000
USED_MEM size =     8520 left =        0 cur = 0x7fb5d1801000 next = 0x0
 
accquire size = 3500
malloc from new chunk size = 3520
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fdced001000 next = 0x7fdced002400
USED_MEM size =     4088 left =     3064 cur = 0x7fdced002400 next = 0x7fdced003400
USED_MEM size =     8520 left =        0 cur = 0x7fdced003400 next = 0x7fdced005600
USED_MEM size =     3520 left =        0 cur = 0x7fdced005600 next = 0x0
 
accquire size = 3000
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fdced001000 next = 0x7fdced002400
USED_MEM size =     4088 left =       64 cur = 0x7fdced002400 next = 0x7fdced003400
USED_MEM size =     8520 left =        0 cur = 0x7fdced003400 next = 0x7fdced005600
USED_MEM size =     3520 left =        0 cur = 0x7fdced005600 next = 0x0
 
accquire size = 100
malloc from new chunk size = 4088
-------------------------------
USED_MEM size =     5016 left =        0 cur = 0x7fdced001000 next = 0x7fdced002400
USED_MEM size =     4088 left =       64 cur = 0x7fdced002400 next = 0x7fdced003400
USED_MEM size =     8520 left =        0 cur = 0x7fdced003400 next = 0x7fdced005600
USED_MEM size =     3520 left =        0 cur = 0x7fdced005600 next = 0x7fdced006400
USED_MEM size =     4088 left =     3968 cur = 0x7fdced006400 next = 0x0
````

相关文件：

my_global.h

my_static.h

my_static.c

my_once.c

# MEM_ROOT

MEM_ROOT封装了USED_MEM对象，在 MySQL的Server层中广泛使用。MEM_ROOT作为类中的一个成员变量，伴随对象的整个生命周期，而且，不同的MEM_ROOT之间互相没有影响。使用MEM_ROOT的类有：THD、String、TABLE、TABLE_SHARE、Query_arena、st_transactions等等。

在MEM_ROOT中，分配内存的单元是block（复用了USED_MEM对象），并优化了以下方面：

1. 初始化时可以预分配内存
2. 将一条单链表拆分为2条单链表（free+used），提升了查找效率
3. 限制内存申请上限，并提供是否关闭错误信息的开关
4. 可以同时申请多块内存
5. 提供了多种回收、释放策略
6. 采用启发式分配算法优化malloc
7. 对free链表中的小块和多次不满足的小块申请，进行了优化

首先看一下MEM_ROOT的数据结构：

````
typedef struct st_mem_root
{
  USED_MEM *free;                  // 空闲块链表（有可用空间）
  USED_MEM *used;                  // 满块链表
  USED_MEM *pre_alloc;             // 预分配块链表
 
  size_t min_malloc;               // 如果内存块过小，则从free移到used
  size_t block_size;               // 初始化的块大小（init_alloc_root指定的大小）
  unsigned int block_num;          // malloc分配的内存块计数
  unsigned int first_block_usage;  // free链表中的首节点计数，如果某次申请时超过10次且申请的大小>4k，则将该内存块从free移到used
  size_t max_capacity;              // 申请总量（allocated_size+size）限制，0为不限制
  size_t allocated_size;            // 已分配的内存总量
 
  void (*error_handler)(void);      // 错误处理函数
} MEM_ROOT;
````

## 内存管理

### 初始化

init_alloc_root函数用于初始化MEM_ROOT对象，如果指定了pre_alloc_size，则malloc一块内存并指向到free和pre_alloc链表

### 分配

alloc_root函数用于进行内存的分配管理，处理流程如下：

1. 查看free链表，如果可用空间不足以容纳申请的的大小，且查找次数次数超过10、可用空间小于4k，则将该内存块从free移到used
2. 如果free链表中没有合适的可用块，如果没有合适的块，则malloc，malloc的大小为max(size, block_size)，其中block_size为初始的block_size * block_num的2次幂
3. 如果free链表中存在合适的可用块，则从该块分配，如果分配后剩余空间过小（min_malloc），则将该内存块从free移到used

详细分析日志：

````
-----------------------------------------------------
init_alloc_root block_size = 1024, pre_alloc_size = 1024
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 0 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =     1024 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =     1024 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 1024 length = 56 mem_root->first_block_usage = 0
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 1 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      968 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      968 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 968 length = 56 mem_root->first_block_usage = 1
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 2 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      912 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      912 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 912 length = 56 mem_root->first_block_usage = 2
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 3 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      856 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      856 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 856 length = 56 mem_root->first_block_usage = 3
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 4 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      800 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      800 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 800 length = 56 mem_root->first_block_usage = 4
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 5 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      744 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      744 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 744 length = 56 mem_root->first_block_usage = 5
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 6 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      688 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      688 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 688 length = 56 mem_root->first_block_usage = 6
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 7 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      632 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      632 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 632 length = 56 mem_root->first_block_usage = 7
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 8 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      576 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      576 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 576 length = 56 mem_root->first_block_usage = 8
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 9 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      520 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      520 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 520 length = 56 mem_root->first_block_usage = 9
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 10 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      464 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      464 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 464 length = 56 mem_root->first_block_usage = 10
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 11 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      408 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      408 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 50
$$$ prev->left = 408 length = 56 mem_root->first_block_usage = 11
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 4 first_block_usage = 12 max_capacity = 0 allocated_size = 1040
-------------- mem_root->used -----------------------
-------------- mem_root->free -----------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 2000
$$$ prev->left = 352 length = 2000 mem_root->first_block_usage = 12
$$$ remove from free ==> used
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 5 first_block_usage = 0 max_capacity = 0 allocated_size = 3056
-------------- mem_root->used -----------------------
USED_MEM size =     2016 left =        0 cur = 0x7fe99d800000 next = 0x7fe99d000000
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->free -----------------------
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 500
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 6 first_block_usage = 0 max_capacity = 0 allocated_size = 4048
-------------- mem_root->used -----------------------
USED_MEM size =     2016 left =        0 cur = 0x7fe99d800000 next = 0x7fe99d000000
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->free -----------------------
USED_MEM size =      992 left =      472 cur = 0x7fe99cd02480 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 5000
$$$ prev->left = 472 length = 5000 mem_root->first_block_usage = 0
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 7 first_block_usage = 0 max_capacity = 0 allocated_size = 9064
-------------- mem_root->used -----------------------
USED_MEM size =     5016 left =        0 cur = 0x7fe99e000000 next = 0x7fe99d800000
USED_MEM size =     2016 left =        0 cur = 0x7fe99d800000 next = 0x7fe99d000000
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->free -----------------------
USED_MEM size =      992 left =      472 cur = 0x7fe99cd02480 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 3000
$$$ prev->left = 472 length = 3000 mem_root->first_block_usage = 0
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 8 first_block_usage = 0 max_capacity = 0 allocated_size = 12080
-------------- mem_root->used -----------------------
USED_MEM size =     3016 left =        0 cur = 0x7fe99e001400 next = 0x7fe99e000000
USED_MEM size =     5016 left =        0 cur = 0x7fe99e000000 next = 0x7fe99d800000
USED_MEM size =     2016 left =        0 cur = 0x7fe99d800000 next = 0x7fe99d000000
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->free -----------------------
USED_MEM size =      992 left =      472 cur = 0x7fe99cd02480 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
 
alloc_root length = 500
$$$ prev->left = 472 length = 504 mem_root->first_block_usage = 0
-------------- mem_root -----------------------------
min_malloc = 32, block_size = 992 block_num = 9 first_block_usage = 2 max_capacity = 0 allocated_size = 14064
-------------- mem_root->used -----------------------
USED_MEM size =     3016 left =        0 cur = 0x7fe99e001400 next = 0x7fe99e000000
USED_MEM size =     5016 left =        0 cur = 0x7fe99e000000 next = 0x7fe99d800000
USED_MEM size =     2016 left =        0 cur = 0x7fe99d800000 next = 0x7fe99d000000
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
-------------- mem_root->free -----------------------
USED_MEM size =      992 left =      472 cur = 0x7fe99cd02480 next = 0x7fe99d002000
USED_MEM size =     1984 left =     1464 cur = 0x7fe99d002000 next = 0x0
-------------- mem_root->pre_alloc ------------------
USED_MEM size =     1040 left =      352 cur = 0x7fe99d000000 next = 0x0
````

### 回收释放

free_root函数用于进行内存的回收是否，提供了3种回收策略：

- MY_MARK_BLOCKS_FREE：free、used只是标记（指针重置），并不归还给操作系统
- MY_KEEP_PREALLOC：是否保留pre_alloc
- 释放全部free、used、pre_alloc

空间利用率上来讲，MEM_ROOT的内存管理方式在每个block 上连续分配，内部碎片基本在每个block的尾部，由 min_malloc 成员变量和参数 ALLOC_MAX_BLOCK_USAGE_BEFORE_DROP，ALLOC_MAX_BLOCK_TO_DROP 共同决定和控制，但是 min_malloc 的值是在代码中写死的，有点不够灵活，可以考虑写成可配置的，同时如果写超过申请长度的空间，就很有可能会覆盖后面的数据，比较危险。但相比 PG 的内存上下文，空间利用率肯定是会高很多的。

从时间利用率上来讲，不提供 free 一个 Block 的操作，基本上一整个 MEM_ROOT 使用完毕才会全部归还给操作系统。

### 其他函数

strdup_root、safe_strdup_root、strmake_root、memdup_root用于字符串和内存数据的空间申请和复制

set_memroot_max_capacity：设置max_capacity

set_memroot_error_reporting：申请不到内存时，是否关闭报错

multi_alloc_root：申请多块内存

reset_root_defaults：重置并复用MEM_ROOT，如果pre_alloc_size不够，则重新分配（free+malloc）

clear_alloc_root：重置MEM_ROOT中的free、used、pre_alloc指向

alloc_root_inited：判断MEM_ROOT是否初始化

claim_root：PSI使用

相关文件：

my_sys.h

my_alloc.h

my_alloc.c
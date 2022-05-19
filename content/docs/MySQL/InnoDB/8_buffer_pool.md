---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

基于磁盘的数据库系统通常都是用缓冲池（buffer pool）技术来弥补CPU和磁盘之间的速度鸿沟。

通过以下机制来达成目标：

**spatial control 空间控制**

- where to write pages on disk.
- the goal is to keep pages that are used together often as physically close together as possible on disk.

**temporal control 时间控制**

- when to read pages into memory, and when to write them to disk.
- the goal is minimize the number of stalls from having to read data from disk.

# 概述

## buffer pool

InnoDB是基于磁盘存储的存储引擎，磁盘中的记录以页为单位进行管理。

简单来说，buffer pool就是存储引擎自管理的一块内存区域，缓存频繁访问的数据，从而减少读取写入磁盘对于数据库的影响。

{{< hint danger>}}

这里预留一个问题，为什么不依赖操作系统或者文件系统提供的page cache机制，或者MMAP来进行缓存？

{{</hint>}}

当数据库读取页时，首先分配一个页的控制块，然后通过该控制块将页"FIX"住，然后发起磁盘IO从磁盘将实际的页读入buffer pool，访问完数据后再”UNFIX“该控制块。下次再读取该页时，先从buffer pool中查找，否则再走上面的流程从磁盘读取。

当数据库修改页时，首先通过上面的机制保证页在buffer pool中后，进行"FIX“，修改页内的数据后，”UNFIX“，此时前台的数据修改完成。后台再以一定的频率（至少在checkpoint时）同步/异步刷回磁盘（原地写/追加写）。

从这里可以看出，读取和写入都为了最大化的利用缓存，对于读取，尽量延长数据保持在内存中的时间，而对于写入，则delay写回磁盘的时机。

对于InnoDB来说，buffer pool中缓存的页有6种类型：索引页、数据页、undo页、change buffer、AHI、lock info，而不仅仅是缓存数据页和索引页。如下图所示：

![InnoDB_buffer_pool_page_type](/InnoDB_buffer_pool_page_type.png)

其中的data page、index page、change buffer page都是需要异步持久化到外存的，undo page分为两种情况：已提交的事务的undo page已经在事务提交时保证其持久化，未提交的事务的undo page可能刷入磁盘也可能没有。

而AHI和lock info是不持久化的，生命周期仅限于运行时。锁也是在事务的生命周期里产生的，在crash recovery时会根据事务信息重建锁状态，所以锁不需要持久化，另外，每个锁信息通常是在trx_lock_t.lock_heap中分配。而AHI是根据访问的数据行动态建立的hashtable，无需保留。

buffer pool通过page table（也称为page hashtable，其Key是space_id, page_no，value是page在内存中的地址）可以快速找到已经被读入内存的数据页，而不用线性遍历LRU List去查找。这里的page table不是AHI，AHI是为了减少B+树的扫描，而page hash是为了避免扫描链表（LRU List）。

AHI在B+tree一章详细介绍。

## 页和控制块

前面讲过，buffer pool的资源单位为页。为了管理页，页在缓冲池中按照控制块（buf_block_t）为单元进行管理（称为PCB（Page Control Block）），其中的页元信息用buf_page_t表示，数据（数据页）在内存中对应的内存空间用frame表示，指向实际的内存数据区域。从这里可以看出，InnoDB参考了Jim Gray的命名方式。

控制块和其中的元信息、数据页之间的关系如下图所示：

![InnoDB_buffer_pool_page_&_block](/InnoDB_buffer_pool_page_&_block.png)

从上面的页和控制块布局可以看出，buf_block_t中的第一个字段（必须）就是buf_page_t，以保证page table指向的buf_block_t和buf_page_t两种类型的指针可以相互转换。block→frame指向真正存放数据的数据页。

buf_page_t实际上存放的是数据页的元信息：page size、state、oldest_modification、newest_modification、access_time等等。如果某个压缩页被解压了，解压页的数据指针是存储在buf_block_t的frame字段里。

如果对表进行了压缩，则对应的数据页称为压缩页，如果需要从压缩页中读取数据，则压缩页需要先解压，形成解压页，解压页为16KB。压缩页的大小是在建表的时候指定，目前支持16K，8K，4K，2K，1K。即使压缩页大小设为16K，在blob/varchar/text的类型中也有一定好处。假设指定的压缩页大小为4K，如果有个数据页无法被压缩到4K以下，则需要做B-tree分裂操作，这是一个比较耗时的操作。正常情况下，Buffer Pool中会把压缩和解压页都缓存起来，当Free List不够时，按照系统当前的实际负载来决定淘汰策略。如果系统瓶颈在IO上，则只驱逐解压页，压缩页依然在Buffer Pool中，否则解压页和压缩页都被驱逐。

## buffer chunk

从MySQL 5.7开始，支持动态调整buffer pool的大小（resizing），由此引入了buffer chunk的概念，可以简单的将buffer chunk理解为一块花布，里面是page里的数据网格，每次调整buffer pool按照花布为粒度进行批量处理。

chun size的大小可以通过[innodb_buffer_pool_chunk_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_chunk_size)设定（默认128M），buffer chunk的数量=缓冲池instance大小/chunk size，同样，为了避免性能问题，buffer chunk不应该超过1000。

一个buffer chunk的结构如下：

![InnoDB_buffer_pool_buffer_chunk](/InnoDB_buffer_pool_buffer_chunk.png)

buffer chunk的初始化只在InooDB启动或者resizing时进行（buf_chunk_init），向memory allocator申请chunk size大小的内存（mmap），并按照页的大小串到free链表上，即把buffer pool打好格子，格子内划分为一组buf_block_t。通过遍历一个chunk可以访问大多数的数据页（比如查找压缩页 buf_chunk_contains_zip），有两种状态的数据页除外：没有被解压的压缩页（BUF_BLOCK_ZIP_PAGE）以及被修改过且解压页已经被驱逐的压缩页（BUF_BLOCK_ZIP_DIRTY）。

{{< hint info>}}

用mmap分配的内存都是虚存，在top命令中占用VIRT这一列，而不是RES这一列，只有相应的内存被真正使用到了，才会被统计到RES中，提高内存使用率。这样是为什么常常看到MySQL一启动就被分配了很多的VIRT，而RES却是慢慢涨上来的原因。

os_mem_alloc_large()

​	mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | OS_MAP_ANON, -1, 0)

{{</hint>}}

{{< hint danger>}}

这里留个小问题：在数据库这种对数据的一致性要求的方案中应激励避免使用MMAP，这里为什么chunk的申请要通过mmap来进行，不会有问题吗？

{{</hint>}}

{{< hint info>}}

NUMA的内存分配策略（[mbind](https://man7.org/linux/man-pages/man2/mbind.2.html)）也在这里控制。

{{</hint>}}

## buffer pool中的页链表

buffer pool中的内存空间以页的倍数来申请，即以页为单位进行管理。

在InnoDB中，页通常通过free、LRU和flush这三个逻辑链表组织并管理起来。free链表用于保存未使用的页，当free链表中的页用尽后，则根据LRU算法淘汰已经使用的页，最频繁使用的页在LRU链表的头部，最少使用的页在LRU链表的尾部。

在InnoDB的LRU链表设计中（LRU-K），通过加入midpoint位点划分出老区（历史队列）和新区（缓冲区）。新读取到的页，虽然是最近访问的页，并不直接放到LRU链表的头部，而是放到midpoint的（默认3/8）位置，这种算法在InnoDB中称为midpoint insertion strategy。这样做而不采用朴素的LRU算法的原因，是因为某些低频但是数据量大的SQL操作或者预读可能会将缓冲池中频繁使用的页刷出（sequential flooding），从而影响缓冲池的效率。

此外，有些页不属于数据页或索引页，从free链表申请，但使用完毕后，并不放入LRU链表，比如AHI，lock info。

例如在下面的例子中，buffer pool - free - LRU = 65528 - 43916 - 21603 = 9，还缺少9个页，这些页其实分配给了AHI：

````
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 1126694912
Dictionary memory allocated 941200
Buffer pool size   65528
Free buffers       43916
Database pages     21603
Old database pages 7810
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 202, created 21401, written 14543
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 21603, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
````

这里显示AHI占用了9个页

````
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 276671, used cells 121, node heap has 1 buffer(s)
Hash table size 276671, used cells 585, node heap has 2 buffer(s)
Hash table size 276671, used cells 21, node heap has 1 buffer(s)
Hash table size 276671, used cells 5, node heap has 1 buffer(s)
Hash table size 276671, used cells 13, node heap has 1 buffer(s)
Hash table size 276671, used cells 10, node heap has 1 buffer(s)
Hash table size 276671, used cells 13, node heap has 1 buffer(s)
Hash table size 276671, used cells 15, node heap has 1 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
````

buffer pool中的页不仅用于读取，也可能被修改，被修改的页称为脏页（dirty page），脏页肯定在LRU链表中，同时也在flush链表中。LRU链表用于管理缓冲池中页的可用性，flush链表用于管理页的刷回，两者互不影响。

下图是free链表，LRU链表，flush链表之间的关系：

![InnoDB_buffer_pool free_LRU_flush](/InnoDB_buffer_pool free_LRU_flush.png)

RU链表中还包含没有被解压的压缩页，这些压缩页刚从磁盘读取出来，还没来的及被解压。

关于LRU算法的详情下面再讨论。

另外，如果启用页压缩，则还使用了以下的页链表：

- Unzip LRU List：这个链表中存储的数据页都是解压页，也就是说，这个数据页是从一个压缩页通过解压而来的。
- Zip Free：压缩页有不同的大小，比如8K，4K，InnoDB使用了类似内存管理的伙伴系统来管理压缩页（binary buddy allocator算法）。Zip Free可以理解为由5个链表构成的一个二维数组，每个链表分别存储了对应大小的内存碎片，例如8K的链表里存储的都是8K的碎片，如果新读入一个8K的页面，首先从这个链表中查找，如果有则直接返回，如果没有则从16K的链表中分裂出两个8K的块，一个被使用，另外一个放入8K链表中。
- Zip Clean List：这个链表只在Debug模式下有，主要是存储没有被解压的压缩页。这些压缩页刚刚从磁盘读取出来，还没来的及被解压，一旦被解压后，就从此链表中删除，然后加入到Unzip LRU List中。

阿里云RDS MySQL 5.6增加了Quick List：使用带Hint的SQL查询语句，可以把所有这个查询的用到的数据页加入到Quick List中，一旦这个语句结束，就把这个数据页淘汰，主要作用是避免LRU List被全表扫描污染。

## buffer pool instance

在MySQL 5.5中，为了提高buffer pool的并发性能，将pool划分为多个instance，一页只会存在于一个instance中（通过page_id（space_id+page_no）hash后按instance_num取模），这样，buf_pool_t表示了一个buffer pool instance。各个instance之间没有竞争关系，之前介绍的buffer pool中的各个逻辑链表、buffer chunks，以及mutex都在每个instance中。

特别的，instace不能过多也不能过小，否则会有性能问题：当buffer pool小于1GB时，只会有1个instance，而最多有64个（使用6 bit表示instance no）。

# buffer pool的管理

## buffer pool的组织

从上面的介绍可以看出，buffer pool把页作为基本单位，并通过block作为控制块，然后再组合成一个chunk，多个chunk组合成一个instance。因此，buffer pool的层次结构如下图所示：

![InnoDB_buffer_pool_hierarchical](/InnoDB_buffer_pool_hierarchical.png)

## 页的状态

页的状态一共有8种。理解这8种状态和其相互转换的关系，对buffer pool的页管理机制会有更清晰直观的理解。

页的状态流转图如下：

![InnoDB_buffer_pool_buf_page_state](/InnoDB_buffer_pool_buf_page_state.png)


下面依次解释一下这8种状态：

- BUF_BLOCK_NOT_USED：处于free链表中的数据页，都是此状态。此状态为长态。

- BUF_BLOCK_READY_FOR_USE：当从free链表中获取一个空闲的数据时，状态会从BUF_BLOCK_NOT_USED变为BUF_BLOCK_READY_FOR_USE（buf_LRU_get_free_block)。这个状态为暂态，处于这个状态的数据页不处于任何逻辑链表中。

- BUF_BLOCK_FILE_PAGE：也称为buffered file page，正常被使用的数据页都是这种状态。LRU链表中的大部分数据页都是这种状态。此状态为长态。

- BUF_BLOCK_MEMORY：也称为memory object，缓冲池中的非用户数据页（存储系统信息，例如InnoDB行锁，自适应哈希索引以及压缩页的数据等）被标记为BUF_BLOCK_MEMORY。处于这个状态的数据页不处于任何逻辑链表中。此状态为长态。

- BUF_BLOCK_REMOVE_HASH：当加入free链表之前，需要先从page hash移除。这种状态就表示此页面已经从page hash移除，但是还没被加入到free链表时的中间状态。

- BUF_BLOCK_POOL_WATCH：被哨兵看管的数据页。这种类型的page是提供给purge线程用的。InnoDB为了实现MVCC，需要把之前的数据记录在undo log中，如果没有读请求再需要它，就可以通过purge线程删除。换句话说，purge线程需要知道某些数据页是否被读取，实现的方式（buf_pool_watch_set）是首先查看page hash，看看这个数据页是否已经被读入，如果没有读入，则获取一个BUF_BLOCK_POOL_WATCH类型的哨兵数据页控制体，同时加入page_hash中（但其中并没有真正的数据：buf_block_t::frame为空），并把其类型置为BUF_BLOCK_ZIP_PAGE（表示已经被使用了，其他purge线程就不会用到这个控制体了）。如果查看page hash后发现有这个数据页，只需要判断控制体在内存中的地址是否属于Buffer Chunks即表示对应数据页已经被其他线程读入（buf_pool_watch_occurred）。另一方面，如果用户线程需要这个数据页，先查看page hash是否为BUF_BLOCK_POOL_WATCH类型的数据页，如果是则回收，从free链表（即在buffer chunks中）分配一个空闲的控制体，填入数据。这里的核心思想就是通过控制体在内存中的地址来确定数据页是否还在被使用。此状态为暂态。

  {{< hint info>}}

  每个缓冲池instance配置配置[purge thread](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_purge_threads)（默认：4）个哨兵数据页控制体，通过buffer_pool->watch数组管理。

  {{</hint>}}

- BUF_BLOCK_ZIP_PAGE：也称为clean compressed page（不全面，未包含purge语音）。当从磁盘读取压缩页时，先通过malloc分配一个临时的buf_page_t，然后从伙伴系统中分配出压缩页存储的空间，把磁盘中读取的压缩数据存入其中，然后把这个临时的buf_page_t标记为BUF_BLOCK_ZIP_PAGE状态（buf_page_init_for_read），当这个压缩页被解压时，将页状态修改为BUF_BLOCK_FILE_PAGE，并加入LRU List和Unzip LRU List（buf_page_get_gen）。如果一个压缩页对应的解压页被evict，但是需要保留这个压缩页并且该压缩页不是脏页，则这个压缩页被标记为BUF_BLOCK_ZIP_PAGE（buf_LRU_free_page）。所以正常情况下，处于BUF_BLOCK_ZIP_PAGE状态的不会很多。前面两种被标记为BUF_BLOCK_ZIP_PAGE的压缩页都在LRU list中。另外，上面提到的WATCH状态可以得知，如果被某个purge线程使用了，也会被标记为BUF_BLOCK_ZIP_PAGE。

- BUF_BLOCK_ZIP_DIRTY：即在flush链表中的compressed page。如果一个压缩页对应的解压页被evict，但是需要保留这个压缩页并且该压缩页是脏页，则被标记为BUF_BLOCK_ZIP_DIRTY（buf_LRU_free_page）。如果该压缩页随后又被解压，则状态会变为BUF_BLOCK_FILE_PAGE。因此BUF_BLOCK_ZIP_DIRTY也是暂态。这种类型的数据页都在flush list中。

整体来说，大部分的数据页都处于BUF_BLOCK_NOT_USED状态（在free链表中）和BUF_BLOCK_FILE_PAGE状态（大部分处于LRU链表中，LRU链表中还包含除被purge线程标记的BUF_BLOCK_ZIP_PAGE状态的数据页），少部分处于BUF_BLOCK_MEMORY状态，极少数处于其他状态。前三种状态的数据页都不在buffer chunks上，对应的控制体都是临时分配的，这三种状态也称为invalid state（buf_block_state_valid）。

## LRU算法

InnoDB有两种LRU算法：

- 朴素的LRU算法：当LRU链表的长度小于BUF_LRU_OLD_MIN_LEN（512）时启用，即读到的buf_page放入LRU链表的头部，即不区分冷区和热区。
- LRU-K（midpoint insertion strategy）：当链表长度大于BUF_LRU_OLD_MIN_LEN时启用，区分冷热区。

LRU链表的布局如下图所示：

![InnoDB_buffer_pool_LRU_algo](/InnoDB_buffer_pool_LRU_algo.png)

old的位置由buf_pool_t中的变量LRU_old（点位）、LRU_old_len（OLD区长度）进行维护。随着链表的不断增大，InnoDB还需要维护这些变量（buf_LRU_old_adjust_len），目的就是将buf_pool→LRU_old指向LRU链表长度的3/8处。同时，引入了BUF_LRU_OLD_TOLERANCE（20）来防止链表调整过频繁，只有查过该barrier才进行调整。其中，buf_page->old表示LRU中的页（buf_page->in_LRU_list=true）是处于OLD/NEW区。

## LRU链表的维护

当读取页时，需要对LRU链表进行维护（buf_page_peek_if_too_old）。根据LRU算法，数据库总是希望经常访问的页保持在LRU中。当LRU List链表大于512（BUF_LRU_OLD_MIN_LEN）时，在逻辑上被分为两部分：冷区和热区。新读取进来的页面默认被放在冷区，在经过innodb_old_blocks_time（1000ms）后，如果再次被访问了，就移到热区。另外，如果一个数据页在读入缓冲池后，在innodb_old_blocks_time时间窗口内被多次访问，之后却不再访问，也认为需要驱逐（待在冷区）。而如果一个页面已经处在热区，再次访问时，只有处于热区1/4（近似值）部分外，才会移动到热区的头部。这样的目的是为了避免LRU的频繁修改，因为每次访问都移动LRU，会影响性能。

![InnoDB_buffer_pool_LRU](/InnoDB_buffer_pool_LRU.png)

LRU相关的函数调用链如下：

![InnoDB_buffer_pool_LRU_chain](/InnoDB_buffer_pool_LRU_chain.png)
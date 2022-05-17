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

buf_page_t实际上存放的是数据页的元信息：page size、state、oldest_modification、newest_modification、access_time等等。

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
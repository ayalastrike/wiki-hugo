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

在以下章节中，InnoDB数据结构中的buf_block_t，buf_page_t、frame按照Jim Gray的命名进行转换，称为block（以下称为PCB，包含meta+data）、meta和frame（data）。

# 概述

## buffer pool

InnoDB是基于磁盘存储的存储引擎，磁盘中的记录以页为单位进行管理。

简单来说，buffer pool就是存储引擎自管理的一块内存区域，缓存频繁访问的数据，从而减少读取写入磁盘对于数据库的影响。

{{< hint danger>}}

这里预留一个问题，为什么不依赖操作系统或者文件系统提供的page cache机制，或者MMAP来进行缓存？

{{</hint>}}

当数据库读取页时，首先分配一个PCB，然后通过该PCB将页"FIX"住，然后发起磁盘IO从磁盘将实际的页读入buffer pool，将frame指向加载的页数据，访问完数据后再”UNFIX“该PCB。下次再读取该页时，先从buffer pool中查找，否则再走上面的流程从磁盘读取。

当数据库修改页时，首先通过上面的机制保证页在buffer pool中后，进行"FIX“，修改页内的数据后，”UNFIX“，此时前台的数据修改完成。后台再以一定的频率（至少在checkpoint时）同步/异步刷回磁盘（原地写/追加写）。

从这里可以看出，读取和写入都为了最大化的利用缓存，对于读取，尽量延长数据保持在内存中的时间，而对于写入，则delay写回磁盘的时机。

对于InnoDB来说，buffer pool中缓存的页有6种类型：索引页、数据页、undo页、change buffer、AHI、lock info，而不仅仅是缓存数据页和索引页。如下图所示：

![InnoDB_buffer_pool_page_type](/InnoDB_buffer_pool_page_type.png)

其中的data page、index page、change buffer page都是需要异步持久化到外存的，undo page分为两种情况：已提交的事务的undo page已经在事务提交时保证其持久化，未提交的事务的undo page可能刷入磁盘也可能没有。

而AHI和lock info是不持久化的，生命周期仅限于运行时。锁也是在事务的生命周期里产生的，在crash recovery时会根据事务信息重建锁状态，所以锁不需要持久化，另外，每个锁信息通常是在trx_lock_t.lock_heap中分配。而AHI是根据访问的数据行动态建立的hashtable，无需保留。

## PCB、meta和frame

前面讲过，buffer pool的资源单位为页。为了管理页，页在缓冲池中按照PCB为单元进行管理，其中包括meta和data（frame）。

PCB和其中的meta、frame（数据页）之间的关系如下图所示：

![InnoDB_buffer_pool_page_&_block](/InnoDB_buffer_pool_page_&_block.png)

从上面的内存布局可以看出，PCB中的第一个字段（必须）是meta，以保证page table指向的buf_block_t和buf_page_t两种类型的指针可以相互转换，block→frame指向真正存放数据的数据页。

meta中的元信息有：page size、state、oldest_modification、newest_modification、access_time...，对于压缩页，解压后的实际数据指针也是用frame指向。

{{< hint info>}}

如果对表进行了压缩，则对应的数据页称为压缩页，如果需要从压缩页中读取数据，则压缩页需要先解压，形成解压页，解压页为16KB。压缩页的大小是在建表的时候指定，目前支持16K，8K，4K，2K，1K。即使压缩页大小设为16K，在blob/varchar/text的类型中也有一定好处。假设指定的压缩页大小为4K，如果有个数据页无法被压缩到4K以下，则需要做B-tree分裂操作，这是一个比较耗时的操作。正常情况下，Buffer Pool中会把压缩和解压页都缓存起来，当Free List不够时，按照系统当前的实际负载来决定淘汰策略。如果系统瓶颈在IO上，则只驱逐解压页，压缩页依然在Buffer Pool中，否则解压页和压缩页都被驱逐。

{{</hint>}}

## page table

buffer pool通过page table（也称为page hashtable，其Key是space_id, page_no，value是PCB地址）可以快速找到已经被读入内存的数据页，而不用线性遍历LRU List去查找。这里的page table不是AHI，AHI是为了减少B+树的扫描，而page hash是为了避免扫描链表（LRU List）。

AHI在B+tree一章详细介绍。

## buffer chunk

从MySQL 5.7开始，支持动态调整buffer pool的大小（resizing），由此引入了buffer chunk的概念，可以简单的将buffer chunk理解为一块花布，里面是page里的数据网格，每次调整buffer pool按照花布为粒度进行批量处理（即支持遍历访问）。

chunk size的大小可以通过[innodb_buffer_pool_chunk_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_chunk_size)设定（默认128M），buffer chunk的数量=缓冲池instance大小/chunk size，同样，为了避免性能问题，buffer chunk不应该超过1000。

一个buffer chunk的结构如下：

![InnoDB_buffer_pool_buffer_chunk](/InnoDB_buffer_pool_buffer_chunk.png)

~~~~
struct buf_chunk_t{
    ulint       size;           一共有多少页（PCB[]+frame[]）
    unsigned char*  mem;        指向frame[]首地址
    ut_new_pfx_t    mem_pfx;    指向allocator的地址（包括cookie），用于deallocator
    buf_block_t*    blocks;     PCB[]
};
~~~~

buffer chunk的初始化只在InooDB启动或者resizing时进行（buf_chunk_init），向memory allocator申请chunk size大小的内存（mmap），并按照页的粒度串到free链表上，即把buffer pool打好格子，格子内划分为一组buf_block_t。通过遍历一个chunk可以访问大多数的数据页（比如查找压缩页 buf_chunk_contains_zip），有两种状态的数据页除外：没有被解压的压缩页（BUF_BLOCK_ZIP_PAGE）以及被修改过且解压页已经被驱逐的压缩页（BUF_BLOCK_ZIP_DIRTY）。

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

## 页链表

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

buffer pool中的页不仅用于读取，也可能被修改，被修改的页称为脏页（dirty page），脏页肯定在LRU链表中，同时也在flush链表中。LRU链表用于管理缓冲池中页的可用性，flush链表用于管理页的刷回（LRU也会触发刷回），两者互不影响。

下图是free链表，LRU链表，flush链表之间的关系：

![InnoDB_buffer_pool free_LRU_flush](/InnoDB_buffer_pool free_LRU_flush.png)

LRU链表中还包含没有被解压的压缩页，这些压缩页刚从磁盘读取出来，还没来的及被解压。

关于LRU算法的详情下面讨论。

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

从上面的介绍可以看出，buffer pool把页作为基本资源单位，并通过一系列PCB+frame作为chunk，多个chunk组合成一个instance，形成一个完整的buffer pool。buffer pool的层次结构如下图所示：

![InnoDB_buffer_pool_hierarchical](/InnoDB_buffer_pool_hierarchical.png)

其中，buffer pool中分配的内存地址需要16KB对齐，这是基于以下两方面的考虑：

- 性能：内存地址对齐可以减少内存的传输次数
- 支持DIRECT_IO：数据页的DIRECT_IO要求内存地址4KB对齐

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

## buffer pool concurrency control

buffer pool中并发访问通过mutex来进行保护，包括：

- PCB
- 各种页链表的管理
- buffer pool

buffer pool的并发控制机制和性能密切相关。最开始，InnoDB通过buffer_pool→mutex来控制并发，即buffer pool中的所有读取、查询、刷新、状态修改等操作都需要持有该mutex，极易形成hotspot。随后，InnoDB将其拆分为多个并发访问的保护对象：缓冲池mutex（buf_pool_t->mutex）、压缩页（buf_pool_t->zip_mutex）、flush链表mutex（buf_pool_t→flush_list_mutex）、PCB.mutex。

内存的并发控制要点：

1. 多个对象之间的封锁传递性
2. 加锁从大到小crabbing
3. 可以采用引用计数减少竞争
4. 通过相容性提供更高并发
5. 前后台路径的正交影响

### free/LRU/flush list mutex

所有的数据页都在free/LRU/flush list上，所以如果操作这些list，首先需要获得list mutex，然后在进行IO的操作前，把list mutex释放。

MySQL 5.7提供了flush_list_mutex

MySQL 8.0新增了free_list_mutex、LRU_list_mutex

### page table rwlock

page table提供了内存页的快速查找，提供的是page table slot的rw lock（hash_table_t→sync_obj.rw_locks），5.6之前是整个page table.mutex。

加锁路径为：page table slot lock→ block/frame lock

### block mutex

block mutex保护的是PCB中meta的buf_fix_count、io_fix、state等等元信息，其目的是减少buffer pool mutex、frame rwlock的竞争。

在代码里的说明：

~~~~
buf_block_t {
  ...
  BPageLock	lock;		/* read-write lock of the buffer frame */
  BPageMutex	mutex;		/* mutex protecting this block:
                                  state (also protected by the buffer pool mutex), io_fix, buf_fix_count, and accessed;
                                  we introduce this new mutex in InnoDB-5.1 to relieve contention on the buffer pool mutex */
}
~~~~

加锁流程如下：

1. FIX page （S/X）
2. buf_page_t.buf_fix_count++ / buf_page_t.io_fix++（block mutex只在操作++--时使用）
3. UNFIX page （--）

这里看一下buf_fix_count和io_fix。

io_fix 表示当前的page frame正在进行读写IO操作：

- BUF_IO_READ：为读取从磁盘加载中
- BUF_IO_WRITE：为写入从磁盘加载中

这里不同的fix代表：读写时buffer pool中没有该页，需要从磁盘加载上来（BUF_IO_READ/BUF_IO_WRITE）

PIN代表已在buffer pool中，正在被访问，相当于引用计数（使用buf_fix_count），不能从buffer pool中刷掉。每次访问都会++，在最后mtr.commit时–。判断该page是否可以访问不用通过frame rwlock，而是通过buf_fix_count>0，这代表已被FIX，这可以减少frame rwlock的竞争。

这里有两方面的优化目的：

- 在用户的正向路径上减少frame rwlock的调用
- 在后台路径上减少frame rwlock的调用：刷脏，replace

比如在flush page时要检测其是否可以被flush，直接判断io_fix：

~~~~
buf_flush_ready_for_flush
if (bpage->oldest_modification == 0
	    || buf_page_get_io_fix(bpage) != BUF_IO_NONE) {
		return(false);
}
~~~~

检查page是否可以被replace，判断io_fix，并且确保当前没有其他线程在引用，也要保证没有修改过：

~~~~
buf_flush_ready_for_replace
  if (buf_page_in_file(bpage)) {
		return(bpage->oldest_modification == 0
		       && bpage->buf_fix_count == 0
		       && buf_page_get_io_fix(bpage) == BUF_IO_NONE);
	}
~~~~

### frame rwlock

在实际获取一个page时（buf_page_get_gen），流程如下：

1. FIX block
2. block→buf_fix_count++ / block→io_fix++
3. 同步读取，block->io_fix--
4. lock page frame rwlock
5. UNFIX block（block->buf_fix_count–，unlock page frame rwlock）、后台异步读取后（buf_page_io_complete）（block->buf_fix_count–，unlock page frame rwlock）

从这里也可以看出，如果io_fix设置了，则用户线程需要等待IO完成，不会获取page frame rwlock，这样做尽可能的减少持有page frame rwlock的机会。因为page frame rwlock的调用在B+树的核心路径上。

那我们再接着讨论rwlock的加锁类型。InnoDB在访问B+树的时候，会采用乐观访问的方式，先对page frame加S lock，如果可能修改，再加SX lock，确认要修改的时候加X lock。

另外，只有叶子节点才使用page frame rwlock来保护page frame的一致性，对于非叶子节点通过索引（dict_index_t→lock）来进行保护。

InnoDB还做了很多优化，比如之前MySQL 5.6会拿着整颗B+树的index lock，而是尽可能的只拿着会引起B+树结构变化的子树，比如引入SX lock，在真正要修改的时候才会获得X lock。

{{< hint info>}}

引入SX lock是对读取的优化, 对写入并没有优化. 因为持有SX lock 的时候, S lock 操作是可以进行的, 但是X lock 操作不可以。即允许更多的读取，X lock的并发度和之前一样。

{{</hint>}}

从上面我们可以看出，这些优化只是对用户的前台访问路径有效果（减少了page frame rwlock的获取），而在后台路径上并没有过多的优化。

比如上面提到的，在后台路径的异步IO场景下，page frame rw_lock是在buf_page_io_complete 之后才会放开的。因此page frame rw_lock 的持有周期是整个异步IO的周期，直到IO操作完成。而page frame rw_lock又是用户访问路径（B+树）的核心路径（btr_cur_search_to_nth_level），所以可能会因为后台异步IO而对用户的访问造成大量堵塞，如果是非叶子节点，影响范围更大，更明显。而异步IO卡顿的出现可能是因为InnoDB simulated AIO 的队列长度是（io_read_threads+io_write_threads）* 256，可能会出现大量的page在IO等待队列中，page 因为在IO 等待队列中等待，如果存储设备的IO latency恶化，问题会更加明显。

当然我们也通过simulated AIO 优化, copy page等等减少持有page frame 的时长.

buf_page_io_complete主要做两个工作：

1. page io_fix设置成NONE，表示这个page的io操作已经完成
2. 将page frame rw_lock释放（如果是读，释放X lock，如果是写，释放SX lock）

这里读操作要拿X lock主要是为了避免多个线程同时去读这个page，如果同时有其他线程要访问该page（在1，2之间），会尝试加S lock（buf_wait_for_read），如果S lock加成功，则说明该page已IO读取完成（2步完成）。

最后总结一下buffer pool的并发控制：

- free/LRU/flush list的操作通过相关的list mutex保护
- 后面4个mutex（引入block mutex、buf_fix_count/io_fix避免不必要的获得page frame rw_lock的开销）
  - 先加page hash slot rwlock
  - 获得block mutex
  - 释放page hash slot rwlock
  - 修改buf_fix_count/io_fix
  - 释放block mutex
  - 持有page frame rw_lock
  - 操作完成后修改buf_fix_count，释放page frame rw_lock

## buffer pool warmup

MySQL 5.6引入了buffer pool warmup功能，解决以下问题：

- MySQL冷启动以后数据的预热问题
- 运行时的OLAP污染buffer pool

为此，提供dump+load机制进行warmup。

功能实现上也简单，通过buf_dump_thread后台线程，如果收到srv_buf_dump_event事件通知，则遍历缓冲池instance，将缓冲池中的页按照<space_id, page_no>的方式dump到文件中。在遍历过程中，需要对buffer_pool→mutex加锁，所以会引起性能抖动，需要注意。同时，在load时，因为会有大量的IO读发生，所以最好warmup后再提供服务。

在buf_load实现中，先把外储的文件读入内存，然后使用归并排序对数据排序（因为page no代表了物理位置），以64个数据页为单位进行IO合并，然后发起一次真正的读取操作。排序的作用就是便于IO合并。

warmup功能的相关参数如下：

{{< hint info>}}

[innodb_buffer_pool_dump_at_shutdown](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_at_shutdown)

[innodb_buffer_pool_load_at_startup](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_at_startup)

[innodb_buffer_pool_dump_now](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_now)

[innodb_buffer_pool_load_abort](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_load_abort)

[innodb_buffer_pool_dump_pct](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_dump_pct)

[innodb_buffer_pool_filename](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_filename)

{{</hint>}}

## buffer pool resizing

为了支持计算资源（内存）的弹性控制，MySQL 5.7提供了在运行时动态调整缓冲池大小（resizing）的功能。

我们从之前的介绍可以得知，每个缓冲池instance都是由相同数量的buffer chunks组成的，每个buffer chunk的大小通过[innodb_buffer_pool_chunk_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_chunk_size)指定（实际上会稍大5%，因为需要存放PCB）。在resizing的时候以chunk size为单位进行动态增大/缩小，调整完毕后缓冲池的大小=innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances的倍数。

resizing通过buf_resize_thread完成，在resizing的过程中可以通过[Innodb_buffer_pool_resize_status](https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html#statvar_Innodb_buffer_pool_resize_status)查看状态。

下面来看一下resizing的处理过程：

1. 根据新的buffer pool size计算每个缓冲池instance新的chunks数量
2. 禁用AHI（并清空AHI）
3. 收缩：
   1. 锁buffer_poo→mutex
   2. 从free链表中把待删除chunk上的page放入待删除链表（buffer_poo→withdraw）
   3. 如果上述page为脏页，则刷脏
   4. 解锁
   5. 重复a~d直至收缩收集完成
4. resizing
   1. 锁buffer_pool->mutex、page hash
   2. 从待删除链表中以chunk为单位收缩buffer pool
   3. 清空待删除链表
   4. 扩大buffer pool：分配新的chunks
   5. buffer pool重新分配所有的chunks
   6. 如果扩大/收缩超过2倍，重置page hash，改变hash桶大小
   7. 解锁
   8. 果扩大/收缩超过2倍，重启和buffer pool大小相关的内存结构，如锁系统（lock_sys_resize），AHI（btr_search_sys_resize），数据字典（dict_resize）
5. 开启AHI

从上面可以看到，扩容比缩容相对容易。在收缩时，如果有事务一致未提交，并且占用了待收缩的页，导致收缩一直重试，则会通过日志输出。同时，为了避免频繁重试，重试的时长处采用指数形式增长。

另外，resizing后，serach过程中的btr cursor保存的page可能重新进行了加载，AHI也都进行了清空，都需要重新读取。

# LRU

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
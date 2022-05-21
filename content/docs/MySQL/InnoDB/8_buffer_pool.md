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

这里给读者留一个问题，为什么不依赖操作系统或者文件系统提供的page cache机制，或者MMAP来进行缓存？

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

{{< hint danger>}}

这里给读者留一个问题：frame和PCB的生命周期是一致的吗？

{{</hint>}}

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

frame通过PCB.block（buf_block_t.block）保护

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

- 朴素的LRU算法：当LRU链表的长度小于BUF_LRU_OLD_MIN_LEN（512）时启用，读到的buf_page放入LRU链表的头部，不区分冷区和热区。（两方面的考虑：空间腾挪机会小，less is more）
- LRU-K（midpoint insertion strategy）：当链表长度大于BUF_LRU_OLD_MIN_LEN时启用，区分冷热区。

LRU链表的布局如下图所示：

![InnoDB_buffer_pool_LRU_algo](/InnoDB_buffer_pool_LRU_algo.png)



这张图来自官方，逻辑上说明了LRU的布局。

在LRU中，保证使用最多最频繁使用的数据页尽可能keep在内存中，同时还要考虑dirty page evict的反压，这时LRU算法的核心。

在InnoDB中，通过全局时钟策略（clock）来作为算法的核心，有两个时钟的概念：

- access clock：用于数据访问的标识
- evict clock：用于数据evict的标识

## access clock

全局access clock时钟：每次往LRU热区中add，则++，即buffer_pool->ulint_clock

每个PCB记录全局时钟中其自己的位置，即PCB.LRU_position

这样，数据访问的热点就可以评估出来：

- 全局access clock代表全局热度
- 每个页可以通过LRU_position-ulint_clock得知其在LRU中的位置，从而得知其是否被频繁访问，同时意味着evict的概率大小（也要考虑dirty page flush）





## evict clock

全局evict clock：每次LRU evict++，即buf_pool->freed_page_clock

每个PCB记录上一次移到LRU头的global evict clock，记入block->freed_page_clock

这样，evict的信息就可以展现出来：

- 全局evict clock代表evict的频率，即置换出去的页的频繁程度
- PCB和evict clock的差值：频繁（access_clock）而置换概率低的（evict clock），越应该在LRU的上部

整体上如图所示：



这样的设计也不是没有问题，维护clock的开销和访问成正比，这会是一个hotspot。

## 维护策略

OLD的维护是通过ratio/次数（BUF_LRU_OLD_TOLERANCE）的barrier来保证链表的调整不过于频繁，这个barrier通过ratio或者次数。

新读取进来的页面默认被放在OLD（LRU_old位置）。

再次访问时：

- OLD经过一段时间窗口（innodb_old_blocks_time 1s），才被移到NEW；否则还待在OLD
- 位于NEW区中前1/4移动到头部
- 位于NEW区中3/4不移动

flush后放入尾部，表示可以evict

尾部evict

如图所示：

![InnoDB_buffer_pool_LRU_list](/InnoDB_buffer_pool_LRU_list.png)





![InnoDB_buffer_pool_LRU](/InnoDB_buffer_pool_LRU.png)

LRU相关的函数调用链如下：

![InnoDB_buffer_pool_LRU_chain](/InnoDB_buffer_pool_LRU_chain.png)

{{< hint danger>}}

留给读者2个小问题：

1. 官方图中的OLD区头部是静态的吗？
2. 如何计算评估evict和keep in memory的策略？

{{</hint>}}

# 页的读取

数据页的读取分为2种：

- 逻辑读：从buffer pool中读取数据页
- 物理读：从磁盘中读取数据页，然后再在内存中修改数据

从中可以看出，物理读在逻辑读的过程之中，所以我们从逻辑读开始。

## 逻辑读取

逻辑读取是指从buffer pool中访问指定的页(space, offset)。首先在page table中查找（buf_page_hash_get），如果没有，意味逻辑读miss，则走物理读从磁盘上将页加载到buffer  pool中，然后再访问。

### 读取页面

页面的读取逻辑由buf_page_get_gen实现，我们在深入内部逻辑之前，先看一下buf_page_get_gen的2个参数：读取模式（mode）和page-latch类型（rw_latch），以便我们有一个全局的视角作为参考。

读取模式参数mode（其中前三种使用较多）：

- BUF_GET：默认获取数据页的方式，如果数据页不在buffer pool，则从磁盘读取；如果在，则判断是否要把其加入到LRU list中，以及判断是否需要进行线性预读。读取加s-latch，修改加x-latch。
- BUF_GET_IF_IN_POOL：只在buffer pool中查找，不在直接返回null，不走磁盘，后续内存操作同上
- BUF_PEEK_IF_IN_POOL：只在buffer pool中查找，不走磁盘，不进行后续内存操作
- BUF_GET_NO_LATCH：不加任何latch，其他同#1，B+树的非叶子节点页的读取走这种模式。
- BUF_GET_IF_IN_POOL_OR_WATCH：purge thread使用，只在buffer pool中查找（在：加LRU list+线性预读，不在：设置watch）
- BUF_GET_POSSIBLY_FREED：与BUF_GET类似，只是允许相应的数据页在函数执行过程中被释放，主要用在估算B+tree两个slot之间的数据行数。

参数rw_latch为读到数据页后所持有的latch，有以下几种：

- RW_S_LATCH
- RW_SX_LATCH
- RW_X_LATCH
- RW_NO_LATCH

根据FIX Rules，每个页在放入buffer pool后都需要加上latch来保护（buf_block_t->lock），因此需要rw_latch：RW_S_LATCH/RW_SX_LATCH/RW_X_LATCH，另外还有RW_NO_LATCH，这是因为在InnoDB中，对于B+树索引的所有非叶子节点的页访问都是通过索引本身的latch（dict_index_t→lock）来保护的，因此对于非叶子节点（页）的访问，页可以不需要持有任何latch，从而节省CPU的资源以提高性能。

然而，不管访问模式如何，根据FIX Rules规则，都需要FIX，在InnoDB中不是根据是否持有该页的latch来进行判断的，而是根据buf_page_t->buf_fix_count（引用计数）来决定的。因此，只要读取了页，就需要更新buf_fix_count（buf_block_buf_fix_func）。这也就意味着，如果页被FIX了，那么其对应的buf_fix_count一定是>0的。

{{< hint danger>}}

留给读者1个小问题：page-latch何时释放

{{</hint>}}

另外，再次重生PCB.meta中的buf_fix_count和io_fix两个变量，这两个变量主要用来做并发控制，减少mutex锁定的范围。当从buffer pool读取数据页时，会其加读锁，然后递增buf_page_t::buf_fix_count，同时设置buf_page_t::io_fix为BUF_IO_READ，然后即可以释放读锁。后续如果其他线程在驱逐数据页(或者刷脏)的时候，需要先检查一下这两个变量，如果buf_page_t::buf_fix_count不为零且buf_page_t::io_fix不为BUF_IO_NONE，则不允许驱逐(buf_page_can_relocate)。这里的技巧主要是为了减少PCB上mutex的争抢，而对数据页的内容（frame），读取的时候依然要加读锁，修改时加写锁。

读取页面的函数调用链如下：

~~~~
buf_page_get_gen
	buf_page_hash_get_low				// 从page hash中查找（缓冲池）
	miss: block= null
		buf_read_page					// 物理读取
			buf_read_page_low			
				buf_page_init_for_read	// 从缓冲池中通过LRU算法分配对象buf_block_t，加block->mutex，并加x-latch（block->lock）,用以保护其指向的内存（block->frame，即页的实际存储的内容）
				fil_io					// 读取磁盘上的数据页
				buf_page_io_complete	// 同步/异步IO完成，释放x-latch
					buf_page_set_io_fix	// 释放io_fix
			buf_read_ahead_random		// 随机预读
		failed retry 100...				// 物理读取失败则重试100次
	hit: fix_block= block
	buf_block_fix						// FIX page
	update buf_page->access_time		// 更新page ts
	buf_read_ahead_linear				// 线性读取
	buf_page_make_young_if_needed		// 更新LRU
~~~~

以下是buf_page_get_gen的详细流程：

1. 通过page_id（space_id+page_no）查找数据页位于哪个buffer pool instance中，instance_no = (space_id << 20 + space_id + page_no >> 6) % instance_number，即通过space_id+page_no算出一个fold值然后按照instance取余。这里有个小细节，page_no的第六位被砍掉，这是为了保证一个extent的数据（2的6次方=64）能被缓存到同一个buffer pool instance中，便于后面的预读操作。
2. 在page table中查找page，返回PCB（buf_page_hash_get_low）
3. 如果没有在buffer pool中，并且mode为BUF_GET_IF_IN_POOL_OR_WATCH则设置watch数据页（buf_pool_watch_set）
4. 如果没有在buffer pool中，并且mode为BUF_GET_IF_IN_POOL | BUF_PEEK_IF_IN_POOL | BUF_GET_IF_IN_POOL_OR_WATCH，则返回null，表示没有找到数据页。
5. 如果没有在buffer pool中，并且mode为其他，发起磁盘同步读取（buf_read_page）(buf_read_page)。
   在读取磁盘数据之前，如果是非压缩页，则先从free链表中获取空闲的数据页，如果没有，则需要通过同步刷脏来释放数据页，最后将获取到的空闲数据页加到LRU链表上（buf_page_init_for_read）
   如果是压缩页，则临时分配一个buf_page_t用来做控制体，通过伙伴系统（buf_buddy_alloc）分配到压缩页存数据的空间，最后同样加入到LRU List中。
6. 调用IO子系统的接口同步读取页面数据（fil_io + IORequest::READ）。
   如果读取失败，重试100次（BUF_PAGE_READ_MAX_RETRIES，每次重新从#1开始），然后触发断言中止mysqld（Unable to read page + abort）
   如果成功，则判断是否要进行随机预读（buf_read_ahead_random）
7. 判断读取的数据页是不是压缩页，如果是的话，因为从磁盘中读取的压缩页的控制体是临时分配的，所以需要重新分配block（buf_LRU_get_free_block），把临时分配的buf_page_t给释放掉，用buf_relocate函数替换掉，接着进行解压，解压成功后，设置state为BUF_BLOCK_FILE_PAGE，最后加入Unzip LRU链表中。
8. 接着，我们判断这个页是否是第一次访问（buf_page_is_accessed），如果是则设置buf_page_t::access_time。
   如果mode不为BUF_PEEK_IF_IN_POOL，判断是否把这个数据页移到young list中（buf_page_make_young_if_needed）。
   如果mode不为BUF_GET_NO_LATCH，给数据页加上读写锁。
   如果mode不为BUF_PEEK_IF_IN_POOL，并且这个数据页是第一次访问，则判断是否需要进行线性预读。

从这里可以看出，#5是性能的重点，如果buffer pool中没有数据页，会产生同步IO，并且，如果没有空闲的PCB，则还可能需要进行同步刷脏，这会造成明显的stall，所以我们在日常的运维中需要格外关注这种场景。

### 为物理读取准备PCB

从上面可以看到，如果逻辑读miss，则需要准备好PCB，以便后续通过文件IO填充frame（buf_page_init_for_read），进行物理读取，其流程如下：

~~~~c++
buf_page_init_for_read(){
  ...

  // 1. 核心函数，用于获取一个空闲的PCB，可能会触发LRU List和Flush List的淘汰
  block = buf_LRU_get_free_block(buf_pool);

  // 2. 获得buffer pool的互斥保护
  buf_pool_mutex_enter(buf_pool);

  // 3. 对page table slot rwlock加x-latch
  hash_lock = buf_page_hash_lock_get(buf_pool, page_id);
  rw_lock_x_lock(hash_lock);

  // 4. 获得block mutex
  buf_page_mutex_enter(block);

  // 5. 初始化Page，并加到page table中
  buf_page_init(buf_pool, page_id, page_size, block);

  // 6. 设置page的io_fix类型为BUF_IO_READ，同时解除page table slot锁定（设置io_fix而不用对block fix是因为page hash的slot已提供互斥保护）
  buf_page_set_io_fix(bpage, BUF_IO_READ);
  rw_lock_x_unlock(hash_lock);

  // 7. 加入到LRU List
  buf_LRU_add_block(bpage, TRUE /* to old blocks */);

  // 8. 持有page frame rwlock x-latch
  rw_lock_x_lock_gen(&block->lock, BUF_IO_READ);

  // 9. 释放block mutex
  buf_page_mutex_exit(block);

  // 10.释放buffer pool的互斥保护
  buf_pool_mutex_exit(buf_pool);
  ...
  // 11. 返回PCB
  return (bpage);
}
~~~~

这里注意两点：

- 对buffer pool的保护在最外面，是为了其中的free list、LRU list和flush list的保护
- block->mutex只是为了保护io_fix和buf_fix_count，数据页（内容）的保护通过frame lock来保证，这个直到io完成才释放

### 获取空闲PCB

我们再接着从#1深入下去，来探究获得空闲PCB的具体逻辑（buf_LRU_get_free_block）。

页首先从free链表中分配，如果free链表已用尽（为空），则从LRU链表中申请（buf_LRU_search_and_free_block）。

而从上面的LRU算法可以知道，尾部的页可以被替换，但替换需要满足以下的前提条件：

- 不是脏页（bpage->oldest_modification == 0）
- 页没有被其他线程使用（bpage->buf_fix_count == 0 && buf_page_get_io_fix(bpage) == BUF_IO_NONE）

其中，脏页需要先进行刷盘再替换；页如果正在被其他线程使用，则不能立即被替换出去。（buf_flush_read_for_replace）

如果在LRU链表中找到可以替换的页，则对页进行free操作，即将block→state设置为BUF_BLOCK_MEMORY，从page table和AHI中删除。

那么，获取空闲PCB的具体流程如下：

1. 整体健康检查：从free list中取首先检查free list和LRU list的长度，如果发现他们在buffer chunks占用太少的空间，则表示太多的空间被lock info，自AHI等内部结构给占用了（一般是是大事务导致的。候会报错误日志）

2. 从free list中取，大部分情况下可以取到并直接返回

3. 取不到再通过LRU list和flush list中evict（buf_LRU_get_free_block），在这个过程中，通过迭代的方式获取

   通过定义n_iterations，用于标识是第几次进行迭代获取的空闲block。总共分为三种情况作处理，具体如下：

   - iteration 0：
     1. 从free list中获取（buf_LRU_get_free_only），返回
     2. 如果free list中没有，设置try_LRU_scan，开始扫描LRU list尾部的100个page，如果找到一个可以回收的page，则进行回收（buf_LRU_free_page），并将其加入到free list中，然后再从free list上取
     3. 如果还是没有找到空闲的page，则尝试从LRU list的尾部flush一个page（buf_flush_single_page_from_LRU），并将其加入到free list中，然后再从free list上取
     4. 将n_iterations++，并重复1-3的步骤

   - n_iterations = 1：

     和iterator 0几乎一样，只是在扫描LRU List时是扫描整个链表而不是只扫描尾部的一部分了，并唤醒刷脏线程（page_cleaner），其余流程完全一致。如果未找到则将n_iterations++

   - n_iterations > 1：

     和round 1一样，只是会在在flush之前每次sleep 10ms。如果还是找不到空闲的page，则继续n_iterations++。

   - 当n_iterations > 20时，会打印日志

总结一下：

1. 整体健康检查
2. 从free取
3. 从LRU扫100，从flush刷1
4. 从LRU扫全部，从flush刷1
5. 从LRU扫全部，从flush刷1，并等一等，休眠10ms
6. 重复上一步...
7. 20次以上，打日志

## 物理读取

从磁盘中将页读入缓冲池称为页的物理读取（physical read）。

函数调用链如下：

~~~~
buf_read_page_low
    buf_page_init_for_read // 从buffer pool中通过LRU算法分配PCB，加frame x-latch）用以保护其指向的页的实际存储数据）
    fil_io                 // 读取磁盘上的数据页（文件IO）
    buf_page_io_complete   // 释放frame lock x-latch
~~~~

从上面可以看出，frame block有以下作用：

- 当发起读取操作时，能够告诉当前线程发起的I/O操作是否完成，只有当buf_page_io_complete完成后，才能确保block→frame对象中的页内容是完整的
- 如果有其它线程并发的发起对该页的请求，则需要等待本次I/O操作完成，以保证并发访问的页的数据一致性

同时，当有多个线程并发访问同一个页时，即存在多个线程调用buf_page_init_for_read，也不会有问题，其中会通过buf_pool→mutex来进行并发控制的保护。第一个进行物理读取的线程在调用buf_page_init_for_read时会从LRU链表中返回PCB对象。

当PCB对象通过buf_page_init_for_read 初始化操作完成后，仅仅需要调用fil_io，即可将磁盘上页的内容读取到block→frame中。这里的读取操作分为同步和异步两种（sync变量区分）。对于同步读取，操作完成后直接调用buf_page_io_complete释放持有的frame lock x-latch。对于异步读取，则通过io_thread释放所持有的x-latch（fil_aio_wait）。

另外，对于insert buffer的bitmap页、事务系统页，读取操作必须为同步IO操作。

## 随机预读

随机预读（buf_read_ahead_random）只会在物理读取之后触发。通过[innodb_random_read_ahead](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_random_read_ahead)控制是否开启随机预读（默认：关闭）

随机预读是指判断某个区域内的页（1个extent）是否其大多数已被访问（13个），并且这些被访问的页是否为热点数据（热区前1/4），如果满足条件，InnoDB则认为该区域内的页都可能被访问，于是提前进行读取操作，提高数据库的读取性能。预读根据区域内的（space, offset）进行以异步IO的方式进行顺序读取。总结起来，预读就是将之后可能被访问的页通过顺序的方式进行读取，从而提高数据库性能。

随机预读的判断代码如下：

~~~~c++
/* Count how many blocks in the area have been recently accessed,
	that is, reside near the start of the LRU list. */

	for (i = low; i < high; i++) {
		const buf_page_t*	bpage = buf_page_hash_get(
			buf_pool, page_id_t(page_id.space(), i));

		if (bpage != NULL
		    && buf_page_is_accessed(bpage)
		    && buf_page_peek_if_young(bpage)) {

			recent_blocks++;

			if (recent_blocks
			    >= BUF_READ_AHEAD_RANDOM_THRESHOLD(buf_pool)) {

				buf_pool_mutex_exit(buf_pool);
				goto read_ahead;
			}
		}
	}
~~~~

此外，如果当前InnoDB读取压力较大时（buf_pool→n_pend_reads），也不会启用随机预读，即如果缓冲池中一半的页都在等待读取操作完成。

~~~~c++
/** If there are buf_pool->curr_size per the number below pending reads, then
read-ahead is not done: this is to prevent flooding the buffer pool with
i/o-fixed buffer blocks */
#define BUF_READ_AHEAD_PEND_LIMIT	2

buf_read_ahead_random() {
    ...
	if (buf_pool->n_pend_reads
	    > buf_pool->curr_size / BUF_READ_AHEAD_PEND_LIMIT) {
		buf_pool_mutex_exit(buf_pool);

		return(0);
	}
}
~~~~

最后一点，预读会把整个范围内的页都读取上来，甚至还包括原来读取过的页。

## 线性读取

线性预读（read-ahead algorithms）（buf_read_ahead_linear）通过判断页的访问是否为顺序的来触发。

线性预读的算法为：读取一个页，如果该页是某个区域的边界（border），并且该区域内的部分页（56个，[innodb_read_ahead_threshold](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_read_ahead_threshold)）都已经被顺序访问（正序/逆序：通过判断数据页access time为升序/逆序），则触发线性预读操作，以异步IO的方式顺序读取之前/之后的一个extent。

从这里可以看出，线性读取的判断比较苛刻，这是因为其主要是为了解决全表扫描的性能问题。

InnoDB中除了有预读功能，在刷脏页的时候，也能进行预写（buf_flush_try_neighbors）。当一个数据页需要被写入磁盘的时候，查找其前面或者后面邻居数据页是否也是脏页且可以被刷盘（没有被IOFix且在old list中），如果可以的话，一起刷入磁盘，减少磁盘寻道时间。预写功能可以通过[innodb_flush_neighbors](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_flush_neighbors)参数来控制。不过在现在的SSD磁盘下，这个功能可以关闭。

线性预读也是通过区域的方式管理，区域大小由BUF_READ_AHEAD_LINEAR_AREA定义（默认32）。判断页是否在该区域的边界：是否在该32个页的第一个或者最后一个页。

~~~~c++
	low  = (page_id.page_no() / buf_read_ahead_linear_area)
		* buf_read_ahead_linear_area;
	high = (page_id.page_no() / buf_read_ahead_linear_area + 1)
		* buf_read_ahead_linear_area;

	if ((page_id.page_no() != low) && (page_id.page_no() != high - 1)) {
		/* This is not a border page of the area: return */

		return(0);
	}
~~~~

最后，也是最容易被忽视的，线性预读要求物理上也是连续的页：通过当前读到的页，判断FIL_HEADER中保存的下一个页的位置，判断是否为物理连续：

~~~~c++
	/* Read the natural predecessor and successor page addresses from
	the page; NOTE that because the calling thread may have an x-latch
	on the page, we do not acquire an s-latch on the page, this is to
	prevent deadlocks. Even if we read values which are nonsense, the
	algorithm will work. */

	pred_offset = fil_page_get_prev(frame);
	succ_offset = fil_page_get_next(frame);

	buf_pool_mutex_exit(buf_pool);

	if ((page_id.page_no() == low)
	    && (succ_offset == page_id.page_no() + 1)) {

		/* This is ok, we can continue */
		new_offset = pred_offset;

	} else if ((page_id.page_no() == high - 1)
		   && (pred_offset == page_id.page_no() - 1)) {

		/* This is ok, we can continue */
		new_offset = succ_offset;
	} else {
		/* Successor or predecessor not in the right order */

		return(0);
	}
~~~~

在上面有一点特别的：通过block->frame判断前一个/后一个页的偏移量时没有通过frame保护（持有block->lock），但这样也不会有问题，这是因为，即使并发导致读取的数据不一致，最坏的情况也就是将不应该线性读取的页进行了读取，或者应该预读的页没有进行线性读取，因为只是读而不是写，对数据本身不会破坏。另一个考虑是，有可能进行预读的线程已经持有了对该页的frame x-latch，那么将会导致死锁产生（参加上面的注释部分）。

线性预读和随机预读是完全不同的预读算法，具体差异见下：

|                  | 随机预读   | 线性预读     |
| :--------------- | :--------- | :----------- |
| 触发阈值         | 13         | 56           |
| 触发判断条件     | 页是否活跃 | 访问是否顺序 |
| 目标页是否被读取 | 否         | 是           |
| 触发时间         | 页被读取前 | 页被读取后   |

预读是一种启发式的优化方式，但并不是一定会带来性能的提升（并不所有场景都适用），也可能会给性能带来反作用。比如，如果预读到的页之后并未被访问，则LRU机制会把有用的页evict出去，导致buffer pool的命中率下降。

{{< hint info>}}

以下部分补充自阿里内核月报

**InnoDB 预读 VS Oracle 多块读**

目前，IO 仍然是数据库的性能杀手，为了提高 IO 利用率和吞吐量，不同的数据库都设计了不同的方法，InnoDB提供了预读功能（read-ahead），Oracle提供了多块读功能（multiblock-read），在这里来进行一个对比和分析。

**InnoDB read-ahead**

InnoDB提供了两种预读的方式，一种是线性预读（Linear read ahead），当连续读取一个extent的threshold个page时，会触发上/下一个extent（BUF_READ_AHEAD_PAGES=64个页）的预读。另一种是随机预读（Random read-ahead），当连续读取设定数量的page后，会触发读取这个extent的剩余page。

InnoDB的预读是在后台异步完成的。

**Oracle multiblock-read**
对堆表进行全表扫描时，需要大量IO，Orache可以在session级别设置db_file_multiblock_read_count，这样在读取堆表结构的数据块的时候，一次IO读取多个数据块，大大减少了IO的次数。在针对大表的汇总分析查找中，效果非常明显。这里要注意不要设置过大，否则会造成buffer cache flooding。

**场景分析**

下面我们看两个非常典型的场景:

1. **高并发，小IO**：在高并发的场景下，latency主要取决于IO请求的时间，而InnoDB的预读是异步的，只能起到预热（warmup）的效果，对latency收益很小；而Oracle如果触发了multiblock-read，latency反而会上升，因为一次同步IO请求的时间增加了。
2. **低并发，高IO吞吐**：通常情况下，对线上数据的汇总查询尽量安排在业务低峰期，同时希望能够完全使用主机的资源来完成查询。在使用全表扫描时，InnoDB会触发read-ahead，每次提前异步读取下一个extent的page，加快读取的速度。 Oracle会一次IO读取多个block，提高读取的吞吐量。

所以从这里可以看到，对于第二个场景，这个优化才会产生效果。

**为什么在聚集查询的时候，Oracle的效果会比InnoDB要好？**

如果是机械盘，则又回到了 IOPS 和 throughput 的讨论上去了。InnoDB的read-ahead，在触发的时候，针对下一个extent，对每一个page提交了异步IO请求，也就是增加了IO request次数，虽然Native AIO和disk会有针对性合并IO，但仍然非常有限，而Oracle每次提交合并多个连续数据块的IO请求，能够更好利用disk的吞吐能力。

所以，InnoDB在针对aggregation类型的查询的时候，想要完全使用IO的吞吐能力，相比较Oracle的multiblock-read，会偏弱一点。

**优化**

针对目前的InnoDB机制，可以有以下几种优化方法:

- 在session级别，提供可设置预读的触发条件，并使用多个后台线程来完成异步IO请求。因为没有减少小IO请求，收益甚小
- 采用和Oracle类似的方法，独立一个buffer pool，专门进行多块读，针对previous/next extent，一次读取到buffer pool中
- 终极优化方法，使用并行查询，Oracle在全表扫描的时候，使用/* parallel */ hint方法启动多个进程完成查询，这里需要将InnoDB的聚簇索引进行逻辑分片，采用多线程对逻辑分区进行并行查询

{{</hint>}}



{{< hint info>}}

**InnoDB logical read-ahead**

上面提到的InnoDB linear read-ahead和Oracle的multiblock read，其之所以带来了更高的吞吐量，都是基于数据存储的连续性的假设，比如InnoDB使用自增字段作为PK，或者是Oracle使用默认的堆表。

![InnoDB_buffer_pool_logical_read-ahead](/InnoDB_buffer_pool_logical_read-ahead.png)

上图是一个典型的InnoDB聚簇索引表，这样的结构很容易构造，比如使用一个非连续的字段作为索引字段，随机对记录进行插入，因此，叶子节点链表上的page no就会产生非连续性，如果进行一次全表扫描，会因为page no的非连续性而无法触发InnoDB的预读，更没有办法合并IO请求。

因此，对于存在时间比较长，变更又比较多的大表，除非我们对于这个表进行重建，否则叶子节点的离散性会随着时间的推移越来越严重。但对于在线应用来说，重建又会产生比较大的运维风险，这里有一种平衡的方法：logical read-ahead。

**logical read-ahead**

逻辑预读的思路是根据branch节点来预读leaf节点。

逻辑预读使用两个扫描路径:

- 一个cursor定位到leaf page，然后根据leaf page之间的双链表，moves_up进行扫描数据；
- 另一个cursor定位到branch节点，因为InnoDB B-Tree结构的每一层都由双向链表进行连接，然后这个cursor就沿着branch节点进行扫描，保存扫描到的page no，然后使用异步IO，发起这些leaf page的预读。

实现

1. 在row_search_for_mysql进行moves_up的过程中进行logical read-ahead
2. branch节点扫描的cursor保存到trx结构中，生命周期到一个sql语句结束
3. branch cursor扫描用户可配置的page count，临时保存到数组中，对page_no进行排序
4. 使用libaio发起异步IO读取，完成logical read-ahead。
   logical read-ahead很好的提升了离散存储数据的吞吐能力，Facebook在其MySQL的逻辑备份过程中，对于大表的dump备份开启了此特性，备份速度有非常大的提升。

{{</hint>}}

# 页的更新

## 页的更新流程

页的更改流程为：页面的更新通过FIX rules进行保护，并在FIX期间生产变更的日志（mini-transaction）存入private repo，并在mtr.commit时写入redo log buffer，然后依赖force-log-at-commit在事务提交时将日志持久化。而实际的脏页随后再以异步的方式写回磁盘。

### 加入flush list

flust list中保存的是虽有未写回磁盘原处的脏页，这些脏页在mtr.commit时加入flush list（release_latches），并释放latch（具体流程在mini-transaction中详细介绍）。一旦写入到flush list中，flush list的页面顺序就不再更改，即flush list的顺序是按照mtr.commit，也就是LSN的顺序排列的（oldest_modification），这从语义上代表着写入序（变更序）。如果页面再次发生修改，则更新newest_modification。

# 页的刷回

## WAL和检查点

开篇已经提到，buffer pool设计是为了协调CPU和磁盘之间的速度鸿沟，因此页的操作都是先在buffer pool完成的，然后再刷新回磁盘。

如果页每次变化都立即刷回磁盘，这个开销是巨大的，并且，如果热点集中在某几个页上，性能会变得更差。另外，如果从缓冲池中将变更后的页（新版本）刷新到磁盘时发生了宕机，那么数据就会损坏（corrupted）。为了解决以上的问题，数据库系统普遍采用了WAL（Write Ahead Log）策略：当事务提交时，先写redo log，再修改页；如果宕机，则可以通过redo log来完成数据的恢复。这也是事务ACID特性中D的要求。

理想情况下，如果redo log的空间是无限的，buffer pool可以容纳所有的数据，那么就不需要将buffer pool中页的新版本刷回磁盘，当宕机时，可以通过redo log将数据恢复到宕机发生的时刻（内存态），因此，理想情况可以归纳为两个条件：

- buffer pool可以容纳所有的数据
- redo log空间无限

第一个条件在实际的环境中很难满足，这是因为数据是在持续增长的，并且，内存的价格也比磁盘昂贵，从成本上是不合算的。

第二个条件可以做到，但是要配合checkpoint来定期将数据刷回磁盘，缩短宕机的恢复时间，并回收日志空间。

因此，checkpoint技术可以解决以下问题：

- 缩短数据库的恢复时间
- 在buffer pool空间有限时，将脏页刷回磁盘
- redo log空间有限时，将脏页刷回磁盘

这里有几个压力点：系统宕机、内存空间不足，磁盘空间不足。

1. 数据库宕机时，不需要重做所有的redo log，因为检查点之前的页已经刷回磁盘，只需要对检查点之后的redo log进行恢复，大大减少了恢复时间
2. buffer pool空间不足时，根据LRU算法淘汰最近最少使用的页，如果此页是脏页，还需要强制执行checkpoint，将页的最新版本刷回磁盘
3. redo log file常采用循环使用的方式，而不是无限增长（从成本/管理上都会增加困难），则在重用这部分日志的话，需要保证对应的脏页刷到redo log的点位。

checkpoint机制可以有两种实现方式（即页的刷回机制）：

- sharp checkpoint
- fuzzy checkpoint

sharp checkpoint可以有两种场景：

1. 在数据库关闭时将所有的脏页刷回磁盘（默认），即参数innodb_fast_shutdown = 1。
2. 在运行时使用sharp checkpoint，即每次变更就立即刷回磁盘，这会极大的影响数据库的性能

InnoDB（也是大多数关系型数据库的选择）使用fuzzy checkpoint进行页的刷回，即只刷回一部分脏页，而不是刷回所有脏页。

InnoDB中触发fuzzy checkpoint的地方有：

- master thread checkpoint
- async/sync flush checkpoint
- flush_lru_list checkpoint

master thread checkpoint是指master thread以每秒/每10秒为单位将flush list中的脏页以一定比例异步刷回磁盘，用户线程不会阻塞。

async/sync flush checkpoint是指当redo log空间即将用尽时，需要强制将flush list的一部分脏页刷回磁盘，以腾挪出空闲的redo log空间：async的点位为75%，sync为90%，checkpoint_age = redo_file_lsn - last_checkpoint_lsn。即

-  < 75%，无动作
- 75% ~ 90%，异步刷回
- \> 90%，同步刷回

这种方式是保证重做日志的可用性（可循环使用），保证宕机可恢复性，也是为了做到D。

flush_lru_list checkpoint是指InnoDB要保证LRU链表和free链表中至少要保留一些页（BUF_FLUSH_FREE_BLOCK_MARGIN + BUF_FLUSH_EXTRA_MARGIN个可被立即使用的页（146））。因此，在页被读取后，都会调用buf_flush_free_margin保证这点。如果free链表中没有足够的数量，就需要从LRU链表中进行脏页的刷回，一旦完成刷回，页就被放入LRU链表的尾部，以保证下次页的分配使用。

{{< hint danger>}}

这里留给读者一个小问题：可能出现flush的dirty page位于flush list中部，而不是按flush list刷脏吗？会有问题吗？

{{</hint>}}

## 异步刷脏

从上面可以看出，如果脏页堆积，则会对性能造成严重的影响，所以InnoDB需要高效的刷回flush链表中的脏页（异步刷脏），尽可能的避免用户线程参与刷脏（同步刷脏）。在MySQL 5.6之前，刷脏的工作是由master thread负责的。在MySQL 5.6之后，刷脏交由专门的page cleaner线程负责。为了提高刷脏效率，在MySQL 5.7.4中，采用多个page cleaner进行刷脏（[innodb_page_cleaners](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_page_cleaners)：默认4 1C3W）。

我们下面以MySQL 5.7.4后的多线程刷脏为例来介绍异步刷脏机制。

page cleaner线程分为一个协调线程和多个工作线程，协调线程本身也是工作线程，也需要进行刷脏的工作。工作队列长度为缓冲池instance的个数，使用一个全局slot数组表示（page_clean_t→slots: page_cleaner_slot_t）。page cleaner线程并未和缓冲池instance绑定，而是每次进入REQUESTED状态后，寻找一个空闲的slot进行刷脏。

其中的核心数据结构是page_cleaner_t、page_cleaner_slot_t和page_cleanere_state_t。

### 整体纵览

我们以一张图的方式从整体和细节上来纵览：

![InnoDB_buffer_pool_flush_page](/InnoDB_buffer_pool_flush_page.png)



# MySQL 8.0改进

MySQL在宕机时，会生成巨大的core文件。在MySQL 8.0.14中，引入了[innodb_buffer_pool_in_core_file](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_buffer_pool_in_core_file)在core文件中剔除缓冲池页面，极大的缩小了core文件的大小。（需要linux kernel 3.4以上，支持MADV_DONTDUMP non-POSIX extension to madvise()）

把全局大锁buffer pool mutex拆分了，各个链表由其专用的mutex保护，大大提升了访问扩展性。实际上这是由percona贡献给上游的，而percona在5.5版本就实现了这个特性（[WL#8423](https://dev.mysql.com/worklog/task/?spm=a2c4e.10696291.0.0.6a8919a4Tczoe3&id=8423): InnoDB: Remove the buffer pool mutex 以及 [bug#75534](https://bugs.mysql.com/bug.php?spm=a2c4e.10696291.0.0.21b619a4njE2AI&id=75534)）。

原来的一个大mutex被拆分成多个为free_list, LRU_list, zip_free, 和zip_hash单独使用mutex:

- LRU_list_mutex for the LRU_list;
- zip_free mutex for the zip_free arrays;
- zip_hash mutex for the zip_hash hash and in_zip_hash flag;
- free_list_mutex for the free_list and withdraw list.
- flush_state_mutex for init_flush, n_flush, no_flush arrays.

由于log system采用lock-free的方式重新实现，flush_order_mutex也被移除了，带来的后果是flush list上部分page可能不是有序的，进而导致checkpoint lsn和以前不同，不再是某个log record的边界，而是可能在某个日志的中间，给崩溃恢复带来了一定的复杂度（需要回溯日志）

log_free_check也发生了变化，当超出同步点时，用户线程不再自己去做preflush，而是通知后台线程去做，自己在那等待(log_request_checkpoint), log_checkpointer线程会去考虑log_consider_sync_flush，这时候如果你打开了参数innodb_flush_sync的话, 那么flush操作将由page cleaner线程来完成，此时page cleaner会忽略io capacity的限制，进入激烈刷脏

8.0还增加了一个新的参数叫innodb_fsync_threshold，，例如创建文件时，会设置文件size,如果服务器有多个运行的实例，可能会对其他正常运行的实例产生明显的冲击。为了解决这个问题，从8.0.13开始，引入了这个阈值，代码里在函数os_file_set_size注入，这个函数通常在创建或truncate文件之类的操作时调用，表示每写到这么多个字节时，要fsync一次，避免对系统产生冲击。这个补丁由facebook贡献给上游。

其他 当然也有些辅助结构来快速查询buffer pool:

adaptive hash index: 直接把叶子节点上的记录索引了，在满足某些条件时，可以直接定位到叶子节点上，无需从根节点开始扫描，减少读的page个数 page hash: 每个buffer pool instance上都通过辅助的page hash来快速访问其中存储的page，读加s锁，写入新page加x锁。page hash采用分区的结构，默认为16，有一个参数innodb_page_hash_locks，但很遗憾，目前代码里是debug only的，如果你想配置这个参数，需要稍微修改下代码，把参数定义从debug宏下移出来 change buffer: 当二级索引页不在时，可以把操作缓存到ibdata里的一个btree(ibuf)中，下次需要读入这个page时，再做merge；另外后台master线程会也会尝试merge ibuf。
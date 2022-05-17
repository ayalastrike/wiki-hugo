---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

存储管理是数据库系统最基本的功能，本章将介绍InnoDB存储引擎中数据在外存中的组织形式（即数据文件）、数据文件在内存中的管理方式以及数据在文件层次的读/写。

# 存储组织

存储管理需要解决以下2个问题：

- 如何在存储设备上表示和组织数据库中的数据
- 如何在内存和外存上控制数据的存取

一般来说，DBMS不直接使用操作系统提供的文件系统作为直接的存储，而是在文件系统之上封装了一层自己对于存储设备的管理，以保证数据的完整性和存取效率。目前大部分的文件系统即使提供了日志的支持，可以保证写的原子性，但是数据库的数据页可能大于文件系统中的block大小，仍然避免不了出现的半写（partial-write）问题，所以数据库也要解决半写问题。

## 存储层次

为了解决以上两个问题，根据分层和抽象的思想，存储的组织和管理可以划分为3个层次：

- 文件存储（file storage）
- 页（page layout）
- 记录（tuple layout）

{{< hint info >}}
暂不讨论raw device，这里假设数据库的文件存放在操作系统提供的文件系统之上。

{{</hint>}}

文件存储负责将文件系统上的一组物理文件进行逻辑组织，按照使用场景（日志、数据、临时数据...）、以及数据IO特征的不同（日志是append-only的，数据可能是原地写/append-only），将物理文件进行划分，抽象出表空间，并形成统一的数据库层的逻辑文件子系统。

根据局部性原理，为了I/O的高效，我们需要将数据聚簇并划块，这就形成了页，所以第二层可以理解为页的集合，负责按照页为单位从外存、内存读写数据，并分配使用空间。

对于用户来说，操作的是数据，也就是记录的集合，那么在微观上需要对记录进行操作。这也是最后一层，我们在这层实现数据的更改和查询。

所以，存储引擎中内部存储单元（页）和操作系统、以及磁盘上的物理存储单元关系如下图所示：

![InnoDB_storage_page_block_sector](/InnoDB_storage_page_block_sector.png)

## 文件存储

文件存储负责将操作系统文件系统上的数据库相关物理文件组织在一起，形成数据库的逻辑文件子系统，整体结构如下图所示：

![InnoDB_storage_file_storage](/InnoDB_storage_file_storage.png)

从上图可以看到，不同的物理文件组成了多个表空间，然后一起形成file system子系统。官方表空间的描述在[这里](https://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace.html)。

InnoDB存储引擎对于文件的管理通过fil_system_t、fil_space_t、fil_node_t进行描述，并定义了四种文件空间（file space）类型：

- FIL_TYPE_TABLESPACE：持久表空间类型，包括系统表空间、独立表空间、undo表空间（innodb_system、user、innodb_undo<000>）
- FIL_TYPE_LOG：重做日志（ib_logfile<0~100>）
- FIL_TYPE_TEMPORARY：临时表空间（innodb_temporary）
- FIL_TYPE_IMPORT：只在导入表空间时使用（导入前为FIL_TYPE_IMPORT，导入完成后为FIL_TYPE_TABLESPACE）

每个文件空间可以包含若干个文件节点（file node）。file node是文件存储的最小单元。逻辑存储模块管理文件系统（file system）下的各个文件空间（file space），并对文件空间下的file node的读/写操作进行管理。

fil_system_t、fil_space_t、fil_node_t的关系如图所示：

![InnoDB_stroage_fil_system_t_fil_space_t_fil_node_t](/InnoDB_stroage_fil_system_t_fil_space_t_fil_node_t.png)

在file system中的name：

- 在file_space→name为文件名（1）或者文件名统称（n）
- file_node→name为具体的文件路径+文件名

比如重做日志中的file_space→name和file_node分别为：innodb_redo_log和path/ib_logfile<0 ~ 100>。

# 表空间

首先先让我们从用户和系统管理的角度来了解表空间的使用场景，以便对表空间有一个直观的认识。

## 使用场景

在MySQL 5.7中，表空间按照使用场景分为五种：

- 系统表空间
- 独立表空间
- 通用表空间
- undo表空间
- 临时表空间

### 系统表空间

系统表空间位于datadir下，存放了InnoDB存储引擎的核心信息，包括数据字典、事务系统信息、double write buffer、change buffer。如果没有配置独立的undo表空间，则undo日志也存放在这里（临时表空间也是如此）。同样，如果没有开启独立表空间（file-per-table），则用户表的数据和索引也存放在这里。因此，系统表空间也被称为共享表空间。

系统表空间由多个ibdata*的物理文件组成。默认情况下只会创建一个ibdata1文件（初始最小12M），也支持配置多个文件。系统表空间支持自动增长，也就是说，当空间不够时，会自动进行文件的增长（默认每次增长8M，即4个区），可以通过[innodb_autoextend_increment](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_autoextend_increment)设置每次增长的大小。

系统表空间的最大问题是空间无法收缩。因为其中存放了undo logs，如果遇到大事务，产生的undo logs会造成表空间的膨胀，即使purge undo log后空间也不会收缩（同样还有临时表空间）。同样的，用户表也无法收缩，即使删除了空间也无法释放。只能新建实例→导入数据这样的方式来重建表空间实现收缩。正是因为如此，MySQL随后划分出出独立的用户表空间、undo表空间和临时表空间。

### 独立表空间

用户表可以存放在独立表空间中，也可以多个表一起存放在通用表空间中（MySQL 5.7引入）。

独立表空间用于存放用户表的数据、索引和change buffer，物理文件创建在datadir/database/table_name.ibd。

好处

- 可以把表创建在不同的磁盘上，充分利用不同设备的IO性能
- 可以使用Barracuda格式，即行格式可以为DYNAMIC/COMPRESSED
- 可以使用import tablespace直接导入ibd
- 使用共享表空间，且innodb_flush_method = O_DIRECT时，无法并发写入一个文件

缺点

- fsync在共享表空间下可以合并IO，独立表空间会增加fsync的个数
- 打开的文件句柄变多
- 独立表空间的增长每次为4MB，不受innodb_autoextend_increment参数的控制
- 在删除表时，会扫描buffer pool，这会影响其他操作的效率

### 通用表空间

因为上述独立表空间的缺点，于是由发展出来通用表空间来兼顾独立表空间和共享表空间的优点。通用表空间可以将多个表放置到一个表空间中。支持Antelope/Barracuda格式，同时，同一表空间下可以支持不同的格式和数据页大小。

````
CREATE TABLESPACE `ts2` ADD DATAFILE 'ts2.ibd' FILE_BLOCK_SIZE = 8192 Engine=InnoDB;
CREATE TABLE t4 (c1 INT PRIMARY KEY) TABLESPACE ts2 ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
````

通用表空间不再局限于datadir，可以创建在数据目录以外。在这种情况下，会在datadir下创建一个.isl文件（软链接）来关联实际的ibd文件。

### undo表空间

undo log存储在undo段中，多个undo segments组成了undo表空间。

在共享表空间中，只能创建一个undo表空间。而采用独立的undo表空间后，可以创建多个undo表空间（后面废弃了）。

只能在MySQL install阶段开启undo表空间，这样才能用到MySQL 5.7引入的[online undo truncate](https://dev.mysql.com/doc/refman/5.7/en/innodb-undo-tablespaces.html)。

首先来看一下undo表空间的相关参数：

- innodb_undo_directory，指定单独存放undo表空间的目录，默认为.（即datadir），可以设置相对路径或者绝对路径。该参数实例初始化之后虽然不可直接改动，但是可以通过先停库，修改配置文件，然后移动undo表空间文件的方式去修改该参数；
- innodb_undo_tablespaces，指定单独存放的undo表空间个数，例如如果设置为3，则undo表空间为undo001、undo002、undo003，每个文件初始大小默认为10M。该参数我们推荐设置为大于等于3，原因下文将解释。该参数实例初始化之后不可改动；
- innodb_undo_logs，指定回滚段的个数（早期版本该参数名字是innodb_rollback_segments），默认128个。每个回滚段可同时支持1024个在线事务。这些回滚段会平均分布到各个undo表空间中。该变量可以动态调整，但是物理上的回滚段不会减少，只是会控制用到的回滚段的个数。
- innodb_undo_tablespaces>=2。因为truncate undo表空间时，该文件处于inactive状态，如果只有1个undo表空间，那么整个系统在此过程中将处于不可用状态。为了尽可能降低truncate对系统的影响，建议将该参数最少设置为3；
- innodb_undo_logs>=35（默认128）。因为在MySQL 5.7中，第一个undo log永远在系统表空间中，另外32个undo log分配给了临时表空间，即ibtmp1，至少还有2个undo log才能保证2个undo表空间中每个里面至少有1个undo log；
- innodb_max_undo_log_size，undo表空间文件超过此值即标记为可收缩，默认1G，可在线修改；
- innodb_purge_rseg_truncate_frequency,指定purge操作被唤起多少次之后才释放rollback segments。当undo表空间里面的rollback segments被释放时，undo表空间才会被truncate。由此可见，该参数越小，undo表空间被尝试truncate的频率越高。

参数可以设置如下：

````
innodb_max_undo_log_size = 100M
innodb_undo_log_truncate = ON
innodb_undo_logs = 128
innodb_undo_tablespaces = 3
innodb_purge_rseg_truncate_frequency = 10
````

### 临时表空间

用户创建的临时表和磁盘临时表都会造成共享表空间的膨胀，由此发展出临时表空间，所有非压缩的临时表都存放在这个表空间中。

临时表空间支持自动增长，重启后，临时表空间会重新创建。

对于云服务提供商而言，通过ibtmp文件，可以更好的控制临时文件产生的磁盘存储。

## 设计

文件系统子系统对于整个的存储管理具有重要的意义，用于保证内存和外存的物理一致性。对于外存（即和文件系统的交互），涉及到文件IO（文件句柄、flush、）、数据一致性、性能；对于内存，涉及到快速查找、并发控制。更为重要和复杂的，则是文件内部数据的组织和管理。

### space id的分配策略

space id的分配策略为：掐头去尾，中间动态分配的方式，并且目前未采用回收重用机制（space_id_reuse_warned = N/A）：

| 0          | 中间   | SRV_LOG_SPACE_FIRST_ID     |
| :--------- | :----- | :------------------------- |
| 系统表空间 | 可分配 | 日志表空间（0xFFFFFFF0UL） |

在InnoDB启动时，space id从系统表空间（space_id = 0的第0个文件的第7个页面，数据字典）中获取，其后在运行时动态产生：

````
dict_hdr_get_new_id
    fil_assign_new_space_id
````

并且，系统表空间和日志表空间要保证始终打开，以避免死锁（mutex）。

这是因为，在fil_io的调用中，首先需要持有文件系统的锁，如果在一个线程里，首先写日志，持有了，然后在insert buffer中，需要读取系统表空间，又试图获取锁，这种情况下会产生一个人获取两次，从而产生死锁。

````c++
fil_mutex_enter_and_prepare_for_io
        if (space_id == 0 || space_id >= SRV_LOG_SPACE_FIRST_ID) {
            /* We keep log files and system tablespace files always
            open; this is important in preventing deadlocks in this
            module, as a page read completion often performs
            another read from the insert buffer. The insert buffer
            is in tablespace 0, and we cannot end up waiting in
            this function. */
            return;
        }
````

### 文件空间类型

文件空间类型共有4种：

````
/** temporary tablespace (temporary undo log or tables) */
FIL_TYPE_TEMPORARY,
/** a tablespace that is being imported (no logging until finished) */
FIL_TYPE_IMPORT,
/** persistent tablespace (for system, undo log or tables) */
FIL_TYPE_TABLESPACE,   
/** redo log covering changes to files of FIL_TYPE_TABLESPACE */
FIL_TYPE_LOG
````

其中除了FIL_TYPE_LOG都属于数据表空间，由函数fil_type_is_data判断。

### tablespace flags

存储了表空间的元信息标识，比如对于行格式为compact/redundant，该值为0，对于compressed/dynamic，该值有效。

对于undo page，除了页大小外，都是false。

其中元信息如下：

- FSP_FLAGS_MASK_POST_ANTELOPE：使用的是antelope行格式
- FSP_FLAGS_MASK_ZIP_SSIZE：压缩页的block size（0为表空间不采用压缩）
- FSP_FLAGS_MASK_ATOMIC_BLOBS：使用的是Compressed/Dynamic行格式
- FSP_FLAGS_MASK_PAGE_SSIZE：page size
- FSP_FLAGS_MASK_DATA_DIR：显式指定了data_dir
- FSP_FLAGS_MASK_SHARED：是否为共享表空间
- FSP_FLAGS_MASK_TEMPORARY：是否是临时表空间
- FSP_FLAGS_MASK_ENCRYPTION：是否为加密表空间（MySQL 5.7.11引入）

### 快速查找

从最上面的图上我们可以快速进行以下查找：

- 通过file_system快速查找file_space by id/name：fil_system→spaces、fil_system→name_hash
- 通过file_system遍历所有的file_space：fil_system→space_list
- 通过file_system遍历IO操作要flush的file_space：fil_system->unflushed_spaces
- 通过file_system遍历所有变化的file_space：fil_system→named_spaces
- 通过file_system遍历LRU列表：fil_system→LRU
- 通过file_space遍历所有的file_node：fil_space→chain

### LRU

频繁文件的句柄打开和关闭是高耗时操作，并且，句柄也是珍贵的资源，当一个文件节点上的操作完成时，该文件节点不会立即被关闭，而是加入到file_system的LRU链表中。

### named_spaces

named_spaces用于记录在上一次CP后有哪些file_space修改过，并通过redo log（MLOG_FILE_NAME）记录下这些变化，max_lsn记录了其MLOG_FILE_NAME的lsn点位（每次mtr commit时，都推进max_lsn到其当时的log_sys→lsn）。

如果在CP时还没有生成这部分信息（MLOG_FILE_NAME），需要在下一次CP时首先生成这部分信息，正常情况下在mtr prepare_write时已写入redo log buffer，同时file_space→max_lsn为0，调用函数为fil_names_dirty_and_write。

采用MLOG_FILE_NAME记录更改的文件列表是因为在recovery时，需要在apply page变更之前确保打开所有的文件，然后可以进行并行恢复，而不被打断。

### flush

在page回写磁盘后，需要调用fsync来保证磁盘上数据页的持久性。

{{< hint warning>}}

数据文件的IO为O_DIRECT, 但每次修改后依然需要去做fsync来持久化元数据信息，但是对于某些文件系统而言并没有必要做fsync，因此MySQL 5.7加入了新的选项：O_DIRECT_NO_FSYNC，这个需求来自于facebook. 他们也对此做了特殊处理：除非文件size变化，否则不做fsync。（最近在buglist上对这个参数是否安全的讨论也很有意思，官方文档做了新的说明，感兴趣的可以看看 [O_DIRECT_NO_FSYNC poss](#)

因此引入了file_node→flush_size

````c++
for loop in space->space_chain {
	/* Skip flushing if the file size has not changed since
	   last flush was done and the flush mode is O_DIRECT_NO_FSYNC */
	if (fbd && (node->flush_size == node->size)) {
		continue;
	}
}
````

{{</hint>}}

#### unflushed list

因为支持异步IO，所以需要通过unflushed_spaces来跟踪未flush的file_space，即在write时计入（此时modification_counter > flush_counter），在flush（fsync）后移除。

其中的点位信息有：

- modification_counter 全局点位（推进）：file_system→modification_counter 作为单调递增的IO计数器，当IO操作开始时++，并将该IO点位更新到file_node→modification_counter
- modification_counter 文件点位（开始）：file_node  →modification_counter
- flush_counter        文件点位（结束）：fil_flush后将flush_counter更新为modification_counter。如果采用Direct IO禁用了fsync（SRV_UNIX_O_DIRECT_NO_FSYNC），当fil_node→flush_size == fil_node→size，则直接更新file_node→flush_counter而无需flush。

{{< hint info>}}

file_system和file_node的点位信息无需持久化，在重启后置零，保证在运行时（内存态）单调递增即可

{{</hint>}}

#### group fsync

在文件sync时，也采用group fsync的方式来减少fsync的代价。在fil_flush中，首先抢到file_system→mutex的负责group fsync，其他未抢到的则通过node→modification_counter是否和file_node→flush_counter来判断是否已经被leader fsync，从而可以跳过fsync。

### 表空间操作的并发控制

因为支持异步IO，所以在对表空间的并发控制，需要考虑异步操作（pending operations），所以这里将表空间支持的操作分为两类：

- 维护操作：包括delete、close、truncate、rename和extend
- 异步操作：读写page，IS统计信息、以及进行io操作（异步io起始结束、flush计数）

首先来看一下异步操作，又可以分为pending ops和pending ios，如下图所示：

![InnoDB_storage_pending_operations](/InnoDB_storage_pending_operations.png)

pending ops即为file_space.n_pending_ops，具体的+-时机如下：

````
在buffer pool从磁盘上同步读取page，或者change buffer修改page时，都需要将file_space的pending ops++--
buf_read_ahead_linear(计算space是否有hole)
buf_read_ahead_random(计算space是否有hole)
ibuf_merge_or_delete_for_page
lock_rec_block_validate(innodb monitor->lock_print_info_all_transactions)
lock_rec_fetch_page(buf_page_get_gen)
以及在IS展示统计信息时，需要将file_space的pending ops++--
````

pending io的指标体现在读写和flush上。

接着看一下维护操作，这里的操作指的是对表空间或者物理文件进行的操作，

- 表空间delete
- 表空间close：只用于import表空间时进行清理
- 表空间truncate：用于用户表空间和undo表空间
- 表空间rename
- 表空间extend：因为是对物理文件进行操作，所以flag设在了file_node上（file_node→being_extended）

在进行维护操作时，需要对操作进行互斥，并发控制如下图所示：

![InnoDB_storage_concurrent_operations](/InnoDB_storage_concurrent_operations.png)

## file_system子系统

文件子系统的整体关系图如下所示：

![InnoDB_storage_file_system-data_structure](/InnoDB_storage_file_system-data_structure.png)

### fil_system_t

fil_system_t用于表示文件子系统的逻辑结构，在InnoDB运行期间，内存中只有一个fil_system_t对象，统一管理所有文件的操作。

| 变量                  | 类型                             | 说明                                                         |
| :-------------------- | :------------------------------- | :----------------------------------------------------------- |
| mutex                 | ib_mutex_t                       | 保护fil_system_t                                             |
| spaces                | hash_table_t                     | 表空间hash table，实现对表空间按id的快速访问 <space->id, fil_space_t> |
| name_hash             | hash_table_t                     | 表空间hash table，实现对表空间按name的快速访问 <space->name, fil_space_t> |
| LRU                   | UT_LIST_BASE_NODE_T(fil_node_t)  | 文件系统最近打开的文件节点链表，维护此链表的目的是减少文件节点打开和关闭的次数 |
| unflushed_spaces      | UT_LIST_BASE_NODE_T(fil_space_t) | 在IO操作后记录需要flush的表空间链表                          |
| n_open                | ulint                            | 文件系统当前打开的文件节点数                                 |
| max_n_open            | ulint                            | 文件系统最大可以打开的文件节点数（[innodb_open_files](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_open_files)） |
| modification_counter  | int64_t                          | 全局写入点位（单调递增）                                     |
| max_assigned_id       | ulint                            | 分配的最大space_id(fil_space_t.id)                           |
| space_list            | UT_LIST_BASE_NODE_T(fil_space_t) | 文件空间链表，实现对文件空间的遍历访问                       |
| named_spaces          | UT_LIST_BASE_NODE_T(fil_space_t) | 在上一次CP后有哪些file_space产生了变化                       |
| space_id_reuse_warned | bool                             | /                                                            |

### fil_space_t

fil_space_t用于表示文件空间的逻辑结构。文件空间是有若干文件节点组成的一个逻辑文件，并不是指单个文件。对于每种文件空间类型，都有一个对应的fil_space_t。

| 变量                   | 类型                            | 说明                                                         |
| :--------------------- | :------------------------------ | :----------------------------------------------------------- |
| 变量                   | 类型                            | 说明                                                         |
| name                   | char*                           | file_space名称                                               |
| id                     | ulint                           | 表空间ID，规则见上                                           |
| max_lsn                | lsn_t                           | 与named_spaces相关，已flush为0，否则为MLOG_FILE_NAME的lsn点位 |
| stop_ios               | bool                            | 在rename表空间时设置，阻止其他操作                           |
| stop_new_ops           | bool                            | 在delete、close、truncate表空间时设置，阻止其他操作          |
| is_being_truncated     | bool                            | 在truncate表空间时设置，阻止其他操作                         |
| redo_skipped_count     | ulint                           |                                                              |
| purpose                | fil_type_t                      | 文件空间类型                                                 |
| chain                  | UT_LIST_BASE_NODE_T(fil_node_t) | 此文件空间中包含的文件节点链表                               |
| size                   | ulint                           | 表空间中所有文件节点的页的总数，文件空间的大小为size * 16K   |
| size_in_header         | ulint                           | 已使用的页的数量（space header.FSP_SIZE）                    |
| free_len               | ulint                           | 空闲区链表的长度（space header.FSP_FREE）                    |
| free_limit             | ulint                           | 已逻辑分配的位置（space header.FSP_FREE_LIMIT）              |
| flags                  | ulint                           | 表空间flags，可以计算出pageSize(space->flags)                |
| n_reserved_extents     | ulint                           | 为表空间操作预留的空闲区个数，比如B+树索引的节点分裂。这些操作会导致表空间的size增加，为了防止增加后超过表空间size的最大值，预先增加这么多个区 |
| n_pending_flushes      | ulint                           | pending flush计数器                                          |
| n_pending_ops          | ulint                           | pending op计数器                                             |
| hash                   | hash_node_t                     | fil_system_t.spaces的节点                                    |
| name_hash              | hash_node_t                     | fil_system_t.name_hash的节点                                 |
| latch                  | rw_lock_t                       | 对表空间的并发操作进行保护                                   |
| unflushed_spaces       | UT_LIST_NODE_T(fil_space_t)     | fil_system_t.unflushed_spaces链表节点                        |
| named_spaces           | UT_LIST_NODE_T(fil_space_t)     | fil_system_t.named_spaces链表节点                            |
| is_in_unflushed_spaces | bool                            | 标记是否未flush，如果该file_space在file_system->unflushed_spaces中，则为true |
| space_list             | UT_LIST_NODE_T(fil_space_t)     | fil_system_t.space_list链表节点                              |

当系统初始化时，会会为数据表文件、重做日志文件创建对应的fil_space_t，并加入到fil_system_t的space_list、spaces和name_hash中。

创建的物理文件会通过文件的第一个page（fsp page）中的space id和file space进行关联。

latch的作用是对表空间的并发操作进行保护，比如之前介绍的页、区、段的管理，都需要现持有表空间对应的latch（mtr_x_lock_space）。有一点特殊的是，对于临时表空间，因为临时表的创建是用户线程私有的，则不需要持有latch，即latch_t.m_temp_fsp = true。

{{< hint info >}}
新建的独立表空间的tablespace初始只有4个页：

- page 0 is the fsp header and an extent descriptor page,
- page 1 is an ibuf bitmap page,
- page 2 is the first inode page,
- page 3 will contain the root of the clustered index of the table we create here.

{{</hint>}}

### fil_node_t

fil_node_t用于文件节点的管理，以便于对文件节点进行读/写操作。

注意：对于不同的表空间类型，file_node（物理文件）的数量不同：日志表空间有多个物理文件；系统表空间，用户表空间、undo表空间和临时表空间：都只有一个物理文件对应。

| 变量                 | 类型                       | 说明                                                         |
| :------------------- | :------------------------- | :----------------------------------------------------------- |
| 变量                 | 类型                       | 说明                                                         |
| space                | fil_space_t*               | 文件节点所属的文件空间                                       |
| name                 | char*                      | 文件节点的名称 <路径+文件名>                                 |
| is_open              | bool                       | 文件是否已被打开                                             |
| handle               | pfs_os_file_t              | 文件节点打开后的fd                                           |
| sync_event           | os_event_t                 | group fsync时通知其他处于等待中的file node fil_flush()调用者 |
| is_raw_disk          | bool                       | 是否创建在raw device上（即用OS_FILE_OPEN_RAW创建）           |
| size                 | ulint                      | 文件节点的页数，为表空间前面所有file的size之和+该文件节点的size的相对偏移量（比如20+20+4）== |
| flush_size           | ulint                      | 用于支持O_DIRECT_NO_FSYNC，详见【2.2.7 flush】               |
| init_size            | ulint                      | 初始页数（FIL_IBD_FILE_INITIAL_SIZE = 4）                    |
| max_size             | ulint                      | 最大页数（ULINT_MAX）==                                      |
| n_pending            | ulint                      | pending io计数器                                             |
| n_pending_flushes    | ulint                      | pending flush的数量，在file node fil_flush时++–，同一时刻可能有其他fil node在fil_flush时等待flush完成 |
| being_extended       | bool                       | 是否正在扩展表空间                                           |
| modification_counter | int64_t                    | 文件准备写入点位（IO操作前）                                 |
| flush_counter        | int64_t                    | 文件完成写入点位（IO操作后）                                 |
| chain                | UT_LIST_NODE_T(fil_node_t) | fil_space_t.chain链表节点                                    |
| LRU                  | UT_LIST_NODE_T(fil_node_t) | fil_system_t.LRU链表节点                                     |
| punch_hole           | bool                       | 文件是否支持打洞                                             |
| block_size           | ulint                      | punch_hole开启情况下，洞的大小（即file的sector size）        |
| atomic_write         | bool                       | 是否支持FusionIO的原子写                                     |

一个文件节点必定在一个文件空间的chain链表上，但不一定在文件系统的LRU链表上。当一个文件节点上的操作完成时，该文件节点不会立即被关闭，而是加入到文件系统的LRU链表中。

## fsp0file & fsp0space & fsp0sysspace

fsp0file中定义了存储的物理表达形式，即物理文件。fsp0space和fsp0sysspace中定义了物理上表空间和系统表空间。

整体物理文件系统的层次组织如下：

![InnoDB_storage_fsp0file_fsp0space_fsp0sysspace](/InnoDB_storage_fsp0file_fsp0space_fsp0sysspace.png)

### Datafile

Datafile中的字段说明：

| 变量                             | 说明                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| m_name                           | 去掉路径名的文件名                                           |
| m_filepath                       | 路径+文件名+扩展名                                           |
| m_filename                       | 文件名+扩展名，指向m_filepath的内存区域                      |
| m_handle                         | fd                                                           |
| m_open_flags                     | 打开文件时的flags                                            |
| m_size                           | 该文件存储使用的page数量                                     |
| m_order                          | 该文件在tablespace中的顺序（0,1,2...） 只有系统表空间为n，其他表空间都为0 |
| m_type                           | NOT_RAW 不是raw partition NEW_RAW 需要进行初始化的raw partition OLD_RAW  已完成初始化的raw partition |
| m_space_id                       | 表空间ID                                                     |
| m_flags                          | 表空间的FSP_SPACE_FLAGS                                      |
| m_exists                         | 在启动时文件是否就已经存在                                   |
| m_is_valid                       | 表空间是否有效                                               |
| m_first_page m_first_page_buf    | 表空间第0页的内存缓存，分配2倍最大页的空间，两个指针指向到一个区域，只是用第一个指针 |
| m_atomic_write                   | 是否支持原子写                                               |
| m_last_os_error                  | 读写文件时的最后一次错误信息                                 |
| m_file_info                      | 文件的inode number                                           |
| m_encryption_key m_encription_iv | 表空间的加密key信息                                          |

### Remote Datafile

软链接文件

在datadir下创建一个软链接文件，即InnoDB Symbolic Link (ISL)，对于共享表空间，软链接文件位于datadir/basename.isl；对于独立表空间，软链接文件位于datadir/database/tablename.isl。

### Tablespace

使用create tablespace都会创建一个tablespace对象，其包括一组Datafile，即vector<Datafile> m_files。

### SysTablespace

继承Tablespace，并可以自动增长



# 文件IO

在InnoDB中，对于文件的IO操作分为运维操作和数据读写操作。其中运维操作是指创建、删除等针对文件的操作，而数据读写操作指的是对文件内部的数据进行读写。

运维操作这里不再详述，我们把精力主要集中在数据读写操作上，以下所提到的IO操作如果不做特殊说明，指的是数据的读写操作。

在进行数据读写时，用户线程通过同步IO进行读写数据，后台线程通过异步IO的方式来读写数据的。

同步IO和异步IO都是通过pread/pwrite来保证文件系统的并发读写正确性，区别只是实际的读写线程是用户线程还是IO线程。

由于早期的操作系统不支持原生的异步IO，所以InnoDB在早期通过模拟的形式来进行异步IO的操作，以提高性能。而在Windows和Linux支持native AIO后，基本上已经不再使用模拟异步IO的方式来读写数据了。

对于InnoDB来说，前台的用户操作和日志采用同步IO（checkpoint为异步IO），后台的数据操作采用异步IO，并且，异步IO也不是FIFO的模式。

文件IO的整体处理流如下图所示：

![InnoDB_storage_IO_data_flow](/InnoDB_storage_IO_data_flow.png)

更详细的函数调用如下图所示：

![InnoDB_storage_IO_process_flow](/InnoDB_storage_IO_process_flow.png)

## 异步IO

存储引擎传入的IO请求放入等待队列，通过唤醒独立的IO线程来同步处理这些IO请求，IO线程实际上通过同步的file read/write读取文件中的数据。这里的异步IOhandler有3种：

- Windows原生异步IO（Windows native AIO）

- Linux原生异步IO（Linux native AIO）
  
  通过libaio的方式进行异步读写
  
- 模拟异步IO
  
  通过os_aio_simulated_handler进行模拟异步读写。

数据结构如下图所示

![InnoDB_storage_aio](/InnoDB_storage_aio.png)

这里可以看到每个io thread都有256个slot，可以认为是每个io thread的并发数控制，如果超过，则后续的异步请求需要等待。

{{< hint info >}}

块设备层也有相应的io并发数控制：

The **nr_requests** is a parameter for block device, it controls maximum requests may be allocated in the block layer for read or write requests, the default value is 128. Occasionally, it may be suggested to adjust the value, generally speaking:

- Increasing the value will improve the I/O throughput, but will also increase the memory usage.
- Decreasing the value will benefit the real-time applications that are sensitive to latency, but it also decreases the I/O throughput.

{{</hint>}}

函数调用：

````
fil_io
	fil_mutex_enter_and_prepare_for_io 预处理
	找到page.space
	通过node->size和page_no找node
	fil_node_prepare_for_io				io prepare
	sync + fil_node_complete_io
	async + os_aio						发送aio request
		os_aio_func
			OS_AIO_SYNC os_file_read_func/os_file_write_func os_file_pread/pwrite 同步IO
			OS_AIO_LOG  日志
			OS_AIO_IBUF	change buffer read
			OS_AIO_NORMAL	其他
			select_slot_array
			fill slot
			wake up IO thread
````

## Punch Hole

punch hole （打洞）功能，就是可以把文件中间的一部分内容释放掉，但是剩余部分的文件偏移不变。

{{< hint info >}}

Hole Punch Size on Linux

On Linux systems, the file system block size is the unit size used for hole punching. Therefore, page compression only works if page data can be compressed to a size that is less than or equal to the InnoDB page size minus the file system block size. For example, if innodb_page_size=16K and the file system block size is 4K, page data must compress to less than or equal to 12K to make hole punching possible.

{{</hint>}}

### 什么是Punch Hole

在UNIX文件操作中，文件位移量可以大于文件的当前长度，在这种情况下，对该文件的下一次写将延长该文件，并在文件中构成一个空洞，这一点是允许的。位于文件中但没有写过的字节都被设为 0。

如果 offset 比文件的当前长度更大，下一个写操作就会把文件“撑大（extend）”。这就是所谓的在文件里创造“空洞（hole）”。没有被实际写入文件的所有字节由重复的 0 表示。空洞是否占用硬盘空间是由文件系统（file system）决定的。大部分文件系统是不占用的。

### 怎样获得一个Punch Hole文件

以Linux来说，**使用lseek或truncate到一个固定位置生成的“空洞文件”是不会占据真正的磁盘空间的**。 

空洞文件特点就是offset大于实际大小，也就是说一个文件的两头有数据而中间为空，以‘\0‘填充。那文件系统会不会不做任何处理的将其存放在硬盘上呢？大部分文件系统是不会将其存放在硬盘上。

### 文件预留

为什么需要文件预留

在开发过程中有时候需要为某个文件快速地分配固定大小的磁盘空间，为什么要这样做呢？

1. 可以让文件尽可能的占用连续的磁盘扇区，减少后续写入和读取文件时的磁盘寻道开销
2. 迅速占用磁盘空间，防止使用过程中所需空间不足
3. 后面再追加数据的话，不会需要改变文件大小，所以后面将不涉及metadata的修改

前面提到使用lseek或truncate到一个固定位置生成的“空洞文件”是不会占据真正的磁盘空间的。

快速的为某个文件分配实际的磁盘空间在Linux下可通过fallocate（对应的posix接口为posix_fallocate）系统调用来实现，大部分主流文件系统如ext4，xfs还是支持fallocate。

### 文件打洞

最近遇到了这样的一种需求，一个大文件中的某段范围的内容已经失效了，想把这段失效的文件部分所占用的磁盘空间还给文件系统。linux下可以通过fallocate实现归还一个文件所占用的部分磁盘空间。

````
#include <fcntl.h>
int fallocate(int fd, int mode, off_t offset, off_t len);
````

fd就是open产生的文件描述符，offset就是进行fallocate的文件偏移位置，len为fallocate的的长度。offset和len一起构成了要释放的文件范围。

重点介绍的是mode，它决定了fallocate的行为。

- Allocating disk space 

  这是默认的操作，对应mode等于0。它所作的工作是如果分配从offset开始到offset+len的一段空间，这个是真的分配磁盘空间，不是hole，新分配的空间以0填充数据。当然这个操作一般在offset+len大于现有文件长度时才会起到增加文件数据空间的作用。 
  一般情况下新增加空间后文件的size也会随着调整，但是有一个特殊情况，就是当FALLOC_FL_KEEP_SIZE出现在mode中时，在增加文件空间后不会改变文件的size。这样的操作算是一种在文件结尾处的预分配，对于后期的append写入操作有优化作用。 
  （但遗憾的是ubuntu 12.04 ext4文件系统好像并不支持fallocate的预分配）

- Deallocating file space 

  释放文件的某段范围的磁盘空间 （文件打洞） 

  FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE 

  此mode虽然并不会改变文件的大小，但其实却释放了offset和len所在范围的磁盘块，将它们归还给了文件系统。fallocate成功后，后续对offset和len所在的文件范围进行读操作，将会读到0。

## FusionIO

FusionIO公司（2014年被SanDisk收购）在2012年声称他们的基于flush memory的存储产品，可以提供百万IOPS，这款产品和SSD的区别可以参看[这篇文章](https://blog.51cto.com/alanwu/865235)。随后[MariaDB](https://mariadb.com/kb/en/fusion-io-introduction/)也提供了FusionIO设备的支持，即对文件句柄加上DFS_IOCTL_ATOMIC_WRITE_SET的ioctl标记为，就可以启用文件的原子写特性，这样，就不会出现半写的问题，也就不再需要double write。

在MySQL中，随后也做了相应的支持，代码如下：

````c++
#if !defined(NO_FALLOCATE) && defined(UNIV_LINUX)

#include <sys/ioctl.h>
/** FusionIO atomic write control info */
#define DFS_IOCTL_ATOMIC_WRITE_SET	_IOW(0x95, 2, uint)

/**
Try and enable FusionIO atomic writes.
@param[in] file		OS file handle
@return true if successful */
bool
fil_fusionio_enable_atomic_write(pfs_os_file_t file)
{
	if (srv_unix_file_flush_method == SRV_UNIX_O_DIRECT) {

		uint	atomic = 1;
		ut_a(file.m_file != -1);
		if (ioctl(file.m_file, DFS_IOCTL_ATOMIC_WRITE_SET, &atomic) != -1) {

			return(true);
		}
	}

	return(false);
}
#endif /* !NO_FALLOCATE && UNIV_LINUX */
````

Fusion IO的高性能一方面是通过PCI-E通道进行数据的传输，一方面在IO路径上也和传统的存储设备不一样，如下图所示：

![InnoDB_storage_FusionIO](/InnoDB_storage_FusionIO.png)

## InnoDB文件操作

### 文件的并发访问控制

InnoDB需要通过文件系统的文件锁来保证只有一个进程（mysqld）对某个文件进行读写操作（os_file_lock），在实现中使用了建议锁（Advisory locking），而不是强制锁（Mandatory locking），因为强制锁在不少系统上（包括linux）有bug。在非只读模式下，所有文件打开后，都用文件锁锁住，以保护只有mysqld进程对文件的正确访问。

{{< hint info >}}

os_file_lock使用了fcntl+F_WRLCK的方式对文件加排他锁

{{</hint>}}

### 建目录

InnoDB中目录的创建使用递归的方式(os_file_create_subdirs_if_needed和os_file_create_directory)。例如，需要创建/a/b/c/这个目录，先创建c，然后b，然后a，创建目录调用mkdir函数。此外，需要注意创建目录上层需要调用os_file_create_simple_func函数，而不是os_file_create_func。

### 临时文件

InnoDB也需要临时文件，临时文件的创建逻辑比较简单(os_file_create_tmpfile)，就是在tmp目录下成功创建一个文件后直接使用unlink函数释放掉句柄，这样当进程结束后（不管是正常结束还是异常结束），这个文件都会自动释放。InnoDB创建临时文件，首先复用了server层函数mysql_tmpfile的逻辑，后续由于需要调用server层的函数来释放资源，其又调用dup函数拷贝了一份句柄。

### 其他文件操作

如果需要获取某个文件的大小，InnoDB并不是去查文件的元数据(stat函数)，而是使用lseek(file, 0, SEEK_END)的方式获取文件大小，这样做的原因是防止元信息更新延迟导致获取的文件大小有误。

InnoDB会预分配一个大小给所有新建的文件(包括数据和日志文件)，预分配的文件内容全部置为零(os_file_set_size)，当前文件被写满时，再进行扩展。此外，在日志文件创建时，即install_db阶段，会以100MB的间隔在错误日志中输出分配进度。

# 表空间的磁盘组织结构

对于物理文件，内部也需要通过高效的数据组织来有效的利用文件空间。所以，接下来我们深入文件内部，来看一下内部的数据组织形式。

从上面可以看到，物理文件隶属于不同的表空间，不同的表空间，除了日志表空间外，都有统一的page layout（具有通用的FIL_HEADER和FIL_TRAIL），固定的page size（对于压缩表，可以在建表时指定block size，但在内存中表现的解压页依旧为统一的页大小），并普遍使用B+树来管理数据。

在表空间的管理上，首先表空间是由一个或多个物理文件组成的，一个物理文件按照区（extent）来进行管理，区是物理上连续分配的一段空间，由64个页组成，用于存储实际的数据，分配给数据的区逻辑上叫做段（分为non-leaf segment和leaf segment以及碎片页）。

{{< hint danger>}}

日志表空间的log block大小为512字节，这时为了保证日志页写入的原子性。因为现代的存储设备（SSD）中的sector size已经为4K了，所以将log block size对齐到4K可以避免read-modify-write现象。MySQL通过innodb_log_write_ahead_size设置来避免read-modify-write，而Percona则直接设置log block size的值。

MySQL:

Setting the [innodb_log_write_ahead_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_log_write_ahead_size) value too low in relation to the operating system or file system cache block size results in read-on-write. Setting the value too high may have a slight impact on fsync performance for log file writes due to several blocks being written at once.

Percona

[Effect from innodb log block size 4096 bytes](https://www.percona.com/blog/2011/01/03/effect-from-innodb-log-block-size-4096-bytes/)

**read-modify-write**

当要向硬盘写的数据小于4K时，会先把4k的一个扇区的数据读出，修改相应的部分后在写入。这个过程称之为read-modify-write (RMW)，也叫写放大。

![InnoDB_storage_read-modify-write_(RMW)](/InnoDB_storage_read-modify-write_RMW.png)

{{</hint>}}

## 页

对于InnoDB存储引擎，数据文件最小的存储单位是页（默认16K），在页的基础上逻辑的划分为区（extent）、段（segment），并形成一个逻辑上的统一整体：表空间（tablespace）。

同时，为了使数据库获得更好的I/O性能，InnoDB存储引擎对于空间的申请不是按照页，而是按照区的方式，一次64页（1MB）的单位申请。这样的目的是：提高空间申请的效率、一定程度上保证磁盘上数据存放的顺序性。

另外，page size的大小设置也和存储设备的IO性能息息相关，存储设备慢，page size可以设的大一些，以保证IO操作可以读取到更多的数据。

页是InnoDB访问磁盘的最小I/O单元。页的默认大小是16K（UNIV_PAGE_SIZE）。除去页头和页尾的元数据，页的绝大部分空间用来存储数据，页的结构如图所示：

![InnoDB_storage_page_layout](/InnoDB_storage_page_layout.png)

从上图中可以看到，页有不同的类型，但都有统一固定的头部（page header 38 bytes）和尾部（page trailer 8 bytes）。通过space_id + page_offset可以定位页的具体位置（FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID + FIL_PAGE_OFFSET）。由于FIL_PAGE_OFFSET的长度是4个字节，所以一个表空间的最大存储空间是64TB（231*2*16K）。索引页之间的逻辑顺序（需要保证有序性：键值顺序）通过双向链表指针连接起来（FIL_PAGE_PREV、FIL_PAGE_NEXT），即内部存储的是前页/后页在表空间中的偏移量。

{{< hint info >}}

页的位置和页之间的位置都通过offset来存储，即相对位置存储的好处是，当表空间数据移动时不会受到影响，如果存储的是绝对位置，则需要进行变更。

{{</hint>}}

FIL_HEADER和FIL_TRAIL的字段如下：

|                                  | 字段                        | 大小                                                         | 说明                                                         |
| :------------------------------- | :-------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| FIL_PAGE_DATA（38）              | FIL_PAGE_SPACE_OR_CHKSUM    | 4                                                            | checksum                                                     |
| FIL_PAGE_OFFSET                  | 4                           | 页号，也是页在表空间中的偏移量                               |                                                              |
| FIL_PAGE_PREV                    | 4                           | 前一个页的偏移量（仅对索引页有效）                           |                                                              |
| FIL_PAGE_NEXT                    | 4                           | 后一个页的偏移量（仅对索引页有效）                           |                                                              |
| FIL_PAGE_LSN                     | 8                           | 页LSN                                                        |                                                              |
| FIL_PAGE_TYPE                    | 2                           | 页类型                                                       |                                                              |
| FIL_PAGE_FILE_FLUSH_LSN          | 8                           | 仅在系统表空间的第1个页（0,0）中使用，用来判断数据库是否正常关闭 |                                                              |
| FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID | 8                           | 表空间ID（缓冲池中靠space_id+page_no来标识页）               |                                                              |
| FIL_PAGE_DATA_END（8）           | FIL_PAGE_END_LSN_OLD_CHKSUM | 8                                                            | 前4个字节存放checksum，后4个字节存放FIL_PAGE_LSN的后4个字节，用于检测页是否损坏 |

{{< hint warning>}}

在数据库正常关闭时，会做一次checkpoint，并把CP lsn写入到页（0,0）中。这样，在数据库恢复时，根据这两个值是否相等来判断是否为正常关闭。

{{</hint>}}

{{< hint warning>}}

**判断页面是否损坏**

通过FIL_PAGE_END_LSN_OLD_CHKSUM来做到"前后呼应"，判断页面的修改是否是完整的：

1. 通过后4个字节，检查和FIL_PAGE_LSN的后半部分（后4个字节）是否一致
2. 通过页面内容来计算出checksum，用前4个字节以及页头的checksum（FIL_PAGE_SPACE_OR_CHKSUM）是否一致

{{</hint>}}

{{< hint info>}}

在页面头上记录一个标志，如时间戳，然后在页面尾上也记录一个相同的标志，当读取数据页面后，只要检查一下标志是否相同即可证明页面是否存在部分写。需要注意的是每次页面更新后，标志也必须同时进行修改，不能使用原有页面的标志。还有一些代价更大的方法，如生成整个页面的摘要，然后记录到页头中，除了可以检查页面的部分写之外，还能防止页面被恶意篡改，但这对系统资源的消耗比较大。一种热衷的办法是计算部分页面数据的摘要，但要包含页面头和页面尾，这样就可以能检测页面的部分写，也能部分达到防止篡改的目的。

{{</hint>}}

其中页类型如下：

````
#define FIL_PAGE_INDEX      17855           /*!< B-tree node */
#define FIL_PAGE_RTREE      17854           /*!< B-tree node */
#define FIL_PAGE_UNDO_LOG       2           /*!< Undo log page */
#define FIL_PAGE_INODE          3           /*!< Index node */
#define FIL_PAGE_IBUF_FREE_LIST 4           /*!< Insert buffer free list */
/* File page types introduced in MySQL/InnoDB 5.1.7 */
#define FIL_PAGE_TYPE_ALLOCATED 0           /*!< Freshly allocated page */
#define FIL_PAGE_IBUF_BITMAP    5           /*!< Insert buffer bitmap */
#define FIL_PAGE_TYPE_SYS       6           /*!< System page */
#define FIL_PAGE_TYPE_TRX_SYS   7           /*!< Transaction system data */
#define FIL_PAGE_TYPE_FSP_HDR   8           /*!< File space header */
#define FIL_PAGE_TYPE_XDES      9           /*!< Extent descriptor page */
#define FIL_PAGE_TYPE_BLOB      10          /*!< Uncompressed BLOB page */
#define FIL_PAGE_TYPE_ZBLOB     11          /*!< First compressed BLOB page */
#define FIL_PAGE_TYPE_ZBLOB2    12          /*!< Subsequent compressed BLOB page */
#define FIL_PAGE_TYPE_UNKNOWN   13          /*!< In old tablespaces, garbage in FIL_PAGE_TYPE is replaced with this value when flushing pages. */
#define FIL_PAGE_COMPRESSED     14          /*!< Compressed page */
#define FIL_PAGE_ENCRYPTED      15          /*!< Encrypted page */
#define FIL_PAGE_COMPRESSED_AND_ENCRYPTED 16/*!< Compressed and Encrypted page */
#define FIL_PAGE_ENCRYPTED_RTREE 17         /*!< Encrypted R-tree page */
````

## 区

页是innoDB访问和存储的最小单位，区是InnoDB申请空间的最小单位，一个区由64个连续的页组成，大小为1MB。区的管理和分配由FSP_HDR页（第0页）和XDES页共同完成，其中表空间的元信息存储在FSP_HDR页的space header中，占用112个字节。

区分为可用区（用于分配给段 extent）和碎片区（frag extent），可用区通过保存在空闲区链表（FSP_FREE）中。InnoDB为了节约存储空间，数据首先保存在32个碎片页中，碎片页从碎片区中分配，超过32个碎片页后，再以区的方式从空闲区链表中申请空间。碎片区不属于任何段，保存在碎片半满区链表（FSP_FREE_FRAG）和碎片全满区链表（FSP_FULL_FRAG）中。

表空间可以看做是由多个区组成的一个“大文件块”，使用时按照从低到高的页偏移量顺序地进行区空间的申请。FSP_FREE_LIMIT表示当前已经已经申请到的位置。超过FSP_FREE_LIMIT的部分表示区还未进行初始化。

space header保存的元信息如下：

| 字段                | 大小 | 说明                                                         |
| :------------------ | :--- | :----------------------------------------------------------- |
| FSP_SPACE_ID        | 4    | 表空间ID，由数据字典分配                                     |
| FSP_NOT_USED        | 4    | -                                                            |
| FSP_SIZE            | 4    | 表空间总的page数量，扩展文件时需要更新（fsp_try_extend_data_file_with_pages），物理分配界限 |
| FSP_FREE_LIMIT      | 4    | 未分配的最小page no，该offset之后的都尚未加到空闲区链表（FSP_FREE）上，逻辑分配界限 |
| FSP_SPACE_FLAGS     | 4    | 表空间flags                                                  |
| FSP_FRAG_N_USED     | 4    | 碎片区（FSP_FREE_FRAG）中已经使用的页的数量，每当从FSP_FREE_FRAG分配一个空闲页出去时，+1，可快速计算表空间的可用碎片页数 |
| FSP_FREE            | 16   | 空闲区链表，用户从这里以区为单位申请加入到段链表中           |
| FSP_FREE_FRAG       | 16   | 碎片半满区链表，该链表中的区中的页要么属于不同的段，要么还未分配 |
| FSP_FULL_FRAG       | 16   | 碎片全满区链表，当由page从该链表的区中释放时，则将该区移回碎片半满区链表 |
| FSP_SEG_ID          | 8    | 下一个段的ID，在表空间中，每个段都有唯一的编号，即段ID，每次分配段后+1 |
| FSP_SEG_INODES_FULL | 16   | 已经完全用满的segment inode页链表（也称为段inode全满页链表） |
| FSP_SEG_INODES_FREE | 16   | 至少存在一个空闲的segment inode entry的segment inode页链表（也称为段inode未满页链表） |

{{< hint info>}}

对于用户表空间来说，当小于64个页时，FSP_FREE_LIMIT却为64（因为区是分配的最小粒度），但是物理上并没有分配这么多的空间，可能只分配了4个页（初始状态）。

{{</hint>}}

区空间的申请通过函数fsp_fill_free_list实现，如果空间大小允许，每次申请4个区（FSP_FREE_ADD），如果申请的区包含碎片区，则申请5个区。申请的区的信息更新到space header。

每个区包含了64个页，由区描述符（XDE entry）表示。每 个区（意味着256个XDE entry）存储在一个XDE page里。

**区描述符**

区描述符（XDES extent descriptor 40 bytes）使用位图表示64个页的使用状态，每个页的状态占用2位（XDES_FREE_BIT/XDES_CLEAN_BIT），一共需要64*2 = 128 bits = 16 bytes。区描述符的结构如图所示：

![InnoDB_storage_XDES](/InnoDB_storage_XDES.png)

| 字段           | 大小 | 说明                                                         |
| :------------- | :--- | :----------------------------------------------------------- |
| XDES_ID        | 8    | 如果区已分配给段，则记录其段ID（segment inode entry.FSEG_ID） |
| XDES_FLST_NODE | 12   | 区所在的链表节点：FSP_FREE、FSP_FREE_FRAG / FSP_FULL_FRAG、或者位于某个B+树的segment inode entry链表中 |
| XDES_STATE     | 4    | 区状态XDES_FREE     ：空闲区，待分配给段，在FSP_FREE链表中 XDES_FREE_FRAG：碎片半满区，在FSP_FREE_FRAG链表中 XDES_FULL_FRAG：碎片全满区，在FSP_FULL_FRAG链表中 XDES_SEG      ：已分配给段，记录段ID |
| XDES_BITMAP    | 16   | 区中64个页的使用状态，用2个bit表示一个页 XDES_FREE_BIT  ：该页是否空闲 XDES_CLEAN_BIT：未使用，保留位 |

一个区描述符占40字节，一页可以保存256个区描述符（40 * 256 = 10240，不会用满，剩余5986字节未用），该XDES页可以描述总共16384个页面的信息（64个页 * 256个区描述符），这样，每隔16384个页分配一个XDES页（第0页，同时也分配第1页IBUF_BITMAP page），即每个XDES页的页号都是16384的倍数，函数xdes_calc_descriptor_page用于计算区描述符所在的页面位置（offset），即XDES entry可以用page no+boffset（在XDES page中的相对位置）描述。

虽然一个XDES page保存256个XDES，但是第1个用于存放XDES元信息，只用了后面的255个区，也就是说，之后后面的255个区放到了表空间的链表管理中。并且，区描述符的管理不需要链表进行串联，这是因为区是连续分配的，每64个区就有一个XDES page，直接通过*64就可以定位到任意一个XDES page。

同时由于space header存储在页（0，0）中，所以区描述符保存的开始位置在页的偏移量150字节处（FIL_HEADER + SPACE_HEADER 38 + 112）。同时，如果一个区中的页含有区描述符，则该区为碎片区。

另外，FSP_HDR页中的剩余5986字节的未使用空间中，可以存储该表空间的加密信息（encryption）：

| 字段                       | 大小 | 说明                                                         |
| :------------------------- | :--- | :----------------------------------------------------------- |
| ENCRYPTION_MAGIC_SIZE      | 4    | 加密版本，即version ENCRYPTION_KEY_MAGIC_V1 ENCRYPTION_KEY_MAGIC_V2 |
| master_key_id              | 1    | key_id                                                       |
| ENCRYPTION_SERVER_UUID_LEN | 36   | server_uuid                                                  |
| ENCRYPTION_KEY_LEN         | 32   | key                                                          |

## 段

段用来保存特定对象的数据。在InnoDB中，表是最常见的对象，同时表中的数据是通过索引组织表的方式组织的，即通过主键值以B+树索引的方式存储数据的。在InnoDB中，每个表至少有两个段，叶子节点段（leaf segment）和非叶子节点段（non leaf segment），段依据区的形式组织存储空间。

但是为了更有效的管理存储空间，如果表非常小，或者undo段，都不需要一个完整的区来保存数据。那么，在存储数据时，每个段设计了32个碎片页，段中的空间首先保存在这32个碎片页中，如果不够了再以区为单位申请空间。碎片页从空闲碎片区列表（FSP_FREE_FRAG）中的一个碎片区中分配，具体的函数为fsp_alloc_free_page。段中的区从空闲区链表（FSP_FREE）中申请分配。

一个段可以管理32个独立的页和若干个区。段的这种页+区混合管理的方式可以更有效的节约存储空间。表空间一旦将一块空间（一个页或者一个区）分配给一个段后，就不能再给别人使用了。所以，InnoDB存储引擎管理数据的思路是：从创建表开始，随着表中数据的增加，段每次从表空间中获取一个页，当获取到32个页后，则从表空间中获取一个区，这样，既保证了空间使用率，又兼顾了空间分配效率。

那么抽象来看，一个段是可以“无限扩展的”，并且段是由若干区+碎片页组成的，是一个逻辑概念，而区是实实在在的物理存储，物理上是连续的。表空间中的若干段都由若干独立的区链接起来，这些链接起来的区链表长短不一，并且磁盘位置是随机的，逻辑上是连续的。

segment inode entry用于保存段的信息：

| 字段                 | 大小 | 说明                                                         |
| :------------------- | :--- | :----------------------------------------------------------- |
| FSEG_ID              | 8    | 段ID（分配从1开始，0表示未分配）                             |
| FSEG_NOT_FULL_N_USED | 4    | FSEG_NOT_FULL链表中使用的页数量                              |
| FSEG_FREE            | 16   | 已分配给该段，但完全没有使用的区链表                         |
| FSEG_NOT_FULL        | 16   | 已分配给该段，全部页未被用满的区链表（也称为段未满区链表）   |
| FSEG_FULL            | 16   | 已分配给该段，全部页已被使用的区链表（也称为段已满区链表）   |
| FSEG_MAGIC_N         | 4    | magic number                                                 |
| FSEG_FRAG_ARR 0..31  | 128  | 碎片页链表，一共有32个页，保存了每个碎片页在表空间中的偏移量（4），因此总共需要32 * 4个字节 |

从上面可以看到，一个segment inode entry占用192个字节，这样一个segment inode页可以存储85个segment inode entry，页类型为FIL_PAGE_INODE。

segment inode页结构如下：

| 字段                   | 大小 | 说明                                                         |
| :--------------------- | :--- | :----------------------------------------------------------- |
| FSEG_INODE_PAGE_NODE   | 12   | segment inode页的链表节点，链表是FSP_SEG_INODES_FULL/FSP_SEG_INODES_FREE |
| segment inode entry 0  | 192  | segment inode entry                                          |
| segment inode entry 1  | 192  | segment inode entry                                          |
| ...                    |      |                                                              |
| segemnt inode entry 84 |      | segment inode entry                                          |

但是和XDES页不同的是，每个segment inode entry的位置不是固定的，不像XDES页都是间隔16384，同时，一个表可能存在多个索引，每个索引有2个段，所以一个表会有多个segment inode entry。为了找到segment inode entry的位置，还需要有一个segment header（10字节），由segment header指向segment inode entry。

| 字段             | 大小 | 说明                                           |
| :--------------- | :--- | :--------------------------------------------- |
| FSEG_HDR_SPACE   | 4    | segment inode页所在的表空间ID                  |
| FSEG_HDR_PAGE_NO | 4    | segment inode页所在的表空间的偏移量            |
| FSEG_HDR_OFFSET  | 2    | segment inode entry在segment inode页中的偏移量 |

对于用户表来说，segment header总是保存在其索引的root页中，指向叶子节点段的inode页（leaf segment）和非叶子节点段的inode页（non leaf segment）。但是segment header也并不是总是这一页，比如在change buffer中放在一个单独的页中。

root页中的segment info描述了non leaf segment和leaf segment的segment header（10+10字节）：

| 字段              | 大小                  | 说明                                         |
| :---------------- | :-------------------- | :------------------------------------------- |
| PAGE_BTR_SEG_LEAF | 10 (FSEG_HEADER_SIZE) | leaf segment在segment inode page中的位置     |
| PAGE_BTR_SEG_TOP  | 10 (FSEG_HEADER_SIZE) | non leaf segment在segment inode page中的位置 |

当创建一个索引的时候，实际上就是在构建一个新的B+树（btr_create），先为non leaf segment分配一个segment inode entry（从FSP_SEG_INODES_FREE找到一个空闲的segment inode页，从中分配segment inode entry），然后创建root page（根节点页，其位于segment inode entry的第一个碎片页），并将该segment inode entry的位置信息更新到root page上；之后在分配leaf segment的segment inode entry，过程同上。

## 表空间

从上面可以看到，表空间是一个逻辑概念，由文件系统的物理文件组成，然后组织成页、区、段，如下图所示：

![InnoDB_storage_tablespace_organization](/InnoDB_storage_tablespace_organization.png)

表空间的逻辑结构如下图所示：

![InnoDB_storage_tablespace_layout](/InnoDB_storage_tablespace_layout.png)

从上图可以看出，表空间管理的是空闲区、碎片半满区和碎片全满区，segment inode page中的segment inode entry中存放的是具体某一段分配的空闲区、半满区和全满区。
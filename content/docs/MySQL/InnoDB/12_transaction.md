---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: transaction
---

# 事务

## 概述

事务是访问数据库中数据的一个程序执行单元。在一个事务中的操作，要么都做，要么不做，这是事务的目的，也是事务模型区别于文件系统的重要特征之一。

从理论上来说，事务有着极其严格的定义，必须同时满足ACID四个特性。但是数据库厂商出于各种目的，并没有严格满足事务的ACID标准。比如对于MySQL的NDB Cluster，没有满足D；对于Oracle，默认的事务隔离级别是Read Committed，不满足I。在InnoDB中，默认的事务隔离级别是Read Repeatable，完全遵循和满足事务的ACID特性。

这里要注意，D保证的是事务系统的高可靠性（High Reliability），而不是高可用性（High Availability）。对于高可用性，事务本身并不能保证，需要一些系统共同配合来完成。

## 分类

参见Jim Gray

## 隔离级别

令人惊讶的是，大部分数据库系统都没有提供真正的隔离性，最初或许是因为系统实现者并没有真正理解这些问题。如今这些问题已经弄清楚了，但是数据库的实现者在正确性和性能之间做了妥协。ISO和ANSI SQL标准制定了四种事务隔离级别的标准，但是很少有数据库厂商遵循这些标准，比如Oracle不支持Read Uncommitted和Repeatable Read的事务隔离级别。

SQL标准定义的四个隔离级别为：

- Read Uncommitted
- Read Committed
- Repeatable Read
- Serializable

SQL和SQL2标准的默认事务隔离级别是Serializable。

InnoDB支持的默认事务隔离级别是Repeatable Read，但是和SQL标准不同的是，InnoDB在此隔离级别下，通过使用next-key locking算法，避免了幻读的产生，即可以完全保证事务的隔离性要求，达到了SQL标准的Serializable隔离级别。在append-only storage上，采用snapshot isolation也是避免幻读的一种实现方式。

{{< hint danger>}}

这里给读者留一个扩展性的思考：

采用MV可以实现snapshot isolation进而避免幻读吗？

InnoDB既然也有采用了MV，不用next-key locking而用Percolator的方式能不能做到？

{{</hint>}}

关于事务可以洋洋洒洒写很多，这里主要聚焦InnoDB中的事务子系统的实现，详细的事务概念和事务控制技术这里略过。

# 事务子系统

事务子系统中的数据包括事务信息、undo log、binlog信息以及doublewrite信息。

这些信息由trx_sys page牵头，存储在各自的segment中：

| segment             | 存储内容                              | segment header        |
| ------------------- | ------------------------------------- | --------------------- |
| txn segment         | 事务信息                              | trx_sys page          |
| rollback segment    | undo segment、history链表、undo slots | rollback segment page |
| undo segment        | undo log                              | undo segment page     |
| doublewrite segment | doublewrite数据                       | trx_sys_page          |

整体层次如下：

~~~~
trx_sys page:
  → txn_segment header 
  → doublewrite_segment header
  → rollback slots (page)
  rollback segment page:
    → rollback_segment header
    → history_list
    → undo slots (page)
    undo segment page (1st page in undo segment)
      → undo log
    undo page
      → undo log
~~~~

{{< hint info >}}

如果启用了undo表空间，其第3页会存储rollback segment array header（rollback slots），布局和现有介绍不同。

{{</hint>}}

整体上系统表空间中页的分布如下：

![InnoDB_trx_sys_tablespace](/InnoDB_trx_sys_tablespace.png)

## trx_sys page

事务系统段在InnoDB第一次启动时创建（trx_sysf_create），trx_sys segment header保存在系統表空間的(0, 5)位置，该页称为trx_sys page，保存以下事务信息：

- 事务相关信息
- rollback segment slots
- MySQL的binlog信息
- doublewrite segment信息

trx_sys page如下图所示：

![InnoDB_txn_trx_sys_page](/InnoDB_txn_trx_sys_page.png)

TRX_SYS_TRX_ID_STORE保存最大事务ID，为了性能考虑，事务ID每隔256才持久化，然后在事务子系统启动时jump同样区间，跳过这个gap实现”健忘性“。

TRX_SYS_FSEG_HEADER保存事务系统段的segment header（space+page_no+offset）。

TRX_SYS_RSEGS保存回滚段信息（slot array），一共128个rollback slot（也称为回滚段对象），每个slot用8个字节存储space+page_no。

TRX_SYS_MYSQL_LOG_INFO保存server层的binlog信息（-1000），用以保证innoDB的日志和server层日志的一致性。

TRX_SYS_DOUBLEWRITE保存了doublewrite信息（-200）。

## doublewrite segment

在buffer pool一章已经介绍，InnoDB在flush page时，为了避免partial-write，而采用doublewrite的方式进行flush，其本质上使用的是shadow page+duplex replica的方式。

doublewrite的segment header页保存在trx_sys page中。为了保证其在磁盘的顺序性以便顺序IO，doublewrite segment先一次性申请32个碎片页（不使用），再申请2个区，并将每个区的起始地址更新到BLOCK1、BLOCK2（buf_dblwr_create），因此总共申请了160个页（32 + 64*2），实际使用的是2个区128个页。

初始化代码如下：

~~~~c++
for (i = 0; i < 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE
         + FSP_EXTENT_SIZE / 2; i++) {
    new_block = fseg_alloc_free_page(
        fseg_header, prev_page_no + 1, FSP_UP, &mtr);
    if (new_block == NULL) {
        ib::error() << "Cannot create doublewrite buffer: "
            " you must increase your tablespace size."
            " Cannot continue operation.";
 
        return(false);
    }
 
    /* We read the allocated pages to the buffer pool;
    when they are written to disk in a flush, the space
    id and page number fields are also written to the
    pages. When we at database startup read pages
    from the doublewrite buffer, we know that if the
    space id and page number in them are the same as
    the page position in the tablespace, then the page
    has not been written to in doublewrite. */
 
    ut_ad(rw_lock_get_x_lock_count(&new_block->lock) == 1);
    page_no = new_block->page.id.page_no();
 
    if (i == FSP_EXTENT_SIZE / 2) {
        ut_a(page_no == FSP_EXTENT_SIZE);
        mlog_write_ulint(doublewrite
                 + TRX_SYS_DOUBLEWRITE_BLOCK1,
                 page_no, MLOG_4BYTES, &mtr);
        mlog_write_ulint(doublewrite
                 + TRX_SYS_DOUBLEWRITE_REPEAT
                 + TRX_SYS_DOUBLEWRITE_BLOCK1,
                 page_no, MLOG_4BYTES, &mtr);
 
    } else if (i == FSP_EXTENT_SIZE / 2
           + TRX_SYS_DOUBLEWRITE_BLOCK_SIZE) {
        ut_a(page_no == 2 * FSP_EXTENT_SIZE);
        mlog_write_ulint(doublewrite
                 + TRX_SYS_DOUBLEWRITE_BLOCK2,
                 page_no, MLOG_4BYTES, &mtr);
        mlog_write_ulint(doublewrite
                 + TRX_SYS_DOUBLEWRITE_REPEAT
                 + TRX_SYS_DOUBLEWRITE_BLOCK2,
                 page_no, MLOG_4BYTES, &mtr);
 
    } else if (i > FSP_EXTENT_SIZE / 2) {
        ut_a(page_no == prev_page_no + 1);
    }
 
    if (((i + 1) & 15) == 0) {
        /* rw_locks can only be recursively x-locked
        2048 times. (on 32 bit platforms,
        (lint) 0 - (X_LOCK_DECR * 2049)
        is no longer a negative number, and thus
        lock_word becomes like a shared lock).
        For 4k page size this loop will
        lock the fseg header too many times. Since
        this code is not done while any other threads
        are active, restart the MTR occasionally. */
        mtr_commit(&mtr);
        mtr_start(&mtr);
        doublewrite = buf_dblwr_get(&mtr);
        fseg_header = doublewrite
                  + TRX_SYS_DOUBLEWRITE_FSEG;
    }
 
    prev_page_no = page_no;
}
~~~~

从这里可以看出，一共申请了64 * 2 +32个页（2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE + FSP_EXTENT_SIZE / 2），这正如前面所分析的，doublewrite segment需要保证写入的顺序性。

# undo log

## 设计

在InnoDB中，undo log有两个用途：

1. 用来实现事务的原子性，即当事务由于主动/被动的原因而失败时，可以实现回滚，从而使数据恢复到事务开始运行时的状态
2. 实现一致性非锁定读（consistent non-locking read）

undo log的作用贯穿事务的整个声明周期：故障恢复（crash & txn recovery）和并发控制（concurrency control）。而undo的持久性是靠redo保证的，这也是crash recovery要先做redo apply的原因之一。

目前绝大多数数据库都通过locking+MV的方式提供事务的并发控制，这样，只有WW阻塞，RW、WR都可以并发，这样极大提高了事务的并发度。

用于存储数据MV的地方称为version storage，在InnoDB中，采用delta的方式存储（因为其undo log记录的是行的diff），在PostgreSQL中，是在数据的行上实现的MV，即append-only。关于MVCC在后面一节详述。 

InnoDB的一致性非锁定读通过多版本控制的方式（multi-versioning）来读取当前执行期间数据库中行的数据，如果需要读取的行正在执行更新操作，读取操作无需等待行上锁的释放（nonlocking），而是读取行的一个快照（read snapshot），并可以保证数据的读取一致（consistent）。

InnoDB的一致性非锁定读如下图所示，快照是指该行之前版本的数据，存储在undo log中。同时，因为undo log本身就会用于事务的回滚，因此快照本身是没有额外存储开销的，而且，读取快照也不需要加锁（因为不会变更数据）。

一致性非锁定读如下图所示：

![InnoDB_txn_consistent_nonlocking_read](/InnoDB_txn_consistent_nonlocking_read.png)

在redo log一章我们介绍过，redo log考虑的一个重点是recovery的时间，因此通过多种手段来做出保证，比如可以基于page来做redo log的并行recovery。而undo log和redo log却大不相同，完全不必在crash recovery阶段就完成undo（数据库运行期间就保有undo log），可以在后台异步purge，这样就可以更快的完成启动，提供服务。

undo log注重的是事务间的并发，以及维护MV的代价，并且与物理存储解耦。因此，undo log采用了逻辑日志的方式。

一个undo segment在同一时刻只属于一个事务，一个读写事务至少会持有一个undo segment，这样，当大量事务并发时，就需要存在多个undo segment为事务存储undo log，因此，会将多个undo segment组成rollback segment，并通过history链表进行GC。

undo log的组织有几个维度：

- 事务维度的undo log list，用于快速rollback transaction，一个事务的所有undo log都连续存储在最多2个undo segment中
- 全局维度的已提交事务undo log list，称为history链表，用于purge GC
- undo log的物理布局：包括存储在哪个表空间中，undo page，undo log type，undo log的组织

接下来，通过从总体到局部的顺序，依次介绍rollback segment→undo segment→undo page→undo log。

## 物理布局

undo log的存储由rollback segment和undo segment共同完成,，并且，这两个段的segment header都保存在各自的segment内。rollback segment中仅保存undo segment page所在页的位置，一个rollback segment一共保存1024个undo segment的信息，加上trx_sys page可以保存128个rollback segment slot，因此理论上可以支持最大128 * 1024/2个并发事务（因为同一时刻只能有一个active事务使用一个undo page，假定每个事务DML有INSERT、UPDATE、DELETE）。

整体关系如下图所示：

![InnoDB_txn_undo_layout](/InnoDB_txn_undo_layout.png)



### rollback segment

trx_sys page描述了128个rollback segment的信息（space+page_no）。这128个rollback segment保存在rollback segment header page中，其中RSEG_0 page位于(0, 6)位置。

每个rollback segment header page维护了段内的history链表和1024个undo slot位置。

字段信息如下：

| 字段                  | 大小     | 说明                                                   |
| :-------------------- | :------- | :----------------------------------------------------- |
| TRX_RSEG_MAX_SIZE     | 4        | rollback segment可以拥有的最大undo page数（ULINT_MAX） |
| TRX_RSEG_HISTORY_SIZE | 4        | history链表中undo page的数量                           |
| TRX_RSEG_HISTORY      | 16       | history链表（committed undo log list）                 |
| TRX_RSEG_FSEG_HEADER  | 10       | rollback segment header                                |
| TRX_RSEG_UNDO_SLOTS   | 1024 * 4 | undo slot directory（undo segment page位置）           |

总结一下，rollback segment主要存储三个信息：

1. history链表，存储已提交事务的undo log list（按事务提交顺序逆序存放）
2. rolback segment header
3. undo slot directory

在MySQL 5.7中，REG_0在系统表空间中，REG_1 ~ REG_32在临时表空间，REG_33 ~ REG_127如果配置了undo表空间（space_id固定，从1开始）则放置其内，否则在系统表空间。

### undo segment

每个rollback segment描述了1024个undo segment page，其中每个undo segment又由一系列undo page组成，存储实际的undo log，因此undo segment可以认为是一个"逻辑概念"，由一组undo page构成。

在每个undo segment中，第一个undo page称为undo segment page，用于保存undo segment header（30字节）和相应的段元信息：

| 字段                 | 大小 | 说明                                                         |
| :------------------- | :--- | :----------------------------------------------------------- |
| TRX_UNDO_STATE       | 2    | UNDO segment状态（5）：TRX_UNDO_ACTIVE：段内有活跃事务的undo logTRX_UNDO_CACHE：cached for quick reuseTRX_UNDO_TO_FREE：存放的都是insert DML，可以快速freeTRX_UNDO_TO_PURGE：不能重用，交由purge线程freeTRX_UNDO_PREPARED：段内有prepared txn的undo log |
| TRX_UNDO_LAST_LOG    | 2    | 最后一个undo log header offset                               |
| TRX_UNDO_FSEG_HEADER | 10   | undo segment header                                          |
| TRX_UNDO_PAGE_LIST   | 16   | undo page链表（将同一事务的segment page和normal page串在一起） |

### undo page

undo segment中的所有page都为undo page，都有undo page header（18字节）：

| 字段                | 大小 | 说明                                                    |
| :------------------ | :--- | :------------------------------------------------------ |
| TRX_UNDO_PAGE_TYPE  | 2    | 页中保存的undo log类型：TRX_UNDO_INSERT TRX_UNDO_UPDATE |
| TRX_UNDO_PAGE_START | 2    | 页中最新事务的undo log offset                           |
| TRX_UNDO_PAGE_FREE  | 2    | 页中的空闲空间offset                                    |
| TRX_UNDO_PAGE_NODE  | 12   | undo page链表节点（同一事务的undo page串起来）          |

undo page中其余部分则存储实际的undo log。这样，一个undo page保存以下几部分信息：

- undo page header
- undo segment header（段中第一页）
- undo log...

每个事务在需要记录undo log时都会申请一个或两个undo segment（insert_undo/update_undo分开），同时把事务的第一个undo page放入对应undo segment中，这个页同时也会作为undo segment header page。一个undo segment header page同一时刻只隶属于同一个活跃事务，但是一个undo header page上面存储的undo log可能包含多个已经提交的事务和一个活跃事务。

当活跃事务产生的undo record超过undo header page容量后，单独再为此事务分配的undo page（trx_undo_add_page）。undo page只隶属于一个事务。

从上面可以看出，InnoDB会复用undo page存放不同事务的undo log，是因为：

- OLTP事务产生的undo log相对较小
- 其他事务可能正在引用当前的undo log，事务在提交时并不能直接删除相应的undo log（insert例外）

复用undo page有以下优势：

- 可以减少undo page的分配

我们计算一下，假设OLTP TPS=1000，每个事务产生200字节的undo log，如果undo page不复用，那么一分钟就要产生1000 * 60个undo page，每个undo page 16K，要分配16K * 60000 = 937M的存储空间；并且，如果purge线程每秒只能处理20个undo page，则存储用量会持续上涨。这种设计下磁盘的开销将会非常巨大。

因此复用undo page是合理的。undo page的复用，是指一个undo page可以存放不同事务的undo log。具体来讲，在事务提交时，首先将undo log放入history链表，然后判断当前undo page是否可以复用：

1. undo page的可用空间是否小于3/4（TRX_UNDO_PAGE_REUSE_LIMIT ）
2. 该事务只用了一个undo page

如果满足则复用该undo page，并将undo segment状态设为TRX_UNDO_CACHE，之后的undo log append-only进行追加。

这里可以看出history链表是undo log串起来的，而undo log又分布在不同的undo page中，因此purge线程在处理history链表时会产生离散IO。

复用条件的判定：

~~~~c++
/** An update undo segment with just one page can be reused if it has
at most this many bytes used; we must leave space at least for one new undo
log header on the page */
 
#define TRX_UNDO_PAGE_REUSE_LIMIT   (3 * UNIV_PAGE_SIZE / 4)
 
trx_undo_set_state_at_finish() {
    ...
    if (undo->size == 1
        && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE)
           < TRX_UNDO_PAGE_REUSE_LIMIT) {
 
        state = TRX_UNDO_CACHED;
 
     }
     ...
}
~~~~

### undo log

undo log是一种逻辑日志，存储的是记录修改的before image（前镜像）。因此事务的rollback意味着是将数据逻辑的恢复到原来的样子。

但是数据结构和page在回滚之后可能大不相同。因为，在数据库系统中，事务是并发访问的。同一页中的不同行并发，可能已经让page产生分裂/合并进而改变了B+ tree的结构，或者page的布局发生了变化。不能通过apply undo log将page回滚到该事务开始的样子，会影响其他事务在该页上的并发操作。

另外，redo log是基于LSN实现的，LSN具有单调递增属性，如果回滚页的物理操作，会破坏这个特性，使设计变得复杂。

每个undo记录由两部分内容构成：

- undo log header
- undo log record

undo log record又分为insert_undo/update_undo两种，insert操作产生insert undo log record，其他DML操作产生update undo log record，也存在一些特殊情况，后面另行介绍。

undo log的结构如下图所示：

![InnoDB_txn_undo_log](/InnoDB_txn_undo_log.png)

undo log header保存每个事务undo日志的通用信息。并且，因为undo页可以重用，因此回滚段的第一个undo页中可以存在多个undo log header（活跃事务的header在最后）。每个undo log header占用46个字节：

| 字段                  | 大小 | 说明                                                         |
| :-------------------- | :--- | :----------------------------------------------------------- |
| TRX_UNDO_TRX_ID       | 8    | 事务ID（事务开始的逻辑时间）                                 |
| TRX_UNDO_TRX_NO       | 8    | 事务提交顺序，也用来判断是否能purge（事物结束的逻辑时间）    |
| TRX_UNDO_DEL_MARKS    | 2    | undo log中是否有DEL_MARK_REC，避免purge不必要的扫描          |
| TRX_UNDO_LOG_START    | 2    | undo log header结束的offset（也是第一个undo log record的offset） |
| TRX_UNDO_XID_EXISTS   | 1    | XID                                                          |
| TRX_UNDO_DICT_TRANS   | 1    | 是否为DDL事务                                                |
| TRX_UNDO_TABLE_ID     | 8    | DDL事务修改的表                                              |
| TRX_UNDO_NEXT_LOG     | 2    | undo log list                                                |
| TRX_UNDO_PREV_LOG     | 2    | undo log list                                                |
| TRX_UNDO_HISTORY_NODE | 12   | history链表节点                                              |

在事务每次进行DML操作时，首先分配一个undo log header（trx_undo_create），事务可能包含多个insert/update操作，但是不能存放在一个undo页中。每个类型的undo记录需要分配单独的undo段（UNDO LOG PAGE HEADER.TRX_UNDO_PAGE_TYPE）。这里的insert/update undo log record的删除时机不同：由于插入的记录对其他事务不可见，索引insert undo log record在事务提交后就可以被删除，而update/delete操作产生的update undo log record需要维护MV，只能在purge线程中随后清理。也正是因为这样，分为了两个undo segment，并且这两种undo log record分别称为：insert_undo & update_undo。

undo log header之后存放实际的undo log record。

根据产生的undo log就可以将一行记录恢复到之前的版本（trx_undo_prev_version_build），这也是MVCC（或者说非锁定读）的实现过程。

整个undo log的存储内容如下图所示：

![InnoDB_txn_undo_log_detail](/InnoDB_txn_undo_log_detail.png)

#### insert undo log record

insert undo log record通常是指事务在INSERT操作中产生的undo日志。因为insert操作的记录只对事务本身可见，对其他事务不可见（没有历史版本），因此可以在事务提交后直接删除，不需要进行purge操作。

undo log record的类型为TRX_UNDO_INSERT_REC，只需为rollback做准备，而不需要考虑MVCC，因此只记录行的before image（key fields: col 1...n len+data）。

insert undo log的结构：

![InnoDB_txn_undo_log_insert_undo_log_record](/InnoDB_txn_undo_log_insert_undo_log_record.png)

rollback时根据key fields在index中定位record

#### update undo log record

update undo log record通常保存的是对DELETE和UPDATE操作产生的undo日志。该undo日志可能需要提供MVCC机制，因此与insert undo log record不同，其不能在事务提交时立即进行删除。当事务提交时，它会放入rollback segment的history链表的头部。然后等待purge。

update undo log record的结构如图所示：

![InnoDB_txn_undo_log_update_undo_log_record](/InnoDB_txn_undo_log_update_undo_log_record.png)

update undo log record相对于insert undo log record要复杂的多，其根据不同的事务操作，会产生不同的update undo log record。primary key是都要存储的，而上图中的最后两部分（update vector、index columns）不一定每个update undo log record都会存储。

update undo log record一共有3种不同的类型：

- TRX_UNDO_UPD_EXIST_REC：更新已经存在的记录
- TRX_UNDO_UPD_DEL_REC  ：更新已经删除的记录
- TRX_UNDO_DEL_MARK_REC：将记录标识为已删除

除了一些"事务信息"，update undo log record主要存储以下3部分的信息：

- 记录的主键值列表（primary key）
- 发生更新的列（update vector）
- 索引列

主键值是一定存在的，后面2部分信息取决于操作是否产生这些修改，如果有则记录下来。比如，对于delete操作，只是将记录标记为删除，没有发生更新的列，因此没有第2部分的信息。对于update操作，则需要保存更新前记录的值（upd_t）。

第3部分保存的是索引列的信息。有两种情况需要保存这部分的信息：

1. DELETE MARK操作
2. 更新操作更新了索引列

这样设计的目的是保存这部分信息可以在purge时删除记录所对应的secondary index。

当一个undo page放不下update undo log record时，会通过将已经产生的部分update undo log record删除（trx_undo_erase_page_end），并以十六进制0xFF进行填充。同时将新的undo page添加到undo segment中，并使用新申请的undo page存放update undo log record。同时，该undo page不可再进行重用。

##### TRX_UNDO_UPD_EXIST_REC

更新现有记录，需要记录update vector和index columns这两部分信息

##### TRX_UNDO_DEL_MARK_REC

因为要支持MVCC多版本，所以需要将DELETE的记录伪删除（更新info_bits），而真正的删除操作由purge线程完成。

因为没有修改任何列，因此不需要update vector的信息。一般来说，TRX_UNDO_DEL_MARK_REC类型的update undo log record占用的空间最小。

对于有主键值的更新操作来说，会产生TRX_UNDO_DEL_MARK_REC类型的update undo log record，同样也会产生insert undo log record。这意味着，一个SQL语句可能产生多个undo日志。

##### TRX_UNDO_UPD_DEL_REC

TRX_UNDO_UPD_DEL_REC表示更新已经删除的记录，但是，如果记录真的被删除，由于需要被purge，因此不可以对其进行更新操作。所以，产生该类型的update undo log应该是在同一事务内对记录进行更新操作的（DELETE+INSERT）。

~~~~
BEGIN;
DELETE FROM t where a = 1;
INSERT INTO t values (1, 'm'); 
~~~~

在上面的事务中，首先删除了主键值=1的记录，随后又再次插入了主键值=1的新纪录，这时该事务会产生2个update undo log record。第一个为TRX_UNDO_DEL_MARK_REC，第二个update undo log record需要保存update vector的信息（更新了b列）。另外，由于列b是索引列，因此还需要保存索引列的信息（index columns）。

从这里可以发现，insert操作不一定都产生insert update undo log，也可能产生TRX_UNDO_UPD_DEL_REC类型的update undo log record。另外，在这个例子中，第二次插入的并非是主键值=1的记录，或者之后插入的记录的字节大小发生了变化，那么就不会产生TRX_UNDO_UPD_DEL_REC的undo日志，其产生的依旧是insert update undo log。

### undo的redo

在前面已经提到，undo的持久化要通过redo保证，因此undo也有对应的redo log type：

- MLOG_UNDO_INSERT：写入undo log record
- MLOG_UNDO_ERASE_END：undo log跨undo page时做的抹黑（0xFF）
- MLOG_UNDO_INIT：undo page初始化
- MLOG_UNDO_HDR_DISCARD：废弃undo log header
- MLOG_UNDO_HDR_REUSE：复用undo log header
- MLOG_UNDO_HDR_CREATE：创建undo log header

## 内存布局

在事务子系统中，undo的管理非常重要，我们在上面介绍了物理布局，在内存中，也需要构建其对应的内存对象，来进行undo管理。

在这里，我们略过undo表空间对象，聚焦在rollback segment和undo log对象上。

rollback segment内存对象 trx_rseg_t

~~~~
struct trx_rseg_t {
  ulint       id;
  RsegMutex     mutex;
  ulint       space;            rollback segment page所在的位置（space+page_no）
  ulint       page_no;
  page_size_t page_size;        所在表空间的page size
  ulint       max_size;         rollback segment可以拥有的最大undo page数
  ulint       curr_size;        当前undo page数

  UT_LIST_BASE_NODE_T(trx_undo_t) update_undo_list;   活跃事务产生的update_undo链表
  UT_LIST_BASE_NODE_T(trx_undo_t) update_undo_cached; 之前事务提交后可复用的undo log list（update_undo）

  UT_LIST_BASE_NODE_T(trx_undo_t) insert_undo_list;   活跃事务产生的insert_undo链表
  UT_LIST_BASE_NODE_T(trx_undo_t) insert_undo_cached; 之前事务提交后可复用的undo log list（update_undo）

                                从trx_undo_t中提取，用于维护history链表的LWN
  ulint       last_page_no;     history链表中最后一个没有被purge的undo page no
  ulint       last_offset;      最后一个没有被purge的undo log header offset
  trx_id_t    last_trx_no;      最后一个没有被purge的事务逻辑时间
  ibool       last_del_marks;   最后一个没有被purge的undo log需要purge

  ulint       trx_ref_count;    活跃事务的引用计数，=0则rollback segment可truncate
  bool        skip_allocation;  truncate rollback segment时，trx_sys不分配该段
};
~~~~

整体布局如下：

![InnoDB_txn_undo_memory_layout](/InnoDB_txn_undo_memory_layout.png)

## undo的一生

我们从整个undo的生命周期来完整系统的介绍：
1. 分配rollback segment
2. 使用rollback segment
3. 写入undo log
4. 事务提交，事务回滚，MVCC，purge，crash recovery在事务一节介绍

### 分配rollback segment

当事务"产生"对数据进行修改时，会相应产生undo log。也就是说，rollback segment在事务第一次"需要"存储undo log时才分配。

我们知道，事务按读写分为只读事务和读写事务，我们分别介绍这两种事务如何分配rollback segment。

#### 只读事务

涉及到对临时表读写，会从临时表空间分配rollback segment：REG_1 ~ REG_32。（trx_assign_rseg）

#### 读写事务

事务默认都是只读事务，只有当随后判定为读写模式时，才切成读写事务，并分配TrxID和rollback segment（trx_set_rw_mode）。

分配流程如下：

1. 采用round-robin的轮询方式来分配，如果rollback segment被标记为skip_allocation，则跳过
2. 分配后，rseg->trx_ref_count++
3. 临时表rseg挂到trx->rsegs->m_noredo，普通rseg挂到trx->rsegs->m_redo

代码调用链如下：

~~~~
trx_assign_rseg
  trx_assign_rseg_low(srv_rollback_segments, srv_undo_tablespaces, TRX_RSEG_TYPE_NOREDO)
    rseg = get_next_noredo_rseg(srv_tmp_undo_logs + 1);

trx_set_rw_mode   设置为读写事务
  trx_assign_rseg_low(srv_rollback_segments, srv_undo_tablespaces, TRX_RSEG_TYPE_REDO)
    rseg = get_next_redo_rseg(max_undo_logs, n_tablespaces);
~~~~

### 使用rollback segment

流程如下（trx_undo_report_row_operation）：

1. 当前变更如果是临时表（dict_table_is_temporary），则使用临时表rseg（trx->rsegs.m_noredo）；否则使用普通rseg（trx->rsegs.m_redo）
2. 判断变更类型
   1. insert_undo，单独分配undo slot
   2. update_undo，单独分配undo slot

分配undo slot（undo segment）（trx_undo_assign_undo）

1. 首先复用rseg上的cached undo list，
   1. insert_undo：修改seg_hdr（改为active）和page_hdr（start, free），预留XID位置
   2. update_undo：在undo segment page创建undo log header，预留XID位置
2. 初始化trx_undo（设为active）
3. 如果没有cached undo list，从rollback segment分配一个undo slot，并初始化undo page（如果1024分配完了，返回DB_TOO_MANY_CONCURRENT_TRXS错误）
4. 将trx_undo加入active undo list
5. 如果是DDL事务，还要记录元信息（TRX_UNDO_DICT_TRANS=true && table_id）

{{< hint danger>}}

这里留给读者一个问题：在#1中对于update_undo，为什么要在undo segment page中创建undo log header

{{</hint>}}

### 写入undo log record

写入undo log record流程如下（trx_undo_report_row_operation）：

1. 选择undo page（undo->last_page_no）
2. 按照insert_undo/update_undo进行插入记录
3. 如果写入过程中空间不足，则将已写部分涂黑0xFF（trx_undo_erase_page_end），然后申请新的undo page重新写入
4. 构建rollback pointer，并更新到clustered index record中

# 事务

大多数数据库采用STEAL+NO-FORCE的方式，在事务推进国产中，不断写入undo log，然后在事务提交时，保证redo log持久化到磁盘。这样，当发生被动故障时，就可以通过crash recovery将数据恢复到一致的状态。

## rollback

### rollback pointer

在record一章我们介绍过，行记录通过roll_ptr将行的历史变化串联起来，实现MV。

在MVCC中，通过当前行不断apply undo log record，来构造出之前的版本。

rollback pointer指针占用7个字节：undo segment no需要1个字节，且最高位保存undo log type；undo page需要4个字节；undo log需要2个字节。因此可以通过rseg_id+page_no+offset构造出rollback pointer（trx_undo_build_roll_ptr），也可反解（trx_undo_decode_roll_ptr）。

![InnoDB_txn_rollback_pointer](/InnoDB_txn_rollback_pointer.png)

![InnoDB_txn_rollback_chain](/InnoDB_txn_rollback_chain.png)

### 回滚操作

根据回滚操作的发起者来分类，可以分为用户态的回滚和内核态的回滚。

用户态的回滚指的是用户通过rollback命令发起。

内核态的回滚指的是InnoDB内部发起的回滚，一共有两种：

- 仅回滚事务最近的一个语句：比如违反唯一性约束，那么仅回滚最近的一个SQL语句，事务的状态依然是活跃的。
- 回滚整个事务，和用户态的回滚一样：比如发生死锁时回滚victim的事务。

InnoDB内部定义了三种回滚类型

- TRX_SIG_TOTAL_ROLLBACK：回滚整个事务
- TRX_SIG_ROLLBACK_TO_SAVEPT：回滚到最近一个保存点（InnoDB支持保存点的事务，并且，保存点事务也是基于undo实现的（trx_t::undo_no，每次事务写入undo++））
- TRX_SIG_ERROR_OCCURRED：当错误发生时进行回滚（没有找到触发的事件）

在进行回滚操作时，InnoDB主要做两件事情：

1. 根据trx_id找到undo log record（从undo segment page中根据当前事务的trx_id找到对应的undo log header）
2. 根据undo log record顺序找到最后一个undo log record，逆序回滚记录

若回滚操作进行的是TRX_SIG_TOTAL_ROLLBACK，即重复上述两个操作直到事务中所有的undo log record都已经被回滚。回滚时根据undo log record进行逆序操作（trx_roll_pop_top_rec_of_trx）。当回滚整个事务完成之后，undo log record的空间被释放，可以供之后的事务存放undo log，但需要特别注意的是：回滚后undo log header所占用的空间是不被释放的，但是这并不影响purge。

当取得undo log record后，需要根据不同类型的undo log record进行处理。

- insert undo log record：进行逆操作DELETE即可，首先删除secondary index中的记录，在删除clustered index中的记录（调用btr_cur_optimistic_delete或btr_cur_optimistic_optimistic，而不是像redo log recovery哪有直接对physical page进行直接操作）。DELETE操作是真正的删除操作，而不是DELETE MARK的伪删除。这是因为事务并没有提交，所以记录对所有其他事务是不可见的，因此可以立即删除，不需要等purge线程来清理。此外，回滚是一个逻辑操作，这和redo log不同。
- update undo log record：三种类型，回滚因人而异。但一样是是逻辑操作，处理顺序也是secondary index→clustered index（row_undo_mod）
  - 从update_undo中解析出update vector回退seconary index：去除delete mark标记，或者用update vector中的diff信息修改成之前的值（row_undo_mod_del_unmark_sec_and_undo_update）
  - 同样用update vector中的diff信息修改clustered index（row_undo_mod_clust）

总结一下，逆向操作可以分为以下几种：

- delete mark = 1改为0
- in-place更新直接改成旧值
- 对于插入操作，依次删除seoncdary index和clustered index
  - row_undo_ins_remove_sec_rec
  - row_undo_ins_remove_clust_rec
  

当回滚处理完最后一个undo log record后，释放之前undo log record所占用的空间（trx_undo_truncate_end），但是并不回收undo log header。

回滚同样需要提交，因为与正常提交一样，需要释放事务所持有的资源：undo segment、lock、read view，并flush redo log buffer。

## commit

事务的提交核心注意两个点位：commit point和complete point。

事务提交分为两种：隐式提交和显式提交。

当执行DDL，或者采用事务控制语句（BEGIN，START TRANSACTION时），为隐式提交。

显式提交，发送COMMIT。

另外，MySQL支持XA事务，这里我们只谈内部XA事务。

### 内部XA事务

MySQL因为架构分为server层和engine层，为了保证两层日志的一致性，需要使用2PC原子协议，而这通过内部XA事务保证。

采用2PC，就要有协调者（Coordinator），而协调者可以有三种：

- mysql_bin_log：开启binlog，至少一个支持事务的存储引擎，使用binlog记录事务状态，称为Binlog/Engine XA
- tc_log_mmap：不开binlog，至少两个支持事务的存储引擎，使用内存记录事务状态，称为Engine/Engine XA
- tc_log_dummy：不开binlog，即只有InnoDB或其他，事务状态下沉到engine中，称为Engine Commit

选择协调者的具体逻辑代码如下：在MySQL启动时进行判断：

~~~~
init_server_components() {
  ...
  if (total_ha_2pc > 1 || (1 == total_ha_2pc && opt_bin_log))
  {
    if (opt_bin_log)
      tc_log= &mysql_bin_log;
    else
      tc_log= &tc_log_mmap;
  }
  ...
}
~~~~

存储引擎是否支持事务（total_ha_2pc）判断如下：

~~~~
int ha_initialize_handlerton(st_plugin_int *plugin) {
	if (hton->prepare)
    total_ha_2pc++;
}
~~~~

提供的XA命令（接口）：

- XA start：open()
- XA end：close()
- XA rollback：rollback()
- XA prepare：prepare()
- XA commit：commit()

各个具体的协调者则实现这些接口。

tc_log_dummy直接调用下发到engine层（调用engine的rollback, prepare, commit），而另2种协调者则需要做具体的事情，具体区别如下：

|               | prepare        | commit                       | rollback        |
| ------------- | -------------- | ---------------------------- | --------------- |
| mysql_bin_log | ha_prepare_low | write binlog + ha_commit_low | ha_rollback_low |
| tc_log_mmap   | ha_prepare_low | write XID + ha_commit_low    | ha_rollback_low |
| tc_log_dummy  | ha_prepare_low | ha_commit_low                | ha_rollback_low |

其中ha_prepare_low，ha_commit_low，ha_rollback_low都是调用存储引擎注册的prepare，commit和rollback。

从上面可以看到，对于协调者是binlog和mmap时，都需要支持2PC，2PC是否支持（m_no_2pc）的封装在TransactionContext中的THD_TRANS对象中：

~~~~
class Transaction_ctx
{
...
private:
  struct THD_TRANS
  {
    /* true is not all entries in the ht[] support 2pc */
    bool        m_no_2pc;
    int         m_rw_ha_count;
    /* storage engines that registered in this transaction */
    Ha_trx_info *m_ha_list;
~~~~

每个事务在存储引擎注册时设置该值：

~~~~
trans_register_ha() {
  if (ht_arg->prepare == 0)
    trn_ctx->set_no_2pc(trx_scope, true);
}
~~~~

在事务推进过程中标记读写：

~~~~
mark_trx_read_write
  ha_info->set_trx_read_write();
    m_flags|= (int) TRX_READ_WRITE;
~~~~

并在事务提交时计算是否真正需要2PC：

~~~~
ha_commit_trans() {
  if (ha_info)
  {
    uint rw_ha_count;
    rw_ha_count= ha_check_and_coalesce_trx_read_only(thd, ha_info, all);
      for loop in ha_list:
        if (ha_info->is_trx_read_write())
          ++rw_ha_count;
    trn_ctx->set_rw_ha_count(trx_scope, rw_ha_count);
~~~~

### Binlog/Engine XA

在这里我们聚焦在支持2PC且协调者为mysql_bin_log。

在这种场景下，TM为mysql_bin_log，RM为binlog engine和InnoDB engine。结合MySQL的架构可以看出，TM在server层，RM在engine层。

#### 事务协调机制

这样，在XA事务中，就需要有通信机制确认是否进行两阶段提交（2PC）：

1. 注册事务：即上面的trans_register_ha，InnoDB会在innobase_register_trx中调用
2. 标记发生事务读写：在server层的handler调用时标记
3. 提交时判断标记，即ha_commit_trans()

其中InnoDB engine注册的接口为：

~~~~
innobase_hton->commit = innobase_commit;
innobase_hton->rollback = innobase_rollback;
innobase_hton->prepare = innobase_xa_prepare;
innobase_hton->recover = innobase_xa_recover;
innobase_hton->commit_by_xid = innobase_commit_by_xid;
innobase_hton->rollback_by_xid = innobase_rollback_by_xid;
~~~~

binlog engine注册的接口为：

~~~~
binlog_hton->commit= binlog_commit;
binlog_hton->commit_by_xid= binlog_xa_commit;
binlog_hton->rollback= binlog_rollback;
binlog_hton->rollback_by_xid= binlog_xa_rollback;
binlog_hton->prepare= binlog_prepare;
binlog_hton->recover=binlog_dummy_recover;
~~~~

标记发生事务读写的地方：

~~~~
handler::ha_bulk_update_row
handler::ha_write_row
handler::ha_update_row
handler::ha_delete_row
handler::ha_delete_all_rows()
handler::ha_truncate()
handler::ha_rename_table
handler::ha_delete_table
handler::ha_drop_table
handler::ha_create
...
~~~~

#### 2PC

2PC协议不再详述。这里只说几个key point：

1. commit point：因为TM是binlog，因此事务成功的标志是binlog写入成功
2. forward recovery：过了commit point，即XID在binlog里的，commit
3. backwared recovery：没过commit point，即XID不在binlog里的，rollback
4. complete point：事务完全完成

因此，两阶段核心做的事情就清晰了：

- prepare：
  - binlog：no-op
  - InnoDB：undo中标记事务状态prepared，记XID，以上都包裹在redo log中，redo log持久化
- commit
  - binlog：写XID binlog，sync binlog，记commit point
  - InnoDB：InnoDB commit，记complete point（history链表，undo slot，释放资源）

从这里也可以看出，redo包住undo，commit point一定要redo先binlog后，complete point一定要redo log在最后。

{{< hint danger>}}

这里留给读者1个问题：binlog中一定会有XID吗？更进一步，数据操作一定会有XID吗？

{{</hint>}}

两阶段提交中InnoDB的细节如下：

- prepare：
  - 将事务状态从active->prepared，即将所有undo slot中的undo segment page的seg_hdr.TRX_UNDO_STATE=TRX_UNDO_PREPARED（trx_undo_set_state_at_prepare）
  - 在事务的undo log header中填充XID（执行事务的第一条SQL时，就会注册XA，并根据thd->query_id分配XID）（trans_register_ha）
  - redo log持久化
- commit：
  - 将所有undo slot设为完成态（CACHED/TO_FREE/TO_PURGE）
    - 如果undo segment iff 1 undo page && 使用量小于3/4，复用该undo segment（将该undo segment状态设为TRX_UNDO_CACHE）
    - 含有insert undo log record，事务提交后快速free掉（将该undo segment状态设为TRX_UNDO_TO_FREE）
    - 含有update undo log record，undo segment交由purge free（将该undo segment状态设为TRX_UNDO_TO_PURGE）
  - 将rseg加入purge队列（purge_sys->purge_queue）
  - 更新rseg->last_*四元组
  - 串到cached undo list（insert_undo知道事务释放完全部资源才free）
  - rseg->trx_ref_count--
  - 如果开启了binlog（trx->mysql_log_file_name != NULL && trx->mysql_log_file_name[0] != '\0'），在trx_sys page写入binlog信息
  - 清理read view对象
  - flush redo log buffer（WAL）
  - 将trx从事务列表中删除

redo log中并没有一个"特殊"的redo log record表示事务结束（complete point）：事务结束的标志是事务的undo log放入rollback segment的history链表。这个操作将redo log写入mtr中，mtr.commit后在事务提交时flush redo log buffer完成。

调用链如下：

~~~~
trx_commit_low
  trx_write_serialisation_history
    trx_undo_set_state_at_finish
~~~~

{{< hint danger>}}

这里留给读者2个思考：

为什么不直接设置一个特殊的redo log标识事务结束？

为什么可以事务结束的标识放到history链表中？

{{</hint>}}

#### group commit

在MySQL 5.6之前，每次2PC都是串行的，即上一个事务prepare+commit完成后，下一个事务才能开始。

从上面的2PC我们可以知道，其中会发生3次IO，redo log, binlog,  redo log，这极大的影响了事务处理的性能。

在MySQL 5.6中，引入了binlog的group commit，即prepare阶段不变，commit阶段变为3个（ordered_commit）：

1. flush stage：各个用户线程按序从thd合入binlog文件
2. sync stage：对binlog做fsync
3. commit stage：各个用户线程按序做InnoDB commit

每个stage都有一个队列，第一个进入队列的thd为leader，后续进入的线程为follower阻塞并等待该stage完成。leader负责领导队列中的所有线程执行该stage所要进行的工作，并带领所有follower进入下一个stage。

这三个阶段类似于CPU执行指令的pipeline一样，并发执行，从而提升commit效率。

有两个参数控制group的形成：

- 时间：binlog_group_commit_sync_delay (μs)
- 空间：binlog_group_commit_sync_no_delay_count=N

MySQL 5.7继续优化，将prepare中InnoDB写redo log delay到group commit阶段：这是因为prepare阶段的IO，即每个用户线程的redo log sync成为了瓶颈，

于是，redo log sync移到了commit阶段的flush stage：

1. 排队，确认leader和follower
2. (+) leader线程进行redo log write+sync，一次性将所有线程的redo log持久化（ha_flush_logs）
3. 各个用户线程按序从thd合入binlog文件

即redo log delay（不会破坏正确性，因为redo log持久化还是在binlog持久化之前），redo log group commit。

{{< hint info>}}

这里注意binlog_order_commits，其控制binlog和InnoDB提交同序，在单机下不会有问题，因为一致性靠XID保证，但是如果备份使用xtrabackup工具，其依赖trx_sys page上记录的binlog点位，如果位点发生乱序，就会导致备份的数据不一致。

{{</hint>}}

这里稍微提一句，MGR中的Paxos在写binlog前（commit point之前）各个用户线程单独进行Group Replication，成功才写binlog，否则直接回滚事务。

## MVCC

从前面的undo chain图上可以很直观的看到，undo记录包含了行数据的历史版本，因此，MVCC可以：

- 根据rollback pointer可以回溯undo log，进而可以构建出那个版本的数据
- 根据trx_id判断可见性

简单来说就是对于RC及以上隔离级别（RU直接读取当前行记录），当前事务只能看到其他已经提交的事务的数据，不能看到active事务修改中的数据。

因为trx_id是单调递增的，因此，我们可以在一个线性序列上依托于trx_id构建可见性，因此：

- trx_id：事务开始的逻辑时间，在事务开始时分配
- trx_no：事务结束的逻辑时间，在事务提交commit阶段分配，这个用于构建purge的可见性
- trx_sys->rw_trx_ids：当前active的读写事务列表，开启读写事务时加入，读写事务提交时移除

以及基于以上构建readview

{{< hint info>}}

MySQL 5.6将事务列表拆分为只读事务列表和读写事务列表

{{</hint>}}

读写异常是因为本事务的R和其他事务的R+W有交集，MVCC也是本着这个本质来解决问题的，解决问题的核心是定序。

只读事务的trx_id始终为0（不分配），读写事务分配trx_id，另外，readview都分配，

### readview

readview的生命周期：

1. 分配：在显式start transaction with consistent snapshot或RR时，事务开始就分配（在事务期间不变，保证可见性不变）；其他情况读取数据时才分配（trx_assign_read_view）
2. 回收：事务commit后，清理资源阶段进行回收（view_close）

全局的readview通过一个readview链表维护（trx_sys->mvcc->readview->m_views），在头部插入新的readview，因此，链表中最后一个还没有close的readview就是oldest readview（这是purge的基础）。

readview由以下几部分构成：

- m_low_limit_id：高水位，分配时取trx_sys::max_trx_id，也就是取当前还没有被分配的事务ID
- m_up_limit_id：低水位，如果m_ids不为空，取其最小值，否则取trx_sys::max_trx_id
- m_ids：当前活跃事务列表（不包括自己）
- m_creator_trx_id：本事务（自己）

本质就是读取时获得一个快照版本，active的，自己的可见，别人的不可见；（高水位）未来的不可见，（低水位）过去的可见。

活跃事务：

1. trx_id > read_view::m_low_limit_id
2. read_view::m_up_limit_id < trx_id < read_view::m_low_limit_id，并且 trx_id 属于 trx_t::read_view::m_ids

已提交事务：

1. trx id < read_view::m_up_limit_id
2. read_view::m_up_limit_id < trx id < read_view::m_low_limit_id，并且 trx id 不属于 trx_t::read_view::m_ids

### 数据可见性

事务为T，trx_id为记录R中的DATA_TRX_ID：

- trx_id > 高水位，T 不可见 R
- trx_id < 低水位，T 可见 R
- 在高低之间，如果是自己，T可见R，否则不可见

比如下图示例：

![InnoDB_txn_MVCC](/InnoDB_txn_MVCC.png)

这里要理解insert_undo为什么事务提交后可以抛弃，readview的遍历方式。

{{< hint danger>}}

这里留给读者2个问题：prepared的事务可见吗？如何理解commit point、complete point和MVCC之间的关系？

{{</hint>}}

### 构建版本

因为在clustered index的行上存储了trx_id和roll_ptr，因此可以通过trx_id判断可见性，如果不可见，则通过roll_ptr继续向前构建，直至可见或者达到尾部。（row_vers_build_for_consistent_read）。

这也是线性系统的一种体现。

{{< hint info>}}

因为构建MV需要持有page latch，如果active事务很多，则undo chain可能会很长，要注意latch的持有时长。

undo log record如果是insert_undo，不用查看undo，因为未提取事务的新插入记录，对其他事务一定不可见。

对于secondary index记录，如果page上的最大事务ID（PAGE_MAX_TRX_ID）对当前事务是可见的，也无需构建MV，否则还要去走clustered index record构建。

{{</hint>}}

## purge

purge操作负责清理已提交的不再被使用的数据，包括记录和日志。

清理这些数据依托于undo log，更进一步，依托于构建在提交序上的点位（trx_no），再结合SQL graph，展开实际的清理。

### 清理操作

purge清理属于事后打扫，这一方面保证forward fast，另一方面为了MVCC的GC，因此，清理工作包括：

- 清理记录：清理delete mark的行记录或者secondary index record（update undo log record中标记修改了secondary index相应值）
- 清理undo log：清理undo log，如果undo page中的所有undo log都清理后，则删除对应的undo segment。在清理undo log时，需要判断当前是否有其他事务正在进行MVCC。

undo有几个序：

1. undo申请序：事务在进行DML操作时申请undo page用于存放undo log，在一个undo segment中一个事务会占用1...n个undo page，但在undo segment是无序的。
2. undo使用序：undo page可以复用，存储不同事务的undo log，在page维度的存储序是committed-committed-active。
3. 提交序：history链表标识已提交事务序，这可以关联到事务中的undo log序，这个undo log序分布在同一个undo segment中的不同undo page中，通过undo log header串联，history和各个undo log header串联起来已提交序。

总结一下：

- undo segment整体是无序的
- undo page按事务状态排序
- 提交序离散分布在rollback segment中的各个undo page中

purge的执行序列通过构建SQL execution graph来实现。

这里注意：

- 事务的提交序和执行序不一样，因此在purge undo和purge记录时有依赖关系，需要构建graph

并且，在进行purge操作时，如果undo page在buffer pool还好，如果已经flush到磁盘，则会产生大量的随机读，这可能会产生性能问题。另外，如果出现了长事务正在使用mvcc（比如最新的active事务），则会导致history链表最头部的已提交事务的undo log的回收无法进行。

### purge实现

purge_sys全局对象（trx_purge_t）保存当前purge的位置和信息。

purge流程如下：

1. 通过purge point确认可见性（purge_sys::view）

   通过clone oldest active readview（clone_oldest_view），来保证purge point之前的事务变更都是可以清理的

   具体解释一下，在事务子系统中，committing事务保存在serialisation_list中，并按照提交序排列（trx_no）。在事务prepare时加入，事务commit时移除

   并且，在readview中还会维护一个serialisation_list LWN（read_view::m_low_limit_no：取自serialisation_list trx_no最小值），其含义是如果其他事务的trx_no比其小，一定已提交。
   这样，一个事务的readview的LWN就确定了，那么全局的LWN即为所有readview的最小点（trx_no，直接按提交序倒排第一个即是），这个点即为purge point

2. 从各个rollback segment中的history链表中依次（从后）读取最多300个undo log（trx_purge_fetch_next_rec递归），分配到各个purge线程的工作队列中（vector thr->child->undo_recs）

   这些history链表已经在事务提交时加入puge_sys::purge_queue中了，即可以将该queue理解为以trx_no为key的优先级队列

   因此，purge_sys::view和所有histroy链表已经存在交集。

   因此这里可以理解成两层关系，purge_queue维护了提交序，负责推进purge，其来源来自于rollback segment的history链表，。

3. 实际purge（细节见下）：分为undo purge和undo truncate

4. 清理history list（每128）

   insert_undo的undo segment事务提交后就释放，update undo在做完history后，也到了释放时机。但因为undo page的复用，一个undo segment是离散分布的，所以需要全局推进。因此，每做完128次purge后，进行一次purge history链表。

   从history链表中遍历释放undo segment，因为history是按照提交序排列的，所以遇到一个还未purge的undo log（trx_no比当前purge point大）则停止。

第三步的purge函数调用栈如下：

~~~~
row_purge_step                        // purge single undo log record(全部清理完成返回node，即purge线程)
  row_purge                           // fetch & purge (300),从purge_queue取第一个rseg，从history链表中取最老的还还没有purge的undo log header，一次读取该事务的undo log record，依次往复，直到300或没有
    row_purge_parse_undo_rec          // parse update_undo
      row_purge_record                // undo记录（secondary->clustered）
        row_purge_del_mark               1. delete mark 1->0 物理删除所有涉及的记录 undo purge
        row_purge_upd_exist_or_extern    2. clustered index in-place update, 可能更新secondary index（delete+insert），清理, undo truncate
~~~~

当一个undo segment段中的最后一个undo log被清理完毕后，则将该undo segment删除（trx_purge_free_segment）。这里有两个细节需要特别注意：

1. 如果一个undo segment包含多个undo page时，当进行undo slot的回收时，首先删除最后一个undo页，并进行mtr_commit。

   如果在这时，即undo segment的最后一个undo log所在的最后一个undo page从段中被删除时，数据库crash，那么重启后，重新进行purge时，会导致最后一个undo log的最后一部分undo log record可能会出现不一致的问题（因为回收的undo page可能已经分配给其他段使用），这时，再进行purge操作会导致错误发生。因此，InnoDB在设计时对每个undo log header，设置了一个TRX_UNDO_DEL_MARKS标记。当最后一个undo log删除时，首先将其置为true，那么crash recovery而重新purge时，由于该变量已置为true，所以不需要继续进行purge操作

2. rollback segment的history链表的更新与undo segment的删除需要在一个mtr中。否则，当数据库宕机时，可能发生rollback segment的history链表已经将undo log删除，而undo segment还存在的情况。在这种情况下，undo segment将没有机会被删除，却需要一直占用存储空间。

purge_is_running用于保护DROP TABLE的操作，当进行purge undo日志时，需要持有该对象的x-latch；当需要进行DROP TABLE时，需要持有该对象的s-latch，以此保证当进行purge操作时，表不会被删除。

latch用于保证删除undo日志的正确性，保证删除时没有其他事务正在引用该undo日志。当开启purge操作时，首先需要持有该x-latch，然后通过read_view_oldest_copy_or_open_new判断哪些undo日志可以被清理（注意：purge是一个特殊的事务，事务类型为TRX_PURGE，其他都是用户事务）。当用户事务需要通过undo日志进行多版本并发控制（一致性非锁定读）时，需要首先获得该对象的s-latch。

## crash recovery

crash recovery的恢复分为两个阶段：

1. forward，恢复到内存态，即将redo log从checkpoint到最新，进行apply，将磁盘上的数据恢复到crash时的内存态，其中也包括undo log。这样就获得ongoing事务列表。

2. backward：因为buffer pool的STEAL+NO-FORCE，需要将active STEAL的数据rollback掉，即找出active+prepared事务，将其rollback，也称为failure atomic。

   具体来讲，遍历所有rollback segment，读取其undo segment中的undo segment page中TRX_UNDO_STATE，可以得知其事务状态（还记得前面说过的吗，一个undo segment只会最多有一个active transaction），如果为活跃事务，则需要遍历该事务的undo来rollback，以及构建出事务子系统的内存布局：trx_sys，trx_t，trx_rseg_t和trx_undo_t
   
   读取最后一个binlog文件，判断文件头是否有LOG_EVENT_BINLOG_IN_USE_F，有则意味着MySQL异常关闭，则依次读取binlog中的event，提取XID用于XA事务恢复

总结一下，数据库系统整体的恢复节奏如下：

1. redo recovery
2. 数据字典子系统初始化
3. 事务子系统初始化&重建
4. XA事务恢复

redo recovery详细的函数调用栈如下：

~~~~
innobase_start_or_create_for_mysql
	recv_sys_var_init();
	recv_sys_create();
		malloc recv_sys
		create mutex/writer_mutex
	recv_sys_init(buf_pool_get_curr_size());
		创建一个MEM_HEAP_FOR_RECV_SYS的heap 用于存放 log records和file
		create flush_start/flush_end event
		recovery时在buffer pool instance中预留好的frame数量 512 recv_n_pool_free_frames
		malloc recv_sys->buf (parser buffer) 2M
		malloc recv_sys->addr_hash 大小为 buffer_pool_size/512
		recv_sys->last_block_buf_start = 2 * 512
		malloc recv_sys->dblwr
	recv_recovery_from_checkpoint_start(flushed_lsn);
	recv_sys->dblwr.pages.clear();
	recv_apply_hashed_log_recs(TRUE);						从log records hash table应用到page
		在buffer pool中， recv_recover_page -> recv_parse_or_apply_log_rec_body
		不在内存中，recv_read_in_area
			buf_read_recv_pages
			buf_read_page_low
				buf_page_io_complete
	recv_recovery_from_checkpoint_finish();

recv_recovery_from_checkpoint_start
	创建buffer pool instances中的flush list（红黑树）buf_flush_init_flush_rbt
	在redo log file group中查找最新的CP的file和其CP位置 recv_find_max_checkpoint
	从file的CP位置读取CP信息放入 log_sys->checkpoint_buf
	从CP信息中解析出 checkpoint_lsn checkpoint_no
	contiguous_lsn = checkpoint_lsn
	从contiguous_lsn开始扫描，把redo log放入hash table recv_group_scan_log_recs
	...

	recv_synchronize_groups();	更新group->lsn, lsn_offset，做checkpoint
	更新log_sys->点位

recv_group_scan_log_recs
	end_lsn = contiguous_lsn = 512对齐后的checkpoint_lsn
	do {
		start_lsn = end_lsn
		end_lsn += 4 * 16K = 128 log block
		log_group_read_log_seg(log_sys->buf, group, start_lsn, end_lsn);	从redo log file中将start_lsn - end_lsn的数据读入log_sys->buf
	} while (!recv_scan_log_recs(
			 available_mem,				page_size * (buffer pool总page-512*instance)
			 &store_to_hash,			STORE_YES / STORE_IF_EXISTS redo log record是否需要存在hash table里
			 log_sys->buf,
			 RECV_SCAN_SIZE,			128 log block
			 checkpoint_lsn,
			 start_lsn, contiguous_lsn, &group->scanned_lsn)

recv_scan_log_recs									解析128个log block Scans log from a buffer and stores new log data to the parsing buffer.
	recv_sys_add_to_parsing_buf						处理每个log block，一直处理完128 log block，拷贝到recv_sys->buf Adds data from a new log block to the parsing buffer of recv_sys
	recv_parse_log_recs								处理recv_sys->buf中的mtr record
		single page/
		recv_parse_log_rec							parse a single log record
		recv_add_to_hash_table						将mtr record放入hash table，元素为recv_t，规则为space_id,page_no
		multiple page
		loop in recv_parse_log_rec （多次）
		loop in recv_parse_log_rec + recv_add_to_hash_table
			recv_parse_or_apply_log_rec_body		在recv_parse_log_rec中只分析

		从log records hash table应用到page
	recv_sys_justify_left_parsing_buf
~~~~


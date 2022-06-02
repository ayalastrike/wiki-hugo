---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: lock
---

日志或者说logging schema主要是为crash recovery algorithms服务的，即为了保证事务的ACD特性。crash recovery是通过多种机制协调解决的，这里面包括buffer pool的flush策略，WAL、logging schema和checkpoint。并且，InnoDB采用MV2PL，还通过undo log在MVCC中实现delta storage用于存储行的多版本快照。

从这里可以看出，日志在事务处理中的重要性。这里的日志主要分为undo log和redo log，undo log在下一章事务中详细介绍，本章聚焦在redo log。

# 设计

crash recovery algorithms由IBM在90年代发表的一篇ARIES论文提出（**A**lgorithms for **R**ecovery and **I**solation **E**xploiting **S**emantics）。InnoDB也借鉴了ARIES的思想。思想的核心是利用日志（undo+redo）来实现可恢复性。

## redo log

redo log是在数据库的运行时，随着数据的更改而产生的变更日志，其目的是两个：

1. 体现数据的变更序
2. 通过变更序持久化的能力，可以让数据库系统在crash recovery后恢复到内存态

因此，我们分别来详细描述如何做到这两点。

首先是体现数据的变更序。

在数据库中，数据都是通过page为单位组织的。因此，在体现数据变更的时候，这个变更序首先是page的变更序。并且，在InnoDB中，page被组成成B+ tree结构，因此，还要体现SMO，也就是说，还要考虑一个逻辑操作可能会设计B+ tree的多个页面，以及不同B+ tree之间的操作关系，这个关系的组织通过mini-transaction来实现，即提供原子粒度的一组redo log（只保证forwrd）保证动作-页面的一致性。。这些操作组合起来形成了事务，换句话说，事务中的每个逻辑操作产生了一系列page的变更序列，即一组redo log。

在整体上，数据库中并发事务之间也需要在整体上进行page排序，这一方面依靠FIX Rules保证page间的并发控制，另一方面B+ tree concurrency control protocol也要从数据结构维度维护并发序，加上MV2PL机制，整体上这个page变更序列就确定了。这个全局page的变更序以一个全局的redo log buffer呈现。

第二点，持久化实际上就是把redo log biffer sync到持久化存储（磁盘）中，sync到磁盘的时机我们称之为sync point。一般来讲，sync point同时也是事务commit点，即WAL机制（force log at commit），也就是保证事务的所有日志（redo+undo）sync到磁盘完成后，事务提交才真正完成，这意味着sync point = commit point。

这个sync point也有group commit的体现，每个用户线程在各自事务中的一个逻辑操作通过mini-transaction提交到redo log buffer，然后再在sync point时一起持久化。换句话说，每次sync都会把当前sync point之前的所有redo log都持久化到磁盘，这一组（group）持久化的日志包含了各个事务并发修改的数据。

## 设计取舍

在设计redo log时，需要保证数据的完整性，也要兼顾效率。

数据的完整性方面，既要考虑磁盘上文件系统所提供的原子写的粒度，不能出现半写（partial-write），也要有校验机制来检验log block的完整，以及是否为异常关闭，还要考虑介质是否会出现损坏。

在效率方面，从上面可以看出，commit point = sync point，这意味着事务处理的速度取决于redo log sync的效率，即使redo log file为顺序IO，也需要充分利用缓存（page cache+sync），兼以group commit，最大化的提升写入效率。

从另一方面，事务处理中所产生的redo log量越小，性能越好。因此，大多数数据库系统（也包括InnoDB）在log schema设计上采用物理逻辑日志（Physiological Logging），不仅保证了日志量足够少，而且还提供幂等性（steal、recovery中二次crash）。也是因为log schema的这个设计，page flush需要保证page的完整性，以保证page在可用的基础上才能apply redo log。

crash recovery也要考虑恢复的日志量，通过checkpoint来维护LWN，page粒度的日志则可以实现page级的并行恢复。

下面来介绍InnoDB中的具体实现。

# 技术

## mini-transaciton

### 概念

[MySQL文档上关于mini transaction的定义](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_mini_transaction)：

An internal phase of InnoDB processing, when making changes at the ***physical*** level to internal data structures during ***DML*** operations. A mini-transaction (mtr) has no notion of ***rollback***; multiple mini-transactions can occur within a single ***transaction***. Mini-transactions write information to the ***redo log*** that is used during ***crash recovery***. A mini-transaction can also happen outside the context of a regular transaction, for example during ***purge*** processing by background threads.

从这里的定义可以看出mini-transaction的几个特征，以下简称mtr：

1. 主要用于DML操作的场景，诣在保证在物理层面上对内部数据结构的更改有效（一致）并通过redo log记录
2. 其本质就是WAL（基于redo log），所以不用进行rollback（因为如果mtr写入过程中失败，修改只在内存中，不会落盘，只有mtr commit也就是完成WAL后，数据才会落盘）
3. 一个事务可能包含多个DML，所以会有多个mtr
4. crash recovery依赖flush后的WAL进行恢复

InnoDB中的ACID，最终要和底层的物理存储上对齐，而通过mini transaction来保证并发情况下和crash后逻辑操作对于页上数据变更的一致性，而事务保证多个逻辑操作数据的一致性和持久性需要通过mtr来实现。

以DML举例，比如插入操作，如果表定义有primary index和secondary index，insert操作需要涉及到两颗index tree的更改，每个index tree可能还会发生SMO，这些操作都要保证物理结构上和DML的一致，即动作-页面的一致性。保证的方式就是通过将一系列页操作包含进一个mini-transaction，可以一起mtr commit到1，否则就是0。

为了使mtr可以保证物理页在数据结构上的一致性，mtr需要遵循以下几个规则：

- FIX Rules：保证并发操作的隔离性
- Write-Ahead Log（WAL）：可以持续写入日志，保证日志在mtr粒度合入redo log
- Force-log-at-commit：事务提交前必须先写入日志，保证持久性

### FIX Rules

当数据库访问或者修改一个页时，需要持有该页的latch（frame rwlock），以此保证并发情况下数据的一致性。acquire page latch的操作称为fixing the page。当latch granted后，称这个页已经fixed，unlock page latch的操作，称为unfixing。

FIX Rules定义如下：

- 修改一个页时，需要获得X-latch
- 访问一个页时，需要获得X-latch或者S-latch
- 持有latch直到页的修改或者访问完成

如果操作涉及到多个页，那么根据FIX Rules的要求，需要持有多个页的latch，并在所有页的操作完成后，再释放latch。

这里的latch类型有：

- PAGE_S_FIX：page latch
- PAGE_X_FIX：page latch
- PAGE_SX_FIX：page latch
- BUF_FIX：only buf fix
- S_LOCK：index latch
- X_LOCK：index latch
- SX_LOCK：index latch

关于frame rwlock和index concurrency control参见buffer pool一章和B+ tree一章。

### Write-Ahead Log

Write-Ahead-Log要求在把page写入磁盘之前，必须首先将日志写入磁盘中。

Write-Ahead Log流程如下：

- 每个页有一个LSN（有次序）
- 每次页的修改都需要维护该LSN（维护次序），并将页FIX
- 当一个页写入到持久存储设备时，必须将内存中所有小于该LSN的日志首先写入到磁盘中
- 当日志写入后，再将页写入到磁盘
- 在将页写入到持久存储设备时，需要将页FIX，以此保证内存页中数据的一致性

这里的日志就是redo log，redo log写入的磁盘设备是redo log file，page写入到对应的文件表空间。

### Force-log-at-commit

Write-Ahead Logging保证了写入的顺序：先日志再数据，但是，并不能保证日志”真正“写入到了持久存储设备上。为了严格保证持久性，还需要Force-log-at-commit，即当事务提交时，日志必须“刷新”到持久存储设备上。这样，在日志刷盘后，page刷盘前发生crash，来保证crash recovery时可以通过日志将页回放到修改的内存态。

事务的D需要Force-log-at-commit来保证，细节在commit point一节介绍。

log_write_up_to()调用链

~~~~c++
ha_innobase::write_row(buf)
    innobase_commit(ht, m_user_thd, 1)
        trx_commit_complete_for_mysql(trx->commit_lsn, trx))
            trx_flush_log_if_needed(lsn)
                trx_flush_log_if_needed_low(lsn, flush)
                    switch (srv_flush_log_at_trx_commit)
                    case 2:
                            /* Write the log but do not flush it to disk */
                            flush = false;
                    case 1:
                            /* Write the log and optionally flush it to disk */
                            log_write_up_to(lsn, flush);
                            return;
                    case 0:
                            /* Do nothing */
                            return;
~~~~

### mtr数据结构

~~~~
struct mtr_t {
    struct Impl {
        mtr_buf_t   m_memo;                 // 管理latch信息（以栈的方式，方便释放，遍历从栈顶到栈底，也方便释放保序）
        mtr_buf_t   m_log;                  // 管理日志信息
        bool        m_made_dirty;           // buffer pool中的page是否为脏页 TRUE (X_FIX/SX_FIX) && block_dirty 或者FIX && buf_block.made_dirty_with_no_latch （临时表只有当前会话线程修改）
        bool        m_inside_ibuf;          // 是否为insert buffer使用
        bool        m_modifications;        // 是否产生了脏页
        ib_uint32_t m_n_log_recs;           // 有多少页的日志放入了m_log
        mtr_log_t   m_log_mode;             // MTR_LOG_ALL：    记录所有会修改磁盘数据的操作
                                               MTR_LOG_NONE：   不记录redo，脏页也不放到flush list上
                                               MTR_LOG_NO_REDO：不记录redo，但脏页放到flush list上
                                               MTR_LOG_SHORT_INSERTS：插入记录操作REDO，在将记录从一个page拷贝到另外一个新建的page时用到，此时忽略写索引信息到redo log中 (只有page_copy_rec_list_end_to_created_page使用)
        ulint       m_user_space_id         // 修改的哪个文件（初始化为TRX_SYS_SPACE）
        // 这三个表空间是事务持久化到外存的联系纽带
        fil_space_t*    m_user_space;       // 指向被mtr修改的用户表空间
        fil_space_t*    m_undo_space;       // 指向被mtr修改的UNDO日志表空间
        fil_space_t*    m_sys_space;        // 指向被mtr修改的系统表空间
        mtr_state_t m_state;                // mtr状态，见状态流转图
        FlushObserver*  m_flush_observer;
        mtr_t*      m_mtr;                  // 指向外面的mtr_t
    };
    Impl m_impl;
    lsn_t m_commit_lsn;                     // mtr commit时的lsn，提交时Command.m_end_lsn = m_commit_lsn
    class Command;                          // 负责具体的mtr动作，比如提交，释放latch/资源，unfix page
}
~~~~

从mtr的内部数据结构可以看出mtr最核心的两个功能就是latch（m_memo）和日志（m_log），这两种信息都是通过mtr_buf_t来组织的，内部是动态分配的block_t双向链表(每个block是512字节，也是redo log block的大小)，如下图所示：

![InnoDB_redo_log_mtr_mtr_buf_t](/InnoDB_redo_log_mtr_mtr_buf_t.png)

mtr_buf_t的函数列表：

| 函数                                                       | 说明                                                         |
| :--------------------------------------------------------- | :----------------------------------------------------------- |
| 函数                                                       | 说明                                                         |
| ctor() earse()                                             | 初始化状态，只分配第一个block，没有分配m_heap                |
| is_small()                                                 | 判断是否为初始化状态                                         |
| add_block()                                                | 分配m_heap，并分配block加到链表尾部                          |
| open(size)                                                 | 在block中预留空间，返回空闲block的end()                      |
| close(ptr)                                                 | 将m_used重置到ptr位置，m_buf_end归0                          |
| has_space(size)                                            | block是否有空间存储size个字节                                |
| find(pos)                                                  | 在所有block中的空闲block上查找pos（pos<end），返回该block    |
| for_each_block(functor) for_each_block_in_reverse(functor) | 遍历所有block并执行functor(block) 反向遍历所有block并执行functor(block) |
| at(ptr)                                                    | find()的block在ptr的位置                                     |
| push(size)                                                 | 返回open预留的开始位置（end()）                              |
| push(ptr, len)                                             | 使用上面的push移动len个字节                                  |

mini-transaction的状态流转图：

![InnoDB_redo_log_mtr_state](/InnoDB_redo_log_mtr_state.png)

### 使用mtr

~~~~
mtr_t mtr;
mtr_start(&mtr);
// FIX
// 变更页
// 写入日志
mtr_commit(&mtr); // UNFIX
~~~~

用伪代码的形式描述mtr的内部流程：

~~~~
mini_transaction_for_single_page() {
   create mtr object and init         // ① ctor & mtr.start()
   lock the page in exclusive mode    // ⓶ mtr_memo_push()
   transform the page                 // ③
   generate undo and redo log record  // ⓸
   unlock the page                    // ⓹ mtr.commit() -> release_latches()
}
~~~~

如果一个mtr包含多个页的修改：

~~~~
mini_transaction_for_multiple_page() {
   create mtr object and init         // ① ctor & mtr.start()
   page1                              // ⓶ ③ ⓸
   page2                              // ⓶ ③ ⓸
   ...
   pageN                              // ⓶ ③ ⓸
   write MLOG_MULTI_REC_END           // 记录了多个页的变更日志，合成整体，并入redo log buffer
   unlock the pageN...2,1             // ⓹ mtr.commit() -> release_latches()
}
~~~~

写入数据到mtr的m_log

~~~~
byte *mlog_open(mtr_t *mtr, ulint size)  // 打开mtr的m_log，即定位到最新一个block_t（有m_buf_end可用），并返回上一次写入的位置
mlog_write_initial_log_record_low()      // 函数向m_log中写入type，space id，page no，并增加修改的页的数量（m_n_log_recs）
mtr->get_log()->push()                   // 按不同的log item type写数据
mlog_close()                             // 更新m_log.m_size
~~~~

mtr record type有60多种（mlog_id_t），但所有mtr record的头部元素是相同的，都包含type（1字节），space_id（compressed）和page_no（compressed）（mlog_write_initial_log_record_low）

![InnoDB_redo_log_mtr_record](/InnoDB_redo_log_mtr_record.png)

type为MLOG_TRUNCATE的示例代码如下：

~~~~
mtr_start(&mtr);
log_ptr = mlog_open(&mtr, 11 + 8);
log_ptr = mlog_write_initial_log_record_low(MLOG_TRUNCATE, m_table->space, 0, log_ptr, &mtr);
mach_write_to_8(log_ptr, lsn);
log_ptr += 8;
mlog_close(&mtr, log_ptr);
mtr_commit(&mtr);
~~~~

mtr commit

~~~~
void mtr_t::commit()
{
    m_impl.m_state = MTR_STATE_COMMITTING;
    Command cmd(this);
    if (m_impl.m_modifications && (m_impl.m_n_log_recs > 0 || m_impl.m_log_mode == MTR_LOG_NO_REDO)) {
        cmd.execute();          // 页修改，执行mtr提交
    } else {                  
        cmd.release_all();      // 否则只单纯的释放资源
        cmd.release_resources();
    }
}
~~~~

mtr_t的整体数据结构如下图所示：

![InnoDB_redo_log_mtr_t](/InnoDB_redo_log_mtr_t.png)

提交详细流程：

~~~~c++
void mtr_t::Command::execute()
{
    // prepare_write
    // 1. MTR_LOG_NONE、MTR_LOG_NO_REDO直接将m_end_lsn=m_start_lsn=log_sys->lsn
    // 2. 计算len = m_impl->m_log.size();
    // 3. 如果修改了单页的记录，更新MLOG_SINGLE_REC_FLAG（日志头的type第1个bit）
    // 4. 如果修改了多页的记录，写入MLOG_MULTI_REC_END（1字节）
    if (const ulint len = prepare_write()) {
        // finish_write
        // 1. 获取全局lsn（log_sys->lsn）作为m_start_lsn
        // 2. 将所有的block_t写入redo log buffer，按照log block粒度拷贝
        // 3. 获得m_end_lsn（log_sys->lsn）
        finish_write(len);                
    }
    //
    if (m_impl->m_made_dirty) {
        log_flush_order_mutex_enter();
    }
    log_mutex_exit();
    // 将mtr的m_commit_lsn设为m_end_lsn
    m_impl->m_mtr->m_commit_lsn = m_end_lsn;
    release_blocks();
    if (m_impl->m_made_dirty) {
        log_flush_order_mutex_exit();
    }
    release_latches();
    release_resources();
}
~~~~

### 实例

以一条insert语句为例：

~~~~
第一个 mtr 从 row_ins_clust_index_entry_low 开始
  
mtr_start(mtr_1) // mtr_1 贯穿整条insert语句
row_ins_clust_index_entry_low
  
mtr_s_lock(dict_index_get_lock(index), mtr_1) // 对index加s锁
btr_cur_search_to_nth_level
row_ins_clust_index_entry_low
  
mtr_memo_push(mtr_1) // buffer RW_NO_LATCH 入栈
buf_page_get_gen
btr_cur_search_to_nth_level
row_ins_clust_index_entry_low
  
mtr_memo_push(mtr_1) // page RW_X_LATCH 入栈
buf_page_get_gen
btr_block_get_func
btr_cur_latch_leaves
btr_cur_search_to_nth_level
row_ins_clust_index_entry_low
      
    mtr_start(mtr_2) // mtr_2 用于记录 undo log
    trx_undo_report_row_operation
    btr_cur_ins_lock_and_undo
    btr_cur_optimistic_insert
    row_ins_clust_index_entry_low
      
        mtr_start(mtr_3) // mtr_3 分配或复用一个 undo log
        trx_undo_assign_undo
        trx_undo_report_row_operation
        btr_cur_ins_lock_and_undo
        btr_cur_optimistic_insert
        row_ins_clust_index_entry_low
          
        mtr_memo_push(mtr_3) // 对复用（也可能是分配）的 undo log page 加 RW_X_LATCH 入栈
        buf_page_get_gen
        trx_undo_page_get
        trx_undo_reuse_cached // 这里先尝试复用，如果复用失败，则分配新的 undo log
        trx_undo_assign_undo
        trx_undo_report_row_operation
  
        trx_undo_insert_header_reuse(mtr_3) // 写 undo log header
        trx_undo_reuse_cached
        trx_undo_assign_undo
        trx_undo_report_row_operation
         
        trx_undo_header_add_space_for_xid(mtr_3) // 在 undo header 中预留 XID 空间
        trx_undo_reuse_cached
        trx_undo_assign_undo
        trx_undo_report_row_operation
          
        mtr_commit(mtr_3) // 提交 mtr_3
        trx_undo_assign_undo
        trx_undo_report_row_operation
        btr_cur_ins_lock_and_undo
        btr_cur_optimistic_insert
        row_ins_clust_index_entry_low
      
    mtr_memo_push(mtr_2) // 即将写入的 undo log page 加 RW_X_LATCH 入栈
    buf_page_get_gen
    trx_undo_report_row_operation
    btr_cur_ins_lock_and_undo
    btr_cur_optimistic_insert
    row_ins_clust_index_entry_low
      
    trx_undo_page_report_insert(mtr_2) // undo log 记录 insert 操作
    trx_undo_report_row_operation
    btr_cur_ins_lock_and_undo
    btr_cur_optimistic_insert
    row_ins_clust_index_entry_low
      
    mtr_commit(mtr_2) // 提交 mtr_2
    trx_undo_report_row_operation
    btr_cur_ins_lock_and_undo
    btr_cur_optimistic_insert
    row_ins_clust_index_entry_low
      
/*
    mtr_2 提交后开始执行 insert 操作
    page_cur_insert_rec_low 具体执行 insert 操作
    在该函数末尾调用 page_cur_insert_rec_write_log 写 redo log
*/
  
page_cur_insert_rec_write_log(mtr_1) // insert 操作写 redo log
page_cur_insert_rec_lowpage_cur_tuple_insert
btr_cur_optimistic_insert
  
mtr_commit(mtr_1) // 提交 mtr_1
row_ins_clust_index_entry_low
~~~~

事务提交时也会涉及mini transaction：

~~~~
提交事务时，第一个 mtr 从 trx_prepare 开始
  
mtr_start(mtr_4) // mtr_4 用于 prepare transaction
trx_prepare
trx_prepare_for_mysql
innobase_xa_prepare
ha_prepare_low
MYSQL_BIN_LOG::prepare
ha_commit_trans
trans_commit_stmt
mysql_execute_command
  
mtr_memo_push(mtr_4) // undo page 加 RW_X_LATCH 入栈
buf_page_get_gen
trx_undo_page_get
trx_undo_set_state_at_prepare
trx_prepare
  
mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, undo->state, MLOG_2BYTES, mtr_4) 写入TRX_UNDO_STATE
trx_undo_set_state_at_prepare
trx_prepare
  
mlog_write_ulint(undo_header + TRX_UNDO_XID_EXISTS, TRUE, MLOG_1BYTE, mtr_4) 写入 TRX_UNDO_XID_EXISTS
trx_undo_set_state_at_prepare
trx_prepare
  
trx_undo_write_xid(undo_header, &undo->xid, mtr_4) undo 写入 xid
trx_undo_set_state_at_prepare
trx_prepare
  
mtr_commit(mtr_4) // 提交 mtr_4
trx_prepare
  
mtr_start(mtr_5) // mtr_5 用于 commit transaction
trx_commit
trx_commit_for_mysql
innobase_commit_low
innobase_commit
ha_commit_low
MYSQL_BIN_LOG::process_commit_stage_queue
MYSQL_BIN_LOG::ordered_commit
MYSQL_BIN_LOG::commit
ha_commit_trans
trans_commit_stmt
mysql_execute_command
  
mtr_memo_push(mtr_5) // undo page 加 RW_X_LATCH 入栈
buf_page_get_gen
trx_undo_page_get
trx_undo_set_state_at_finish
trx_write_serialisation_history
trx_commit_low
trx_commit
  
trx_undo_set_state_at_finish(mtr_5) // set undo state， 这里是 TRX_UNDO_CACHED
trx_write_serialisation_history
trx_commit_low
trx_commit
  
mtr_memo_push(mtr_5) // 系统表空间 transaction system header page 加 RW_X_LATCH 入栈
buf_page_get_gen
trx_sysf_get
trx_sys_update_mysql_binlog_offset
trx_write_serialisation_history
trx_commit_low
trx_commit
  
trx_sys_update_mysql_binlog_offset // 更新偏移量信息到系统表空间
trx_write_serialisation_history
trx_commit_low
trx_commit
  
mtr_commit(mtr_5) // 提交 mtr_5
trx_commit_low
trx_commit
~~~~

至此insert语句涉及的mini transaction全部结束。

## logging schema

Physiological Logging的设计思想是：physical-to-a-page, logical-within-a-page（对页是物理的，页内部的操作是逻辑的），其兼顾了物理日志和逻辑日志的优点。

采用Physiological Logging，意味着既要描述物理信息，也要描述逻辑信息。我们在page那章已经给出了记录在insert、delete下的日志格式。从格式上可以看出，物理信息用space_id+page_no+offset表示，逻辑信息则根据op的不同，因操作而异。换句话说，以page为单位记录变更位置，在page内以逻辑的方式记录具体的数据变化。

即logging schema如下：

~~~~
redo log entry：(op, space_id, page_no, logical description for data)
mini-transaction：redo log entry sequence < 1 ... n > // 先保存在private repo中，在mtr commit中合入全局
redo log buffer // 体现内存更改序（提交）：mini-transaction sequence<1 ... n>
~~~~

同样，由于Physiological Logging的方式，需要解决2个问题：

1.  需要基于正确的page状态上re-play redo log
   由于在一个page内，REDO是以逻辑的方式记录了前后两次的修改，因此重放redo log必须基于正确的Page状态，也就是page要保证是完整的。然而InnoDB的page是16KB，大于文件系统能保证的原子写粒度（4KB），可能出现partial-write，因此，InnoDB通过shadow page的方式提供了double-write机制来保证partial-write出现时可以通过recovery恢复到正确的page状态。
2. 需要保证redo log re-play的幂等
   因为InnoDB的buffer pool policy采用了STEAL+NO_FORCE，因此可能有未提交的事务的redo log flush到了磁盘，在恢复时无需再次重放redo log；另外，可能在crash recovery过程中出现二次故障，这也需要redo log重放提供幂等的能立。因此，InnoDB通过LSN（Log Sequence Number）来维护全局点位，然后在page上也记录一个LSN（page LSN），在恢复时，通过笔记redo log LSN是否小于page LSN来判定是否需要在page上apply redo log，从而实现redo log重放的幂等。

在数据库系统中，redo log可以有以下几种实现方式：

- 物理日志
- 逻辑日志
- 物理逻辑日志

物理日志（physical logging）保存一个页中发生改变的字节，也称这种方式为old value-new value logging。通常来说，其数据结构可参考下面的实现：

~~~~
struct value_log{
   int     opcode;
   long    page_no;
   long    offset;
   long    length;
   char    old_value[length];
   char    new_value[length];
}
~~~~

物理日志的好处是：其保存的是页中发生变化的字节。这样重复多次执行该日志不会导致数据发生不一致的问题，这也就意味着该日志是幂等的。然而物理日志最大的问题是，其日志产生的量相对较大。例如对一个页进行重新整理（reorganize），那么这时的日志大小可能就为页的大小。如果一个操作涉及对多个页的修改，那么需要分别对不同页进行记录。

逻辑日志记录的是对于表的操作，这非常类似于MySQL数据库上层产生的二进制日志。由于是逻辑的，因此其日志的尺寸非常小。例如对于插入操作，仅需类似如下的格式：

~~~~
<insert op, table name, record value>
~~~~

UNDO操作仅需对记录的日志操作进行逆操作。例如INSERT对应DELETE操作，DELETE对应INSERT操作。然而该日志的缺点同样非常明显，那就是在恢复时可能无法保证数据的一致性。例如当对表进行插入操作时，表上还有其他辅助索引。当操作未全部完成时系统发生了宕机，那么要回滚上述操作可能很困难。因为，这时数据可能处在一个未知的状态。无法保证回滚之后数据的一致性。

## LSN

[LSN](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_lsn)是log sequence number的缩写，代表的是日志序列号，具有单调递增的属性。LSN贯穿整个redo log的生命周期，从mini-transaction提交开始，到redo log buffer及其log block的组织，以及外村中redo log file中的点位，以及page LSN，直至checkpoint。

在InnoDB中，LSN分配8个字节，具体定义为：

~~~~
# log0log.h
/* Type used for all log sequence number storage and arithmetics */
typedef ib_uint64_t     lsn_t;
 
#define LSN_MAX         IB_UINT64_MAX
 
# univ.i
/** Maximum value for ib_uint64_t */
#define IB_UINT64_MAX     ((ib_uint64_t) (~0ULL))
~~~~

{{< hint info >}}

在MySQL 5.6.3之后LSN从4个字节扩充为8个字节，随之redo log file group的大小也从4GB扩大到512GB。

{{</hint>}}

在InnoDB中，LSN的初始值由LOG_START_LSN定义：

~~~~
/* The counting of lsn's starts from this value: this must be non-zero */
#define LOG_START_LSN       ((lsn_t) (16 * OS_FILE_LOG_BLOCK_SIZE)) // 初始化为MAX（这样初始分配从0开始），使用区间是0 ~ 16 * OS_FILE_LOG_BLOCK_SIZE
~~~~

上面已经提到，LSN贯穿整个redo log的生命周期，其存在于多个对象中，表示的含义各不相同：

- redo log
- checkpoint
- page

LSN表示事务写入到redo log的字节量。例如当前redo log的LSN为1000，事务T1写入了100个字节的redo log，那么LSN就变为1100，如果事务T2又写入了200字节的redo log，那么LSN变为1300。由于redo log有内存和外存两种状态，因此会记录两个点位：

- 写入到redo log buffer（log_sys→lsn）的点位
- 写入redo log file（log_sys→flushed_to_disk_lsn）的点位

LSN还记录在page中，表示page持久化的日志点位，即page LSN。redo log依据page LSN来判断该page是否需要进行恢复操作

checkpoint也通过LSN的形式来保存，其表示页已经刷新到磁盘的LSN位置，当数据库重启时，仅需从checkpoint开始进行恢复操作。若checkpoint LSN与redo log file LSN相同，表示所有page都已经刷新到磁盘，不需要进行恢复操作。

~~~~
show engine innodb status
---
LOG
---
Log sequence number 256862524   // redo log buffer LSN
Log flushed up to   256810600   // redo log file LSN
Pages flushed up to 256051783   // 最老的dirty page的LSN
Last checkpoint at  256050987   // checkpoint LSN，即脏页刷新到磁盘的LSN
  
内部的数据结构表示和更新时机：
log_sys->lsn // mtr刷入redo log buffer时更新（mtr_commit()）
log_sys->flushed_to_disk_lsn // redo log buffer刷入redo log file时更新（log_write_up_to()）
flush_list->buf_page_t->oldest_modification // 取自所有buffer_pool里flush_list oldest_modification最小的，buf_page_t->oldest_modification在页加入到flush list时赋值
log_sys->last_checkpoint_lsn); // 写入checkpoint后更新（log_checkpoint()）
  
除此以外，页也有LSN信息：
FIL_PAGE_LSN：页最后flush时的LSN
~~~~

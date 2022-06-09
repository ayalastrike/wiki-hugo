---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: redo log
---

日志或者说logging schema主要是为crash recovery algorithms服务的，即为了保证事务的ACD特性。crash recovery是通过多种机制协调解决的，这里面包括buffer pool的flush策略，WAL、logging schema和checkpoint。并且，InnoDB采用MV2PL，还通过undo log在MVCC中实现delta storage用于存储行的多版本快照。

从这里可以看出，日志在事务处理中的重要性。这里的日志主要分为undo log和redo log，undo log在下一章事务中详细介绍，本章聚焦在redo log。crash_recovery在事务一章介绍。

# 设计

crash recovery algorithms由IBM在90年代发表的一篇`ARIES`论文提出（**A**lgorithms for **R**ecovery and **I**solation **E**xploiting **S**emantics）。InnoDB也借鉴了ARIES的思想。思想的核心是利用日志（undo+redo）来实现可恢复性。

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

## redo log layout

从上面可以看出，redo log分为内存态的数据和外存态的数据。

在内存中，redo log在mini-transaciton中产生，以stack的结构保存在各个用户线程自己的内存空间中。然后再mini-trnasaction commit时，合并到全局的redo log buffer中，由一个个redo log record组成。

这一层称为逻辑redo log层。

外村中的redo log分为两层，最下层是以文件方式存在（redo log file），块设备（文件）中块的粒度就是redo log block（512 bytes），这是为了保证日志的原子写。这层称为redo log文件层。

为了避免创建文件以及初始化空间、预防文件自增带来的开销，redo log file最常见的一种组织方式是提供n个redo log file首尾相接一个逻辑的文件（ring file），作为redo log表空间，这层称为redo log buffer层。

内存和外存的一致性通过force log at commit机制保证的。

如下图所示：![InnoDB_redo_log_layout](/InnoDB_redo_log_layout.png)

在逻辑redo log层用全局唯一递增的SN表示，并在日志子系统中维护当前SN的最大值（log_sys→lsn）

而在物理层（buffer & 文件）中，抽象出了为了IO读写的log block，并且为了维护log block的元信息（log block header & tailer），在物理层用LSN标识。

LSN和SN的换算关系为：

````
constexpr inline lsn_t log_translate_sn_to_lsn(lsn_t sn) {
  return (sn / LOG_BLOCK_DATA_SIZE * OS_FILE_LOG_BLOCK_SIZE +
          sn % LOG_BLOCK_DATA_SIZE + LOG_BLOCK_HDR_SIZE);
}
````

SN加上之前所有的block的header以及tailer的长度就可以换算到对应的LSN，反之亦然。

而在文件层中，InnoDB在每个文件的开头固定预留4个block来记录一些额外的信息，其中第一个block称为header block，之后的3个block在0号文件上用来存储duplex checkpoint信息，而在其他文件上留空。如下图所示：![InnoDB_redo_log_file_group](/InnoDB_redo_log_file_group.png)

逻辑redo是真正需要的数据，用SN索引，逻辑redo按固定大小的block组织，并添加block的头尾信息形成物理redo，以LSN索引，这些block又会放到循环使用的文件空间中的某一位置，文件中用offset索引。

从中可以看出，逻辑redo log层时真正的日志数据，用SN索引；逻辑redo log按固定大小的log block组织在一起，并添加block header+tailer形成物理redo log，用LSN索引；而这些log block又会由循环使用的日志表空间中，在文件中用offset索引，而offset对应的的文件的读写IO。

offset和LSN的转换关系如下：

````
const auto real_offset =
      log.current_file_real_offset + (lsn - log.current_file_lsn);
````

切换文件时会在内存中更新当前文件开头的文件offset，current_file_real_offset，以及对应的LSN，current_file_lsn，通过这两个值可以方便地用上面的方式将LSN转化为文件offset。注意这里的offset是相当于整个redo文件空间而言的，由于InnoDB中读写文件的space层实现支持多个文件，因此，可以将首位相连的多个redo文件看成一个大文件，那么这里的offset就是这个大文件中的偏移。

### redo log buffer

redo log buffer的大小由参数innodb_log_buffer_size控制，默认大小为16MB。

从整体上看，redo log buffer可以看作是由redo log block组成的一个线性数组，每个数组尾头存储数组元素的元信息。

redo log block header中各个字段的含义：

| 字段                      | 大小 | 说明                                                         |
| :------------------------ | :--- | :----------------------------------------------------------- |
| LOG_BLOCK_HDR_NO          | 4    | block number，也就是当前log block在redo log buffer（线性数组）中的位置，最高的1个bit用于标识是否flush过，所以整个线性数组的容量是2G (2^（8*4-1） = 2G) |
| LOG_BLOCK_HDR_DATA_LEN    | 2    | 当前log block已写入了多少字节，如果为0x200则已写满（0x200 = 512） |
| LOG_BLOCK_FIRST_REC_GROUP | 2    | 当前log block中第一个mtr log record的偏移量                  |
| LOG_BLOCK_CHECKPOINT_NO   | 4    | log_sys->next_checkpoint_no的低4位，恢复时使用（从CP1/CP2上读取CP后，CP后的redo log block上的该字段都应该大于>CP） |

redo log block tailer中各个字段的含义：

| 字段               | 大小 | 说明                                                         |
| :----------------- | :--- | :----------------------------------------------------------- |
| LOG_BLOCK_CHECKSUM | 4    | 该log block的checksum，用于recovery时检测log block是否损坏；在3.23.52之前为LOG_BLOCK_HDR_NO |

我们以一个具体的例子来说明redo log block的组织：有两个事务T1和T2的redo log写入redo log buffer，事务T1的redo log，也就是mtr log record，为1254字节，下图标记为青色，事务T2的mtr log record为100字节，下图标记为黄色。T1需要3个log block才能盛下，T2小于492字节，且T1写入后第3个log block的剩余空间大于100字节，所以放到第三个log block中。那么在这种情况下，我们来看一下头尾的metadata是如何表示的：

- LOG_BLOCK_HDR_NO：从第一个log block到第四个log block的编号依次为0、512、1024、1536，换算成16进制为0x0000、0x0200、0x0400、0x0600
- LOG_BLOCK_HDR_DATA_LEN：第一个和第二个log block都用完了，所以为512字节（0x0200），第三个块写入了370字节（270+100，0x172），第四个快，否则记录真实的字节数，未使用记录为0（0x0000）
- LOG_BLOCK_FIRST_REC_GROUP：记录该log block中第一个mtr log record的相对偏移量，对于事务T1，第一个log block相对于该log block的偏移量为12（0x000C），第二个log block用满了492字节，且实际相对于第一个log block数据末尾的偏移量为8+12+492=512（0x0200），即LOG_BLOCK_FIRST_REC_GROUP=LOG_BLOCK_HDR_DATA_LEN，可以根据该公式判断该log block是否有新的mtr log record（即该块为上一个mtr log record的延续），在第三个log block中，T1写入了270就结束了（1254-492-492），事务T2（下图标记为橙色）的mtr log record为该log block的第一个记录，其相对于该log block的偏移量为282（270+12，0x011A），第四块log block没有存放任何mtr log record，为0。

````c++
if (log_block_get_flush_bit(log_block)) {
    /* This block was a start of a log flush operation:
    we know that the previous flush operation must have
    been completed for all log groups before this block
    can have been flushed to any of the groups. Therefore,
    we know that log data is contiguous up to scanned_lsn
    in all non-corrupt log groups. */
 
    if (scanned_lsn > *contiguous_lsn) {
        *contiguous_lsn = scanned_lsn;
    }
}
 
data_len = log_block_get_data_len(log_block);
 
if (scanned_lsn + data_len > recv_sys->scanned_lsn
    && log_block_get_checkpoint_no(log_block)
    < recv_sys->scanned_checkpoint_no
    && (recv_sys->scanned_checkpoint_no
    - log_block_get_checkpoint_no(log_block)
    > 0x80000000UL)) {
 
    /* Garbage from a log buffer flush which was made
    before the most recent database recovery */
    finished = true;
    break;
}
````

数据组织为下图所示：

![InnoDB_redo_log_block_example](/InnoDB_redo_log_block_example.png)

### redo log files

redo log日志表空间中的每个redo log file头部会预留2KB（LOG_FILE_HDR_SIZE = (4 * OS_FILE_LOG_BLOCK_SIZE)）信息用于存储文件的元信息（但只有文件0使用），其余部分用于存储log block：

| 存储内容        | 大小 |
| :-------------- | :--- |
| log file header | 512  |
| checkpoint 1    | 512  |
| /               | 512  |
| checkpoint 2    | 512  |

如下图所示：

![InnoDB_redo_log_file_group](/InnoDB_redo_log_file_group-1654323649389.png)

从这里可以看到，redo log file的写入并不完全是顺序IO的，因为在将redo log buffer sync到文件block（append only）后，还需要更新头部的checkpoint信息。

log file header内容如下：

| 字段                 | 大小 | 说明                                  |
| :------------------- | :--- | :------------------------------------ |
| LOG_HEADER_FORMAT    | 4    | redo log格式版本，目前是v1            |
| LOG_HEADER_START_LSN | 8    | redo重做日志文件中的第一个日志的lsn   |
| LOG_HEADER_CREATOR   | 16   | MySQL版本版本信息，比如"MySQL 5.7.26" |

### checkpoint值

checkpoint是crash recovery的关键路径（key point），要最大程度上的保证checkpoint的可用性。因此，设计了两个checkpoint的目的是采用交替写入来避免介质失败。

checkpoint预留了512字节，但实际上使用了32字节：

| 字段                        | 大小 | 说明                                              |
| :-------------------------- | :--- | :------------------------------------------------ |
| LOG_CHECKPOINT_NO           | 8    | checkpoint NO，单调递增，每次做完checkpoint+1     |
| LOG_CHECKPOINT_LSN          | 8    | checkpoint的LSN                                   |
| LOG_CHECKPOINT_OFFSET       | 8    | checkpoint的LSN对应的在redo log file中的偏移量    |
| LOG_CHECKPOINT_LOG_BUF_SIZE | 8    | 做checkpoint时，redo log buffer的大小，无实际意义 |

在重启恢复时，只需要恢复LOG_CHECKPOINT_LSN后面的redo log。由于有两个checkpoint，重启时读取两个，采用较大的LOG_CHECKPOINT_LSN。

## sync point & commit point

为了保证redo log buffer刷入redo log file的持久性，每次都要fsync。这是因为InnoDB打开redo log file时并没有使用direct IO（O_DIRECT），所以为了确保数据写入到磁盘上，需要使用fsync的方式保证synchronized IO file integrity completion。而fsync的效率取决于磁盘的性能，而fsync的效率决定了事务提交的性能，也就是写入性能。

sync的时机如下：

- 事务commit时
- 写入checkpoint时
- 当log buffer中有已使用空间超过某个阈值时

另外，配置选项[innodb_flush_log_at_trx_commit](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)和 innodb_flush_method分别控制着**何时（when）**以及**如何（how）**对redo log file进行sync。

### innodb_flush_log_at_trx_commit

InnoDB允许用户通过innodb_flush_log_at_trx_commit参数控制redo log buffer写入和sync的时机，即sync point。

default：1

 innodb_flush_log_at_trx_commit有3个选项，可以根据速度和安全的不同要求加以选择。

- 0：把redo log file每隔1s写入文件，但sync由操作系统控制。换句话说，0不能确保ACID中的D
- 1：每次事务commit写入文件并sync 
- 2：折衷，每次事务commit写入文件，每隔1s sync

如下图所示：

![InnoDB_redo_log_innodb_flush_log_at_trx_commit](/InnoDB_redo_log_innodb_flush_log_at_trx_commit.png)

{{< hint danger>}}

DDL changes and other internal InnoDB activities flush the log independently of the innodb_flush_log_at_trx_commit setting.

{{</hint>}}

{{< hint info>}}

**MySQL · 参数故事 · innodb_flush_log_at_trx_commit 摘自阿里数据库内核月报**

**背景**

innodb_flush_log_at_trx_commit 这个参数可以说是InnoDB里面最重要的参数之一，它控制了重做日志（redo log）的写盘和落盘策略。

简单说来，可选值的安全性从0->2->1递增，分别对应于mysqld 进程crash可能丢失 -> OS crash可能丢失 -> 事务安全。

以上是路人皆知的故事，并且似乎板上钉钉，无可八卦……

**innodb_use_global_flush_log_at_trx_commit**

直到2010年的某一天，Percona的CTO Vadim同学觉得这种一刀切的风格不够灵活，最好把这个变量设置成session级别，每个session自己控制。

但同时为了保持Super权限对提交行为的控制，同时增加了innodb_use_global_flush_log_at_trx_commit参数。 这两个参数的配合逻辑为：

　　1、若innodb_use_global_flush_log_at_trx_commit为OFF，则使用session.innodb_flush_log_at_trx_commit;

　　2、若innodb_use_global_flush_log_at_trx_commit为ON,则使用global .innodb_flush_log_at_trx_commit（此时session中仍能设置，但无效）

　　3、每个session新建时，以当前的global.innodb_flush_log_at_trx_commit 为默认值。

**业务应用**

这个特性可以用在一些对表的重要性做等级定义的场景。比如同一个实例下，某些表数据有外部数据备份，或允许丢失部分事务的情况，对这些表的更新，可以设置 Session.innodb_flush_log_at_trx_commit为非1值。

在阿里云RDS服务中，我们对数据可靠性和可用性要求更高，将 innodb_use_global_flush_log_at_trx_commit设置为ON，因此修改session.innodb_flush_log_at_trx_commit也没有作用，统一使用 global.innodb_flush_log_at_trx_commit = 1。

{{</hint>}}

{{< hint warning>}}

- O_SYNC: requires that any write operations block until **all data and all metadata** have been written to persistent storage.
- O_DSYNC: like O_SYNC, except that there is no requirement to wait for any metadata changes which are not necessary to read the just-written data. In practice, O_DSYNC means that the application does not need to wait until ancillary information (the file modification time, for example) has been written to disk. Using O_DSYNC instead of O_SYNC can often eliminate the need to flush the file inode on a write.
- O_RSYNC: this flag, which only affects read operations, must be used in combination with either O_SYNC or O_DSYNC. It will cause aread() call to block until the data (and maybe metadata) being read has been flushed to disk (if necessary). This flag thus gives the kernel the option of delaying the flushing of data to disk; any number of writes can happen, but data need not be flushed until the application reads it back.

{{</hint>}}

### innodb_flush_method

[innodb_flush_method](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_flush_method)选项为InnoDB数据文件和日志文件的同步策略：

- fsync：默认选项，数据文件和日志文件都使用deafult打开，都通过fsync确保写入成功
- O_DIRECT：使用O_DIRECT打开数据文件，使用default打开日志文件，都通过fsync来确保写入成功

fsync函数只对由文件描述符filedes指定的单一文件起作用，并且等待写磁盘操作结束，然后返回。

open时的参数O_SYNC有着和fsync类似的语义：使每次write都会阻塞等待硬盘IO完成，因为InnoDB在打开数据文件和日志文件时没有指定O_SYNC，所以需要显式调用fsync已确保元信息和数据刷入磁盘。

### binlog sync

在MySQL Server层还有binlog日志，其用来进行point-in-time（PIT）恢复以及在节点间（主从）进行复制。

MySQL本身通过2PC原子协议保证数据提交时redo log file和binlog file的一致。

MySQL也有参数提供了何时sync（when）：[sync_binlog](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_sync_binlog)

## checkpoint

从上面我们知道，InnoDB存储引擎为了事务的D，采用write ahead log（WAL）。后续数据页（dirty page）的flush大多数情况下是异步进行的，最低水位线由checkpoint负责，即checkpoint保证其LSN点位之前的所有dirty page都必须持久化到磁盘上。这样，在crash recovery时，只需从checkpoint开始进行redo log apply恢复到内存态即可。

关于checkpoint的两种方式（sharp & fuzzy），在buffer pool的page flush章节，已经做过介绍，在此不再赘述。

# MySQL 8.0优化（转）

本节内容转自网络及阿里内核月报。

## WAL流程优化

### **REDO产生**

事务在写入数据的时候会产生REDO，一次原子的操作可能会包含多条REDO记录，这些REDO可能是访问同一Page的不同位置，也可能是访问不同的Page（如Btree节点分裂）。InnoDB有一套完整的机制来保证涉及一次原子操作的多条REDO记录原子，即恢复的时候要么全部重放，要不全部不重放，这部分将在之后介绍恢复逻辑的时候详细介绍，本文只涉及其中最基本的要求，就是这些REDO必须连续。InnoDB中通过min-transaction实现，简称mtr，需要原子操作时，调用mtr_start生成一个mtr，mtr中会维护一个动态增长的m_log，这是一个动态分配的内存空间，将这个原子操作需要写的所有REDO先写到这个m_log中，当原子操作结束后，调用mtr_commit将m_log中的数据拷贝到InnoDB的Log Buffer。

### **写入Log Buffer**

从MySQL 8.0开始，设计了一套无锁的写log机制，其核心思路是允许不同的mtr，同时并发地写Log Buffer的不同位置。不同的mtr会首先调用log_buffer_reserve函数，这个函数里会用自己的REDO长度，原子地对全局偏移[log.sn](http://log.sn/)做fetch_add，得到自己在Log Buffer中独享的空间。之后不同mtr并行的将自己的m_log中的数据拷贝到各自独享的空间内。

~~~~
/* Reserve space in sequence of data bytes: */
const sn_t start_sn = log.sn.fetch_add(len);
~~~~

### **写入Page Cache**

写入到Log Buffer中的REDO数据需要进一步写入操作系统的Page Cache，InnoDB中有单独的log_writer来做这件事情。这里有个问题，由于Log Buffer中的数据是不同mtr并发写入的，这个过程中Log Buffer中是有空洞的，因此log_writer需要感知当前Log Buffer中连续日志的末尾，将连续日志通过pwrite系统调用写入操作系统Page Cache。整个过程中应尽可能不影响后续mtr进行数据拷贝，InnoDB在这里引入一个叫做link_buf的数据结构，如下图所示：

![InnoDB_redo_log_link_buf_1](/InnoDB_redo_log_link_buf_1.png)

link_buf是一个循环使用的数组，对每个lsn取模可以得到其在link_buf上的一个槽位，在这个槽位中记录REDO长度。另外一个线程从开始遍历这个link_buf，通过槽位中的长度可以找到这条REDO的结尾位置，一直遍历到下一位置为0的位置，可以认为之后的REDO有空洞，而之前已经连续，这个位置叫做link_buf的tail。下面看看log_writer和众多mtr是如何利用这个link_buf数据结构的。这里的这个link_buf为log.recent_written，如下图所示：

![InnoDB_redo_log_link_buf_2](/InnoDB_redo_log_link_buf_2.png)

图中上半部分是REDO日志示意图，write_lsn是当前log_writer已经写入到Page Cache中日志末尾，current_lsn是当前已经分配给mtr的的最大lsn位置，而buf_ready_for_write_lsn是当前log_writer找到的Log Buffer中已经连续的日志结尾，从write_lsn到buf_ready_for_write_lsn是下一次log_writer可以连续调用pwrite写入Page Cache的范围，而从buf_ready_for_write_lsn到current_lsn是当前mtr正在并发写Log Buffer的范围。下面的连续方格便是log.recent_written的数据结构，可以看出由于中间的两个全零的空洞导致buf_ready_for_write_lsn无法继续推进，接下来，假如reserve到中间第一个空洞的mtr也完成了写Log Buffer，并更新了log.recent_written*，如下图：

![InnoDB_redo_log_link_buf_3](/InnoDB_redo_log_link_buf_3.png)

这时，log_writer从当前的buf_ready_for_write_lsn向后遍历log.recent_written，发现这段已经连续：

![InnoDB_redo_log_link_buf_4](/InnoDB_redo_log_link_buf_4.png)

因此提升当前的buf_ready_for_write_lsn，并将log.recent_written的tail位置向前滑动，之后的位置清零，供之后循环复用：

![InnoDB_redo_log_link_buf_5](/InnoDB_redo_log_link_buf_5.png)

紧接log_writer将连续的内容刷盘并提升write_lsn。

### 刷盘

log_writer提升write_lsn之后会通知log_flusher线程，log_flusher线程会调用fsync将REDO刷盘，至此完成了REDO完整的写入过程。

### 唤醒用户线程

为了保证数据正确，只有REDO写完后事务才可以commit，因此在REDO写入的过程中，大量的用户线程会block等待，直到自己的最后一条日志结束写入

大量的用户线程调用log_write_up_to等待在自己的lsn位置，为了避免大量无效的唤醒，InnoDB将阻塞的条件变量拆分为多个，log_write_up_to根据自己需要等待的lsn所在的block取模对应到不同的条件变量上去。同时，为了避免大量的唤醒工作影响log_writer或log_flusher线程，InnoDB中引入了两个专门负责唤醒用户的线程：log_wirte_notifier和log_flush_notifier，当超过一个条件变量需要被唤醒时，log_writer和log_flusher会通知这两个线程完成唤醒工作。下图是整个过程的示意图：

![InnoDB_redo_log_log_thread_notify_1](/InnoDB_redo_log_log_thread_notify_1.png)

多个线程通过一些内部数据结构的辅助，完成了高效的从REDO产生，到REDO写盘，再到唤醒用户线程的流程，下面是整个这个过程的时序图：

![InnoDB_redo_log_log_thread_notify_2](/InnoDB_redo_log_log_thread_notify_2.png)

### 推进checkpoint

我们知道REDO的作用是避免只写了内存的数据由于故障丢失，那么打Checkpiont的位置就必须保证之前所有REDO所产生的内存脏页都已经刷盘。最直接的，可以从Buffer Pool中获得当前所有脏页对应的最小REDO LSN：lwm_lsn。 但光有这个还不够，因为有一部分min-transaction的REDO对应的Page还没有来的及加入到Buffer Pool的脏页中去，如果checkpoint打到这些REDO的后边，一旦这时发生故障恢复，这部分数据将丢失，因此还需要知道当前已经加入到Buffer Pool的REDO lsn位置：dpa_lsn。取二者的较小值作为最终checkpoint的位置，其核心逻辑如下：

~~~~
/* LWM lsn for unflushed dirty pages in Buffer Pool */
lsn_t lwm_lsn = buf_pool_get_oldest_modification_lwm();
 
/* Note lsn up to which all dirty pages have already been added into Buffer Pool */
const lsn_t dpa_lsn = log_buffer_dirty_pages_added_up_to_lsn(log);
 
lsn_t checkpoint_lsn = std::min(lwm_lsn, dpa_lsn);
~~~~

为了去掉flush_order_mutex，就需要允许redo log buffer中对应的脏页无序（并发）添加到flush list，即用户写完redo log buffer，就可以把自己的dirty page(s)添加到flush list而无需抢锁。但是这种方法会造成做checkpoint的时候，无法保证flush list最上面的page lsn是最小的，于是引入了log_closer线程定期检查recent_closed（link_buf）是否连续，如果连续就把 recent_closed buffer 向前推进（提升recent_closed的tail，也就是buffer pool脏页的最大LSN（pda_lsn），类似log.recent_written和log_writer），那么checkpoint 的信息也可以往前推进了。需要注意的是，由于这种乱序的存在，lwm_lsn的值并不能简单的获取当前Buffer Pool中的最老的脏页的LSN，保守起见，还需要减掉一个recent_closed的容量大小，也就是最大的乱序范围，简化后的代码如下：

~~~~
/* LWM lsn for unflushed dirty pages in Buffer Pool */
const lsn_t lsn = buf_pool_get_oldest_modification_approx();
const lsn_t lag = log.recent_closed.capacity();
lsn_t lwm_lsn = lsn - lag;
 
/* Note lsn up to which all dirty pages have already been added into Buffer Pool */
const lsn_t dpa_lsn = log_buffer_dirty_pages_added_up_to_lsn(log);
 
lsn_t checkpoint_lsn = std::min(lwm_lsn, dpa_lsn);
~~~~

这里有一个问题，由于lwm_lsn已经减去了recent_closed的capacity，因此理论上这个值一定是小于dpa_lsn的。那么再去比较lwm_lsn和dpa_lsn来获取Checkpoint位置或许是没有意义的。

在buf_pool_get_oldest_modification_lwm() 还是里面, 会将buf_pool_get_oldest_modification_approx() 获得的 lsn 减去recent_closed buffer 的大小, 这样得到的lsn 可以确保是可以打checkpoint 的, 但是这个lsn 不能保证是最大的可以打checkpoint 的lsn. 而且这个 lsn 不一定是指向一个记录的开始, 更多的时候是指向一个记录的中间, 因为这里会强行减去一个 recent_closed buffer 的size. 而以前在5.6 版本是能够保证这个lsn 是默认一个redo log 的record 的开始位置。

基本思想是，脏页链表的有序性可以被部分的打破，也就是说，在一定范围内可以无序，但是整体还是有序的。这个无序程度是受控的。假设脏页链表第一个数据页的oldest_modification为A, 在之前的版本中，这个脏页链表后续的page的oldest_modification都严格大于等于A，也就是不存在一个数据页比第一个数据页还老。在MySQL 8.0中，后续的page的oldest_modification并不是严格大于等于A，可以比A小，但是必须大于等于A-L，这个L可以理解为无序度，是一个定值。那么问题来了，如果脏页链表顺序乱了，那么checkpoint怎么确定，或者说是，奔溃恢复后，从哪个checkpoint_lsn开始扫描日志才能保证数据不丢。官方给出的解法是，checkpoint依然由脏页链表中第一个数据页的oldest_modification的确定，但是奔溃恢复从checkpoint_lsn-L开始扫描(有可能这个值不是一个mtr的边界，因此需要调整)。

![InnoDB_redo_log_thread](/InnoDB_redo_log_thread.png)

参考资料

[MySQL 8.0.11Source Code Documentation: Format of redo log](https://dev.mysql.com/doc/dev/mysql-server/8.0.11/PAGE_INNODB_REDO_LOG_FORMAT.html)

[MySQL 8.0: New Lock free, scalable WAL design](https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design/)

[How InnoDB handles REDO logging](https://www.percona.com/blog/2011/02/03/how-innodb-handles-redo-logging/)

## LinkBuf

在MySQL8.0中增加了一个新的无锁数据结构Link_buf，主要用于redo log buffer以及buffer pool的flush list。

这个数据结构简单来看就是一个拥有固定大小的数组，而对于InnoDB使用来说里面保存的就是写入redo log buffer或者加入到flush list的数据的大小，数组的每个元素可以被原子的更新。

由于在MySQL 8.0中redo log buffer会有空洞，因此linkbuf用来track当前log buffer的写入情况，也就是说每次写入的数据大小都会保存在linkbuf中，而每次写入的位置通过start lsn来得到（hash），假设有空洞（某些lsn还没有写入），那么其对应在linkbuf中的值就是0，这样就可以很简单的track空洞。

最后要注意的是这个数据结构的前提就是LSN是一直增长且不会重复的，因此在InnoDB中只在redo log和flush list（oldest LSN序）中使用。

**源码分析**

无锁ring buffer数据结构

~~~~
template <typename Position = uint64_t>
class Link_buf {
 public:
  typedef Position Distance;          // 累计保存
.....................................
  */** Capacity of the buffer. */*
  size_t m_capacity;                  // linkbuf的大小
 
  */** Pointer to the ring buffer (unaligned). */*
  std::atomic<Distance> *m_links;     // 保存的内容（动态数组，元素是每次写入的长度）
 
  */** Tail pointer in the buffer (expressed in original unit). */*
  alignas(INNOBASE_CACHE_LINE_SIZE) std::atomic<Position> m_tail; // 目前ring buffer的尾部（即第一个空洞的位置，保证之前都是连续的）
};
~~~~

ctor

根据传递进来的capacity,创建对应大小的数组(m_links)，然后初始化数组的内容（每个slot设为0）

~~~~
template <typename Position>
Link_buf<Position>::Link_buf(size_t capacity)
    : m_capacity(capacity), m_tail(0) {
  if (capacity == 0) {
    m_links = nullptr;
    return;
  }
 
  ut_a((capacity & (capacity - 1)) == 0);
 
  m_links = UT_NEW_ARRAY_NOKEY(std::atomic<Distance>, capacity);
 
  for (size_t i = 0; i < capacity; ++i) {
    m_links[i].store(0);
  }
}
~~~~

添加

add_link函数保存这次的写入信息(mlog)：计算写入的起始点+长度（按照长度计算slot，slot = hash(len)）

~~~~
template <typename Position>
inline void Link_buf<Position>::add_link(Position from, Position to) {
  ut_ad(to > from);
  ut_ad(to - from <= std::numeric_limits<Distance>::max());
 
  const auto index = slot_index(from);
 
  auto &slot = m_links[index];
 
  ut_ad(slot.load() == 0);
 
  slot.store(to - from);
}
~~~~

计算slot

计算方式很简单，起始点和数组的大小取模：

~~~~
template <typename Position>
inline size_t Link_buf<Position>::slot_index(Position position) const {
  return position & (m_capacity - 1);
}
~~~~

判断空间

has_space函数就是用来判断对应的position是否已经被占据：插入点是否大于尾部+整个空间大小是否overlap（贪吃蛇）

~~~~
template <typename Position>
inline bool Link_buf<Position>::has_space(Position position) const {
  return tail() + m_capacity > position;
}
~~~~

更新尾部

~~~~
template <typename Position>
template <typename Stop_condition>
bool Link_buf<Position>::advance_tail_until(Stop_condition stop_condition) {
  auto position = m_tail.load();
 
  while (true) {
    Position next;
 
    bool stop = next_position(position, next);
 
    if (stop || stop_condition(position, next)) {
      break;
    }
 
    */* Reclaim the slot. */*
    claim_position(position);
 
    position = next;
  }
 
  if (position > m_tail.load()) {
    m_tail.store(position);
 
    return true;
 
  } else {
    return false;
  }
}
~~~~

读取position位置的slot内容（数据长度），返回下一个位置（next），并判断是否达到空洞（slot未设置，distince == 0）

~~~~
template <typename Position>
bool Link_buf<Position>::next_position(Position *position*, Position &*next*) {
  const auto index = slot_index(position);
 
  auto &slot = m_links[index];
 
  const auto distance = slot.load();
 
  ut_ad(position < std::numeric_limits<Position>::max() - distance);
 
  next = position + distance;
 
  return distance == 0;
}
~~~~

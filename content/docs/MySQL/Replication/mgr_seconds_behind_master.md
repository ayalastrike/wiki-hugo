---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: MGR增加seconds_behind_master
---

# 客观存在的必要性

对于分布式系统，因为shared nothing架构的特点，内部通过RSM进行数据处理，保证了数据的最终一致性，所以需要关注的一个重要指标是各个replica的数据延迟。这不但关系到读取的一致性，也关系到系统的主从切换策略，从而影响可用性。

# 方法论

对于测算，其实本质就是比较，但现实情况是：

1. 快慢的比较采用的是物理时钟，本身就不具备可比较性（时钟跳变/基准时钟不一致）
2. 可比较的另一个必要条件是全序关系，本身也有挑战，比如节点间无法交互，无数据更新

所以可以看出，最完美的方式是：逻辑时钟，即可以保证全序关系（租约/心跳机制，且可驱逐），但会有节点间交互带来的性能挚肘。

# 策略

测算数据延迟的策略：

1. 尽量不影响线上处理速度，只在观测点计算
2. 时间的客观性
3. 计算的准确性

# 分析

第一点：如果记录节点处理的最新点位，即在节点的每次数据处理时更新点位，那么会有2方面的性能问题：

1. 空耗计算能力，常态计算无意义
2. 如果节点为了缩短延迟，可以采用并行回放的方式，那么线程间的应用点位需要原子操作，这可能会给并行增加串行点（改进方法是由coordinator更新应用点位）

所以，在设计之初最直观的想法是：从库的应用时间点位-主库的提交时间点位，可以变为：当前观测时间点位 - 主库的提交时间点位。

第二点，客观性需要考虑：

1. 节点间时间线归约为一，且出生点位一定早于消费点位，如果节点无法交互，无数据更新，则无法归约
2. 应满足单调性，即不考虑时钟跳变（时钟漂移，特指回移）

第三点，准确性的要求：

1. lag语义：出生点和消费点的lag（三级节点和回环结构）
2. 第二点不满足时置0

综上，可以保证lag的计算一定为非负值。

# 落地

主从lag的测算方式：当前时间（show slave status的命令执行时刻，即时计算）- 主库的执行时刻 - 主从库的时间偏移量
即主从lag公式具化为：clock_of_slave - last_master_timestamp - clock_diff_with_master

# 实现

主从lag的计算公式：

~~~~
long time_diff= ((long)(time(0) - mi->rli->last_master_timestamp) - mi->clock_diff_with_master);
protocol->store((longlong)(mi->rli->last_master_timestamp ? max(0L, time_diff) : 0));
 
调用链
mysql_execute_command
    SQLCOM_SHOW_SLAVE_STAT
        show_slave_status_cmd
            show_slave_status
                show_slave_status_send_data
~~~~

## 主从库的时间偏移量

主从库的时间偏移量的测算方式是：从库的IO_THREAD往主库上发出SELECT UNIX_TIMESTAMP()这个SQL语句，并同步获得结果，和自己的当前时间比较，从而比较从库和主库的时间偏移量（Master_info.clock_diff_with_master）。具体实现函数为get_master_version_and_clock。

**get_master_version_and_clock**

~~~~
if (!mysql_real_query(mysql, STRING_WITH_LEN("SELECT UNIX_TIMESTAMP()")) &&
      (master_res= mysql_store_result(mysql)) &&
      (master_row= mysql_fetch_row(master_res)))
  {
    mi->clock_diff_with_master= (long) (time((time_t*) 0) - strtoul(master_row[0], 0, 10));
  }
~~~~

可能出现的错误：

1. IO_thread异常（mi->abort_slave || abort_loop || thd→killed）
2. 网络异常
3. SELECT UNIX_TIMESTAMP()SQL语句在主库上执行异常，seconds_behind_master不可信

刻意设置延迟消费的情况（MASTER_DELAY，sql_delay_event）

计算sql_delay_end和现在的时间差并进行休眠

sql_delay_end = 主库的事务开始时间 + 偏移量 + delay时间 = 从库的事务开始时间

**sql_delay_event**

~~~~
// The time when we should execute the event.
time_t sql_delay_end= ev->common_header->when.tv_sec + rli->mi->clock_diff_with_master + sql_delay;
~~~~

为了保证理论 -> 实际的lag计算一定为非负值，需要考虑以下情况：

- clock_diff_with_master为负
- clock_of_slave - last_master_timestamp一定为0或正，但是该正<clock_diff_with_master正
- 得知主库提交的信息后，才计算并避免负值，否则为0

## last_master_timestamp的赋值

last_master_timestamp在3个地方更新：

1. SQL_thread处理的log_event
2. 如果relay log已读到尾部/GAQ为空，置为0
3. CP时，GAQ中第一个未完成的group

调用点：

~~~~
exec_relay_log_event
    /*
      Even if we don't execute this event, we keep the master timestamp,
      so that seconds behind master shows correct delta (there are events
      that are not replayed, so we keep falling behind).
 
      If it is an artificial event, or a relay log event (IO thread generated
      event) or ev->when is set to 0, or a FD from master, or a heartbeat
      event with server_id '0' then  we don't update the last_master_timestamp.
 
      In case of parallel execution last_master_timestamp is only updated when
      a job is taken out of GAQ. Thus when last_master_timestamp is 0 (which
      indicates that GAQ is empty, all slave workers are waiting for events from
      the Coordinator), we need to initialize it with a timestamp from the first
      event to be executed in parallel.
    */
    if ((!rli->is_parallel_exec() || rli->last_master_timestamp == 0) &&
        !(ev->is_artificial_event() || ev->is_relay_log_event() ||
          ev->get_type_code() == binary_log::FORMAT_DESCRIPTION_EVENT ||
          ev->server_id == 0)) {
      rli->last_master_timestamp =
          ev->common_header->when.tv_sec + (time_t)ev->exec_time;
 
next_event
如果读到EOF，即relay_log已经读到末尾了，置 last_master_timestamp = 0
 
mts_checkpoint_routine() Coordinator做checkpoint时，用ts更新last_master_timestamp，其中ts的计算方式为：
    GAQ == 0，ts = 0
    GAQ != 0，ts  = GAQ中第一个group的ts
 
  /*
    Update the rli->last_master_timestamp for reporting correct Seconds_behind_master.
 
    If GAQ is empty, set it to zero.
    Else, update it with the timestamp of the first job of the Slave_job_queue
    which was assigned in the Log_event::get_slave_worker() function.
  */
  ts= rli->gaq->empty()
    ? 0
    : reinterpret_cast<Slave_job_group*>(rli->gaq->head_queue())->ts;
  rli->reset_notified_checkpoint(cnt, ts, need_data_lock, true);
 
这里，Slave_job_group以事务为单位，也是从库的回放粒度，当处理完一个事务（group，即执行完T-event）后更新Slave_job_group.ts
ptr_group->ts= common_header->when.tv_sec + (time_t) exec_time; // 从库上的回放
~~~~

从这里可以总结出来，提交时间由两个变量计算得出：

- Log_event.exec_time query执行的秒数
- Log_event.Log_event_header.when 主库上事务的开启时间（thd的创建时间）

## Log_event.exec_time

![MySQL_MGR_seconds_behind_master.exec_time](/MySQL_MGR_seconds_behind_master.exec_time.png)

因为只有query和load命令才会产生执行时间这个数据，所以只有这两个log event才会记录exec_time。

这两个log event区分的原因是query是单个动作，而load命令是一个组合动作（放置文件，读取文件，执行），所以需要通过Q_EXEC_TIME_OFFSET/L_EXEC_TIME_OFFSET区分。

代码

~~~~
Log_event
  /* The number of seconds the query took to run on the master. */
  ulong exec_time;
 
Load_event/Create_file_log_event
  /**
   This is the execution time stored in the post header.
   Make sure to use it to set the exec_time variable in class Log_event
  */
  uint32_t load_exec_time;
~~~~

在主库/从库上，通过调用不同的ctor来初始化log_event
Query_log_event/Load_event/Create_file_log_event

主库上，exec_time = 生成binlog的时间 - 线程的开始时间
exec_time= end_time.tv_sec - thd_arg->start_time.tv_sec;

从库上，exec_time = query_exec_time / load_exec_time

## Log_event.Log_event_header.when

![MySQL_MGR_seconds_behind_master.when](/MySQL_MGR_seconds_behind_master.when.png)

从上图可以清晰的看到，when表示的是用户线程的启动时间，即事务的开始时间。

# MGR中的seconds_behind_master

但是，在mgr中gle（Gtid_log_event）在各个节点由applier线程各自生成，那么gle中的when代表的就是applier的产生时间。这会导致last_master_timestamp计算错误，日志如下：

~~~~
2020-09-22T15:41:17.830907+08:00 230 [Note] ---> time = 1600760477 mi->rli->last_master_timestamp = 1600760299 mi->clock_diff_with_master = 0 time_diff = 178 seconds_behind_master = 178
2020-09-22T15:41:17.777634+08:00 9 [Note] ---> reset_notified_checkpoint last_master_timestamp= 0
4247338 2020-09-22T15:41:17.777647+08:00 9 [Note] ---> exec_relay_log_event rli->is_parallel_exec() = 1 rli->last_master_timestamp = 0 ev->is_artificial_event() = 0 ev->is_relay_log_event() = 0 ev->common_header->when.tv_sec = 1600760299 ev->get_type_code() = 33 ev->server_id = 146266501
4247339 2020-09-22T15:41:17.777657+08:00 9 [Note] ---> exec_relay_log_event rli->last_master_timestamp= 1600760299 ev->common_header->when.tv_sec = 1600760299 ev->exec_time = 0
~~~~

这个问题的修复方式是：记录之前一个last_master_timestamp，如果发现gle的取值破坏了单调性，那么就取last_master_timestamp用于计算。

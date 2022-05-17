---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

# 背景

## 高并发场景

在高并发场景（访问共享资源）下，随着用户（并发）的增加，有以下两个挑战：

1. latency在后期成指数型增长![MySQL_ThreadPool_user-latency](/MySQL_ThreadPool_user-latency.png)

2. 吞吐下降严重

   ![MySQL_ThreadPool_user-throughput](/MySQL_ThreadPool_user-throughput.png)

造成这种现象的根本原因是通信、处理和同步成本大幅升高，最终达到处理极限，导致数据库的服务能力大幅下降，甚至无法响应请求。

为了处理这种情况，有两种思路：

1. 在系统达到预设性能指标时，采用管制措施保护数据库，使之可以持续提供稳定的服务。
   这种方式称之为限流（aka “on-ramp metering”），由交通信号灯来控制多少车辆可以在峰值时通过。如下图所示：

   ![MySQL_ThreadPool_Ramp-Metering](/MySQL_ThreadPool_Ramp-Metering.png)

2. 另一种方式是线程池机制，减少内部的线程上下文切换开销、热点资源的竞争，提升CPU上有效代码的执行效率，同时也具备控制流量的能力。

我们最终期望实现的效果是：![MySQL_ThreadPool_user-throughput_expect](/MySQL_ThreadPool_user-throughput_expect.png)

# 技术选型

## MySQL的处理模型

现有的MySQL的请求处理架构采用的是单进程多线程方案，每个用户连接对应一个MySQL处理线程（后面称为工作线程），在这种情况下，随着用户连接数的增多，MySQL进程会创建大量的工作线程，系统性能在高并发场景下会有大幅的下降。

而导致性能下降主要是这三个原因造成的：

- CPU cache miss
- 线程的上下文切换
- 热点共享资源的竞争（latch、锁...）

采用线程池，则可以有针对性的对上述三点进行优化：将MySQL的工作线程和用户连接解绑，同一个工作线程应对多个用户连接上的请求：当前的用户请求处理完成后，工作线程再选择其他的用户请求进行处理。保证合理数量的工作线程提供可持续的处理能力，并且CPU cache的使用更高效，可以有效的降低线程的上下文切换代价，同时热点资源竞争的开销也随之降低，也降低了死锁发生的概率。

线程池机制更加适用于OLTP场景（short CPU-bound queries），对于OLAP场景可能会产生护航（convey）问题。

从某种程度上来看，线程池分离了连接资源（请求）和线程资源（处理）。

## 不适用场景

线程池并不是万能的，对于以下几类场景并不适用：

- 间歇性的workload：这会导致不断的分配/回收线程资源。
- OLTP大查询：大量并发、长时间的复杂查询，如果占满了所有处理线程，后续的请求会进行排队，造成latency的增加。
- 大量的消耗极小的简单查询：比如select 1，大量瞬间涌入的请求，如果进行排队可能也会造成latency的增加。
- 极高并发的prepared statement：prepared statement所使用的MySQL Binary Protocol会使交互往返多次，可能会造成请求的堆积。

## 效果

线程池的性能对比，这里参考官方给出的性能数据：

60x Better Scalability: Read/Write (8192/128)

![MySQL_ThreadPool_Benchmark_RW](/MySQL_ThreadPool_Benchmark_RW.png)

18x Better Scalability: Read Only (8192/512)

![MySQL_ThreadPool_Benchmark_RO](/MySQL_ThreadPool_Benchmark_RO.png)



从这里可以看出，使用线程池后，可以提供稳定的高性能服务，符合我们的预期。

## 线程池演进

从时间线上看，最早提出线程池方案的是MySQL企业版，然后MariaDB随之提供，Percona在port了MariaDB的实现后，增加了一些自己的功能。

MySQL企业版：在MySQL 5.5企业版开始提供thread pool功能。

MariaDB：在MariaDB 5.5开始提供thread pool功能。

Percona：在MySQL 5.5.30和5.6.10-alpha开始提供thread pool功能。

MySQL企业版、Percona、MariaDB提供的都是动态（自适应）线程池。

{{< hint info >}}

选择静态线程池和动态线程池方案主要取决于：

1. 是否有阻塞等耗时操作

2. 是否会存在互相依赖

在大多数场景中，静态线程池都无法满足要求。

{{</hint>}}

对比几家的实现方案，主要有几点不同：

- MariaDB和MySQL企业版在Windows平台实现不同：MySQL企业版使用的是WSAPoll（为了兼容性考虑），但这样也决定了不支持shared memory/named pipe两种连接方式，而MariaDB使用的是原生的Windows线程池。
- MariaDB相对于MySQL企业版在不同平台使用了更加高效I/O复用模型
- Percona在5.5 ~ 5.7版本增加了线程调度优先级，而这是MariaDB本身就有的，二者在线程调度优先级上的细节上不同。

整体功能性对比：

|                                | MySQL企业版 8.0 | MariaDB 5.7 | Percona-Server 5.7 |
| :----------------------------- | :-------------- | :---------- | :----------------- |
|                                | MySQL企业版 8.0 | MariaDB 5.7 | Percona-Server 5.7 |
| 功能提供方式                   | plugin          | builtin     | buitin             |
| 并发调度算法                   | √               | √           | √                  |
| 监听线程                       | √               | √           | √                  |
| 高低优先级队列                 | √               | √           | √                  |
| 限制最大的并发事务数           | √               |             |                    |
| 限制高优先级队列的事务数量     | √               | √           | √                  |
| stall时长                      | √               | √           | √                  |
| 线程数上限                     | √               | √           | √                  |
| 低->高调度                     | √               | √           | √                  |
| idle线程超时机制               |                 | √           | √                  |
| 网络等待优化                   |                 |             | √                  |
| 额外的服务端口（避免影响探活） |                 | √           | √                  |
| 精确的等待时间统计             |                 | √           |                    |

### MySQL企业版 8.0

MySQL企业版通过插件的形式提供线程池功能、并可以通过系统表查看相应的线程池信息。

{{< hint info >}}

MySQL 8.0.14前，线程池信息存储在INFORMATION_SCHEMA中。随着IS的废弃，相应信息移到PSI：

SELECT * FROM performance_schema.tp_thread_state | | tp_thread_group_state | | tp_thread_group_stats;

SELECT * FROM performance_schema.setup_instruments WHERE NAME LIKE '%thread_pool%';

{{</hint>}}

将工作线程划分为group，每个group对应一组用户连接。当建立连接时，thread poo manager通过RR（Round-Robin）的方式将用户连接和group对应起来。随后，每个group的监听线程用于响应用户的query请求，分为两种情况：

1. 当前没有正在执行的其他语句，监听线程直接执行该语句
2. 否则通过队列（高/低优先级）分发给工作线程

如果语句执行时间过长（认定为stall），则创建另一个线程作为监听线程。

初始情况下，每个group创建一个监听线程，和一个后台线程用于监控线程组的状态。

通过[thread_pool_stall_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_stall_limit)来判断执行时间多长的query为stall，这样可以在work load上做出权衡：更快的执行不仅意味着系统访问更迅捷，也意味着死锁发生概率的降低；慢执行可以限制并发执行的数量。在stall时间未到时，同一group的query需要等待之前的query执行完成；当stall时间达到时，则会放行group中的下一个query。

在某些情况下，比如磁盘I/O或者lock，可能导致block从而使整个group不可用，在这方面，设计了回调用于立即创建一个新的线程处理其他语句。

队列划分为高优先级和低优先级：事务的第一个语句进入低优先级队列，事务的后续语句进入高优先级队列；如果是非事务引擎的语句，或者开启了autocommit的语句，则进入低优先级队列。

提供的配置项：

| 配置项                                                       | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [thread_pool_algorithm](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_algorithm) | 并发调度算法                                                 |
| [thread_pool_dedicated_listeners](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_dedicated_listeners) | 是否为每个group开启一个监听线程用于响应用户的网络事件        |
| [thread_pool_high_priority_connection](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_high_priority_connection) | 是否开启高、低优先级队列                                     |
| [thread_pool_max_active_query_threads](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_max_active_query_threads) | 每个group中可以有多少活跃的工作线程                          |
| [thread_pool_max_transactions_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_max_transactions_limit) | 最大的事务数（活跃+非活跃），只能启动时设置                  |
| [thread_pool_max_unused_threads](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_max_unused_threads) | 线程池中空闲的线程数量（区分consumer 1、reserve线程 N-1）    |
| [thread_pool_prio_kickup_timer](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_prio_kickup_timer) | 为了避免饥饿，请求在低优先级队列等待多长时间后移到高优先级队列 |
| [thread_pool_size](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_size) | group大小，一定程度上代表着可以并发执行的语句数量（默认16）  |
| [thread_pool_stall_limit](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_pool_stall_limit) | stall时长                                                    |

### MariaDB 5.7

MariaDB在MySQL企业版推出线程池机制后随之提供了相应功能。但是有一些区别，这个在上面的方案比较中已经谈到。

另外，MariaDB和Percona的线程调度优先级实现的细节上也有些不同：

- 优先级设置不同：MairaDB thread_pool_priority=auto,high, low；Percona thread_pool_high_prio_mode=transactions,statements,none
- Percona有thread_pool_high_prio_tickets
- Maria有thread_pool_prio_kickup_timer

提供的配置项：

| 配置项                         | 说明                                                         |
| :----------------------------- | :----------------------------------------------------------- |
| thread_pool_size               | 线程池大小                                                   |
| thread_pool_max_threads        | 线程池上限                                                   |
| thread_pool_min_threads        | 线程池下限                                                   |
| thread_pool_stall_limit        | stall时长                                                    |
| thread_pool_oversubscribe      | 高优先级的超订配额                                           |
| thread_pool_idle_timeout       | 空闲线程超时时间                                             |
| extra_port                     | 旁路端口                                                     |
| extra_max_connections          | 旁路的最大连接数                                             |
| thread_pool_dedicated_listener | 是否开启监听线程用于响应用户的网络事件                       |
| thread_pool_exact_stats        | 精确等待时间统计                                             |
| thread_pool_prio_kickup_timer  | 为了避免饥饿，请求在低优先级队列等待多长时间后移到高优先级队列 |
| thread_pool_priority           | 高优先级调度模式                                             |

### Percona-Server 5.7

提供了以下几点优化：

- 连接调度的优先级机制：通过thread_pool_high_prio_tickets来决定进入高优先级队列的connection优先级，更加高效的调度连接。
- 低优先级队列限流：当高优先级超订（thread_pool_oversubscribe）时，根据thread_pool_max_threads对低优先级限流，即不创建新的工作线程。
- 长时间的网络等待：对会出现长时间网络I/O等待（socket reads/writes）的场景（大结果集、BLOB、慢客户端），处理下一个query或者创建一个新的线程专门处理。

提供的配置项：

| 配置项                        | 说明                     |
| :---------------------------- | :----------------------- |
| 配置项                        | 说明                     |
| thread_pool_idle_timeout      | 空闲线程超时时间         |
| thread_pool_high_prio_mode    | 高优先级调度模式         |
| thread_pool_high_prio_tickets | 进入高优先级队列的优先级 |
| thread_pool_max_threads       | 线程池上限               |
| thread_pool_oversubscribe     | 高优先级的超订配额       |
| thread_pool_size              | 线程池大小               |
| thread_pool_stall_limit       | stall时长                |
| extra_port                    | 旁路端口                 |
| extra_max_connections         | 旁路的最大连接数         |

# High Level Design

为了实现连接和处理的解耦，线程池技术的处理模型为：连接被抽象为THD，由工作线程（pthread）在处理时通过attach/detach进行挂载/卸载，pthread通过OS调度。同样，pthread和对应的内核任务（task_env）也是一个m:n的模型，这样，连接、线程、内核态任务都实行了m:n的调度。

![MySQL_ThreadPool_Process_Model](/MySQL_ThreadPool_Process_Model.png)

线程池的高效需要做到以下几点：

- 线程组数量和CPU核数对应，尽量保证每个CPU core上只有一个pthread运行
- THD需要被均匀的分配到pthread上
- pthread可以动态的根据workload弹性扩缩容
- THD可以通过快速通道，优先被pthread处理

线程池本质上就是一个三层的pub/sub模型，第一层由conn_handler将监听接受到的连接（channel_info）分配给某个线程组（放入线程组的队列中），第二层由工作线程获取队列中的连接，或者监听epoll事件。

![MySQL_ThreadPool_High_Level_Architecture](/MySQL_ThreadPool_High_Level_Architecture.png)

从这里可以看到，工作线程一共要处理新、旧两种情况：新是指会话的第1次处理，旧是指会话的第n次处理。同时为了效率，还需要设立快速通道，可以优先处理某些连接上的请求（会话的第n次处理）。处理策略如下：

1. 分为普通队列和高优先级队列，优先处理高优先级队列（快速通道）
2. 从队列中取出新建链的连接或者epoll事件，即会话的第1次处理和后续请求（会话的第n次处理）

{{< hint warning>}}

以下如果没有特别指出，则队列指的是普通队列（thread_group_t.queue），高优先级队列会指明为高优先级队列（thread_group_t.high_prio_queue）。

{{</hint>}}
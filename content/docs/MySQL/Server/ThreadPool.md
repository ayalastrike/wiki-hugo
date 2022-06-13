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

# Low Level Design

根据high level deisgn，详细拆解各个模块。

## 模块分类

根据High Level Design，我们将线程池实现分为以下5个模块：

- 线程池资源的创建和销毁：负责线程池的初始化和销毁
- 连接处理：负责TCP建链后的请求分发，由原来的新起一个线程来1:1对应connection变为分发给线程池内的某个group，由group内的某个pthread处理。
- THD和pthread对接
- IO multiplexing
- pthread调度
- wait感知和处理，并增加对socket I/O wait的感知
- 定时器：对stall的THD进行处理。

线程池模块的文件层次设计如下：

![MySQL_ThreadPool_file_layout](/MySQL_ThreadPool_file_layout.png)

## 线程池初始化和销毁

在我们讨论线程池初始化和销毁之前，先通过下面这张图总览一下MySQL的连接处理框架：

![MySQL_ThreadPool_connection_process](/MySQL_ThreadPool_connection_process.png)

### tp_conn_handler

在MySQL启动过程中，首先需要创建并初始化Connection_handler_manager，并通过thread_handling初始化具体的connection_handler。然后通过Connection_acceptor创建Mysqld_socket_listener，监听socket并负责建链。

因此，这里引入tp_conn_handler负责处理连接到连接队列的转换，以及初始化（tp_init）、销毁（tp_end）线程池。

tp_conn_handler的工作包括：

1. 增加conn_handler：Thread_pool_connection_handler，以下简称tp_conn_handler
2. 在配置handler=SCHEDULER_THREAD_POOL时，new tp_conn_handler，初始化线程池
3. 在mysqld关闭（conn_mgr销毁时），销毁线程池
4. tp_conn_handler进行具体的连接处理（Thread_pool_connection_handler::add_connection）

Thread_pool_connection_handler

| 函数            | 说明                                                     |
| :-------------- | :------------------------------------------------------- |
| ctor/dtor       | 线程池初始化/关闭，启动/停止定时器（pool_timer）         |
| add_connection  | 处理连接（后详述）                                       |
| get_max_threads | 返回线程池可以创建的最大线程数（threadpool_max_threads） |

### 初始化线程池

初始化线程池（tp_init）：

1. 初始化thread_group数组（all_groups：128），thread_group组数组在整个线程池生命周期内不再变化
2. 根据pool_size设置线程池，初始化线程组，并创建每个组的epoll_fd
3. 启动定时器

{{< hint info>}}

此时并没有创建工作线程（worker pthread，细分为listener & worker），只是完成资源的初始化，工作线程是随着连接的到来逐步创建的。

{{</hint>}}

### 关闭线程池

关闭线程（tp_end，与tp_init逆序）：

- 停止定时器
- 依次关闭线程组

在关闭线程组时：

1. 如果当前活跃线程组个数为0，直接close(fd)
2. 通知listener和所有的工作线程
   创建关闭管道（shutdown_pipe），并加入epoll，然后通过管道通知listener
   通过信号通知工作线程

**pseudo code**

~~~~
PSI
mutex:
    thread_group_mutex
    timer_mutex
cond:
    worker_thread_cond
    timer_cond
thread:
    worker_thread
    timer_thread
 
tp_init
    threadpool_started = true
    for loop 0..MAX
        thread_group_init(i)
    set_tp_group_size
    timer start // TODO
    PSI reg
 
set_tp_group_size
    for 0 ... size
        epoll_create
 
tp_end
    timer stop  // TODO
    for loop MAX
        thread_group_close(i)
    threadpool_started = false
 
thread_group_init
    mutex init
    pthread_attr set
    pollfd = -1
    shutdown_pipe[0][1] = -1
 
thread_group_close
    lock
    thread_group->shtudown = true
    // listener = NULL              // TODO
    open shutdown_pipe and associate with epollfd
    write(shutdown_pipe, 1)
    wake all thread，由pthread自己处理关闭（最后一个destory thread_group）
    unlock
 
thread_group_destroy（由group内最后一个pthread调用）
    mutex dtor
    pollfd close
    shutdown_pipe[0][1] close
~~~~

## 连接处理

### TCP建链&连接处理

在客户端连接到达时，Mysqld_socket_listener::listen_for_connection_event负责TCP建链，并在建链成功后创建Channel_Info，交给Connection_handler_manager负责整体的连接控制，然后由connection_handler处理连接，此时交由tp_conn_handler处理。

tp_conn_handler的连接处理：

1. 根据channel_info创建THD（channel_info->create_thd()）
2. 创建连接对象（connection_t），并关联THD
3. set THD.thread_id & create time
4. 将THD分配给某个线程组
5. 压入线程组普通队列

THD分配线程组的策略是取模（thd.thread_id % group_size），因为THD的thread_id是单调递增的，所以可以做到线程组间连接的均匀分配。

{{< hint info>}}

从资源的角度考虑，在途事务会持有（锁）资源，所以需要优先将进行中的事务尽快处理完。因此，新建的连接会被放入普通队列，连接的后续请求可以进入高优队列。

{{</hint>}}

**pseudo code**

~~~~
连接队列    LILO（考虑公平性）
typedef I_P_List<connection_t,
                 I_P_List_adapter<connection_t,
                                  &connection_t::next_in_queue,
                                  &connection_t::prev_in_queue>,
                 I_P_List_null_counter,
                 I_P_List_fast_push_back<connection_t> >
connection_queue_t;
 
tp_conn_handler::add_connection
    create thd
    alloc connection
    delete channel_info
    set thd.thread_id & create_time
    thd_mgr->add_thd
    分组（mod）
    group->connection_count++
    enqueue(group, connection)
 
alloc_connection
    malloc connection_t
    set default value
 
enqueue(group, connection)
    lock
        push_back(group->queue)
        if all sleep(active == 0), wake_or_create_thread
    unlock
 
change group
    group->connection_count ++--
~~~~

## thread和THD对接

在线程池方案中，因为对THD和pthread做了解绑，即当事件发生时，再将THD attach到pthread上。

所以需要做以下几类事情：

- pthread的初始化
- thd的创建和初始化
- thd attach
- 处理请求
- 销毁thd
- 销毁pthred

### pthread的初始化

在线程池中，pthread按照按需的方式进行创建。在创建pthread（create_thread）后立即进行初始化（my_thread_init）

### thd的创建和初始化

thd的创建在tp_conn_handler::add_connection时进行（create_thd）

thd的初始化也在add_connection时进行：

- thd→set_new_id
- thd→set_start_time & thd→set_create_time
- thd_mgr->add_thd

### thd attach

thd attach到pthread上在epoll event发生时（listen/wake），在处理connection phase/command phase时进行attach。

attach时：

- set_stack(&thd)
- thd→store_globals()
- my_thread_set_id()
- my_socket_set_thread_owner()

### 处理请求

MySQL中按照协议，请求的处理分为两个阶段：

- connection phase：进行handshake
- command phase：处理客户端命令

所以这里需要进行THD到pthread的切换：

1. connection phase
   thd attach
   login
2. command phase
   thd attach
   do_command

### 销毁thd

提供一个wrapper将thd和connection的相关资源销毁包在一起：

- end_connection
- close_connection
- thd→release_resources()
- thd_mgr→remove_thd()
- conn--
- delete thd

### 销毁pthread

销毁pthread比较简单：

- my_thread_end()

**pseudo code**

~~~~
工作队列    LIFO（考虑cache亲和性）
typedef I_P_List<worker_thread_t,
                 I_P_List_adapter<worker_thread_t,
                                  &worker_thread_t::next_in_list,
                                  &worker_thread_t::prev_in_list> >
worker_list_t;
 
wake_or_create_thread
    wake_thread
    create_thread
 
wake_thread
    pop front from waiting_list（FILO）
    thread->woken = true
    notify
 
create_thread
    MAX limit
        return
    pthread_create(worker_main)
 
worker_main
    worker_thread_t this_thread
    set this_thread->group
    init cond
    wakeup = false
    for(;;)
        connection_t *conn
        conn = get_event(&this_thread, group)
        if (!conn)
            exit
        handle_event(conn)
    last thread in group, destroy group // 由线程组的最后一个线程负责销毁线程组资源，因为需要thread_group->shutdown_pipe通知listener
    my_thread_end
 
handle_event(conn)
    handle_connection_phase
    handle_command_phase
    io_handle
end:
    connection_abort
 
handle_connection_phase
    thread_attach
    thd_prepare_connection
 
handle_command_phase
    thread_attach
    for(;;)
        do_command
 
thread_attach
    set_stack
    thd->store_globals
    my_thread_set_id
    my_socket_set_thread_owner
 
io_handle
    if !connection->not_bound_to_epollfd
        io_poll_associate_fd()
    io_poll_read
 
connection_abort
    tp_remove_connection
    group->connection_count--
    free connection
 
tp_remove_connection
    thread_attach
    end_connection
    close_connection
    thd->release_resources
    remove_thd
    conn--
    delete thd
~~~~

草图如下：

![MySQL_ThreadPool_thd-pthread](/MySQL_ThreadPool_thd-pthread.jpeg)

## IO multiplexing

在原来的conn_handler中，每个pthread都会通过select等待各自的connection发送网络请求。

在线程池中，每个组有一个epoll fd，在add_connection时，对connection进行group的分配，并将socket fd关联到epoll fd上。然后在epoll_wait发生事件后，再通过vio→read的方式读取请求。这样，综合采用了epoll+select。

IO multiplexing提供如下方法（支持多平台）：

~~~~
                                            implementation
创建poll io fd    io_poll_create             epoll/kqueue/Solaris
绑定epoll fd      io_poll_associate_fd       epoll_ctl(pollfd, EPOLL_CTL_ADD)
解绑epoll fd      io_poll_disassociate_fd    epoll_ctl(pollfd, EPOLL_CTL_DEL)
设置读状态         io_poll_read               epoll_ctl(pollfd, EPOLL_CTL_MOD, fd, EPOLLIN)
等待事件           io_poll_wait               epoll_wait
~~~~

## pthread调度

在pthread的调度上，总体原则如下：

- 尽量保证每个CPU core上只有一个pthread运行，减少线程上下文切换的开销
- pthread可以动态的根据workload弹性扩缩容
- pthread等待队列采用FILO，增加CPU的亲和性

### pthread状态

pthread的状态机设计如下：

![MySQL_ThreadPool_Thread_State_Machine](/MySQL_ThreadPool_Thread_State_Machine.png)

**pseudo code**

~~~~
add_connection
    enqueue
 
enqueue
    push_back(group->queue)
    if all sleep(active == 0), create_or_create_thread
 
get_event   // 处理请求
    lock
    for(;;)
        if (thread_group->shutdown) break;                  // 关闭退出
        pop(group->queue) break;                            // connection拿完就走
        if !listener, listen()                              // 没人监听，坐在收发室
        thread->woken = false, enqueue(group->waiting_list) // idle，休眠
        cond wait ... until notify/timeout
    unlock
 
pthread notify（wake_thread）
    enqueue, if all sleep, notify
    listener收到消息，处理不过来，notify
    关闭线程池，notify all
pthread timeout
    tp_idle_timeout, pthread exit
    tp_stall_timeout, kill THD
 
listen    // 监听
    for(;;)
        cnt = epoll_wait(-1)
        lock
        cnt < 0 break;
        if (thread_group->shutdown) break;  // 关闭退出
        group->io_event_count += cnt
        // do it myself or others
        others: enqueue(connection)
        do it myself, then get connection and leave, handle_event()
        unlock
~~~~

从上面可以看到，同一个group中listener的event dispatch和get_event是互斥的，即同一时刻要么在派发事件，要么在获取事件。

同一组内的线程在同一时刻可能处于以下几种状态：

- listen
- idle
- killed
- active
- stall

状态细节

~~~~
listen
    thread_group->listener
 
idle
    worker_thread->woken
 
killed
    thd->killed = THD::KILL_CONNECTION
 
active
    pthread create      +1
    pthread exit        -1
    listen              --
    listener get event  ++
    sleep               --
    wake                ++
    thd wait begin      --
    thd wait end        ++
 
waiting
    thd wait begin      ++
    thd wait end        --
 
stall
~~~~

### 处理epoll events

在处理网络事件（epoll events）时，策略的选择问题

1. 由listener自己处理事件，还是由listener唤醒等待队列中的idle工作线程来处理事件

   从时间片和上下文切换上来看，listener自己处理更加高效：时间片轮转到了listener，不用浪费不说，再把事件交给其他人处理引入额外的消息传递和上下文切换的代价，一定程度上需要避免唤醒，减少代价。

   但是另一方面，listener离开处理事件，势必会有一个时间窗口无法监听网络事件，其他worker可以在这个时间窗口内自取一个事件，以提高效率，即利用率分到的时间片，也没有引入listener的唤醒代价。

   所以综合考虑，如果不繁忙（这里指的是网络事件很多，建链/请求），则由listener亲自处理事件，否则唤醒工作线程处理事件

2. 如果繁忙（queue不为空），需要唤醒多少个工作线程
   在理想情况下（可以快速处理完，不出现wait），只需要一个active工作线程即可，即"one active thread per group"。

综上所述，worker_thread的工作流程如下：

![MySQL_ThreadPool_Worker_Schedule](/MySQL_ThreadPool_Worker_Schedule.png)

### pthread allocation throttle

这里的allocate特指pthread的数量控制，特指工作线程的创建限制，策略包括：

1. 整体：数量上限（threadpool_max_threads）

2. 局部：每个线程组

   每个线程组不能超过connection数量

   如果当前没有活跃的（active = 0），立即创建

   其他情况下，pthread的创建要受到时间间隔的约束：

   0 ~ 3，立即创建

   4 ~ 7，50ms

   8 ~ 15，100ms

   \> 16 ，200ms

## 请求调度

在connection的请求调度上，要尽量兼顾公平和效率。

公平性：

- THD需要被均匀的分配到pthread上
- 造成stall的请求，其connection可以被主动kill掉

高效率：

- 尽可能快的相应connection的请求
- connection上的请求可以通过快速通道，优先被pthread处理

{{< hint info>}}

pthread在处理完connection的一次请求之前，不能处理其他connection的请求。

{{</hint>}}

### 过载 overload

这里的overload可以细分为两种情况：

- too many：未达到CPU bound，但active工作线程数量过大，且没有出现stall，这时工作线程不处理数据，直接进入waiting_threads，等待idle timeout关闭pthread
- too busy：即达到CPU bound或者出现stall（lock/sync/net），此时active+waiting过大，这时优先处理高优先级队列。并对高优队列进行配额管理；同时对stall timeout的进行关闭connection。

### 高优先级调度

可以进入高优队列的：

- 高优先级语句
- 高优先级事务&配额范围内&活跃事务&持有lock

在worker处理队列时，首先从高优队列中取请求，然后再从普通队列中取。

## idle超时处理

idle超时后，需要通过VIO模块关闭用户的connection。

在线程池实现中，进行以下调整：

1. 增加vio_cancel，用于thread pool定时器检测超时和sql超时时调用
2. shutdown（传入how=SHUT_RDWR）转而调用vio_cancel，进行全关闭
3. 实现vio_cancel，实现protocol+how
4. 对socket io read/write等待提供回调钩子

### VIO implemenation

include/violite.h

- 增加vio_cancel
- shutdown(+how)
- 提供毁掉钩子注册vio_set_wait_callback
- 增加vio->thread_id，用于vio_shutdown（Windows only），+vio_set_thread_id()

vio/vio_priv.h

- 修改vio_ssl_shutdown(传入how)，内部调用shutdown(传入how)

vio/viosocket.c

- 修改vio_shutdown
  将shutdown(fd, SHUT_RDWR)修改为vio_cancel（+），即封装shutdown系统调用（可以传入how控制关闭行为）
- 注册socket io回调

vio/vio.c

- 注册vio->viocancel
- 原来的shutdown显式传入SHUT_RDWR

vio/vioshm.c vio/viopipe.c

- 实现named pipe和shared memory protocol下的viocancel
- 适配shutdown(how)

sql/protocol_classic.cc

- Protocol_classic::shutdown调用vio→shutdown(SHUT_RDWR)

rapid/plugin/x/ngs/ngs_common/connection_vio.cc

- 适配vio→shutdown(SHUT_RDWR)

{{< hint warning>}}

TCP的"四次握手"关闭. TCP是全双工的信道, 可以看作两条单工信道, TCP连接两端的两个端点各负责一条. 当对端调用close时, 虽然本意是关闭整个两条信道, 但本端只是收到FIN包. 按照TCP协议的语义, 表示对端只是关闭了其所负责的那一条单工信道, 仍然可以继续接收数据. 也就是说, 因为TCP协议的限制, 一个端点无法获知对端的socket是调用了close还是shutdown.

The shutdown() call causes all or part of a full-duplex connection on the socket associated with sockfd to be shut down.

- SHUT_RD, further receptions will be disallowed.
- SHUT_WR, further transmissions will be disallowed.
- SHUT_RDWR, further receptions and transmissions will be disallowed.

https://man7.org/linux/man-pages/man2/shutdown.2.html

{{</hint>}}

## stall感知

线程池方案中，由于用户连接和线程的m:n模型，以及pthread在没有处理完当前用户请求之前不能处理其他连接上的请求这两个背景，需要感知当前处理中的线程是否有等待发生，以便线程池对等待进行介入，进行pthread的调度。

实际上，MySQL本身已经设计了等待回调的机制。我们先来了解现有的等待回调机制，然后结合线程池的实现，进行对接和调度控制。

### 相关知识

#### THD

THD可以看做运行时对象，对于MySQL来说，一个运行单元（通常为线程），在内存中通过THD对象来描述。

#### thr_lock

MySQL中的运行单元THD对于操作系统来说是通过Posix pthread作为载体的。MySQL将pthread的read lock和write lock封装为thr_lock，并提供了一个函数回调用于在phread lock wait前后上报。

回调函数：

~~~~
static void (*before_lock_wait)(void)= 0;
static void (*after_lock_wait)(void)= 0;
~~~~

注册回调：

~~~~
void thr_set_lock_wait_callback(void (*before_wait)(void),
                                void (*after_wait)(void))
{
  before_lock_wait= before_wait;
  after_lock_wait= after_wait;
}
~~~~

触发回调：

~~~~
thr_multi_lock
  thr_lock
    wait_for_lock
      before_lock_wait
      enter_cond_hook...
      after_lock_wait
~~~~

MySQL Server层的lock是通过thr_lock实现的，统一通过thr_multi_lock来进行检测来避免死锁。

### 现有的等待回调设计

#### 等待

在DBMS的运行中，可能会出现各种stall/sleep，都会产生等待。

在线程池场景中，需要在当前THD出现等待时，通过上报等待机制，感知到有等待发生，从而可以选择是否启动新的pthread，还是唤醒现有的pthread，以便让CPU可以执行其他任务。在这里，需要考虑保证新调度上CPU的线程预期收益必须高于其付出的成本（CPU cache miss），所以，应该只上报中长时间的等待，以便使收益高于成本。

按照等待时长可分为：

- short wait：   比如mutex
- medium wait：比如disk io
- large wait：   比如row/table lock

诸如mutex之类的short wait，就不应该（无需）上报。而诸如row lock、table lock、global read lock、mdl lock，可能会等待毫秒甚至秒级别，就需要进行上报。

在MySQL当前的处理模型中，在thread pool模式下，当线程即将进入stall/sleep时，threadpool调度器应该选择创建线程、或者将其他线程唤醒，以调度的方式让CPU执行多任务，以保证效率、提高性能。

对于中长等待，MySQL封装了以下的等待事件类型（wait_type）：

~~~~
THD_WAIT_SLEEP= 1,
THD_WAIT_DISKIO= 2,
THD_WAIT_ROW_LOCK= 3,
THD_WAIT_GLOBAL_LOCK= 4,        // 未使用
THD_WAIT_META_DATA_LOCK= 5,
THD_WAIT_TABLE_LOCK= 6,
THD_WAIT_USER_LOCK= 7,
THD_WAIT_BINLOG= 8,
THD_WAIT_GROUP_COMMIT= 9,       // 未使用
THD_WAIT_SYNC= 10,
THD_WAIT_LAST= 11
~~~~

#### 等待开始和结束的接口

提供以下接口用于声明等待开始和等待结束：

- thd_wait_begin(thd, wait_type);
- thd_wait_end(thd);

service_thd_wait.h只负责声明函数（接口），即挂钩。具体的实现由各个模块负责实现，即上钩后的动作。

挂钩的地方（#include <mysql/service_thd_wait.h>）

| 等待类型                | 使用模块                                 | 开始等待                                                     | 结束等待                               | 说明                                                         |
| :---------------------- | :--------------------------------------- | :----------------------------------------------------------- | :------------------------------------- | :----------------------------------------------------------- |
| THD_WAIT_BINLOG         | sql/rpl/                                 | thd_wait_begin(thd, THD_WAIT_BINLOG);                        | thd_wait_end(thd);                     | 等待SQL Thread执行到等待的点位(pos/gtid)                     |
| THD_WAIT_DISKIO         | innobase/buf/buf0fluinnobase/buf/buf0rea | thd_wait_begin(NULL, THD_WAIT_DISKIO);thd_wait_begin(NULL, THD_WAIT_DISKIO); | thd_wait_end(NULL);thd_wait_end(NULL); | InnoDB flush pageInnoDB read page                            |
| THD_WAIT_META_DATA_LOCK | sql/mdl/                                 | thd_wait_begin(NULL, THD_WAIT_META_DATA_LOCK);               | thd_wait_end(NULL);                    | mdl锁等待                                                    |
| THD_WAIT_ROW_LOCK       | innobase/lock/lock0wait                  | thd_wait_begin(trx->mysql_thd, THD_WAIT_ROW_LOCK);           | thd_wait_end(trx->mysql_thd);          | InnoDB等待行锁                                               |
| THD_WAIT_TABLE_LOCK     | innobase/lock/lock0wait                  | thd_wait_begin(trx->mysql_thd, THD_WAIT_TABLE_LOCK);         | thd_wait_end(trx->mysql_thd);          | InnoDB等待表锁                                               |
| THD_WAIT_USER_LOCK      | innobase/srv/srv0conc                    | thd_wait_begin(trx->mysql_thd, THD_WAIT_USER_LOCK);          | thd_wait_end(trx->mysql_thd);          | server层进入InnoDB时控制并发(thread_currency)，InnoDB handler虚函数调用 |

对于conn_mgr (connection_handler_manager.cc)，通过THD_event_functions封装了回调，除了之前定义的开始等待、结束等待之外，并提供了一个额外的扩展接口（post_kill_notification）：

~~~~
struct THD_event_functions
{
  void (*thd_wait_begin)(THD* thd, int wait_type);
  void (*thd_wait_end)(THD* thd);
  void (*post_kill_notification)(THD* thd);
};
~~~~

conn_mgr在init时注册了pthread lock的等待回调：

**Connection_handler_manager::init()**

~~~~
// Init common callback functions.
thr_set_lock_wait_callback(scheduler_wait_lock_begin,
                           scheduler_wait_lock_end);
thr_set_sync_wait_callback(scheduler_wait_sync_begin,
                           scheduler_wait_sync_end);
~~~~

#### 接口实现

在sql_class.cc中，除了实现以上2个接口外，还实现了行锁等待下的死锁检测（thd_report_row_lock_wait）。

~~~~
#ifndef EMBEDDED_LIBRARY
extern "C" void thd_wait_begin(MYSQL_THD thd, int wait_type)
{
  MYSQL_CALLBACK(Connection_handler_manager::event_functions,
                 thd_wait_begin, (thd, wait_type));
}
 
/**
  Interface for MySQL Server, plugins and storage engines to report
  when they waking up from a sleep/stall.
 
  @param  thd   Thread handle
*/
extern "C" void thd_wait_end(MYSQL_THD thd)
{
  MYSQL_CALLBACK(Connection_handler_manager::event_functions,
                 thd_wait_end, (thd));
}
 
void thd_report_row_lock_wait(THD* self, THD *wait_for)
{
  DBUG_ENTER("thd_report_row_lock_wait");
 
  if (self != NULL && wait_for != NULL &&
      is_mts_worker(self) && is_mts_worker(wait_for))
    commit_order_manager_check_deadlock(self, wait_for);
 
  DBUG_VOID_RETURN;
}
 
#else
no-op // 空实现 thd_wait_begin / thd_wait_end / thd_report_row_lock_wait
~~~~

我们从上面可以看到，MYSQL_CALLBACK宏定义了等待发生时的回调方法，即当等待发生时调用OBJ.FUNC(PARAMS)

~~~~
#define MYSQL_CALLBACK(OBJ, FUNC, PARAMS)         \
  do {                                            \
    if ((OBJ) && ((OBJ)->FUNC))                   \
      (OBJ)->FUNC PARAMS;                         \
  } while (0)
~~~~

挂钩之处：

| 代码                                   |                                                              | 说明                                    |
| :------------------------------------- | :----------------------------------------------------------- | :-------------------------------------- |
| sql_class                              | MYSQL_CALLBACK(Connection_handler_manager::event_functions, thd_wait_begin, (thd, wait_type)); | thd_wait_begin挂钩的地方                |
| Set_kill_conn::virtual void operator() | MYSQL_CALLBACK(Connection_handler_manager::event_functions, post_kill_notification, (killing_thd)); | kill连接                                |
| THD::awake                             | MYSQL_CALLBACK(Connection_handler_manager::event_functions, post_kill_notification, (this)); | THD被theadpool调度器唤醒                |
| Connection_handler_manager::init()     | MYSQL_CALLBACK(Connection_handler_manager::event_functions, thd_wait_begin, (current_thd, THD_WAIT_TABLE_LOCK));MYSQL_CALLBACK(Connection_handler_manager::event_functions, thd_wait_begin, (current_thd, THD_WAIT_SYNC)); | 处理HD_WAIT_TABLE_LOCK处理THD_WAIT_SYNC |

但是，Connection_handler_manager::event_functions没有注册任何函数用于对应实现，可以认为官方只是设置好了钩子，只待thread pool实现之。

### 线程池回调对接和实现

#### 定义上钩实现

~~~~
THD_event_functions tp_event_functions=
{
  tp_wait_begin, tp_wait_end, tp_post_kill_notification
};
~~~~

#### THD适配

THD

**sql/sql_class.h**

~~~~
THD_event_functions *scheduler; // 增加 回调函数集合
thd_scheduler event_scheduler;  // 改名 THD调度回来需要的数据，connection
~~~~

THD ctor

**sql/sql_class.cc**

~~~~
scheduler= NULL;
event_scheduler.data= 0;
~~~~

THD getter/settter // 改名

**sql/sql_class.cc**

~~~~
void *thd_get_scheduler_data(THD *thd)
{
  return thd->event_scheduler.data;
}
 
void thd_set_scheduler_data(THD *thd, void *data)
{
  thd->event_scheduler.data= data;
}
~~~~

THD members赋值（tp_conn_handler::add_connection）

**Thread_pool_connection_handler::add_connection()**

~~~~
thd->scheduler= &tp_event_functions;
 
thd->event_scheduler.data= connection;
~~~~

#### 挂钩

增加了一种等待类型：THD_WAIT_NET，用于描述VIO socket的读写等待，并挂钩。

| 等待类型     | 使用模块      | 开始等待          | 结束等待        | 说明                 |
| :----------- | :------------ | :---------------- | :-------------- | :------------------- |
| THD_WAIT_NET | vio/viosocket | START_SOCKET_WAIT | END_SOCKET_WAIT | VIO socket的读写等待 |

修改现有挂钩之处：

| 代码                                   |                                                              | 说明                                                    |
| :------------------------------------- | :----------------------------------------------------------- | :------------------------------------------------------ |
| sql_class                              | MYSQL_CALLBACK(thd->scheduler, thd_wait_begin, (thd, wait_type)); | thd_wait_begin挂钩的地方                                |
| Set_kill_conn::virtual void operator() | MYSQL_CALLBACK(killing_thd->scheduler, post_kill_notification, (killing_thd)); | kill连接                                                |
| THD::awake                             | MYSQL_CALLBACK(this->scheduler, post_kill_notification, (this)); | THD被theadpool调度器唤醒                                |
| Connection_handler_manager::init()     | MYSQL_CALLBACK(thd->scheduler, thd_wait_begin, (current_thd, THD_WAIT_TABLE_LOCK));MYSQL_CALLBACK(thd->scheduler, thd_wait_begin, (current_thd, THD_WAIT_SYNC));MYSQL_CALLBACK(thd->scheduler, thd_wait_begin, (thd, THD_WAIT_NET)); | 处理HD_WAIT_TABLE_LOCK处理THD_WAIT_SYNC增加THD_WAIT_NET |

实现：

~~~~
void tp_wait_begin(THD *thd, int type) {
  connection_t *connection = (connection_t *)thd->event_scheduler.data;
  if (connection)
  {
     wait_begin(...);
  }
}
 
void tp_wait_end(THD *thd) {
  connection_t *connection = (connection_t *)thd->event_scheduler.data;
  {
     wait_end(...);
  }
}
 
void tp_post_kill_notification(THD *thd)
{
  Vio* vio= thd->get_protocol_classic()->get_vio();
  if (vio)
    vio_cancel(vio, SHUT_RD);
  DBUG_VOID_RETURN;
}
 
Thd_timeout_checker
virtual void operator() (THD* thd) {
  connection_t *connection= (connection_t *)thd->event_scheduler.data;
}
~~~~

整体的THD事件回调如下图所示：

![MySQL_ThreadPool_thd_event_callback](/MySQL_ThreadPool_thd_event_callback.png)

#### ThreadPool callback implementations

service_thd_wait.h
services.h.pp

- 增加一种等待类型：THD_WAIT_NET

sql/sql_class

- set_active_vio设置thread_id（用于windows）
- THD增加回调函数集合指针，回调数据改名。declear+ctor+get/set
- CALLBACK调整

**vio_set_thread_id调用链**

~~~~
login_connection
  check_connection
    set_active_vio
      vio_set_thread_id
~~~~

conn_mgr

- CALLBACK调整
- THD_WAIT_NET注册接口（+）

threadpool.h

- extern void tp_wait_begin(THD *, int);
- extern void tp_wait_end(THD*);
- extern void tp_post_kill_notification(THD *thd);
- extern THD_event_functions tp_event_functions;

threadpool_common.cc

~~~~
THD_event_functions tp_event_functions=
{
  tp_wait_begin, tp_wait_end, tp_post_kill_notification
};
~~~~

threadpool.cc

- tp_wait_begin() -> wait_begin()
- tp_wait_end() -> wait_end()
- tp_post_kill_notification() -> vio_cancel(vio, SHUT_RD)

## 定时器

定时器设计为毫秒级精度，其提供以下功能：

1. 循环定时
2. 唤醒后巡检（stall、timeout）

### stall检测

查漏补缺：没有listener（即listener正在处理事件），则唤醒/创建工作线程

认定卡顿：队列堆积的方法是在每次stall巡检周期设置一个出队累计值。当处理过（出队累计值>0）且队列

仍有未处理的数据时，认定为stall

快速响应：唤醒所有的idle线程，或者线程不足时补充线程（创建）

洗心革面：在处理下一个事件时重置stall标记

### timeout检测

其中，超时时间点计算方法为：当前唤醒时间点+stall+等待时间，即：

~~~~
thd->wait_timeout = kill_idle_transaction_timeout / net_wait_timeout
next = current + 1000 * stall + 1000000 * thd->wait_timeout
~~~~

这既是连接的超时时间点，同时，也是下一次的检查时间点。

正确性：唤醒检查点和下一次检查点都需要加上memory barrier

在保证定时器功能性的前提下，也需要考虑性能影响。为了避免在stall检测时，对正向路径的前端请求造成颠簸，这里采用try+lock的优雅方式进行检测。

## 统计信息

thread_group.thread_count

thread_group.active_thread_count

num_worker_threads 当前全局线程数

threadpool_max_threads 最大线程数

## Observability

增加状态：

| status                  | scope  | 说明                            |
| :---------------------- | :----- | :------------------------------ |
| Threadpool_idle_threads | global | 空闲线程数                      |
| Threadpool_threads      | global | 全局线程数（listener & worker） |

## 内部数据结构

### 对象

- 线程组数组 all_groups <thread_group_t>(128)
- 线程组 thread_group_t
- 连接 connection_t
- 连接队列 connection_queue_t
- 工作线程 worker_thread_t active/idle
- 工作线程队列 worker_list_t
- 定时器 pool_timer_t

类图关系

![MySQL_ThreadPool_Class_Relationship_Diagram](/MySQL_ThreadPool_Class_Relationship_Diagram.png)

### 生命周期

全局对象（static）


| 名称           | 类型                              | 时机           |
| :------------- | :-------------------------------- | :------------- |
| 线程组数组     | all_groups <thread_group_t>(size) | tp_init/tp_end |
| 活跃线程组大小 | group_count                       | tp_init        |
| 线程组是否启动 | threadpool_started                | tp_init/tp_end |
| 定时器         | pool_timer                        | tp_init/tp_end |

### 对象数据结构详解

线程组数组

~~~~
all_groups <thread_group_t>(128)
~~~~

线程组 thread_group_t

~~~~
mutex                                     互斥量，保护线程组对象
pthread_attr                              线程私有变量
pollfd                                    epollfd
shutdown_pipe[2]                          用于关闭时通知listener
connection_queue_t queue;                 普通队列（FIFO）
connection_queue_t high_prio_queue;       高优队列（FIFO）
worker_list_t waiting_threads;            空闲（等待）线程队列（FIFO）
worker_thread_t *listener;                监听线程（listener）
int  thread_count;                        已创建的工作线程数量
int  active_thread_count;                 正在执行的工作线程数量
int  connection_count;                    处理的连接数量（累计），处理新链接时++，调整线程组大小时调整
int  waiting_thread_count;                等待的工作线程（lock/sync/net）
int io_event_count;                       需要处理的epoll事件数量（epoll_wait返回的ev个数++）
int queue_event_count;                    stall周期的出队次数
ulonglong last_thread_creation_time;      最近的工作线程的创建时间
bool shutdown;                            关闭线程池时设为true
bool stalled;                             线程组是否出现了stall
~~~~

连接 connection_t

```
THD *thd;                                 对应的THD
thread_group_t *thread_group;             对应的线程组
connection_t *next_in_queue;
connection_t **prev_in_queue;
ulonglong abs_wait_timeout;               超时时间点
bool logged_in;                           是否已经过应用建链，connection phase为false，command phase为true
bool bound_to_poll_descriptor;            是否绑定到线程组对应的epollfd
bool waiting;                             是否处于等待状态（lock/sync/wait）
uint tickets;                             高优队列配额
```

队列 connection_queue_t

```
typedef I_P_List<connection_t,
                 I_P_List_adapter<connection_t,
                                  &connection_t::next_in_queue,
                                  &connection_t::prev_in_queue>,
                 I_P_List_null_counter,
                 I_P_List_fast_push_back<connection_t> >
connection_queue_t;
```

工作线程 worker_thread_t

```
ulonglong  event_count;                   处理的事件数量（累计）
thread_group_t* thread_group;
worker_thread_t *next_in_list;
worker_thread_t **prev_in_list;
mysql_cond_t  cond;
bool          woken;                      进入waiting_threads睡去false，唤醒后为true
```

工作线程队列 worker_list_t

```
typedef I_P_List<worker_thread_t,
                 I_P_List_adapter<worker_thread_t,
                                  &worker_thread_t::next_in_list,
                                  &worker_thread_t::prev_in_list> >
worker_list_t;
```

定时器 pool_timer_t

```
mysql_mutex_t mutex;
mysql_cond_t cond;
volatile uint64 current_microtime;        唤醒时间点
volatile uint64 next_timeout_check;       超时时间点，初始MAX
int  tick_interval;                       stall时长，也是休眠时长
bool shutdown;                            启动时设置为false，停止时设置为true，并通过cond通知
```


---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: latch
---

# latch数据结构

先看一下整体的latch数据结构，以及之间的关系：

![InnoDB_latch](/InnoDB_latch.png)

下面逐个展开。

## latch基本信息

### latch ID

用于标识latch，数据结构为latch_id_t。

### latch ordering

在数据库中，从狭义视角看，latch用于保护修改的内存对象；从全局视角看，全局操作不同对象也需要遵循特定的顺序，因此有了latching order。数据结构为latch_level_t。

比如

~~~~
enum latch_level_t {
    ...
    SYNC_BUF_BLOCK,
    SYNC_BUF_PAGE_HASH,
    ...
}
~~~~

### latch counter

LatchCounter负责进行latch计数，计数信息包括：spin、waits、calls，并可以动态开启/关闭计数功能（MutexMonitor），也可以通过传入的callback函数用计数值进行运算（比如在innodb中统计latch信息）。

数据结构说明

| 变量/函数                          | 说明                         |
| :--------------------------------- | :--------------------------- |
| 变量/函数                          | 说明                         |
| Count  spin  waits  calls  enabled | 是否开启计数                 |
| vector<Count*> m_counters          | m_counters                   |
| bool m_active                      | 是否开启计数                 |
| enable/disable/reset               | 开启/关闭/重置计数           |
| single_register/single_deregister  | 注册/注销单例计数器          |
| sum_register/sum_deregister        | 注册/注销聚合计数器          |
| iterate                            | 迭代计数器，调用callback函数 |

### latch元信息

latch元信息由以上3类数据聚合而成（ID、ordering、counter），数据结构为latch_meta_t。

### 全局latch元信息

用于全局latch计数器统计并展示，其类型为latch_meta（vector<latch_meta_t*>）。

### 全局latch计数器

MutexMonitor用于show engine innodb status显示全局latch计数信息。

函数说明：

| 函数    | 说明                        |
| :------ | :-------------------------- |
| enable  | 开启所有LatchCounter计数    |
| disable | 关闭所有LatchCounter计数    |
| reset   | 重置所有LatchCounter计数    |
| iterate | 遍历LatchMetaData，执行函数 |

使用场景：

~~~~

sync_check_init()
  mutex_monitor = UT_NEW_NOKEY(MutexMonitor());
innodb_monitor_set_option()
  switch (set_option) {
  case MONITOR_TURN_ON:
      mutex_monitor->enable();
  case MONITOR_TURN_OFF:
      mutex_monitor->disable();
  case MONITOR_RESET_VALUE:
      mutex_monitor->reset();
innodb_show_mutex_status()
  ShowStatus  collector;
  mutex_monitor->iterate(collector);
sync_check_close()
  UT_DELETE(mutex_monitor);
~~~~

### latch状态

mutex_state_t，主要用于表示不同mutex的状态，参见下面的TAS mutex。

## latch

### 系统互斥量 OSMutex

OSMutex封装了系统的mutex，在WIN32上是CriticalSection，在Linux上是pthread_mutex_t。

OSMutex也是唯一一个未封装Policy（统计信息）的latch。

其方法包括：

| 函数                | 说明                |
| :------------------ | :------------------ |
| ctor dtor           | /                   |
| init destroy        | 创建 销毁           |
| enter try_lock exit | lock trylock unlock |

用法：

~~~~
1. 声明
Mutex m_mutex;
2. 初始化
ctor
m_mutex.init();
3. 使用
m_mutex.enter();
....
m_mutex.exit();
4. 销毁
dtor
m_mutex.destroy();
~~~~

### 条件互斥量 OSEvent

参见os0event

### TAS mutex

在这里通过PolicyMutex模板TAS互斥量，其中包括：

- MutexImpl：原子操作的实现方式，包括system mutex、TAS、event TAS、TTAS
- Policy：统计信息，包括NoPolicy（NoPolicy）、GenericPolicy（单项统计）、BlockMutexPolicy（聚合统计）

这4种TAS mutex的主要区别如下图所示：

![InnoDB_latch_PolicyMutex](/InnoDB_latch_PolicyMutex.png)

通过以上的二元组合产生TAS mutex：

![InnoDB_latch_TAS_mutex](/InnoDB_latch_TAS_mutex.png)

使用场景：

ib_mutex_t、ib_bpmutex_t在InnoDB中广泛使用，其MutexImpl可以是system mutex、event TAS和TTAS，而TAS没有使用，具体场景参见下面的表格：

| Mutex        | 使用场景                                                     |
| :----------- | :----------------------------------------------------------- |
| SysMutex     | file aio arrayos thread countsync array                      |
| ib_mutex_t   | fts_cachefile_systeminsert buffermaster keyrow loghash table |
| ib_bpmutex_t | BPageMutex buf_block_t                                       |

### reader-writer latch

参见下节rw latch

# rw latch

前面已经提到，MySQL封装了system mutex和system condition，但是没有封装system rwlock，MySQL自己通过TAS和osevent封装了一个rw latch。同时，rw latch还支持写操作的递归锁，即同一个线程可以多次获得写锁（依然不能同时获得读锁和写锁）。另外，为了公平的竞争（和system mutex的PTHREAD_MUTEX_ADAPTIVE_NP异曲同工），没有设计等待队列，不按照FIFO的等待顺序进入critical section（OSEvent调用的是broadcast），而是采用写者优先的方式，记录第一个等待的写者，优先通知。

rw latch提供以下函数：

| 函数                                               | 说明              |
| :------------------------------------------------- | :---------------- |
| rw_lock_create/rw_lock_free                        | 创建/销毁rw latch |
| rw_lock_?_lock (s/x/sx)                            | 加latch           |
| rw_lock_?_unlock (s/x/sx)                          | 释放latch         |
| rw_lock_x_lock_wait rw_lock_?_lock_nowait (x/s/sx) | 等待 不等待       |

{{< hint info >}}

**函数命名规则**

具体实现加上_func后缀，底层实现加上_ow后缀（TAS），附PSI信息加上pfs_前缀

{{</hint>}}

## lock_word设计

![InnoDB_latch_rw-latch_lock_word](/InnoDB_latch_rw-latch_lock_word.png)

从上图中可以看到，rw latch中的lock_word和TAS mutex不同，自旋锁的lock_word取值只有0、1，而rw latch的lock_word取值范围是(-(2 * X_LOCK_DECR), X_LOCK_DECR]，并且0x20000000为5亿+，足够使用。而且，在这个设计中，从lock_word的区间可以直接知道latch的状态，以及持有latch和等待latch的数量。

| lock_word取值                 | 说明                                         |
| :---------------------------- | :------------------------------------------- |
| X_LOCK_DECR                   | latch空闲                                    |
| (0, X_LOCK_DECR)              | 有X_LOCK_DECR-lock-word个读锁                |
| 0                             | 有1个写锁                                    |
| (-X_LOCK_DECR, 0)             | 有-lock-word个读锁，同时还有1个写锁在等待    |
| (-2X_LOCK_DECR, -X_LOCK_DECR] | 递归写锁，有 \|lock_word\|-X_LOCK_DECR个写锁 |

## 数据结构

通过上面的介绍，这里再理解数据结构就比较容易了，所以只列出几个关键的变量。

- waiters：是否处于等待中
- waiter_thread：第一个等待的写者
- recursive 是个bool 变量，用来表示当前的读写锁是否支持递归写模式，在某些情况下，例如需要另外一个线程来释放这个读写锁（insert buffer需要这个功能）的时候，就不要开启递归模式了。

这里的数据结构比较简单

recursive 是个bool 变量，用来表示当前的读写锁是否支持递归写模式，在某些情况下，例如需要另外一个线程来释放这个读写锁（insert buffer需要这个功能）的时候，就不要开启递归模式了。

## 加锁/解锁

![InnoDB_latch_rw-latch_lock_unlock](/InnoDB_latch_rw-latch_lock_unlock.png)

# 等待队列

## 等待队列的设计

在互斥量（InnoDB spin lock, rw latch）等待时，设计了一个等待队列机制。

设计思想是：

因为每个线程只会有两种状态：要么等待，要么已经拿到latch（granted），所以可以设计一个等待队列，其队列中的元素个数为线程数（OS_THREAD_MAX_N）即可。

同时为了避免sync_arra→mutex成为热点，将队列分为等待队列组（默认为1组，可以通过[innodb_sync_array_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_sync_array_size)调整），在latch需要等待时，通过放置等待对象（sync_cell_t）的方式进行等待。在latch释放时通过osevent进行通知。

在这里，为了保护sync_array_t，通过封装的system mutex（SysMutex）来保护，而不是latch，以此来避免可能存在的递归死锁情况的发生。

在放置等待对象时，采用random方式从组中随机挑选一个等待队列。同时，在等待队列内分配slot时，采用如下策略：

1. 首先尝试用中间的空洞（first_free_slot）
2. 用单调向前推进的free slot（next_free_slot）

同时，sync_cell_t采用一次性分配，latch等待时设置，latch释放时置空的方式复用。

为了提高性能的考虑，在acquire latch不成功时，首先进行spin操作，然后再放到等待队列中。

整体设计如下图所示：

![InnoDB_latch_sync_wait_array_diagram](/InnoDB_latch_sync_wait_array_diagram.png)

## 设计细节

数据结构和实现细节如下图所示：

![InnoDB_latch_sync_wait_array_details](/InnoDB_latch_sync_wait_array_details.png)

# 死锁检测和死锁预防

因为latch的死锁和lock的死锁不同，只能靠程序来保证。所以需要有相应的debug机制和检测机制来一定程度上辅助开发者避免死锁的发生。

在latch的设计中也包含了完善的死锁预防机制和死锁检测机制。

在每次需要latch等待时（sync_array_wait_event），即调用os_event_wait之前，需要启动死锁检测机制来保证不会出现死锁，从而造成无限等待。

在每次加锁成功（rw_lock_lock_word_decr，lock_word 递减后，函数返回之前）时，都会启动死锁预防机制，降低死锁出现的概率。

另外，因为死锁预防和死锁检测需要扫描比较多的数据，算法上也有递归操作，所以只在debug模式下开启。

## 死锁检测

死锁检测机制（sync_array_detect_deadlock）通过等待队列（sync_array_t）中的等待对象（sync_cell_t）上保存的等待latch（latch链表（lock->debug_list））和等待thread来判断是否形成有向无环图来确定是否存在死锁。

如果开启了多个等待队列，则该检测机制有缺陷，其只在单个等待队列上进行遍历，将无法发现死锁。

另外，在InnoDB中还会通过srv_error_monitor_thread后台线程来定时检测（1s），来处理无限等待和长时间等待的latch。

出现无限等待的场景是因为在lock_word操作和osevent通知直接可能会出现race condition，在这里通过判断latch是否已经可用（sync_arr_cell_can_wake_up），然后进行通知进行补偿。

如果出现了长时间的等待，InnoDB也会干预：当latch等待超过240秒，会输出到错误日志中；如果同一个latch被检测到等到超过600秒且连续10次被检测到，则InnoDB会通过assert来自杀。

## 死锁预防

死锁预防机制（LatchDebug）通过线程及其已持有的latching order来进行进行检测。同一个线程的加锁顺序必须从优先级高到低，即如果一个线程目前已经加了一个低优先级的锁A，在释放锁A之前，不能再请求优先级比锁A高(或者相同)的锁。

通过锁优先级可以低死锁发生的概率，但是不能完全消除。原因是可以把锁设置为SYNC_NO_ORDER_CHECK 这个优先级，这是最高的优先级，表示不进行死锁预防检查，如果上层的程序员把自己创建的锁都设置为这个优先级，那么InnoDB 提供的这套机制将完全失效，所以要养成给锁设定优先级的好习惯。

为了支持latch的debug，定义了一组基本数据结构：

- Latch_Level：为了避免AB-BA问题造成的死锁定义了order（上面已经介绍）
- sync_check_functor_t提供了基于order的acquire比较方法
- CreateTracker：跟踪latch的创建信息（file、line、thread_id）
- MutexDebug：跟踪latch的持有信息（file、line、thread_id）
- Latched：每个线程所持有的latch
- Latches：所有线程所持有的latches
- LatchDebug：死锁预防检测器

### sync_check_functor_t

sync_check_functor_t作为模板方法，提供了基于latch ordering的比较，供调用线程（calling thread）检查是否持有某些latch。

有3个具体的模板方法：

- btrsea_sync_check 是否持有btr search mutex相关的latch
- dict_sync_check 是否持有dictionay latching相关的latch
- sync_allowed_latches 是否持有某些latch

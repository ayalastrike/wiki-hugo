---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: lazy drop/truncate table
---

# 背景

DBA在线上进行运维时，会遇到drop table，truncate table出现卡顿（维持数秒），在这期间，如果业务访问量大，请求会在短时间内急剧堆积，对线上的稳定性造成很大的挑战。

现有DBA需要drop/truncate table的运维场景有：

1. 业务需要调整schema，发起DDL操作（drop）
2. 业务清理过期表（按照日期定期滚动）（drop）
3. 业务废弃（drop）
4. 业务重用表（drop + create）
5. 业务清理数据（truncate）

其中DDL过程如下：

DBA在线上进行DDL的时候，采用gh-ost的旁路方式进行schema变更，最终会drop table。

1. create ghost table like origin table
2. alter ghost table
3. copy origin table data and apply its binlog to ghost table ...
4. lock origin table with write lock, until all binlog applied
5. rename origin table
6. rename ghost to origin
7. drop origin table

# root cause

drop table、truncate table的目的都是为了清理数据，除了清理磁盘上的数据外，内存中的高速缓存（buffer pool）也需要清理，以免用户访问到了失效的缓存而得到错误的结果。

在MySQL的InnoDB存储引擎中，缓冲池以page为单位组织数据，每个页用space_id,page_no来标记，这些页存放在buffer pool的两个地方：AHI和LRU list（+flush list）。缓存的清理也是在这里进行的。

在清理AHI和LRU list的时候，都需要锁住buffer pool，防止其他线程访问到过期的数据，这也是发生stall的原因。当buffer pool越大，遍历的开销就会越大，用户感受到的stall也会越明显。

# 方案设计

既然我们stall的root cause是清理cache，那么，我们把清理cache延后来做（lazy invalid）就可以避免stall，在正常流程中只清理数据，如下图所示：

![MySQL_lazy_truncate_drop_table_0](/MySQL_lazy_truncate_drop_table_0.png)

## 其他方案比较

### 方案1

drop table不需要复用space_id，但是truncate table会复用space_id，这也是清理buffer pool的不同之处：

- truncate table：清理LRU + flush list
- drop table：清理flush list

从这里可以看出，drop table的清理代价较低，阿里的patch也是通过这样做的，即将truncate转换为drop+create，但是在MySQL 5.7上还需要实现原子DDL，这个需要对事务模型做扩充，改动量很大。

### 方案2

DBA将truncate拆解为drop + create，中间有时间窗口会报表不存在，这和truncate语义是相违背的。

### 方案3

另一个方案是将page按照space_id维度来组织，这样，在进行清理时，只需要依次fix page即可，但是这样维护该链表会影响正常的dml（读写交叉影响）。

补充材料

MySQL 5.5.23和Percona 5.1之前也做过lazy drop table的优化，优化方向是减少latch的持有时长，[链接在此](http://mysql.taobao.org/monthly/2016/01/07/)。

## 引入版本号

那我们要解决的问题变为，如何感知到cache失效。这里我们引入版本号来解决：

- 在内存中保留已删除表对应的表空间对象fil_space_t，并在其中加入m_version
- 在内存中的page上也增加版本号，即buf_page_t上增加m_version，并增加一个指针指向fil_space_t
- 在访问page时比较当前页和表空间上的版本，如果不一致，则page invalid

page的内存结构如下图所示：

![MySQL_lazy_truncate_drop_table_3](/MySQL_lazy_truncate_drop_table_3.png)

version的生命周期为：

|                     | fil_space.m_version                 | page.m_version                |
| :------------------ | :---------------------------------- | :---------------------------- |
| 初始值              | 0（从磁盘上构建出内存的表空间对象） | 0（从磁盘上构建出内存的page） |
| drop/truncate table | 1                                   | 0                             |

从上面可以看到，只有fil_space.m_version会发生变化，更新的时机为：

- truncate/drop table时进行m_version++

这样，随着page的访问，会逐渐进行page lazy invalid，并放入缓冲池的空闲页列表（free list）。

## 正向的drop/truncate table调整

维持原来的流程，调整两点：

- 保留fil_space表空间对象，并将其放入fil_system→deleted_spaces（回收站）中，在正向路径中（fil_system中）摘除（drop table需要此步，truncate table无需此步）
- 不清理buffer_pool中的page（增加BUF_REMOVE_FLUSH_NO_OP，即no-op）

## 表空间对象fil_space的维护

同样，我们不会一直保留fil_space表空间对象（针对drop table，truncate table选择保留），我们需要增加一个引用计数来确定安全释放表空间对象的边界，即当表对应的page数为零时（m_n_ref_count = 0）时，就可以安全的释放这个内存对象了。

并且，新建一个fil_system->deleted_spaces,

如下图所示：

![MySQL_lazy_truncate_drop_table_1](/MySQL_lazy_truncate_drop_table_1.png)

page的访问场景有：

- 用户读写数据/索引：invalid
- 后台的page cleaner线程刷脏：skip
- buffer pool脏页达到阈值（70%）时，强制刷脏：skip
- buffer pool满，LRU进行淘汰：invalid
- buffer pool dumper：skip

综上所述，整个处理模型如下图所示：

![MySQL_lazy_truncate_drop_table_2](/MySQL_lazy_truncate_drop_table_2.png)

##  lazy invalid的边界

那么drop table和truncate table何时完成lazy invalid呢？

### drop table

space_id会一直往上增长，所以不会复用；同时，已经删除的表不再有访问路径（被放置在deleted_spaces中），那么buf page只能等到buffer pool满；或者为脏页被page cleaner或脏页达到阈值强制刷脏

### truncate table

space_id会复用（不变），所以会有用户的访问路径（RW）可以触发page invalid，释放速度会快于drop table。

## 正确性

还有以下几点，需要解释：

- 在正向的drop table/truncate table流程中，已经对持久化的信息进行了清理，只保留cached page和cached file_space，其中cached page会通过版本识别invalid，cached file_space会从file_system子系统中摘除，放入deleted_spaces，无访问路径和其关联，且和fil_system的LRU解绑
- 正常/非正常关机下，因为只有内存中的数据结构，所以关机正常消逝，不影响正确性
- 正向路径的更改除未销毁cache page和cached file_space外，其余对数据的变更均产生WAL（没有修改），这保证了recovery的正确性
- 重新创建/复用fil_space的对象：drop table space_id不会重用，truncate table space_id会重用，但是因为是不同的内存表空间对象，不存在交集，所以不影响表空间对象的正确性
- 对于同步，ddl走的是SBR，在从库上执行的流程和主库相同，可以保证操作和数据的同构
- 临时表的生命周期和会话绑定，因此不涉及临时表的操作
- 没有对数据存储进行改动，所以兼容性没有问题
- 引入新的参数（ddl_lazy_cache_invalid）来控制是否开启优化，关闭保持原生行为

需要考虑到drop table后，再create table时，space_id的使用场景，所以这里附上space_id的分配策略：

在分配space_id时，只在内存中将相应值++，然后将相应的变更（space_id + offset + id）写入mtr log。

{{< hint info >}}

在记录space_id时，需要保证内存和外存的crash一致性。

因为某些情况下（比如临时表），不能将space_id的分配记入log，这样file-per-table下，就出现了三个场景：

初始space_id = x

1. 日志开启，内存space_id = x+1，日志记录了x+1，recovery后space_id = x+1

2. 不开启日志， 内存space_id = x+1，日志不记录，recovery后space_id = x

3. 分配了两次，第一次临时表，space_id=x+1，日志不记录，第二次用户表：space_id = x + 2，recovery后space_id=x+2

因此，space_id会出现回撤，洞和正常三种状态。并且，如果临时表过多，会导致达到max_space_id。

{{</hint>}}

## 性能

性能评估

- 因为要对内存对象加字段（version），要保证不破坏cacheline
- page访问增加一条CPU指令（fil_space→m_version == page→m_version）代价
- 后台线程对前台用户路径无正交

## 限制条件

表必须为独立表空间，即每个用户表有独立的space_id

表不能有附属对象（外键关联、trigger、view等）

不支持同时drop多个表，不支持跨存储引擎的表

限制条件下的drop table/truncate table走的还是MySQL的原生流程，只是不走优化流程。

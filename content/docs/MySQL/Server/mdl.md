---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

# 历史

为了解这个著名的[bug#989](https://bugs.mysql.com/bug.php?spm=a2c6h.12873639.article-detail.4.50e71d38Qf5uk4&id=989)：

{{< hint info >}}

DML和DDL如果并发执行，binlog序错乱：主库并发执行了DDL（T → T'）和DML（T’），但是提交到了binlog中顺序是DDL+DML，这样在从库上回放时，先执行DDL，后执行DML，但此时表结构已经变为T‘，DML执行失败

{{</hint>}}

因此，在MySQL 5.5中，引入了[MDL](https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html)（Meta Data Lock）来控制DDL和DML的并发，保证元数据的一致性。

设计思理：

- [WL#3726：DDL locking for all metadata objects](http://dev.mysql.com/worklog/task/?id=3726)
- [WL#4284：Transactional DDL locking](http://dev.mysql.com/worklog/task/?id=4284)

谈到MDL，需要先谈谈MySQL的锁定机制设计以及thr_lock

# MySQL locking

为了保证数据的一致性和完整性，数据库系统普遍存在封锁机制，而封锁机制的优劣直接关系到数据库系统的并发处理能力和性能，所以封锁机制的实现也成为了各种数据库的核心技术之一。

[MySQL的封锁机制](https://dev.mysql.com/doc/refman/5.7/en/internal-locking.html)有三种：

- row-level locking：InnoDB、NDB Cluster
- table-level locking：MyISAM、MEMORY、MERGE、CSV等非事务存储引擎
- page-level locking：BerkeleyDB

MySQL采用如此多样的封锁机制是由其产品定位和发展历史共同决定的。首先，MySQL的产品定位是通过plugin机制可以接入多个存储引擎。在早期的存储引擎（MyISAM和MEMORY）设计中，设计原则建立在"任何表在同一时刻都只允许单个线程（无论读写）对其访问"之上。随后，MySQL3.23修正了之前的假设：MyISAM支持[Concurrent Insert](https://dev.mysql.com/doc/refman/5.7/en/concurrent-inserts.html)：如果没有hole，可以多个读线程并发读同一张表，同时多个写线程以队列的形式进行尾部insert。之后，BerkeleyDB和InnoDB的引入也挑战了之前的设计假设，要求page-level、和row-level locking。此时，之前的设计方式已经和存储引擎所提供的能力不相衬了。因此MySQL做出了改变，允许存储引擎自己改变MySQL 通过接口传入的锁定类型（也就是上面的3种）而自行决定该怎样封锁数据。

MySQL加锁的顺序

SQL → open table → 加MDL锁 → 加表锁 → 加InnoDB锁 → SQL执行 → 释放MDL锁 → 释放表锁 → 释放InnoDB锁

其中表锁是通过thr_lock提供的，在[MySQL 5.7.5](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-5.html)中，thr_lock被MDL替换：

{{< hint info >}}

Scalability for InnoDB tables was improved by avoiding THR_LOCK locks. As a result of this change, DML statements for InnoDB tables that previously waited for a THR_LOCK lock will wait for a metadata lock:

- Explicitly or implicitly started transactions that update any table (transactional or nontransactional) will block and be blocked by LOCK TABLES ... READ for that table. This is similar to how LOCK TABLES ... WRITE works.
- Tables that are implicitly locked by LOCK TABLES now will be locked using metadata locks rather than THR_LOCK locks (for InnoDB tables), and locked using metadata locks in addition to THR_LOCK locks (for all other storage engines). Implicit locks occur for underlying tables of a locked view, tables used by triggers for a locked table, or tables used by stored programs called from such views and triggers.
  Multiple-table updates now will block and be blocked by concurrent LOCK TABLES ... READ statements on any table in the update, even if the table is used only for reading.
- HANDLER ... READ for any storage engine will block and be blocked by a concurrent LOCK TABLES ... WRITE, but now using a metadata lock rather than a THR_LOCK lock.

{{</hint>}}

相关WorkLog：[WL#6671: Improve scalability by not using thr_lock.c locks for InnoDB tables](https://dev.mysql.com/worklog/task/?spm=a2c6h.12873639.article-detail.5.1ede13fbwVMKN7&id=6671)

## Table Locking

MySQL的表级锁定最开始使用thr_lock，主要分为两种类型，一种是读锁定，另一种是写锁定。table lock模块为每个表维护了四个队列来表示这两种锁定：

- granted read/write
- waiting read/write

如下：
• Current read-lock queue (lock->read)
• Pending read-lock queue (lock->read_wait)
• Current write-lock queue (lock->write)
• Pending write-lock queue (lock->write_wait)

对于DQL，锁类型一般是TL_READ；对于普通的DML，是TL_WRITE_ALLOW_WRITE

代码路径：

````
lock_tables
    mysql_lock_tables
        thr_multi_lock
            thr_lock
````

# MDL locking

MDL Locking通过多层次、多粒度的locking，在满足一致性和完整性的前提下保证并发性能。

## 锁的获取

等待中的锁，按照锁优先级编排锁的获取。默认写者优先，但读者在配置了[max_write_lock_count](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_write_lock_count)（一般不会出现）后，可以优先处理。

MDL锁是依次申请的（one by one），并进行死锁检测：

- DML：按照语句中table的出现顺序获取
- DDL：按照table的字母序申请

比如

````
RENAME TABLE tbla TO tbld, tblc TO tbla; // 按照a->c->d顺序申请MDL锁（name order）
RENAME TABLE tbla TO tblb, tblc TO tbla; // 按照a->b->c顺序申请MDL锁（name order）
这样可以保证表按同序申请
````

但如果DML、DDL并发执行，则先后顺序有差异，见官方文档中的[示例](https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html)。

## 锁的释放

为了保证事务的可序列化，需要保证保证DML和DDL的互斥，MDL锁只能在事务提交后释放（S2PL）。

## 锁的状态变更

观测

可以通过[performance_schema.metadata_locks](https://dev.mysql.com/doc/refman/5.7/en/performance-schema-metadata-locks-table.html)查看MDL信息

````
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE NAME ='global_instrumentation';
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES' WHERE NAME ='wait/lock/metadata/sql/mdl';
select* from performance_schema.metadata_locks
````

MDL状态转换如下图所示：

![MySQL_MDL_State_Machine](/MySQL_MDL_State_Machine.png)

## 死锁检测

造成死锁是因为：

- 访问共享资源
- 多个线程访问
- 互斥冲突采用等待策略
- 不能按同一固定顺序请求锁

从当前线程开始，转换到全局锁列表，然后在深度遍历，当wait_for和当前THD相同时，形成环；然后递归返回时，通过权重确定victim。

````c++

ctx::find_deadlock
    while (1) {
        Deadlock_detection_visitor dvisitor(this); // ctx
        ctx::visit_subgraph();          当前线程是否有锁等待MDL_context::m_waiting_for，有的话就沿着ticket搜下去，没有就退出。
            ticket->accept_visitor      搜索视角的转换，从 MDL_context 经过 MDL_ticket 进入到 MDL_lock
                lock->visit_subgraph    核心逻辑
    }
````

visit_subgraph

先给搜索深度加1，然后判断是否超过最大搜索深度（MAX_SEARCH_DEPTH= 32），超过就无条件认为有死锁，退出；
遍历当前锁的ticket链表，看ticket对应的线程是否和死锁检测的发起线程是同一个，如果是则说明有回路，退出（相当于做了一层的广度搜索）；

从头开始遍历当前锁的ticket链表，对每个ticket对应的线程，递归调用MDL_context::visit_subgraph（深度搜索）。
整个死锁检测逻辑是一个加了深度限制的深搜，中间同时多了一层广搜。

Deadlock_detection_visitor 是死锁检测中重要的辅助类，主要负责：

- 记录死锁检测的起始线程
- 记录被选做victim的线程
- 在检测到死锁，深搜一层层退出的时候，会依次检查回路上各线程的死锁权重，选择权重最小的做为最终的victim（权重由锁的类型决定）

死锁权重：

- DDL：100
- USER_LEVEL_LOCK：50
- DML：0

即优先回滚DDL

# MDL子系统

锁子系统的核心功能：

- 由用户侧提交锁请求
- 提供一个中心化的锁管理（锁对象、granted、waiting、停等通知、死锁检测）
- 通过ticket关联用户和锁对象

## 锁子系统交互

在MDL子系统中，体现在THD和MDL锁子系统的交互上，如下图所示：![MySQL_MDL_object_relationship](/MySQL_MDL_object_relationship.png)

加锁是用户线程（THD）向MDL子系统申请并获得对应锁的ticket的过程，加锁成功标志是MDL模块返回一个对应的ticket，大致逻辑如下：

1. 解析SQL语句，根据语义对每一个表对象设置TABLE_LIST.mdl_request，如对普通的select语句 TABLE_lsit.mdl_request.type 就是MDL_SHARED_READ（st_select_lex::set_lock_for_tables()）
2. 用户线程在打开每个表之前，会请求和这个表对应的MDL锁，通过 thd→mdl_context.acquire_lock()接口将mdl_request请求发给MDL子系统
3. MDL模块根据请求类型和已有的锁来判断请求能否满足，如果可以就返回一个ticket；如果不可以就等待，等待结果可以是成功（别的线程释放了阻塞的MDL锁）、失败（超时、连接被kill或者被死锁检测选为victim）
4. 用户线程根据MDL模块的返回结果，决定继续往下走还是报错退出。

{{< hint danger>}}

DL锁并不是对表加锁，而是在加表锁前的一个预检查，如果能拿到MDL锁，下一步加相应的表锁。

{{</hint>}}

MDL子系统内部加锁流程

![MySQL_MDL_acquire_lock_process](/MySQL_MDL_acquire_lock_process.png)

MDL子系统内部类关系图

![MySQL_MDL_class_relationship](/MySQL_MDL_class_relationship.png)

## 生命周期

整个锁系统各个对象的生命周期如下：

![MySQL_MDL_lifecycle](/MySQL_MDL_lifecycle.png)

## 数据结构

### MDL_key 锁对象标识

Metadata lock object key用于标识MDL对象，由三元组组成：namespace、db、table，作为全局锁表（MDL_map）、锁请求（MDL_request）、锁对象（MDL_lock）的成员变量。

其数据结构（POD）如下：![MySQL_MDL_key](/MySQL_MDL_key.png)

````
ctor        namespace, db, name
            *MDL_key
reset       置空
is_equal    bit equal
cmp         memcmp
禁止拷贝
````

MDL lock提供多粒度锁，分为scoped lock和object lock，namespace层次如下：![MySQL_MDL_key namespace](/MySQL_MDL_key namespace.png)

namespace的详细说明（这里颠倒了枚举定义的顺序，按scoped lock和object lock顺序论述）：

| enum_mdl_namespace | 说明                                                         |
| :----------------- | :----------------------------------------------------------- |
| enum_mdl_namespace | 说明                                                         |
| GLOBAL             | 全局唯一，防止DDL和写操作的过程中执行 set golbal_read_only =on 或flush tables with read lock;IX+S(EXPLICIT) 显式释放 |
| COMMIT             | 全局唯一，执行flush table with read lock后，防止在途写事务的提交IX+S(EXPLICIT) 显式释放 |
| TABLESPACE         | 表空间锁                                                     |
| SCHEMA             | 库锁                                                         |
| TABLE              | 表锁                                                         |
| FUNCTION           | 用于UDF                                                      |
| PROCEDURE          | 用于SP                                                       |
| TRIGGER            | 用于TRIGGER                                                  |
| EVENT              | 用于event-scheduler                                          |
| USER_LEVEL_LOCK    | 用于test sync，GET_LOCK(str,timeout)，RELEASE_LOCK(str)      |
| LOCKING_SERVICE    | plugin service                                               |

其中具体的应用场景：

````
MDL_key::TABLESPACE   X/IX TRX
用于
CREATE or ALTER TABLESPACE
DISCARD or IMPORT TABLESPACE
CREATE TABLE LIKE
 
MDL_key::SCHEMA       X/IX TRX
用于
lock_schema_name
  GLOBAL+IX+STMT
  SCHEMA+X+TRX
 
lock_object_name
  GLOBAL+IX+STMT
  SCHEMA+IX+TRX
  OBJECT+X+TRX
 
lock_table_names
  SCHEMA+IX+TRX
 
MDL_key::GLOBAL       IX  STMT
                      S   EXPLICIT
 
MDL_key::COMMIT       IX  STMT
                      IX  EXPLICIT
                      S   EXPLICIT
````

示例：

锁的申请由上到下，由大到小，由意向到具体：

{{< hint info >}}

session A: begin transaton and select

session B: ALTER TABLE ...

| GLOBAL         | IX   | STMT |
| -------------- | ---- | ---- |
| SCHEMA test    | IX   | TRX  |
| TABLE   test t | SU   | TRX  |

session A: commit

session B:

| TABLE   test t | X    | TRX  |
| -------------- | ---- | ---- |
|                |      |      |

{{</hint>}}

### 锁信息

在MDL_request和MDL_ticket中，都需要描述锁信息（key+type+duration）

type表示MDL锁的类型，enum_mdl_type

| 锁类型                    | 简写 | 说明                       |
| :------------------------ | :--- | :------------------------- |
| MDL_INTENTION_EXCLUSIVE   | IX   | 意向X锁，只用于scoped lock |
| MDL_SHARED                | S    |                            |
| MDL_SHARED_HIGH_PRIO      | SH   |                            |
| MDL_SHARED_READ           | SR   |                            |
| MDL_SHARED_WRITE          | SW   |                            |
| MDL_SHARED_WRITE_LOW_PRIO | SWLP |                            |
| MDL_SHARED_UPGRADABLE     | SU   |                            |
| MDL_SHARED_READ_ONLY      | SRO  |                            |
| MDL_SHARED_NO_WRITE       | SNR  |                            |
| MDL_SHARED_NO_READ_WRITE  | SNRW |                            |
| MDL_EXCLUSIVE             | X    |                            |



\* MDL_INTENTION_EXCLUSIVE IX // 意向X锁，只用于scope 锁

\* MDL_SHARED S // 只能读metadata，当能读写数据，如检查表是否存在时用这个锁

\* MDL_SHARED_HIGH_PRIO SH // 高优先级S锁，可以抢占X锁，只能读metadata，不能读写数据，用于填充INFORMATION_SCHEMA，或者show create table时

\* MDL_SHARED_READ SR // 可以读表数据，select语句，lock table xxx read 都用这个

\* MDL_SHARED_WRITE SW // 可以更新表数据，insert，update，delete，lock table xxx write, select for update，

\* MDL_SHARED_UPGRADABLE SU // 可升级锁，可以升级为SNW或者X锁，ALTER TABLE第一阶段会用到

\* MDL_SHARED_NO_WRITE SNW // 可升级锁，其它线程能读metadata，数据可读不能读，持锁者可以读写，可以升级成X锁，ALTER TABLE的第一阶段

\* MDL_SHARED_NO_READ_WRITE SNRW // 可升级锁，其它线程能读metadata，数据不能读写，持锁者可以读写，可以升级成X锁，LOCK TABLES xxx WRITE

\* MDL_EXCLUSIVE X // 排它锁，禁止其它线程的所有请求，CREATE/DROP/RENAME TABLE

- MDL_INTENTION_EXCLUSIVE(IX) 意向排他锁，锁定一个范围，用在GLOBAL/SCHEMA/COMMIT粒度。
- MDL_SHARED(S) 用在只访问元数据信息，不访问数据。例如CREATE TABLE t LIKE t1;
- MDL_SHARED_HIGH_PRIO(SH) 也是用于只访问元数据信息，但是优先级比排他锁高，用于访问information_schema的表。例如：select * from information_schema.tables;
- MDL_SHARED_READ(SR) 访问表结构并且读表数据，例如：SELECT * FROM t1; LOCK TABLE t1 READ LOCAL;
- MDL_SHARED_WRITE(SW) 访问表结构且写表数据， 例如：INSERT/DELETE/UPDATE t1 … ;SELECT * FROM t1 FOR UPDATE;LOCK TALE t1 WRITE
- MDL_SHARED_WRITE_LOW_PRIO(SWLP) 优先级低于MDL_SHARED_READ_ONLY。语句INSER/DELETE/UPDATE LOW_PRIORITY t1 …; LOCK TABLE t1 WRITE LOW_PRIORITY。
- MDL_SHARED_UPGRADABLE(SU) 可升级锁，允许并发update/read表数据。持有该锁可以同时读取表metadata和表数据，但不能修改数据。可以升级到SNW、SNR、X锁。用在alter table的第一阶段，使alter table的时候不阻塞DML，防止其他DDL。
- MDL_SHARED_READ_ONLY(SRO) 持有该锁可读取表数据，同时阻塞所有表结构和表数据的修改操作，用于LOCK TABLE t1 READ。
- MDL_SHARED_NO_WRITE(SNW) 持有该锁可以读取表metadata和表数据，同时阻塞所有的表数据修改操作，允许读。可以升级到X锁。用在ALTER TABLE第一阶段，拷贝原始表数据到新表，允许读但不允许更新。
- MDL_SHARED_NO_READ_WRITE(SNRW) 可升级锁，允许其他连接读取表结构但不可以读取数据，阻塞所有表数据的读写操作，允许INFORMATION_SCHEMA访问和SHOW语句。持有该锁的的连接可以读取表结构，修改和读取表数据。可升级为X锁。使用在LOCK TABLE WRITE语句。
- MDL_EXCLUSIVE(X) 排他锁，持有该锁连接可以修改表结构和表数据，使用在CREATE/DROP/RENAME/ALTER TABLE 语句。

duration表示MDL锁的持有时长，enum_mdl_duration

| 锁时长          | 说明                                |
| :-------------- | :---------------------------------- |
| MDL_STATEMENT   | 语句结束自动释放                    |
| MDL_TRANSACTION | 事务结束自动释放                    |
| MDL_EXPLICIT    | 显示申请，显示释放（unlock tables） |

### 锁矩阵

MDL的锁矩阵根据scoped lock和object lock分为两类，其中有根据加锁策略分为granted matrix和waiting matrix。

MDL_scoped_lock只使用3种锁类型：IX、S、X

**scoped_lock granted matrix**


````
         | Type of active |
 Request |   scoped lock  |
  type   |  IX   S  X     |
---------+----------------+
IX       |   +   -  -     |
S        |   -   +  -     |
X        |   -   -  -     |
````

**scoped_lock waiting matrix**


````
         |    Pending    |
 Request |  scoped lock  |
  type   |  IX  S  X     |
---------+---------------+
IX       |   +  -  -     |
S        |   +  +  -     |
X        |   +  +  +     |
````

MDL_object_lock使用除IX外的所有锁类型

**object_lock granted matrix**


````
Request   |  Granted requests for lock                  |
 type     | S  SH  SR  SW  SWLP  SU  SRO  SNW  SNRW  X  |
----------+---------------------------------------------+
S         | +   +   +   +    +    +   +    +    +    -  |
SH        | +   +   +   +    +    +   +    +    +    -  |
SR        | +   +   +   +    +    +   +    +    -    -  |
SW        | +   +   +   +    +    +   -    -    -    -  |
SWLP      | +   +   +   +    +    +   -    -    -    -  |
SU        | +   +   +   +    +    -   +    -    -    -  |
SRO       | +   +   +   -    -    +   +    +    -    -  |
SNW       | +   +   +   -    -    -   +    -    -    -  |
SNRW      | +   +   -   -    -    -   -    -    -    -  |
X         | -   -   -   -    -    -   -    -    -    -  |
SU -> X   | -   -   -   -    -    0    0   0    0    0  |
SRO -> X  | -   -   -   -    0    0    0   0    0    0  |
SNW -> X  | -   -   -   0    0    0    0   0    0    0  |
SNRW -> X | -   -   0   0    0    0    0   0    0    0  |
````

0：不可能出现的情况，比如对于SU锁来说其和自身是不兼容的，不可能有2个线程对同一个对象都持有SU锁，所以就不存在当一个线程进行锁升级时，另一个线程持有SU

**object_lock waiting matrix**


````
Request   |         Pending requests for lock          |
 type     | S  SH  SR  SW  SWLP  SU  SRO  SNW  SNRW  X |
----------+--------------------------------------------+
S         | +   +   +   +    +    +   +    +     +   - |
SH        | +   +   +   +    +    +   +    +     +   + |
SR        | +   +   +   +    +    +   +    +     -   - |
SW        | +   +   +   +    +    +   +    -     -   - |
SWLP      | +   +   +   +    +    +   -    -     -   - |
SU        | +   +   +   +    +    +   +    +     +   - |
SRO       | +   +   +   -    +    +   +    +     -   - |
SNW       | +   +   +   +    +    +   +    +     +   - |
SNRW      | +   +   +   +    +    +   +    +     +   - |
X         | +   +   +   +    +    +   +    +     +   + |
SU -> X   | +   +   +   +    +    +   +    +     +   + |
SRO -> X  | +   +   +   +    +    +   +    +     +   + |
SNW -> X  | +   +   +   +    +    +   +    +     +   + |
SNRW -> X | +   +   +   +    +    +   +    +     +   + |
````

SH 比 X 锁的优先级还高，正是其高优先级(high priority)的体现。

### MDL_map 全局锁信息

static，在MySQL启动时初始化，并预分配两个全局唯一的scoped lock（GLOBAL+COMMIT），MDL_map是MDL子系统的内部对象，外部不可见。

作为全局MDL锁的存储，其必定会成为热点，之前采用partition mutex的方式解决热点，后改进采用xxx。并且采用lazy random re-org的方式进行内存整理。

### MDL_context

作为THD和MDL子系统的交互接口，提供锁的管理功能（申请、释放、升级、clone、回滚、死锁检测）。

MDL_context的生命周期如下（和THD同生命周期）：

````
声明定义
class THD {
  MDL_context mdl_context;
};
 
初始化
THD::THD
    mdl_context.init(this);
 
使用：申请锁
acquire lock
thd->mdl_context.acquire_lock(&mdl_request, thd->variables.lock_wait_timeout))
 
释放
THD::release_resources
    if (!cleanup_down)
        THD::cleanup
            mdl_context.release_transactional_locks();
                mdl_context::release_locks_stored_before(MDL_STATEMENT, NULL);
                mdl_context::release_locks_stored_before(MDL_TRANSACTION, NULL);
    mdl_context.destroy();
````

MDL_context.m_tickets[]中存储了3个ticket链表：

- STMT
- TRX
- EXPLICIT

其中STMT和TRX属于automiatc release，EXPLICIT属于manual release，有以下4种锁使用explicit显示锁：

````
LOCK TABLES locks
User-level locks
HANDLER locks
GLOBAL READ LOCK locks
````

数据结构

````
m_tickets       指针数组（里面是3个ticket链表，分别代表STMT,TRX,EXPLICIT）
m_owner         指向THD
m_wait          实现锁等待（MDL_wait）
m_waiting_for   当前线程正在等待的锁（MDL_wait_for_subgraph*）
 
 
find_ticket                                 在当前线程的ticket链表中查找一个ticket(和request->key同一对象&强度>=request->type)
clone_ticket                                clone ticket
 
mdl_savepoint                               生成savepoint
rollback_to_savepoint                       MDL锁回滚到某个savepoint
release_locks_stored_before                 释放ticket链表上在某个ticket之前所有ticket
release_lock                                释放单个MDL锁（全局和自己）
release_locks
release_all_locks_for_name                  把当前线程对某个对象加的所有MDL锁都释放掉
 
acquire_lock                                申请锁
acquire_locks                               一次性申请多个X锁，要么全部成功要么全部失败，用于RENAME, DROP和其他DDL语句
try_acquire_lock                            尝试申请锁，失败就返回out_ticket，没有死锁检测
 
upgrade_shared_lock                         升级共享锁
  
owns_equal_or_stronger_lock                 当前线程是否已持有更强的锁
find_lock_owner                             找到持有锁的第一个owner
 
has_lock                                    当前线程是否在savepoint之前持有指定的锁
has_locks                                   当前线程是否没有持有任何锁（3个链表都为空）
has_locks_waited_for
 
set_explicit_duration_for_all_locks      在LOCK TABLES时使用，因为trx lock生命周期长于explicit，所以将stmt和trx链表中的锁移到explicit链表中
set_transaction_duration_for_all_locks 上面的反操作
 
set_lock_duration                           将当前线程的某个ticket在链表中移动
release_statement_locks                     释放所有语句范围的锁
release_transactional_locks                 释放所有事务范围的锁
get_deadlock_weight                         死锁时拿一个权重值，以此来判断对应线程是否要做为victim
set_force_dml_deadlock_weight
set_needs_thr_lock_abort
get_needs_thr_lock_abort
 
find_deadlock                               检测是否有死锁
visit_subgraph                              和死锁检测相关
````

acquire_locks：一次性申请多个X锁，要么全部成功要么全部失败，用于RENAME, DROP和其他DDL语句
如果后续的锁grant失败，会通过savepoint将前面已经申请到的锁也rollback。

比如drop table test.t1这个DDL会一次加3个锁：

1. GLOBAL，MDL_INTENTION_EXCLUSIVE
2. test 库, MDL_INTENTION_EXCLUSIVE
3. test.t1 表，MDL_EXCLUSIVE

MDL_context::acquire_lock

主加锁函数，调试MDL锁相关问题时，给这个函数加断点比较有效。先调用MDL_context::try_acquire_lock_impl，如果加锁失败就进入等待加锁逻辑：

1. 将MDL_context::try_acquire_lock_impl返回的ticket放进MDL_lock的等待队列
2. 触发一次死锁检测
3. 进入等待，等待又分为2种：
   1. 定时检查等待: 如果当前请求的锁是比较高级的（对于MDL_object_lock是比MDL_SHARED_NO_WRITE类型更高，对于MDL_scoped_lock是MDL_SHARED类型），就会每秒给其它持有当前锁的线程（并且这些连接持有的锁等级比较低）发信号，通知其释放锁，然后再检查是否锁已拿到
   2. 一直等待，直到超时
4. 检查步骤3的等待结果，可以是GRANTED（拿到锁）、VICTIM（被死锁检测算法选为受害者）、TIMEOUT（加锁超时）、KILLED（连接被kill）。拿到锁返回成功，其它返回失败

MDL_context::upgrade_shared_lock

锁升级，从共享锁升级到互斥锁，实现方式是重新申请一个目标锁，拿到新的ticket后替换老的ticket，用在alter table和create table场景中。

如create table test.t1(id int) engine = innodb，会先拿test.t1的MDL_SHARED共享锁，检查表是否存在，如果不存在就把锁升级到MDL_EXCLUSIVE锁，然后开始建表。

对于alter table test.t1 add column name varchar(10), algorithm=copy;，alter用copy到临时的方式来做。整个过程中MDL顺序是这样的：

1. 刚开始打开表的时候，用的是 MDL_SHARED_UPGRADABLE 锁；
2. 拷贝到临时表过程中，需要升级到 MDL_SHARED_NO_WRITE 锁，这个时候其它连接可以读，不能更新；
3. 拷贝完在交换表的时候，需要升级到是MDL_EXCLUSIVE，这个时候是禁止读写的。

所以在用copy算法alter表过程中，会有2次锁升级。

MDL_ticket::downgrade_lock 和MDL_context::upgrade_shared_lock对应的锁降级，从互斥锁降级到共享锁，实现比较简单，直接把锁类型改为目标类型（不用重新申请）。

对于alter table test.t1 add column name varchar(10), algorithm=inplace，如果alter使用inplace算法的话，整个过程中MDL加锁顺序是这样的：

1. 和copy算法一样，刚开始打开表的时候，用的是 MDL_SHARED_UPGRADABLE 锁；
2. 在prepare前，升级到MDL_EXCLUSIVE锁；
3. 在prepare后，降级到MDL_SHARED_UPGRADABLE（其它线程可以读写）或者MDL_SHARED_NO_WRITE（其它线程只能读不能写），降级到哪种由表的引擎决定；
4. 在alter结束后，commit前，升级到MDL_EXCLUSIVE锁，然后commit。

可以看到inplace有2次锁升级，1次降级，不过在alter最耗时的阶段是有可能降级到MDL_SHARED_UPGRADABLE的，对其它线程的影响小。

### MDL_request

MDL_request用于描述THD当前SQL的MDL锁请求语义（请求什么对象什么类型多长时间的MDL锁），负责THD → MDL的交互数据，由6元组组成：![MySQL_MDL_request](/MySQL_MDL_request.png)

MDL_request是POD，通过MDL_REQUEST_INIT宏来完成初始化

````
MDL_request mdl_request;
MDL_REQUEST_INIT(&mdl_request,
                 mdl_type, db, name, MDL_EXCLUSIVE, MDL_TRANSACTION);
````

### MDL_ticket

MDL子系统内部对锁请求的表示，可以想象成一张"门票"。

创建点：由MDL_ticket::create()统一创建ticket

````
申请锁时
MDL_context::try_acquire_lock_impl
    ticket= MDL_ticket::create(this, mdl_request->type, mdl_request->duration);
MDL_context::clone_ticket
    ticket= MDL_ticket::create(this, mdl_request->type, mdl_request->duration);
````

也可以复用ticket（MDL_context.find_ticket()）：

1. 查找ticket MDL_context.find_ticket()：遍历当前线程的3个ticket链表，查找当前锁对象（key）是否有>=duration的ticket
2. 如果duration不相同，或者为显式锁（explicit），则clone ticket；否则复用

其中#2，因为duration不同，锁的释放时机不同，这会破坏锁的严格性；explicit为manual release，所以必须单独一个；而可以复用锁的场景更为普遍，如下：

````
START TRANSACTION;
insert into t1 values (1);
insert into t1 values (2);
````

第二条insert不需要再走一遍复杂的加锁逻辑，因为第一条insert已经成功拿到t1表的ticket，类型都是MDL_SHARED_WRITE，并且MDL锁时间范围也一样（transaction），这个时候直接用已有的ticket。

ticket有三个用途：

- 向THD描述锁申请的结果：通过挂在MDL_request上的ticket指针表示，有指向则granted/waiting，没有指向则是没有申请到锁资源
- 作为THD和MDL子系统的桥梁，构建线程的锁等待图，用于死锁检测

其中第二点再展开一下：

要构建锁线程的锁等待图：

1. 连接线程和全局锁对象：作为双方（MDL_context，MDL_lock）的共同链表元素，提供路径可以access
2. 实现锁等待算法：作为MDL_wait_for_subgraph的子类，实现accept_visitor()

内部的数据结构

````
m_ctx*, m_lock*, type, duration
MDL_lock中的2个ticket链表节点(next,prev)
MDL_context中的3个ticket链表节点(next,prev)
 
ctor
dtor
create
destroy
禁止copy
 
has_pending_conflicting_lock        当前ticket的锁类型是否和对应MDL锁的等待队列中的锁冲突
is_upgradable_or_exclusive          是否是可升级锁或者X锁（SU、SNW、SNRW、X）
downgrade_lock                      锁降级（X/SNR -> ???）
has_stronger_or_equal_type          当前ticket对应的锁和指定的锁比较是否更强 granted matrix
is_incompatible_when_granted        是否和granted matrix冲突
is_incompatible_when_waiting        是否和waiting matrix冲突
/** Implement MDL_wait_for_subgraph interface. */
accept_visitor                     
get_deadlock_weight()               获取死锁权重
````

### MDL_lock MDL锁对象

MDL子系统内部用于描述MDL锁对象的表示。

````
m_rwlock        保护MDL_lock锁对象的读写锁（采用读者优先）
key             锁标识（MDL_key）
m_granted       granted ticket链表（Ticket_list）
m_waiting       waiting ticket链表（Ticket_list）
 
Ticket_list     表示当前锁（key）对应的ticket链表，内部类
    add_ticket
    remove_ticket
    bitmap      当前ticket list中的所有锁类型（bitmap），用于快速检测要申请的ticket和grant matrix和waiting matrix是否冲突
 
struct MDL_lock_strategy
 
create
destroy
 
remove_ticket           从MDL_lock中的granted/waiting链表释放指定的ticket
reschedule_waiters      当锁的ticket释放/降级时，从等待队列中选择ticket看能否grant
 
incompatible_granted_types_bitmap
incompatible_waiting_types_bitmap
get_incompatible_waiting_types_bitmap_idx
switch_incompatible_waiting_types_bitmap_if_needed
 
- has_pending_conflicting_lock     // 已经授权的ticket是否和等待队列中的ticket不兼容
- can_grant_lock                   // 能否加锁，先和等待队列进行优先级比较，然后看和已授权的锁是否兼容
 
 
- visit_subgraph                   // 死锁检测相关
- needs_notification               // 是否需要通知其它线程，当前ticket的锁情况
- notify_conflicting_locks         // 通知其它线程，有一个高级的锁请求
needs_connection_check
needs_hton_notification
is_affected_by_max_write_lock_count
count_piglets_and_hogs
get_unobtrusive_lock_increment
is_obtrusive_lock
fast_path_granted_bitmap
 
- hog_lock_types_bitmap            // 标识哪种锁是高级锁
* m_hog_lock_count                 // 高级锁可以连接拿得锁的个数，超过这个数目就要给低级锁让路，防止低级锁饿死
m_piglet_lock_count
m_current_waiting_incompatible_idx
````

### MDL_wait

锁等待的实现类，作为MDL_context的成员变量

````
enum_wait_status       锁等待退出时的状态
timed_wait             锁等待，mutex+cond+timeout
````

### MDL_savepoint

因为explicit锁不会在savepoint rollback释放，所以只需记录THD对应的stmt和trx的ticket point，供MDL_context生成savepoint使用

```
m_stmt_ticket  指向创建savepoint前的最后一个stmt ticket
m_trans_ticket  指向创建savepoint前的最后一个stmt ticket
```

# global read lock

MySQL为了获得全局一致性的点位，通过FTWRL（FLUSH TABLES WITH READ LOCK）来阻止变化：

1. 新的变更进不来（有且只能有一人抢到全局唯一的锁）
2. 已有变更提交不了

做到这样需要：

1. 有且只能有一人抢到全局唯一的锁，并进行范围锁定
2. 两把锁：阻止老的，阻止新的

锁定范围

- GLOBAL+S+EXPLICIT（通过Global_read_lock::lock_global_read_lock）
- COMMIT+S+EXPLICIT（通过Global_read_lock::make_global_read_lock_block_commit）

并清空表缓存（逼迫更新表元信息必须先走open table获取GLOBAL+IX+STMT），以及其他元信息更新入口
阻止元数据修改和表数据更改（S和IX不兼容）：

- 更新表元信息必须先走open table获取GLOBAL+IX+STMT）
- 事务和xa事务提交（数据变更，写入binlog先获取COMMIT+IX+STMT）

实现：

当FLUSH TABLES table_list [WITH READ LOCK]时，Lex->type添加REFRESH_TABLES和REFRESH_READ_LOCK标记

**sql_yacc.cc**

````c++
flush_options:
          table_or_tables
          {
            Lex->type|= REFRESH_TABLES;
            /*
              Set type of metadata and table locks for
              FLUSH TABLES table_list [WITH READ LOCK].
            */
            YYPS->m_lock_type= TL_READ_NO_INSERT;
            YYPS->m_mdl_type= MDL_SHARED_HIGH_PRIO;
 
opt_flush_lock:
          /* empty */ {}
        | WITH READ_SYM LOCK_SYM
          {
            TABLE_LIST *tables= Lex->query_tables;
            Lex->type|= REFRESH_READ_LOCK;
            for (; tables; tables= tables->next_global)
            {
              tables->mdl_request.set_type(MDL_SHARED_NO_WRITE);
              tables->required_type= FRMTYPE_TABLE; /* Don't try to flush views. */
              tables->open_type= OT_BASE_ONLY;      /* Ignore temporary tables. */
            }
          }
````

**sql_parse.cc**

````c++
case SQLCOM_FLUSH:
  reload_acl_and_cache
      thd->global_read_lock.lock_global_read_lock(thd);
      thd->global_read_lock.make_global_read_lock_block_commit(thd))
      flush table cache
````

blocking其他请求

````
Global_read_lock::lock_global_read_lock                 GLOBAL+S+EXPLICIT
Global_read_lock::make_global_read_lock_block_commit    COMMIT+S+EXPLICIT
 
open_table                                              GLOBAL+IX+STMT
ha_commit_trans 事务提交                                  COMMIT+IX+EXPLICIT
xa_transaction xa事务提交
    prepare                                             COMMIT+IX+STMT
    commit                                              COMMIT+IX+STMT
    rollback                                            COMMIT+IX+STMT
````

TODO:

FAST PATH（unobtrusive） OR SLOW PATH（obtrusive）

LF_HASH
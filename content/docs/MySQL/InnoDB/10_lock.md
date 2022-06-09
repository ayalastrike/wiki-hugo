---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: lock
---

在InnoDB中，采用悲观并发控制结合多版本技术，即MV2PL来提供事务的并发控制。并且，通过nextkey locking在RR级别下解决了幻读。最后，InnoDB没有锁升级机制，这得益于其行锁对象及implicit lock的设计。

# 1 锁与事务

我们在前面谈过latch，并且比较过在并发控制上二者的区别，但是，锁的用途不仅仅如此，锁和事务戚戚相关。

## 1.1 隔离性

与锁相关的概念有：

- 并发控制 concurrency control
- 序列化 serializability
- 隔离性 isolation

这里用ACID特性中的隔离性（I）来阐述锁。锁是用来实现事务一致性和隔离性的一种常用技术。

事务的隔离性要求每个读/写事务对象对其他事务的操作可以互相分离，即该事务提交前对其他事务不可见。

{{< hint info >}}

First Law of Concurrency Control ：

Concurrent execution should not cause application programs to malfunction.

{{</hint>}}

这也是事务隔离性的要求，即当数据库系统中的事务并发运行时，每个事务的运行不会受到其他事务的影响，好像每个并发事务都是"单线程"的在运行。

实现隔离性有许多种方式，最为广泛使用的就是加锁（locking）技术。不过，即使采用加锁技术，仍然可以有多种实现方式，因此就有了并发控制的第二准则：

{{< hint info >}}

Second Law of Concurrency Control ：

Concurrent execution should not have lower throughput or much higher response times than serial execution.

{{</hint>}}

如果采用加锁技术实现的数据库并发系统的影响时间大于串行运行的方式，那么这也不是一种能被接受的加锁方式，即并发控制的第二准则要求一种简单的算法或者开销较小的方式来实现加锁技术。

最简单的加锁技术是对每个要访问的对象加上一个锁。当事务访问一个对象，数据库自动请求并加上一个锁，在事务结束后释放该锁。若请求时该对象上已经被其他事务持有锁，则该事务等待对象上锁的释放。由此可见，锁提供了一种串行机制，用来保证同一时刻一个对象仅能被一个事务访问。

通过多粒度（fine granular）锁可以用来提高数据库系统的并发性。比如，可以让不同事务访问同一页中的不同记录，从而提高数据库的并发度。

InnoDB中锁的实现和Oracle数据库非常类似，提供一致性的非锁定读、行级锁支持，但InnoDB中的行级锁没有相关额外的开销，可以在保证数据一致性的基础上同时获得较高的并发性。

## 1.2 事务的隔离级别

在DBMS发展初期，大部分的数据库系统都没有提供真正的隔离性，最初是因为系统的设计者并没有真正理解这些问题。现在各种读写异常的场景已经清晰，但DBMS需要在正确性和性能之间做出妥协。ISO和ANSI [SQL标准](http://ayalastrike.f3322.net:8090/pages/viewpage.action?pageId=28737556)制定了四种事务隔离级别的标准，但是很少有数据库厂商遵循这些标准，比如Oracle数据库就不支持READ UNCOMMITTED和REPEATABLE READ的事务隔离级别。

SQL标准定义的四个隔离级别为：

- READ UNCOMMITTED
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE

READ UNCOMMITTED称为浏览访问（browse access），仅仅对于只读事务而言的。

READ COMMITTED称为游标稳定（cursor stability）。

REPEATABLE READ是2.9999的隔离，没有幻读的保护。

SERIALIZABLE称为隔离，或3的隔离。

SQL和SQL2标准的默认事务隔离级别是SERIALIZABLE。

InnoDB存储引擎默认的隔离级别是REPEATABLE READ，但是与标准SQL不同的是，InnoDB存储引擎在REPEATABLE READ事务隔离级别下，使用next-key locking的锁算法，可以避免幻读的产生。这与其他数据库系统（比如Microsoft SQL Server数据库）是不同的。所以，InnoDB存储引擎在默认的REPEATABLE READ的事务隔离级别下已经能完全保证事务的隔离性要求，即达到SQL标准的SERIALIZABLE隔离级别。

从理论上说，隔离级别越低，事务请求的锁越少或者保持锁的时间就越短。这也是为什么大多数数据库系统默认的事务隔离级别是READ COMMITTED的缘故。

此外，大家会质疑SERIALIZABLE隔离级别带来的性能问题，但是根据Jim Gray在Transaction Processing：Concepts and Techniques一书中指出，两者的开销几乎是一样的，甚至SERIALIZABLE可能更优！！！因此在InnoDB存储引擎中选择REPEATABLE READ的事务隔离级别并不会有任何性能的损失。同样的，即使使用READ COMMITTED的隔离级别，性能也不会有大幅度的提升。

## 1.3 幻读

解决幻读是通过一种称为谓词锁（predict lock）的方法，其锁住的不是单个记录，而是一个"条件"。通常来说，一个事务发出一个锁请求如下所示：

<t, [xlock | slock], predict>

谓词锁存在一些性能上的问题，而key-range locking算法是谓词锁的一种改进实现。其锁定的不是“条件”，而是范围。根据锁定的边界不同，又可以分为next-key locking和previous-key locking。假设有记录W、Y、Z，则根据next-key locking算法，可将其锁定的范围可有：

(-∞，W]，(W，Y]，(Y，Z]，(Z，+∞)

若插入记录X，则可锁定的范围变为：

(-∞，W]，(W，X]，(X，Y]，(Y，Z]，(Z，+∞)

因此在next-key locking算法下，其锁定的不是记录，而是一个范围。例如锁定记录Y其实锁定的是 (W，Y] 这个范围，当这个范围被锁定时会阻止其他事务向这个范围内插入新的记录，从而避免了幻读问题的产生。

# 2. 锁的设计

## 2.1 锁模式、类型

InnoDB实现了如下两种标准的行级锁：

- 共享锁（S Lock），允许事务读一行数据
- 排他锁（X Lock），允许事务删除或者更新一行数据

但是，在数据库中，数据对象本身存在层次关系（如下图所示），如果只通过行锁来控制page、table以及database，一是锁对象多（本身资源宝贵），二是组合语义会导致设计复杂。为此，需要提供一套抽象的并发控制语义，即多粒度锁。

![InnoDB_lock_granularity](/InnoDB_lock_granularity.png)

在InnoDB中，为了支持多粒度锁定，提供了表锁和行锁两个粒度，允许同一事务在行锁和表锁同时存在。为了支持在不同粒度上进行加锁操作，InnoDB支持意向锁（Intention Lock）。意向锁将锁定的对象分为多个层次，意味着事务希望在更细粒度（fine granularity）上进行加锁。

如果将上锁的对象看成一棵树的话，如果对最下层的对象上锁，也就是对最细粒度的对象进行上锁，那么首先需要对粗粒度的对象上锁。如上图所示，需要对页上的记录record上x-lock，那么首先需要依次对数据库database、表table、页page加ix-lock，最后才对记录record加x-lock。如果在这个过程中的任何一步发生等待，那么该操作需要等待粗粒度锁的完成。举例来说，在对记录record1加x-lock之前，如果已经有其他事务对table1加了s-lock，而当前事务需要对table1加ix-lock，由于ix和s不相容，所以当前事务需要等待其他事务的表锁释放后才能继续推进。

InnoDB的意向锁设计为表级别，即在一个事务中揭示下一行将被请求的锁类型，共有两种意向锁：

- 意向共享锁（IS Lock），事务想要获得一张表中某几行的共享锁
- 意向排他锁（IX Lock），事务想要获得一张表中某几行的排他锁

意向锁不会阻塞除全表扫描以外的任何请求，锁兼容性列表如下：

~~~~
/* LOCK COMPATIBILITY MATRIX
 *    IS IX S  X
 * IS +  +  +  -
 * IX +  +  -  -
 * S  +  -  +  -
 * X  -  -  -  -
~~~~

从上面的表格中可以看出，意向锁互相之间完全兼容，S锁和S以及IS锁兼容，X锁和所有意向锁不兼容。

在行锁上，InnoDB提供3种行锁算法：

- record lock：单个索引记录上的锁
- gap lock：间隙锁，锁定一个范围，但不包括记录本身
- next-key lock：gap lock + record lock，锁定一个范围+记录本身

InnoDB的行锁实质上是一个[索引记录锁](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)（the row-level locks are actually index-record locks），锁定的是索引上的记录。此外，根据锁的实现方式不同，还可以分为隐式锁（implicit lock）和显式锁（explicit lock）。

## 2.2 锁对象

锁对象细分为行锁和表锁。

### 2.2.1 行锁对象

行锁在InnoDB中使用的最为频繁，因此，InnoDB在行锁的设计上尽量减少开销（通过page粒度来构造锁对象，并通过隐式锁，infimum记录锁来降低锁的开销），因此不需要支持锁升级。行锁的数据结构用lock_rec_t表示：

~~~~
/** Record lock for a page */
struct lock_rec_t {
    ib_uint32_t space;      // space id
    ib_uint32_t page_no;    // page no
    ib_uint32_t n_bits;     // bitmap位数
    // 实际的bitmap
};
~~~~

从上面可以看到，InnoDB中的行锁是以page为单位来进行组织管理的，page中的所有行的锁信息都以位图（bitmap）的方式表示，位图的索引与记录的heap no一一对应，这也是heap no顺序分配的原因，以保证不变性。并且，lock bitmap是根据页中的记录数量来进行分配内存空间的，所以不显式地对其进行定义。变量n_bits表示需要bitmap占用多少位以用于管理。由于page中的记录可能在之后还会进行增加，因此这里额外预留LOCK_PAGE_BITMAP_MARGIN（64）个记录的位图信息，以便在少量记录增长的情况下不需要重新分配bitmap。

bitmap的大小

~~~~
/* Safety margin when creating a new record lock: this many extra records
can be inserted to the page without need to create a lock with a bigger
bitmap */
static const ulint  LOCK_PAGE_BITMAP_MARGIN = 64;
 
/**
Create record locks */
class RecLock {
    /**
    Calculate the record lock physical size required, non-predicate lock.
    @param[in] page     For non-predicate locks the buffer page
    @return the size of the lock data structure required in bytes */
    static size_t lock_size(const page_t* page)
    {
        ulint   n_recs = page_dir_get_n_heap(page);
 
        /* Make lock bitmap bigger by a safety margin */
 
        return(1 + ((n_recs + LOCK_PAGE_BITMAP_MARGIN) / 8));
    }
};
~~~~

比如page(20, 100)有250条记录，则n_bit=250+64=314，那么实际位图需要额外40个字节用于位图的管理（n_bytes=1+314/8=40）。如下图所示，页中heap_no为2、4、5的记录都已经上锁：

![InnoDB_lock_lock_rec_t](/InnoDB_lock_lock_rec_t.png)

根据page粒度来组织行锁是一个高效的设计，这样可以极大的减少行锁的开销。因为锁是稀有资源，在传统的数据库系统设计中，如果事务占用太多的锁资源，会采用锁升级（lock escalation）的方式，将大量的细粒度锁升级为更粗粒度的锁（页锁/表锁），而在InnoDB中，这样的设计避免了锁升级。

### 2.2.2 表锁对象

InnoDB的表锁分为意向锁和自增锁，用lock_table_t表示：

~~~~
/** A table lock */
struct lock_table_t {
    dict_table_t*           table;  // DD中的table
    UT_LIST_NODE_T(lock_t)  locks;  // 同一个table上的lock列表
};
~~~~

通常情况下，如果要在行记录上加行锁，首先需要在相应的表上加IX意向锁：

~~~~
lock_rec_lock(
{
...
    ut_ad((LOCK_MODE_MASK & mode) != LOCK_S
          || lock_table_has(thr_get_trx(thr), index->table, LOCK_IS));
    ut_ad((LOCK_MODE_MASK & mode) != LOCK_X
          || lock_table_has(thr_get_trx(thr), index->table, LOCK_IX));
...
}
~~~~

## 2.3 锁子系统

锁是在事务的推进过程中产生的，因此也需要在每个事务中描述相应的锁对象（行锁/表锁），各个事务统一由事务子系统进行管理；同时，锁对象也统一由锁子系统进行管理。因此，InnoDB抽象了一个锁对象（lock_t），来描述事务-锁的对应关系、锁对象的具化（行锁/表锁）、行锁对象所对应的索引、行锁/表锁在全局系统中的元信息，以及具体的锁元信息（type_mode）：

~~~~
/** Lock struct; protected by lock_sys->mutex */
struct lock_t {
    trx_t*                  trx;        // 该锁对象对应的事务
    UT_LIST_NODE_T(lock_t)  trx_locks;  // 活跃事务的锁链表的节点
    dict_index_t*           index;      // 行锁所对应的索引
    lock_t*                 hash;       // 全局锁列表，对应lock_sys->rec_hash
 
    union {                             // 具体的锁对象
        lock_table_t        tab_lock;   // 表锁
        lock_rec_t          rec_lock;   // 行锁
    } un_member;
 
    ib_uint32_t type_mode;              // 具体的锁的辕信息，锁模式&锁类型&行锁类型， lock mode、type, LOCK_GAP or LOCK_REC_NOT_GAP, LOCK_INSERT_INTENTION, wait flag, ORed
};
~~~~

由锁子系统（log_sys_t）来提供全局锁视角：

~~~~
/** The lock system struct */
struct lock_sys_t{
    hash_table_t*   rec_hash;       // 全局行锁哈希表
    hash_table_t*   prdt_hash;      // 全局gis谓词锁哈希表（gis）
    hash_table_t*   prdt_page_hash; // 全局page谓词锁哈希表（gis）
    ...
};
~~~~

锁子系统的初始化：

~~~~
srv_lock_table_size = 5 * (srv_buf_pool_size / UNIV_PAGE_SIZE); // hash cell为5倍buffer pool的页数
 
innobase_start_or_create_for_mysql
    lock_sys_create(srv_lock_table_size);
~~~~

在进行全局的行锁查询时，首先通过space_id+page_no（block→lock_hash_val）在log_sys_t→rec_hash中找到page粒度的行锁对象rec_lock_t，然后根据heap no和n_bits在bitmap中判断某行是否已加锁。

从上面可以看到，对于每个锁对象（lock_t），存在两个维度：

- 事务维度，每个事务都有获得的锁，以及等待中的锁（trx→lock→trx_locks/table_locks，trx→wait_lock）
- 全局维度：所有行锁都保存在锁子系统的哈希表中（lock_sys→rec_hash），lock_rec_t中的（space_id+page_no）保证了同一页的记录锁都会被hash到同一个桶中，表锁在DD的表对象中（dict_table→locks） 

锁相关的数据结构关系如下图所示：

![InnoDB_lock_organization](/InnoDB_lock_organization.png)

## 2.4 锁的元信息

锁的元信息存储在lock_t.type_mode中，包括lock_mode、lock_type、record_lock type三类，如下图所示：

![InnoDB_lock_lock_t_type_mode](/InnoDB_lock_lock_t_type_mode.png)

从后往前看，第1-4位表示锁的模式（lock_mode）：

~~~~
#define LOCK_MODE_MASK  0xFUL   /*!< mask used to extract mode from the type_mode field in a lock */
 
/* Basic lock modes */
enum lock_mode {
    LOCK_IS = 0,            // 意向共享锁
    LOCK_IX,                // 意向排他锁
    LOCK_S,                 // 共享锁
    LOCK_X,                 // 排他锁
    LOCK_AUTO_INC,          // 自增锁（表级排他锁）
    LOCK_NONE,              // consistent read
    LOCK_NUM = LOCK_NONE,   // number of lock modes
    LOCK_NONE_UNSET = 255
};
~~~~

锁的模式使用方式为：

- 行锁加S/X lock
- 表锁一般加IS/IX lock，LOCK TABLE加S/X lock
- <font color=red>在使用SBR binlog格式时，加AI lock</font>

锁的类型（lock_type）细分为行锁和表锁，用第5-8位表示（只使用了2位）：

~~~~
#define LOCK_TYPE_MASK  0xF0UL  // 用于从type_mode中获取锁类型
 
/** Lock types */
#define LOCK_TABLE          16  // 表锁
#define LOCK_REC            32  // 行锁
~~~~

以上的元信息通过宏LOCK_TYPE_MASK、LOCK_MODE_MASK提取对应锁的类型和模式。

最后一部分是锁属性，包括行锁的锁定范围、插入意向锁和谓词属性另外，在行锁的设计上，还需要通过行锁的锁定范围来保证事务的隔离性。因此，具体的行锁锁定范围算法由以下元信息描述(record_lock type)：

~~~~
#define LOCK_WAIT           256     // 处于等待中
/* Precise modes */
#define LOCK_ORDINARY       0       // next-key lock，锁住记录和记录之前的gap
#define LOCK_GAP            512     // 锁住记录之前的gap（不锁记录，锁间隙）
    /*!< when this bit is set, it means that the
                lock holds only on the gap before the record;
                for instance, an x-lock on the gap does not
                give permission to modify the record on which
                the bit is set; locks of this type are created
                when records are removed from the index chain
                of records */
#define LOCK_REC_NOT_GAP    1024    // 锁住记录，不锁记录前面的gap
    /*!< this bit means that the lock is only on
                the index record and does NOT block inserts
                to the gap before the index record; this is
                used in the case when we retrieve a record
                with a unique key, and is also used in
                locking plain SELECTs (not part of UPDATE
                or DELETE) when the user has set the READ
                COMMITTED isolation level */
#define LOCK_INSERT_INTENTION 2048  // 插入意向锁
 /*!< this bit is set when we place a waiting
                gap type record lock request in order to let
                an insert of an index record to wait until
                there are no conflicting locks by other
                transactions on the gap; note that this flag
                remains set when the waiting lock is granted,
                or if the lock is inherited to a neighboring
                record */
#define LOCK_PREDICATE  8192        // 谓词锁，用于gis索引
#define LOCK_PRDT_PAGE  16384       // page上的谓词锁，用于gis索引
~~~~





举个例子，比如有记录：2，4，6，8，10。在next-key locking算法下，如果用户锁住8这条记录，那么其实锁住的是一个范围，即（6，8]（即LOCK_ORDINARY）。如果在锁住记录8的时候，同时将type_mode设置为LOCK_GAP，则这时表示锁住的范围为（6，8），并没有锁8这条记录。

对于插入操作，需要判断下一个记录是否有锁，但是锁的类型为LOCK_GAP即可。因为不需要对插入记录的下一条记录进行加锁，这样就提高了数据库的并发性。此外，对于supremum记录，其总是为LOCK_GAP，因为其是伪记录，表示一个页中最大的记录，亦可理解该值为正无穷，锁区间为（最后一个用户记录, +∞）。

## 2.5 锁的兼容性

InnoDB相对于传统的数据库系统，锁矩阵中增加了自增锁（auto-increment lock，AI）。AI和自身、X/S不兼容，和IS、IX兼容。

锁兼容矩阵和锁强弱矩阵如下：

~~~~
/* LOCK COMPATIBILITY MATRIX
 *    IS IX S  X  AI
 * IS +  +  +  -  +
 * IX +  +  -  -  +
 * S  +  -  +  -  -
 * X  -  -  -  -  -
 * AI +  +  -  -  -
 
/* STRONGER-OR-EQUAL RELATION (mode1=row, mode2=column)
 *    IS IX S  X  AI
 * IS +  -  -  -  -
 * IX +  +  -  -  -
 * S  +  -  +  -  -
 * X  +  +  +  +  +
 * AI -  -  -  -  +
~~~~

锁的兼容矩阵表示的是"能不能"加，用于判断两个事务之间是否存在冲突，如果存在冲突，则需要等待。而锁的强弱矩阵表示的是"需不需要"加，用于在一个事务中判断如果已持有锁，是否还需要升级锁的强度。

## 2.6 locking和MVCC

InnoDB采用MVCC+SS2PL的方式综合进行事务的并发控制，即locking保证W-W的并发控制，R-W通过MVCC来实现读写之间的non-blocking。

各个DBMS的MVCC Protocol如下：

|  Vendor   |   Protocol   | Version Storage | Garbage Collection | Indexes  |
| :-------: | :----------: | :-------------: | :----------------: | :------- |
|  Vendor   |   Protocol   | Version Storage | Garbage Collection | Indexes  |
|  Oracle   |    MV-2PL    |      Delta      |       Vacuum       | Logical  |
| Postgres  | MV-2PL/MV-TO |   Append-Only   |       Vacuum       | Physical |
|  InnoDB   |    MV-2PL    |      Delta      |       Vacuum       | Logical  |
|  HYRISE   |    MV-OCC    |   Append-Only   |         -          | Physical |
|  Hekaton  |    MV-OCC    |   Append-Only   |    Cooperative     | Physical |
|  MemSQL   |    MV-OCC    |   Append-Only   |       Vacuum       | Physical |
| SAP HANA  |    MV-2PL    |   Time-travel   |       Hybrid       | Logical  |
|   NuoDB   |    MV-2PL    |   Append-Only   |       Vacuum       | Logical  |
|   HyPer   |    MV-OCC    |      Delta      |     Txn-level      |          |
| CMU's TDB |    MV-OCC    |      Delta      |     Txn-level      |          |

# 3. 显式锁和隐式锁

## 3.1 显式锁和隐式锁的区别

在InnoDB中，行锁的实现方式，可以有explicit lock（显式锁）和implicit lock（隐式锁）两种，区别如下：

显式锁（explicit lock）很直接，就是实际有锁。而隐式锁（implicit lock）指的是逻辑上索引记录有X-lock，但在实际的内存对象中并不含有锁信息，这也就意味着implicit lock没有任何内存开销，从而进一步减少了InnoDB的锁开销。

在InnoDB中，行锁的实现方式，可以有explicit lock（显式锁）和implicit lock（隐式锁）两种，区别如下：

| 锁属性               | 锁标志               | 锁范围                                                       | 锁模式 |
| :------------------- | :------------------- | :----------------------------------------------------------- | :----- |
| 显式锁 explicit lock | gap explicit lock    | (  ) （只锁范围，通过LOCK_GAP设置）                          | X-lock |
|                      | no gap explicit lock | (  ] （锁范围+记录，通过LOCK_ORDINARY设置）rec （锁记录，通过LOCK_REC_NOT_GAP设置） | S-lock |
| 隐式锁 implicit lock | /                    |                                                              | X-lock |

下面举一个具体的例子：

~~~~
create table t (
    a int not null,
    b int,
    c int,
    primary key(a),
    key(b)
) engine=InnoDB;
 
insert into t values (1,2,3);
 
session 2
begin;
update t set b = b + 1 where a = 1;
 
set global innodb_status_output_locks = on;
 
---TRANSACTION 1819, ACTIVE 698 sec
2 lock struct(s), heap size 1160, 1 row lock(s), undo log entries 1
MySQL thread id 2, OS thread handle 140688668358400, query id 19 localhost root
TABLE LOCK table `tjw`.`t` trx id 1819 lock mode IX
RECORD LOCKS space id 28 page no 3 n bits 72 index PRIMARY of table `tjw`.`t` trx id 1819 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;           // PK (a)
 1: len 6; hex 00000000071b; asc       ;;     // trx id = 1819
 2: len 7; hex 340000013901f1; asc 4   9  ;;  // rollback pointer
 3: len 4; hex 80000003; asc     ;;           // b
 4: len 4; hex 80000003; asc     ;;           // c
~~~~

这个例子中，session 2显式开启了事务，然后通过查看锁信息得知：

active transaction ID = 1819

获取了两个explicit lock锁：

- 表锁 tjw.t IX
- 行锁 page (29, 3) 上，锁了heap_no = 2的记录（where a = 1），锁模式为rec+X

并打印出了行记录的所有详细信息。

从这里可以看到，secondary index (b)上没有explicit lock。所以，DML并不一定会产生explicit lock，如果加锁需要等待，则产生explicit lock锁对象，因为只有这样才能进行granted lock所释放后的wait-notify唤醒逻辑，如果不需要等待，则可能采用implicit lock。由于implicit lock的存在，在并发下情况下，对索引记录需要将implicit lock转化为explicit lock（lock_rec_convert_impl_to_expl），并将锁信息插入到全局变量lock_sys的哈希表中，以便进行锁的并发控制。

之前已经提到，行锁本质上是索引记录锁。那么当锁定一行clustered index记录时，若该记录上还有secondary index，根据谓词锁的要求还应该对相应secondary index上的记录进行加锁。因此，implicit lock既可以存在于clustered index记录中，也可以存在于secondary index记录中。

比如：

- 对于clustered index记录，例如用户插入了一个row id为4的新记录，但事务还未提交。这时row id为4的记录就包含一个implicit lock。但是在全局锁信息（lock_sys）中查询不到此新记录的锁，因此这个锁是隐式的。
- 对于secondary index记录，例如对row id为4的clustered index记录进行了更改，并且更改的列是secondary index的列，那么在该辅助索引上同样含有一个隐式锁。

那既然是implicit lock，在事务子系统和锁子系统上无据可查，那如何判断是否有implicit lock存在呢？

## 3.2 clustered index记录的隐式锁

clustered index记录中implicit lock的判断较为简单。因为每个clustered index记录都有一个trx ID的隐藏列，只需要通过该trx ID判断当前其是否为活跃事务就能得知是否有implicit lock。如果是活跃事务，则此clustered index记录上有implicit lock；反之则不含有implicit lock。

## 3.3 secondary index记录的隐式锁

secondary index记录中不含有trx ID的隐藏列，因此判断辅助索引记录上是否有implicit lock复杂得多。我们在之前的page一章提到过，每个seconary index page的page header都有一个PAGE_MAX_TRX_ID，其保存了一个最大事务ID，每当该secondary index page中的任何记录被更新后，都需要更新PAGE_MAX_TRX_ID。因此，secondary index记录的implicit lock判断分为两个步骤：

- 根据secondary index page的PAGE_MAX_TRX_ID进行判断
- 根据clustered index记录的trx ID进行判断

因为每次修改secondary index的记录，都需要相应的修改其页上的PAGE_MAX_TRX_ID，因此，如果当前的PAGE_MAX_TRX_ID < 当前活跃事务的最小ID，则此secondary index记录不含有implicit lock，也就是之前已经提交的事务修改了该记录。而如果>=，则存在以下可能：                                                                     

- 存在活跃事务，修改了此secondary index记录（正在修改了自己）
- 存在事务（活跃/已经提交），修改了页中的其他secondary index记录（正在/已经修改他人）

因此，对于>=的情况，需要通过此secondary index记录对应的clustered index记录来判断其是否含有implicit lock（row_vers_impl_x_locked），补充函数调用细节。

这个本质上是因为secondary index上只有page粒度的模糊事务信息，只能进一步通过详细事务信息+rec进行判断。

判断secondary index记录是否持有implicit lock，只须判断是否存在未提交的活跃事务对记录进行INSERT、DELETE、UPDATE的操作。若有，则必然持有implicit lock，并返回该活跃事务对象trx_t（通过clustered index记录中的trx id隐藏列来获得）。

我们以图示的方式来说明row_vers_impl_x_locked的判断流程：

![InnoDB_lock_secondary_index_record_implicit_lock_overview](/InnoDB_lock_secondary_index_record_implicit_lock_overview.png)

1. trx_id对应的事务为不活跃的事务，则secondary index记录rec不含有implicit lock（其他已提交的事务insert记录）
2. prev_version = NULL，表示没有之前版本的记录，即其为当前事务插入的记录，则secondary index记录rec含有implicit lock。（当前事务insert记录）
3. rec == entry，两个版本的secondary index记录相等，但是两个记录的delete flag位不同，则表示某活跃事务删除了记录，因此secondary index记录rec含有implicit lock。（当前事务delete记录）
4. rec != entry，两个版本的secondary index不相同，且记录rec的delete flag为0，表示某活跃事务更新了secondary index记录，因此secondary index记录rec含有implicit lock。（当前事务update记录）
5. rec == entry且两个记录的delete flag位相同，则既可能是当前某活跃事务修改了secondary index记录rec，也可能是之前已提交的事务修改了secondary index记录rec。如果trx_id != prev_id，则表示之前的事务已经修改了记录，因此记录rec上不含有implicit lock。否则，需要通过再之前的记录版本进行判断。（其他事务可能修改过）（当前事务、其他事务update记录）
6. rec != entry且rec的delete flag位为1，则既可能是当前某活跃事务修改了secondary index记录rec，也可能是之前已提交的事务修改了secondary index记录rec。如果trx_id != prev_id，则表示之前的事务已经修改了记录，因此记录rec上不含有implicit lock。否则，需要通过再之前的记录版本进行判断。（当前事务、其他事务delete记录）

总结一下，隐式锁的判断逻辑整体如下：

![InnoDB_lock_secondary_index_record_implicit_lock](/InnoDB_lock_secondary_index_record_implicit_lock.png)

我们来看几个具体的例子加深理解：

~~~~
create table t (
    a int not null,
    b int not null,
    c int not null,
    primary key(a),
    index (b)
) engine = InnoDB;
 
insert into t values ((1,2,3); // 事务已提交
~~~~

**case1**

~~~~
select b from t where b = 2 for update;
~~~~

此时secondary index记录的状态如下：

![InnoDB_lock_secondary_index_record_implicit_lock_1](/InnoDB_lock_secondary_index_record_implicit_lock_1.png)

此时trx_id的事务是不活跃的（insert已提交），所以b=2的secondary index记录不含有implicit lock，不会阻塞session A。

**case 2**

| session A                             | session B                                       |
| :------------------------------------ | :---------------------------------------------- |
| begin;insert into t vlaues (2, 3, 4); |                                                 |
|                                       | select b from t where b = 3 for update;（阻塞） |

此时secondary index记录的状态如下：

![InnoDB_lock_secondary_index_record_implicit_lock_2](/InnoDB_lock_secondary_index_record_implicit_lock_2.png)

从上图中可以看到，记录b对应的clustered index记录为(2, 3, 4)，而其事务A是活跃的（未提交），通过clustered index记录构造之前的版本记录是NULL，因此b=3的secondary index记录含有implicit lock，会阻塞session B。

**case 3**

| session A                        | session B                                       |
| :------------------------------- | :---------------------------------------------- |
| begin;delete from t where a = 1; |                                                 |
|                                  | select b from t where b = 2 for update;（阻塞） |

此时secondary index记录的状态如下：

![InnoDB_lock_secondary_index_record_implicit_lock_3](/InnoDB_lock_secondary_index_record_implicit_lock_3.png)

从上图中可以看到，当前记录（b=2）和之前版本的secondary index记录（b=2）的值都是相等的，不同的是二者的delete marker不一样，因此b=2的secondary index记录含有implicit lock，会阻塞session B。

**case 4**

| session A                             | session B                                       |
| :------------------------------------ | :---------------------------------------------- |
| begin;update t set b = 4 where a = 1; |                                                 |
|                                       | select b from t where b = 4 for update;（阻塞） |

此时secondary index记录的状态如下：

![InnoDB_lock_secondary_index_record_implicit_lock_4](/InnoDB_lock_secondary_index_record_implicit_lock_4.png)

从上图中可以看到，当前记录（b=4）和之前版本的secondary index记录（b=2）的值不同，并且当前记录的delete marker=0，所以b=4的secondary index记录含有implicit lock，会阻塞session B。

**case 5**

| session A                                                    | session B                                       |
| :----------------------------------------------------------- | :---------------------------------------------- |
| begin;update t set b = 4 where a = 1;update t set c = 4 where a = 1; |                                                 |
|                                                              | select b from t where b = 4 for update;（阻塞） |

此时secondary index记录的状态如下：

![InnoDB_lock_secondary_index_record_implicit_lock_5](/InnoDB_lock_secondary_index_record_implicit_lock_5.png)

从上图中可以看到，当前记录（b=4）和之前第一个版本的secondary index记录（b=4）的值相同，并且delete marker也相同 ，需要遍历之前的第二个版本。此时出现case 4，所以b=4的secondary index记录含有implicit lock，会阻塞session B。

**case 6**

| session A                                                    | session B                                       |
| :----------------------------------------------------------- | :---------------------------------------------- |
| begin;update t set b = 4 where a = 1;update t set b = 2 where a = 1;delete from t where a = 1; |                                                 |
|                                                              | select b from t where b = 4 for update;（阻塞） |

此时secondary index记录的状态如下：

![InnoDB_lock_secondary_index_record_implicit_lock_6](/InnoDB_lock_secondary_index_record_implicit_lock_6.png)

从上图中可以看到，当前记录（b=4）和之前第一个版本的secondary index记录（b=2）的值不同，并且delete marker都=1，而这两个记录的trx id相同，需要遍历之前的第二个版本。此时出现case 3，所以b=4的secondary index记录含有implicit lock，会阻塞session B。

# 4. 表锁

前面提到，InnoDB的锁粒度分为表锁和行锁，所以加锁操作也分为这两类介绍：

- 表锁：通过函数lock_table完成，表加锁是根据事务和表来进行的。
- 行锁：通过函数lock_rec_lock完成，行记录加锁根据事务和页来进行。这里要注意两点：implicit lock & 锁对象lock_t的重用，下面详述。

首先从表锁开始。

## 4.1 表锁的模式

表锁的目的是为了防止DDL和DML的并发问题，但在MySQL 5.5引入server层的MDL锁后，InnoDB层的表锁意义就没有那么大了，MDL锁本身已经覆盖了其大部分功能。

表锁支持所有的lock_mode：

### 4.1.1 IX/IS 表锁

意向锁，表示"暗示”未来需要什么样的行锁，意向锁之间不冲突，但和S/X表锁冲突（表示DML，DDL互斥语义）。

### 4.1.2 X 表锁

当加了X表锁后，所有其他的表锁请求都需要等待。在以下场景下需要对表加X锁：

- DDL操作的最后一个阶段（ha_innobase::commit_inplace_alter_table）会对表加LOCK_X锁，以确保没有其他事务持有表级锁。一般情况下，server层的MDL锁已经能保证这一点了，其在DDL的commit 阶段已经加了排他的MDL锁。但是，在外键检查或者在crash recovery时，则需要这个保证。因为这些操作都是InnoDB自治的，不走server层，也就无法通过MDL锁保护。
- 会话的autocommit = 0，执行LOCK TABLE tablename WRITE会对表加X锁（ha_innobase::external_lock）。
- 对表空间执行维护时（discard/import），会对表加X锁（ha_innobase::discard_or_import_tablespace）。

### 4.1.3 S 表锁

在以下场景下需要对表加S锁：

- 在DDL的第一个阶段，如果当前DDL不能通过ONLINE的方式执行，则对表加S锁（prepare_inplace_alter_table_dict）。
- 会话的autocommit = 0，执行LOCK TABLE tablename READ会对表加S锁（ha_innobase::external_lock）。

从上面我们可以看出，对表加X/S的场景并不常见，加的更多的还是互相不冲突的IX/IS意向锁。

### 4.1.4 AUTO_INC 表锁

自增列在数据库中是一种常见的属性，也是首选的主键方式。在通常情况下，锁在事务提交后才会释放（SS2PL），但是自增锁如果也这么做会极大的影响插入性能，因此，InnoDB中的自增锁在插入操作完成后立即释放（STMT），以提高并发插入的性能。但是，如果单条语句执行长时间的插入操作（insert into select，load data），并发插入的性能还是会受到影响。

另外，自增锁的增长是以表为单位的，所以自增锁设计为一个表锁，这样，每张表只能有一个自增锁，锁兼容矩阵如下：

~~~~
/* LOCK COMPATIBILITY MATRIX
 *    IS IX S  X  AI
 * AI +  +  -  -  -
~~~~

从上面可以看出，自增锁和自身、S、X锁都是不兼容的。因此，在一个插入操作正在进行的时候，是不允许除意向锁之外的其他加锁操作的。

每个表的自增值并不持久化存储到磁盘上，而是每次在InnoDB启动时通过以下SQL读取后保存到内存的表对象（dict_table_t）中：

~~~~
select MAX(auto_inc_column) from t for update;
~~~~

内存表对象dict_table_t中关于自增的变量有：

~~~~
/** Data structure for a database table.  Most fields will be
initialized to 0, NULL or FALSE in dict_mem_table_create(). */
struct dict_table_t {
 
    /** AUTOINC related members. @{ */
 
    /* The actual collection of tables locked during AUTOINC read/write is
    kept in trx_t. In order to quickly determine whether a transaction has
    locked the AUTOINC lock we keep a pointer to the transaction here in
    the 'autoinc_trx' member. This is to avoid acquiring the
    lock_sys_t::mutex and scanning the vector in trx_t.
    When an AUTOINC lock has to wait, the corresponding lock instance is
    created on the trx lock heap rather than use the pre-allocated instance
    in autoinc_lock below. */
 
    /** A buffer for an AUTOINC lock for this table. We allocate the
    memory here so that individual transactions can get it and release it
    without a need to allocate space from the lock heap of the trx:
    otherwise the lock heap would grow rapidly if we do a large insert
    from a select. */
    lock_t*                 autoinc_lock;
 
    /** Creation state of autoinc_mutex member */
    volatile os_once::state_t       autoinc_mutex_created;
 
    /** Mutex protecting the autoincrement counter. */
    ib_mutex_t*             autoinc_mutex;
 
    /** Autoinc counter value to give to the next inserted row. */
    ib_uint64_t             autoinc;
 
    /** This counter is used to track the number of granted and pending
    autoinc locks on this table. This value is set after acquiring the
    lock_sys_t::mutex but we peek the contents to determine whether other
    transactions have acquired the AUTOINC lock or not. Of course only one
    transaction can be granted the lock but there can be multiple
    waiters. */
    ulong                   n_waiting_or_granted_auto_inc_locks;
 
    /** The transaction that currently holds the the AUTOINC lock on this
    table. Protected by lock_sys->mutex. */
    const trx_t*                autoinc_trx;
}
~~~~

autoinc_lock用来作为自增锁的缓存，因为每张表在同一时刻只能有一个自增锁，所以autoinc_lock可以避免同一表锁对象（自增锁对象）在各个事务中不断的被申请。

autoinc为表的自增值，为8个字节。

autoinc_mutex用来保护autoinc（似乎没有必要，因为自增锁为表锁，且互相不兼容）。

事务对象trx_t也有自增相关的变量：

~~~~
struct trx_t {
    ulint           n_autoinc_rows; /*!< no. of AUTO-INC rows required for an SQL statement. This is useful for multi-row INSERTs */
    ib_vector_t*    autoinc_locks;  // 该事务所持有的所有AI锁，这些锁也存储在trx_locks中。
}
~~~~

自增值的产生和自增锁的控制通过[innodb_autoinc_lock_mode](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_autoinc_lock_mode)设置（ha_innobase::innobase_lock_autoinc），一共有3种模式：

**AUTOINC_OLD_STYLE_LOCKING（0）**

传统加锁模式（在MySQL 5.1引入innodb_autoinc_lock_mode之前的策略）：在分配自增值之前加上AUTO_INC锁，在SQL结束时释放掉。该模式保证了在STATEMENT复制模式下，备库执行类似INSERT … SELECT这样的语句时的一致性，因为这样的语句在执行时无法确定到底有多少条记录，只有在执行过程中不允许别的会话分配自增值，才能确保主备一致。

很显然这种锁模式非常影响并发插入的性能，但却保证了一条SQL内自增值分配的连续性。

**AUTOINC_NEW_STYLE_LOCKING（1）**

默认值。

- 普通的insert/replace操作会先加一个dict_table_t::autoinc_mutex，然后去判断表上是否有别的线程加了LOCK_AUTO_INC锁，如果有的话，释放autoinc_mutex，并使用OLD STYLE；否则，在预留好本次插入需要的自增值之后，快速将autoinc_mutex释放掉。很显然，对于普通的并发INSERT操作，都是无需加LOCK_AUTO_INC锁的。因此大大提升了吞吐量
- 对于无法预计插入量的操作，比如LOAD DATA，INSERT …SELECT等，还需要使用OLD STYLE

和OLD STYLE相比，NEW STYLE也可以保证STATEMENT模式下的复制安全性，但无法保证一条插入语句内的自增值的连续性，并且，如果执行一条混合了显式指定自增值和使用系统分配两种方式的插入语句时，可能存在一定的自增值浪费。

~~~~
insert into t (c1, c2) values (1, 'a'), (NULL, 'b'), (5, 'c'), (NULL, 'd');
~~~~

假设当前的AUTO_INCREMENT = 101，当语句执行完成后，OLD STYLE的下一个自增值为103；NEW STYLE的下一个自增值为105。这是因为在NEW STYLE中，预取了[101, 104] 4个自增值（通过插入行的个数），然后将AUTO_INCREMENT设为105，从而导致自增值103和104被浪费掉。

**AUTOINC_NO_LOCKING（2）**

只在分配时加个autoinc_mutex，并在执行完成释放。不能保证批量插入的复制安全性。

{{< hint info >}}

**自增锁的bug**

这是Mariadb的Jira上报的一个小bug，在row模式下，由于不走parse的逻辑，我们不知道行记录是通过什么批量导入还是普通INSERT产生的，因此command类型为SQLCOM_END，而在判断是否加自增锁时的逻辑时，是通过COMMAND类型是否为SQLCOM_INSERT或者SQLCOM_REPLACE来判断是否忽略加AUTO_INC锁。这个额外的锁开销，会导致在使用ROW模式时，InnoDB总是加AUTO_INC锁，加AUTO_INC锁又涉及到全局事务资源的开销，从而导致性能下降。

修复的方式也比较简单，将SQLCOM_END这个command类型也纳入考虑。

相关的[Jira链接](https://mariadb.atlassian.net/browse/MDEV-7578)

{{</hint>}}

## 4.2 表锁的加锁流程

加表锁调用lock_table，调用流程如下：

1. 判断本事务的之前获取到的表锁（trx→lock.table_locks）强度，如果有比当前lock_mode强的，直接返回该表锁，无需加锁
2. lock_mode为IX/X，则事务设为读写事务
3. 判断要加的lock和该table上的已经加上的其他事务的表锁（dict_table→locks）是否存在冲突，如果冲突，则返回冲突的锁，这也意味着需要锁等待（wait_for）
4. 创建表锁对象
5. 如果需要锁等待，则加入等待队列（并进行死锁检测）（lock_table_enqueue_waiting）

{{< hint info >}}

在#3中，如果在同一个表上更新的并发度很高，会有很多事务持有IX/IS锁，则这个链表就会非常长。

阿里RDS做了如下优化：

基于大多数表锁不冲突的事实，对各种表锁对象进行计数，在检查是否有冲突时，比如当前申请的是意向锁，则如果此时LOCK_S和LOCK_X的锁计数都是0，就可以认为没有冲突，直接忽略检查。由于检查是在持有锁子系统全局大锁lock_sys→mutex下进行的。在单表大并发下，这个优化的效果还是非常明显的，可以减少持有全局大锁的时间。https://mariadb.atlassian.net/browse/MDEV-7578)

{{</hint>}}

在第4步，需要创建表锁对象，流程如下：

1. 根据lock_mode判断是否需要创建lock_table_t

   **自增锁：**

   设置表锁的自增锁指向：table->autoinc_lock

   设置table的自增锁的事务指向：table->autoinc_trx

   加入到事务的持有自增锁列表中：vecor_push(trx->autoinc_locks)

   **S/X/IS/IX锁：**

   先从事务的预分配的表锁池中获取（trx→lock.table_pool），如果池中不够，则alloc创建一个

   S、X表示需要锁住表，即当前层。对表加IS、IX意向锁表示需要对下层进行加锁

2. 将表锁加入该事务的锁列表（trx->lock.trx_locks）和table的锁列表（table→locks）

3. 如果需要等待，设置为事务的当前等待的锁（trx->lock.wait_lock）

我们还是用上面的锁子系统的全局视图，在其中标出在创建表锁时，需要改动的对象：

![InnoDB_lock_lock_table_allocation](/InnoDB_lock_lock_table_allocation.png)

# 5 行锁

## 5.1 行锁的类型

### 5.1.1 LOCK_REC_NOT_GAP

只对记录加锁，不锁之前的gap，即记录锁。在RC下一般都是加的这个类型的记录锁（唯一例外的是unique secondary index上的duplicate key检查，加的是LOCK_ORDINARY，否则无法保证唯一性）

### 5.1.2 LOCK_GAP

只锁记录前的范围，不锁记录本身，即间隙锁（也叫GAP锁）。<font color=red>在RR下使用。可以通过开启innodb_locks_unsafe_for_binlog来避免GAP锁，这样只有在检查外键约束或者duplicate key时才使用GAP锁。</font>

### 5.1.3 LOCK_ORDINARY

next-key lock，锁记录及其之前的间隙，即区间锁。在RR下使用，用于解决幻读问题，即在一个事务中多次执行相同的查询，会看到不同的结果（注意和不可重复读的区别）：即对"还不存在的数据"做出应对，原来没有，（中间有插入）但却读到了；原来不满足条件，（中间有更新）但却读到了（后满足条件）。这意味着需要在读取时对读取的区间加GAP锁。这也是为什么在插入记录时，需要判断下一条记录是否已加锁（是否锁定前向区间），如果已经加锁，则不让插入，以避免产生幻读。

### 5.1.4 LOCK_S

1. 事务读（auto_commit = 0）在隔离级别为SERIALIZABLE时会给记录加LOCK_S锁，但这也取决于场景：非事务读（auto-commit）在SERIALIZABLE隔离级别下，无需加锁。

~~~~
mysql_lock_tables
    lock_external
        handler::ha_external_lock
            ha_innobase::external_lock
 
        if (trx->isolation_level == TRX_ISO_SERIALIZABLE
            && m_prebuilt->select_lock_type == LOCK_NONE
            && thd_test_options(
                thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
 
            /* To get serializable execution, we let InnoDB
            conceptually add 'LOCK IN SHARE MODE' to all SELECTs
            which otherwise would have been consistent reads. An
            exception is consistent reads in the AUTOCOMMIT=1 mode:
            we know that they are read-only transactions, and they
            can be serialized also if performed as consistent
            reads. */
 
            m_prebuilt->select_lock_type = LOCK_S;
            m_prebuilt->stored_select_lock_type = LOCK_S;
        }
 
ha_innobase::general_fetch
    row_search_mvcc
 
        err = sel_set_rec_lock(pcur,
                       rec, index, offsets,
                       prebuilt->select_lock_type,
                       lock_type, thr, &mtr);
~~~~

{{< hint info >}}

在MySQL 5.7中，show engine innodb status不会打印只读事务的信息，只能从informationschema.innodb_trx表中获取到只读事务持有的锁个数等信息。

{{</hint>}}

2. 一致性的锁定读（select ... in share mode），加S-lock，但在不同的隔离级别下，锁的范围有所不同：
   - RC隔离级别： LOCK_REC_NOT_GAP | LOCK_S
   - RR隔离级别：如果查询条件为唯一索引且是唯一等值查询时，加的是 LOCK_REC_NOT_GAP | LOCK_S；对于非唯一条件查询，或者查询会扫描到多条记录时，加的是LOCK_ORDINARY | LOCK_S锁

通常INSERT操作是不加锁的，但如果在插入或更新记录时，检查到 duplicate key（或者有一个被标记删除的duplicate key），对于普通的INSERT/UPDATE，会加LOCK_S锁，而对于类似REPLACE INTO或者INSERT … ON DUPLICATE这样的SQL加的是X锁。而针对不同的索引类型也有所不同：
对于聚集索引（参阅函数row_ins_duplicate_error_in_clust），隔离级别小于等于RC时，加的是LOCK_REC_NOT_GAP类似的S或者X记录锁。否则加LOCK_ORDINARY类型的记录锁（NEXT-KEY LOCK）；
对于二级唯一索引，若检查到重复键，当前版本总是加 LOCK_ORDINARY 类型的记录锁(函数 row_ins_scan_sec_index_for_duplicate)。实际上按照RC的设计理念，不应该加GAP锁（bug#68021），官方也事实上尝试修复过一次，即对于RC隔离级别加上LOCK_REC_NOT_GAP，但却引入了另外一个问题，导致二级索引的唯一约束失效(bug#73170)，感兴趣的可以参阅我写的这篇博客，由于这个严重bug，官方很快又把这个fix给revert掉了。



需要对辅助索引记录的下一条记录进行加锁是为了避免幻读问题。例如下面的语句，其不仅需要锁住辅助索引记录x，还需要锁住x的下一条记录。这样就可以避免其他事务插入同样值为x的记录，从而避免幻读问题：

~~~~
SELECT * FROM table WHERE key_column = x FOR UPDATE;
~~~~

如果key_column有唯一性约束且为等值查询，就再不需要锁定下一个辅助索引记录。

对于非等值插入，不管辅助索引是否包含唯一约束，都需要锁定下一个索引记录，从而避免幻读问题的产生。如下面的语句：

~~~~
SELECT * FROM table WHERE key_column <= x FOR UPDATE;
~~~~

### 4.2.2 加行锁

前面提到，InnoDB的行锁其实是索引记录锁。也就是根据索引中访问得到的记录进行加锁。InnoDB使用的是索引组织表，每张用户表有一个主键值。辅助索引中包含主键，通过辅助索引访问聚簇索引中的记录，需要通过书签查找，即通过辅助索引记录中对应的主键值再次查询聚簇索引。

此外，InnoDB还设计了implicit lock，保证并不是每次查询都需要进行加锁。总结来说，加锁的过程如下：

- 通过主键进行加锁的语句，仅对聚集索引记录进行加锁
- 通过辅助索引记录进行加锁的语句，首先对辅助索引记录加锁，再对聚集索引记录进行加锁
- 通过辅助索引记录进行加锁的语句，可能还需要对下一个记录进行加锁

我们确定了行锁的具体锁类型后，就可以进行加行锁了。加行锁调用lock_rec_lock，通过函数参数可以知道，我们可以确定在哪个索引的哪一页的哪一行上加什么样的行锁，参数如下：

- impl：表示是否加implicit lock，（true为不加锁）
- mode：锁类型
- block：数据页
- heap_no：数据行
- index：记录所在的index

在加行锁中，需要着重注意这两点：

**implict lock的处理**：是否为implicit lock，通过函数lock_rec_lock中的参数impl来决定。如果是implicit lock，并且当前没有该锁的冲突模式存在，则不需要产生一个lock_t的对象。否则，通过函数lock_rec_create创建一个lock_t对象，并加入等待队列。这样的设计是为了当不存在冲突模式时，可以减少锁的创建。比如，如果当前只有一个线程在执行UPDATE t SET key_column=x WHERE row_id=1，如果操作不需要发生等待则不需要对辅助索引记录x进行加锁。

**锁对象的重用**：重用是为了减少锁的开销，锁重用的前提是同一事务锁住的同一个页面中的记录，并且锁的模式相同。例如：

~~~~
BEGIN;
SELECT * FROM t WHERE rowid = x FOR UPDATE;
SELECT * FROM t WHERE rowid = y FOR UPDATE;
~~~~

当第一个SQL语句执行的时候，InnoDB会创建一个锁对象lock_t，并将记录x所在的heap no对应的lock bitmap相应位置1，如果记录y和记录x位于同一页，则可以重用锁对象lock_t，将y相应的bit置1。

此外，如果同一事务访问同一行记录，并且第二次加的锁的强度弱于之前的锁，同样不需要创建锁对象。这也是一种锁重用的场景。

~~~~
BEGIN;
SELECT * FROM t WHERE rowid = x FOR UPDATE;
SELECT * FROM t WHERE rowid = x LOCK IN SHARE MODE;
~~~~

第一个SQL首先创建一个x lock的锁对象，第二次请求的锁类型为s lock，因此不需要再次创建锁对象，可以重用。

下面看一下具体的加锁流程。

行锁的处理流程分为快速加锁（lock_rec_lock_fast）和慢速加锁（lock_rec_lock_slow）两种。

**快速加锁**

对于冲突少的场景，这时比较普遍的加锁方式，以下场景可以走fast lock：

- 记录所在的页上没有任何记录锁（没有explicit lock），即没有锁对象，创建锁对象，加入lock_sy→rec_hash。
- 记录所在的页上只存在一个记录锁，并且属于当前事务，锁模式相同，直接设置对应的记录的bit即可。

对于不满足fast lock的，走slow lock。

**慢速加锁**

如果行数据上没有更强级别的锁，也没有冲突的锁，并且加的不是隐式锁，就加入行锁队列。核心思想是复用锁对象，如果要加锁的行数据上当前没有其它锁等待，并且行所在的数据页上有相似的锁对象（lock_rec_find_similar_on_page）就可以直接设置对应行的 bitmap 位，表示加锁成功。如果有其它锁等待，就重新创建一个锁对象。

加锁流程如下：

- 判断当前事务是否已经持有了一个强度更高或者相同的锁，如果是的话，直接返回成功（lock_rec_has_expl）;
- 检查是否存在和当前申请锁模式冲突的锁（lock_rec_other_has_conflicting），如果存在的话，就创建一个锁对象（RecLock::RecLock），并加入到等待队列中（RecLock::add_to_waitq），并进行死锁检测
- 如果没有冲突的锁，则入队列（lock_rec_add_to_queue）：已经有在同一个Page上的锁对象且没有别的会话等待相同的heap no时，可以直接设置对应的bitmap（lock_rec_find_similar_on_page）；否则需要创建一个新的行锁对象

在上面的第二条判断时，更细节的判断如下：

如果上述两条都不满足，即不是相同的事务，基本锁类型也不兼容，那么满足下面任意一条，同样返回false，不需要等待；否则返回 true，需要等待：

- 如果当前锁锁住的是supremum或者LOCK_GAP为1并且LOCK_INSERT_INTENTION为0。因为不带LOCK_INSERT_INTENTION 的 GAP 锁不需要等待任何东西，不同的用户可以在 gap 上持有冲突的锁。
- 如果当前锁 LOCK_INSERT_INTENTION 为 0 并且锁是 LOCK_GAP 为 1。因为行锁（LOCK_ORDINARY LOCK_REC_NOT_GAP）不需要等待一个 gap 锁。
- 如果当前锁 LOCK_GAP 为 1，锁 LOCK_REC_NOT_GAP 为 1。同样的，因为 gap 锁没有必要等待一个 LOCK_REC_NOT_GAP 锁。

如果锁 LOCK_INSERT_INTENTION 为 1。此处是最后一步，说明之前的条件都不满足，源码中备注描述如下：

~~~~
/* No lock request needs to wait for an insert
intention lock to be removed. This is ok since our
rules allow conflicting locks on gaps. This eliminates
a spurious deadlock caused by a next-key lock waiting
for an insert intention lock; when the insert
intention lock was granted, the insert deadlocked on
the waiting next-key lock.
 
Also, insert intention locks do not disturb each
other. */
~~~~

比如，如果行数据上已经加了 LOCK_S | LOCK_REC_NOT_GAP, 再尝试去加 LOCK_X | LOCK_GAP，LOCK_S 和 LOCK_X 本身是冲突的，但是满足上述第 3 个条件，返回 FALSE，不需要等待。

我们还是用上面的锁子系统的全局视图，在其中标出在创建行锁时，需要改动的对象：

![InnoDB_lock_lock_rec_allocation](/InnoDB_lock_lock_rec_allocation.png)

## 4.2 释放锁及唤醒

根据隔离级别不同，事务中锁的释放时机也不同，但是写锁都是在事务提交时释放的（除RU外），即SS2PL。但有两种例外场景，写锁的释放会提前：

- 自增锁，前面已经提到了，在插入语句结束就释放（lock_unlock_table_autoinc）
- 在RC隔离级别下执行DML语句时，从引擎层返回到Server层的记录，如果不满足where条件，则需要立刻unlock掉（ha_innobase::unlock_row）。

自增锁的释放

~~~~
/*****************************************************************//**
Commits a transaction in an InnoDB database or marks an SQL statement
ended.
@return 0 or deadlock error if the transaction was aborted by another
    higher priority transaction. */
static
int
innobase_commit(...) {
        ...
        /* We just mark the SQL statement ended and do not do a
        transaction commit */
 
        /* If we had reserved the auto-inc lock for some
        table in this SQL statement we release it now */
 
        if (!read_only) {
            lock_unlock_table_autoinc(trx);
        }
 
        /* Store the current undo_no of the transaction so that we
        know where to roll back if we have to roll back the next
        SQL statement */
 
        trx_mark_sql_stat_end(trx);
}
~~~~

写锁的释放由函数lock_trx_release_locks完成，释放时依次遍历trx→lock.trx_locks即可。在释放时InnoDB还做了优化，如果该事务持有的行锁对象太多，每释放1000（LOCK_RELEASE_INTERVAL）个锁对象，会暂时释放下lock_sys->mutex再重新持有，防止InnoDB hang住。

对于行锁的释放（lock_rec_dequeue_from_page），除了将其从全局锁列表（log_sys→rec_hash）中移除外，还需要将后续在等待队列中的其他不冲突的锁唤醒（lock_rec_grant）。比如释放某个记录的X-lock，那么会唤醒其行上等待中的下一个S-lock。

从行锁的释放和唤醒可以看到，其中的逻辑开销比较大，尤其是大量线程在等待少量几个行锁时。当某个行锁从hash链上移除时，InnoDB实际上通过遍历相同page上的所有等待的行锁，并判断这些行锁等待是否可以被唤醒。而判断唤醒的逻辑又会遍历一次。这里的核心原因是因为行锁的链表维护是基于<space_id, page no>的，并不基于行的heap no构建的。这个问题官方讨论过，可以参阅[bug#53825](http://bugs.mysql.com/bug.php?id=53825)。提到使用<space_id, page no, heap no>来构建链表，移除bitmap会浪费更多的内存，但效率更高，而且现在的内存也没有以前那么昂贵。

表锁对象的释放（lock_table_dequeue）会遍历表锁链表（un_member.tab_lock.locks & LOCK_WAIT），也会唤醒后续的下一组不冲突的表锁。比如释放X，等待队列中是S, IX, IS, X，则会将S, IX, IS这三个等待中的表锁一起唤醒。

另外在Query Cache中，如果表级锁的类型不为LOCK_IS，且当前事务修改了数据，就将表对象的dict_table_t::query_cache_inv_id设置为当前最大的事务id。在检查是否可以使用该表的Query Cache时会使用该值进行判断（row_search_check_if_query_cache_permitted），如果某个用户会话的事务对象的low_limit_id（即最大可见事务id）比这个值还小，说明它不应该使用当前table cache的内容，也不应该存储到query cache中。

锁的等待（lock_wait_suspend_thread）和唤醒实际上是线程的等待和唤醒，并通过OS_EVENT机制来实现唤醒和锁超时。

# 5. 行锁的维护

各个事务在其加锁操作完成后，内存中可能产生多个行锁对象，这些行锁对象通过（space_id, page_no）映像到锁子系统的lock_sys→rec_hash中。但是，页也随着事务的推进时刻发展着变化，比如B+树的分裂和合并，这需要对已经存在的行锁对象进行维护。

下面将对各个数据变更的场景分别来看如何来维护行锁对象。

## 5.1 插入

插入操作的步骤如下：

1. 首先对表加上IX表。
2. 根据查询模式PAGE_CUR_LE定位记录next_rec。
3. 判断记录next_rec是否有锁，有的话等待锁的释放，否则直接插入。

插入操作需要定位插入记录的下一条记录，这是next-key locking算法所要求的，因为该算法锁定的是区间，而不仅仅是记录本身。

比如表中有如下记录：

~~~~
1、2、3、4、5、7、8
~~~~

如果要insert 6，则

1. 首先对表加IX锁

2. 根据查询模式PAGE_CUR_LE定位到记录5

3. 判断记录5的下一条记录（7）是否有锁（lock_rec_insert_check_and_lock，参数inherit用来判断是否在插入完成后调用函数lock_update_insert来对已经锁定的范围进行更新）。根据next-key locking算法，锁定范围应该是(5, 7]或者(5, 7)（GAP标记位=1）

   如果记录7上已经有锁，则不允许在这个范围内进行插入，所以操作将阻塞（lock_rec_enqueue_waiting），等待记录7上锁的释放。这时会产生锁对象，锁定的记录位next_rec（7），锁的类型为LOCK_X | LOCK_GAP

   如果记录7上没有锁，则直接插入，不产生任何锁对象

另外，如果下一条记录next_rec上有锁，不管持有该锁的是否为插入操作事务本身，当插入操作完成后（无需事务提交），需要更新锁定的范围（lock_update_insert）。比如在这个例子中，如果插入了6这条记录，则将原来锁定的范围从 (5, 7] 更新为 (5,6) ，(6, 7]，这样就可以阻止其他事务在（5，6）的范围内进行插入操作。

另外需要注意，如果插入的表上有辅助索引，那么还需要对辅助索引记录进行锁的判断，其方法与步骤2、3相同。只是在判断可以进行插入后，还需要额外更新辅助索引页中的page header.PAGE_MAX_TRX_ID。

## 5.2 更新

当事务需要对记录进行更新（包括删除）前，首先尝试对更新的记录加上X（implicit lock），如果待更新的记录上存在其他锁，则事务被阻塞，需要等待记录上的锁被释放。

lock_clust_rec_modify_check_and_lock和lock_rec_rec_modify_check_and_lock分别对聚集索引和辅助索引进行加锁。两者的过程基本相同，都是调用lock_rec_lock对更新的记录进行加锁操作，不同的是：

- 聚集索引记录加锁前首先需要将记录上implicit lock转化为explicit lock
- 辅助索引记录加锁成功后，还需要更新辅助索引页的page header.PAGE_MAX_TRX_ID

在事务对记录进行更新的过程中，如果记录不能进行原地更新，则需要对锁进行维护，其步骤如下：

1. 将更新记录的锁信息移动到页的infimum记录上（lock_rec_store_on_page_infimum）
2. 删除原记录
3. 插入更新完成后的记录
4. 将页的infimum记录上的锁重新移动到新插入的记录上

这里可以发现伪记录infimum的作用，诣在更新时用于临时保存更新记录上的锁信息。这样做的好处是可以提高更新锁的效率，否则需要先删除老的锁对象然后再创建新的锁对象。

## 5.3 purge

InnoDB进行记录的删除时，分为标记删除和真正删除两步。首先将delete flag置为1，然后通过后台的purge线程将记录真正的删除。

当purge线程真正删除记录操作完成后，删除记录的下一个记录需要继承删除记录的锁定范围，并且其模式是GAP，同时释放并重置删除记录上等待锁的信息（lock_update_delete）。

另外，对于标记为delete flag的加锁记录，InnoDB存储引擎在定位记录后需要进一步扫描记录才能确定查询是否为结束。这里举个例子：

~~~~
create table t(
    a int primary key,
    b varchar(30)
) engine=InnoDB;
 
insert into t values (1, 'a');
~~~~

接着执行下面的操作：

~~~~
begin;
update t set b = repeat('a', 30) where a = 1;
select * from t where a <=1 for update;
~~~~

首先执行第一个update语句，因为记录1的大小发生了变化，不能进行原地更新（varchar之前只存储了'a'，放不下更新后的30个'a'），因此对于这个update操作，首先进行标记删除，然后再插入一条新记录。当purge操作发生前，这个页中包含了两个PK=1的记录，这也符合InnoDB的MVCC。但在此时，先执行了select，InnoDB会对PK=1的两个记录都加上一个X-lock。因为第一次加锁的记录，其record header的delete flag被标记为了1，因此还需要访问record header中的next record来判断是否已经结束的扫描。

然而这有时会导致一些用户可能难以理解的问题，比如下面这个例子：

~~~~
drop table if exists t;
 
create table t(
    a int primary key,
    b varchar(30)
)engine=InnoDB;
 
insert into t values (1, 'a');   // heap_no: 2
insert into t values (2, 'b');   // heap_no: 3
insert into t values (3, 'c');   // heap_no: 4
insert into t values (4, 'd');   // heap_no: 5
~~~~

首先进行会话A：

~~~~
begin;
select * from t where a = 4 for update;
~~~~

接着进行会话B：

~~~~
begin;
select * from t where a <= 2 lock in share mode;
delete from t where a = 3;
select * from t where a <= 2 lock in share mode; // blocking
~~~~

用户会“惊奇地”发现会话B中的第二个SELECT语句会被阻塞，而同样第一次的SELECT操作是已经加锁成功的。其实导致这个现象的原因是SELECT游标锁定的最大记录被标记为了删除（未被真正PURGE删除），因此，当第二次再次执行SELECT操作时，需要进一步锁定记录（a = 4)，而该记录已经在session A的事务中被锁定了。

之前已经说过，同一范围的GAP类型锁是可以互相兼容的，例如两个事务分别持有（2，4）范围的GAP锁是被允许的，GAP锁只是用来阻止在这个范围内进行插入，甚至是持有GAP锁的事务本身。例如有2、3、4这三个记录，事务A持有（2，3）这个GAP锁，类型为X。而事务B持有（3，4）这个GAP锁，类型为S。不同事务可能持有不同兼容性的GAP锁如图9-12所示。

图9-12

从图9-12可以看到，若事务A、B开始持有不同类型的GAP锁，若事务C删除了记录3，并且事务提交后进行了purge操作。这时会对锁定的范围进行合并，而事务A和B这时都锁定了（2，4）这个范围，然而持有锁的类型完全不同。

## 5.4 一致性的锁定读

默认情况下，InnoDB使用一致性的非锁定读（consistent non-locking read），即读取不会被阻塞。但是，某些情况下用户希望通过锁定读取（locking reads）的方式来保证数据的一致性，即通过语法lock in share mode和for update主动的对读取进行加锁操作，这样的一种方式称为一致性的锁定读（consistent locking read）。lock_clust_rec_read_check_and_lock和lock_sec_rec_read_check_and_lock分别用来对聚集索引记录和辅助索引记录进行加锁。

对于聚集索引记录，只需对主键值进行加相应的锁就可以了。而对于辅助索引记录，除了需要对辅助索引记录本身加锁，还需要对主键索引记录加锁。

比如对前面的表t，加上一个辅助索引：

~~~~
ALTER TABLE t ADD KEY idx_b(b);
~~~~

然后运行

~~~~
SELECT * FROM t WHERE b='a' LOCK IN SHARE MODE;
~~~~

那么需要做如下加锁操作：

1. 辅助索引加锁：b='a'的记录加S-lock

2. 聚集索引加锁：a=1 的记录加S-lock

3. 防止幻读

   如果辅助索引有唯一性约束，则无须此步（唯一性保证不会有2个记录的b=‘2’）

   如果没有，则需要对辅助索引记录'a'的下一条记录加上S-lock。（为了避免其他事务插入b等于‘a’的记录，从而导致幻读问题的产生）

更详细的行锁type_mode参见行锁的锁类型一节

## 5.5 页的分裂

当一个页中的记录数增加到一定程度后，就需要存储到多个页中，即产生页的分裂，从而导致页中的锁信息发生变化，插入操作会引起页的分裂。

举个例子：

~~~~
create table t(
    a int not null primary key,
    b blob
)engine=InnoDB;
 
insert into t values (1, repeat('a', 7000));
insert into t values (2, repeat('b', 7000));
~~~~

此时，页中的2条记录存储在一个页中（假设为(20, 100)）。

紧接着，进行如下操作：

~~~~
begin;
select * from t where a = 1 lock in share mode;
select * from t where a = 2 lock in share mode;
~~~~

此时行锁对象在内存中的状态为：

![InnoDB_lock_page_split-1](/InnoDB_lock_page_split-1.png)

如果再插入新的记录：

~~~~
insert into t values (3, repeat('c', 7000));
~~~~

这时就会产生页的分裂，第2个记录从第一个页移动到了第二个页中，并且需要对锁信息进行维护。

![InnoDB_lock_page_split-2](/InnoDB_lock_page_split-2.png)

InnoDB在实际进行页分裂时，可以往左或者往右进行，在这个例子中，页是往右分裂的。维护锁信息的步骤（lock_update_split_right）如下：

1. 确定分裂点记录split_rec
2. 将记录split_rec到尾部（supremum）之间的所有锁移动到新页中，修改lock bitmap的值（lock_rec_move）
3. 将原来页中supremum记录持有的锁移动到新页的supremum记录上（lock_rec_move）
4. 将新页中的第一条记录的锁继承给原页的记录supremum（类型为gap）（lock_rec_inherit_to_gap）

如图所示，灰色表示持有锁的记录：

![InnoDB_lock_migration-page_split-right](/InnoDB_lock_migration-page_split-right.png)

接下来详细解释一下上面的步骤成因，因为innoDB使用的是next-key locking算法，因此锁定的是区间，比如页中有以下记录：

PAGE: R1, R2, R3, R4, …… Rn-1, Rn

那么可以锁定的范围有：

PAGE: ( infimum , R1 ],  ( R1 , R2 ], ( R2, R3 ], ( R3, R4 ] …… ( Rn-1, Rn ], ( Rn, supremum)

如果根据记录R3往右分裂，分裂后的区间为：

PAGE: ( infimum , R1 ],  ( R1 , R2 ], ( R2, supremum)

RIGHT PAGE: ( infimum, R3 ], ( R3, R4 ] …… ( Rn-1, Rn ], ( Rn, supremum)

分裂完成后，需要将split_rec R3及之后的所有记录的锁都移动到RIGHT PAGE，同时更新当前页的锁对象lock_rec_t，并产生RIGHT PAGE上的新的锁对象lock_rec_t。

如果原来PAGE的supremum记录有锁（肯定是gap锁），则分裂后RIGHT PAGE的supremum记录也必须有锁，这是为了防止其他事务在这个范围进行修改操作，也就是步骤3的作用。

在分裂时，如果原来PAGE的split_rec R3上有锁，即表示范围( R2, R3 ]的区间内不允许有其他事务进行修改操作。那么在分裂后，上述的区间分裂为两个区间： ( R2, supremum ) 和 ( infimum, R3 ]，因此分裂完成后还需要对 ( R2, supremum )这个区间加锁，这就是步骤4的作用。

往左分裂（lock_update_split_left）和往右分裂大致相同，但是不需要步骤3，这是因为在往左分裂时，supremum记录的锁是稳定的。如下图所示：

![InnoDB_lock_migration-page_split-left](/InnoDB_lock_migration-page_split-left.png)

## 5.6 页的合并

和页的分裂一样，页的合并（merge）操作也可以分为往右和往左两种，也同样需要对行锁信息进行维护。

往左合并的步骤（lock_update_merge_left）如下：

1. 记录LEFT PAGE合并前最大的用户记录orig_pred
2. 将RIGHT PAGE中的用户记录复制到LEFT PAGE，同时更新对应的行锁信息（page_copy_rec_list_start）
3. 将LEFT PAGE的supremum记录上的锁继承给记录orig_pred指向的下一条记录（lock_rec_inherit_to_gap）
4. 将RIGHT PAGE的supremum记录上的锁移动到LEFT PAGE的supremum记录上（lock_rec_move）

如下图所示：

![InnoDB_lock_migration-page_merge-left](/InnoDB_lock_migration-page_merge-left.png)

往右合并（lock_update_merge_right）和往左合并大致相同，但是不需要步骤4，这是因为在往右合并时，supremum记录的锁是稳定的。如下图所示：

![InnoDB_lock_migration-page_merge-right](/InnoDB_lock_migration-page_merge-right.png)

# 6. 死锁

## 6.1 死锁的概念

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象。如果没有外力的作用，这些事务都将无法推进下去。

解决死锁的方法有三种：

1. 不要有等待，任何等待转化为回滚，并重新开始事务。但这会导致性能的下降，极端情况下任何事务都无法推进下去，而这所带来的问题远比死锁更为严重，因为这样很难被发现并且浪费资源。
2. 超时。当多个事务互相等待时，设置一个锁超时的时间，由用户决定是回滚还是继续推进直至提交。
3. 死锁检测。

死锁检测比超时更为主动。DBMS普遍采用等待图（wait-for graph）的方式进行死锁检测，这就要求在数据库中保存以下两种信息：

- 锁的信息链表
- 事务等待链表

通过上述链表可以构造出一张图，如果是一个有向无环图（DAG - Directed Acyclic Graph），则表示不存在死锁；如果在图中存在环（回路），那么就表示存在死锁。在图中，事务为节点，事务T1指向T2的边的定义为：

- 事务T1等待事务T2所占用的资源
- 事务T1最终等待T2所占用的资源，也就是事务之间在等待相同的资源，而事务T1发生在事务T2的后面

下面来看一个例子，当前事务和锁的状态如下图所示：

![InnoDB_lock_wait-for_graph](/InnoDB_lock_wait-for_graph.png)

从上图中可以看到，在事务等待列表中共有T1、T2、T3、T4 4个事务，所以在wait-for graph中有4个节点。持有锁的情况：T2对row1持有X-lock，T1对row2持有S-lock，T4对row2持有S-lock。而有三个锁等待，冲突的锁等待的关系表示为边（等待→持有，等待→ 等待），这样一共有6条边。可以看出T2→T4→T3形成了环，出现了死锁。

## 6.2 死锁概率

死锁应该极少发生，如果经常发生，则系统会处于不可用的状态。另外，死锁还应该少于等待，因为至少2次等待才会产生一次死锁。

下面将从数学的概率来分析，死锁的发生概率非常小。

假设数据库中一共有n+1个线程执行，即一共有n+1个事务。假设每个事务所作的操作相同，每个事务由r+1个操作组成，每个操作从R行数据中随机操作一行数据，并持有相应的行锁。事务在执行完最后一步后释放所占用的所有锁资源。假设nr << R，即所有事物操作的数据只占所有数据的一小部分。

在上述模型中，事务获得

## 6.3 死锁检测

### 6.3.1 死锁检测的场景

前面提到，出现等待是死锁的先决条件。所以，在InnoDB中，当表锁或者行锁需要等待时，会进行死锁检测，函数调用链如下：

~~~~
RecLock::add_to_waitq
    RecLock::deadlock_check
        DeadlockChecker::check_and_resolve
lock_table
    lock_table_enqueue_waiting
        DeadlockChecker::check_and_resolve
~~~~

我们下面以行锁为例，分析死锁检测的流程。

### 6.3.2 死锁检测流程

当发现有冲突的行锁时，就需要把冲突的行锁放入事务的等待队列中（RecLock::add_to_waitq）。在实际放入等待队列之前：

- 如果持有冲突锁的线程是MySQL内部的后台线程，则不会被高优先级事务取消。这是为了优先保证内部线程正常运行。
- 比较当前会话和持有锁的会话的事务优先级，调用函数trx_arbitrate 返回被选作牺牲者的事务；
  - 当前发起请求的会话是后台线程，但持有锁的会话设置了高优先级时，选择当前线程作为牺牲者；
  - 持有锁的线程为后台线程时，在第一步已经判断了，不会选作牺牲者；
  - 如果两个会话都设置了优先级，低优先级的被选做牺牲者，优先级相同时，请求者被选做牺牲者(thd_tx_arbitrate)；
- 如果当前会话的优先级较低，或者另外一个持有锁的会话为后台线程，这时候如果当前会话设置了优先级，直接报错，并返回错误码DB_DEADLOCK；
  - 默认不设置优先级时，请求锁的会话也会被选作victim_trx，但只创建锁等待对象，不会直接返回错误；
- 当持有锁的会话被选作牺牲者时，说明当前会话肯定设置了高优先级，这时候会走RecLock::enqueue_priority的逻辑；
  - 如果持有锁的会话在等待另外一个不同的锁时，或者持有锁的事务不是readonly的，当前会话会被回滚掉；
  - 开始跳队列，直到当前会话满足加锁条件（RecLock::jump_queue）；
    - 请求的锁对象跳过阻塞它的锁对象，直接操作hash链表，将锁对象往前挪；
    - 从当前lock，向前遍历链表，逐个判断是否有别的会话持有了相同记录上的锁（RecLock::is_on_row），并将这些会话标记为回滚（mark_trx_for_rollback）,同时将这些事务对象搜集下来，以待后续处理（但直接阻塞当前会话的事务会被立刻回滚掉）；
  - 高优先级的会话非常具有杀伤力，其他低优先级会话即使拿到了锁，也会被它所干掉。

到目前为止，MySQL 5.7还不支持用户端设置线程优先级，所以，目前所有用户线程的事务优先级都是一样的。

InnoDB的死锁检测算法是深度优先搜索wait-for graph，如果在搜索过程中发现有环，就说明发生了死锁。同时，为了避免死锁检测的开销过大，如果搜索深度超过了200（LOCK_MAX_DEPTH_IN_DEADLOCK_CHECK），也同样认为发生了死锁（victim是发起者）。

InnoDB最早使用的是递归方式搜索，之后为了减少栈空间的开销，改为使用入栈的方式。

死锁检测相关的数据结构：

~~~~
/** Deadlock checker. */
class DeadlockChecker {
 
    /** DFS state information, used during deadlock checking. */
    struct state_t {                                // 栈中的元素，即状态
        const lock_t*   m_lock;                     /*!< Current lock */
        const lock_t*   m_wait_lock;    /*!< Waiting for lock */
        ulint       m_heap_no;      // 如果为行锁，则行锁对应的记录heap no
    };
 
    /** Used in deadlock tracking. Protected by lock_sys->mutex. */
    static ib_uint64_t  s_lock_mark_counter;        局部的死锁计数器，在当前发起死锁检测时设置
 
    ulint           m_cost;                         // 遍历过的节点计数
    const trx_t*    m_start;                        // 发起者
    bool            m_too_deep;                     // 是否深度遍历达到最大深度
    const lock_t*   m_wait_lock;                    // ctx等待的锁（第一次是发起者申请的锁）
 
    /**  Value of lock_mark_count at the start of the deadlock check. */
    ib_uint64_t     m_mark_start;                   // 局部的死锁计数器，在当前发起死锁检测时设置
    size_t          m_n_elems;                      // state_t入栈次数
    static state_t  s_states[MAX_STACK_SIZE];       // 预分配的栈结构，避免malloc/free开销
}
~~~~

DeadlockChecker.state_t就是辅助的栈结构，分配了s_states[MAX_STACK_SIZE]个（4096）。

DeadlockChecker.m_start是第一个请求锁的事务，即死锁检测的发起者，如果深度搜索的过程中所对应的事务等于m_start，那么就说明产生了环。m_wait_lock表示搜索中的事务等待的锁。

我们在这里举个例子，如下图所示：

![InnoDB_lock_deadlock_detection](/InnoDB_lock_deadlock_detection.png)

事务A、B、C各自已经获得了记录1、2、3上的X锁。事务A对记录2的X锁请求因为B而等待，同样，事务B对记录3的X锁请求因为C而等待。当事务C对记录1的发起X锁请求时，发生死锁：

1. m_start初始化为C，m_wait_lock初始化为C.X1（表示数据1上的X锁）（都为死锁检测的发起者，以及发起者相应需要等待的锁）
2. <font color=blue>根据m_wait_lock = C.X1</font>，拿到数据1上的第一个锁lock（DeadlockChecker::get_first_lock）：事务A锁持有的X1锁 A.X1
3. 判断lock对应的事务（A）是否也在等待其它锁（lock->trx->lock.que_state == TRX_QUE_LOCK_WAIT）。因为事务A确实在等待X2锁，把当前的lock入栈（state.lock = A.X1，state.m_wait_lock = C.X1）
4. ctx中的m_wait_lock更新为lock->trx->lock.wait_lock, 也就是 X1 锁的持有者事务 A 所等待的锁 X2（A.X2）
5. <font color=blue>同步骤2</font>，根据m_wait_lock=A.X2, 拿到加在数据2上的第一个锁赋值给lock，也就是事务B持有的X2锁（B.X2）。<font color=blue>此时完成一次循环</font>。
6. 再次<font color=blue>进入循环</font>，lock 对应的事务（B）同样在等待其它锁（state.lock = B.X2，state.m_wait_lock = A.X2），所以把当前的 lock 入栈。
7. ctx中的m_wait_lock 更新为lock->trx->lock.wait_lock, 也就是 X2 锁持有者事务 B 所等待的锁 X3（B.X3）
8. 同步骤 2，根据m_wait_lock=B.X3, 拿到加在数据3上的第一个锁赋值给lock，也就是事务C持有的X3锁（C.X3）。<font color=blue>此时完成一次循环</font>。
9. 再次<font color=blue>进入循环</font>，此时 lock->trx = C = ctx->start。死锁形成。

具体流程如下图所示：

![InnoDB_lock_deadlock_detection_example](/InnoDB_lock_deadlock_detection_example.png)

具体的程序代码如下：

~~~~c++
DeadlockChecker::check_and_resolve(const lock_t* lock, trx_t* trx)
{
    const trx_t*    victim_trx;
 
    /* Try and resolve as many deadlocks as possible. */
    do {
        DeadlockChecker checker(trx, lock, s_lock_mark_counter);    // new死锁检测器，这里只设置发起者和发起者申请的锁
 
        victim_trx = checker.search();                              // 进行死锁检测
 
        /* Search too deep, we rollback the joining transaction only
        if it is possible to rollback. Otherwise we rollback the
        transaction that is holding the lock that the joining
        transaction wants. */
        if (checker.is_too_deep()) {                                // 如果深度搜索超过200，则认定victim为发起者，进行事务回滚
            rollback_print(victim_trx, lock);
            break;
        } else if (victim_trx != NULL && victim_trx != trx) {       // 如果选出的victim不是当前事务
            ut_ad(victim_trx == checker.m_wait_lock->trx);           // 确认成环
            checker.trx_rollback();                                 // 发起事务回滚
            lock_deadlock_found = true;
        }
    } while (victim_trx != NULL && victim_trx != trx);
 
    /* If the joining transaction was selected as the victim. */
    if (victim_trx != NULL) {                                       // 发起者为victim，上报错误，进行事务回滚
        print("*** WE ROLL BACK TRANSACTION (2)\n");
        lock_deadlock_found = true;
    }
 
    trx_mutex_enter(trx);
 
    return(victim_trx);
}
 
const trx_t*
DeadlockChecker::search()
{
    /* Look at the locks ahead of wait_lock in the lock queue. */
    ulint       heap_no;
    const lock_t*   lock = get_first_lock(&heap_no);                    // 从发起者的申请锁（m_wait_lock），拿到该申请锁所在的page上的第一个已持有锁
 
    for (;;) {
 
        /* We should never visit the same sub-tree more than once. */
        ut_ad(lock == NULL || !is_visited(lock));
 
        while (m_n_elems > 0 && lock == NULL) {                          // 出栈操作
 
            /* Restore previous search state. */
 
            pop(lock, heap_no);
 
            lock = get_next_lock(lock, heap_no);
        }
 
        if (lock == NULL) {                                             // 没有锁需要遍历了，结束死锁检测
            break;
        } else if (lock == m_wait_lock) {                               // 遍历到的发起者的申请锁
 
            /* We can mark this subtree as searched */
            ut_ad(lock->trx->lock.deadlock_mark <= m_mark_start);
 
            lock->trx->lock.deadlock_mark = ++s_lock_mark_counter;
 
            /* We are not prepared for an overflow. This 64-bit
            counter should never wrap around. At 10^9 increments
            per second, it would take 10^3 years of uptime. */
 
            ut_ad(s_lock_mark_counter > 0);
 
            /* Backtrack */
            lock = NULL;
 
        } else if (!lock_has_to_wait(m_wait_lock, lock)) {              // m_wait_lock当前冲突的锁没有和page上有的已获取锁冲突，取page的下一个锁
            /* No conflict, next lock */
            lock = get_next_lock(lock, heap_no);
        } else if (lock->trx == m_start) {                               // 发现环（发起者和节点一致）
            /* Found a cycle. */
            notify(lock);
            return(select_victim());                                    // 选出victim
        } else if (is_too_deep()) {                                     // 遍历深度达到max，将发起者设为victim，退出死锁检测
            m_too_deep = true;
            return(m_start);
        } else if (lock->trx->lock.que_state == TRX_QUE_LOCK_WAIT) {  // 如果遍历到的持有的锁对应的事务也有等待的锁，则压栈
 
            /* Another trx ahead has requested a lock in an
            incompatible mode, and is itself waiting for a lock. */
 
            ++m_cost;                                                   // 遍历节点++
 
            if (!push(lock, heap_no)) {                                 // 压栈（等待的锁，冲突对应的持有的锁）
                m_too_deep = true;                                      // 如果预留的栈已用尽，将发起者设为victim，退出死锁检测
                return(m_start);
            }
 
            m_wait_lock = lock->trx->lock.wait_lock;                  // 将冲突对应的持有的锁的事务等待锁设为ctx.m_wait_lock
            lock = get_first_lock(&heap_no);                            // 获取ctx.m_wait_lock对应的page上的第一个已持有锁
 
            if (is_visited(lock)) {                                     // 如果已访问过该锁，获取该page的下一个已持有锁
                lock = get_next_lock(lock, heap_no);
            }
        } else {
            lock = get_next_lock(lock, heap_no);                        // page上有多个锁，取下一个锁
        }
    }
 
    /* No deadlock found. */
    return(0);
}
~~~~

### 6.3.3 选出victim

当发生死锁后，会选择一个代价较少的事务进行回滚操作（select_victim），从发起者（ctx→m_start）和形成环的最后一条边（ctx→m_wait_lock）中二选一。

权重的比较（trx_weight_ge）：

1. 修改了不支持事务的表为高
2. 持有的undo数量+持有的锁之和大者为高

{{< hint info >}}

**Tips**

如果业务经过精心设计，是可以从业务上避免死锁的。并且死锁检测本身会持有大锁（log_sys→mutex），代价非常高昂。在高并发的场景下对热点行的更新往往会造成性能雪崩。

为了避免热点行的死锁检测造成性能雪崩，阿里RDS做过一个feature，在server层对热点行的更新进行排队，以减缓死锁检测带来的性能影响。

另外，在阿里内部的应用中，因为有专业的团队来保证业务SQL的质量，我们可以选择性的禁止死锁检测来提升性能，尤其是在热点更新场景，带来的性能提升非常明显，极端高并发下，甚至能带来数倍的提升。

{{</hint>}}

### 6.3.4 锁等待的错误码处理

当无法获得锁时，会将相应的错误码传到上层进行处理（row_mysql_handle_errors）：

**DB_LOCK_WAIT**

- 具有高优先级的事务已经搜集了会阻塞它的事务链表，这时候会统一将这些事务回滚掉（trx_kill_blocking）
- 将当前的线程挂起（lock_wait_suspend_thread），等待超时时间取决于session级别配置（innodb_lock_wait_timeout），默认50秒
- 如果当前会话的状态设置为running，一种是被选做死锁检测的牺牲者，需要回滚当前事务，另外一种是在进入等待前已经获得了事务锁，也无需等待
- 获得等待队列的一个空闲slot。（lock_wait_table_reserve_slot）
  - 系统启动时，已经创建好了足够用的slot数组（类型为srv_slot_t），挂在lock_sys->waiting_threads上
  - 分配slot时，从slot数组的第一个元素开始遍历，直到找到一个空闲的slot。注意这里存在性能问题：如果挂起的线程非常多，每个新加入挂起等待的线程都需要遍历直到找到一个空闲的slot。 实际上如果每次遍历都从上次分配的位置往后找，到达数组末尾在循环到数组头，这样可以在高并发高锁冲突场景下获得一定的性能提升
- 如果会话在innodb层（通常为true），则强制从InnoDB层退出，确保其不占用innodb_thread_concurrency的槽位。然后进入等待状态。被唤醒后，会再次强制进入InnoDB层
- 被唤醒后，释放slot（lock_wait_table_release_slot）
- 如果被选为victim，返回上层回滚事务；如果等待超时了，则根据参数innodb_rollback_on_timeout的配置，默认为OFF只回滚当前SQL，设置为ON表示回滚整个事务。

**DB_DEADLOCK**

直接回滚当前事务

## 6.4 死锁的例子

两个事务就可以出现死锁，即A等待B，B也在等待A，这种死锁称为AB-BA死锁。

比如：

| session A                                       | session B                                                    |
| :---------------------------------------------- | :----------------------------------------------------------- |
| begin;select * from t where a = 1 for update;   |                                                              |
|                                                 | begin;select * from t where a = 2 for update;                |
| select * from t where a = 2 for update;等待.... |                                                              |
|                                                 | select * from t where a = 1 for update;ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |

又比如：

~~~~
create table t (
    a int not null primary key
) engine = InnoDB;
 
insert into t values (1), (2), (4), (5);
~~~~

| session A                                                    | session B                                             |
| :----------------------------------------------------------- | :---------------------------------------------------- |
| begin;select * from t where a = 4 for update;                |                                                       |
|                                                              | begin;select * from t where a = 4 lock in share mode; |
| insert into t values (3);ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |                                                       |

首先session A持有记录4的x-lock，接着session B尝试获取记录4的s-lock，但出现等待，当session 4尝试插入记录3，即要求插入的下一条记录不能含有任何锁，包括gap锁或处于等待的锁，而出现AB-BA死锁。

<font color=red>为何</font>

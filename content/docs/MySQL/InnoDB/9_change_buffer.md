---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: change buffer
---

change buffer是InnoDB所特有的。

# 设计

change buffer目的是减少随机访问磁盘，提升non-unique secondary index（以下简称NUSI）的变更效率，在IO-bound workload下很有必要。

在InnoDB中，数据顺序存储在clustered index的leaf node，secondary index leaf node则是离散的，因此，对seconary index leaf node的变更操作是随机IO。为了提升secondary index的磁盘IO效率，InnoDB的做法是采用写缓冲机制，将NUSI的变更操作缓冲下来（leaf page oriented），减少随机IO，并在合适的时机将多次对同一页的操作进行merge，这块缓存称为change buffer。

{{< hint info >}}

当导入大量数据时，DBA常见的一个做法是，先disable keys，然后倒入数据后再enable keys，这也是减少随机IO手段的一个体现。

{{</hint>}}

这里借用官方一张图，可以给大家带来直观的理解：

![InnoDB_change_buffer](/InnoDB_change_buffer.png)

在MySQL 5.5之前，因为只支持缓存insert操作，所以叫做insert buffer，后面支持了更多的操作类型（一共有insert、delete mark、delete），才改叫change buffer。

那为什么只对non-unique进行cache呢？这是因为unique index要保证唯一性约束，在插入记录时要判断是否唯一，势必要读取secondary page leaf node，而这样做违反了change buffer的设计目的。但是，对于unique secondary index的delete操作，是可以缓存在change buffer中的。

change buffer也是一颗B+ tree，这颗B+ tree的page也会使用buffer pool的内存空间（但是会限制其占用的比例，最多25% CHANGE_BUFFER_DEFAULT_SIZE）。

在更新NUSI时，首先判断leaf page是否在buffer pool中，如果在，则直接在内存中更新page；否则满足缓存条件后将变更缓存到change buffer中（保存位置+op+data），然后在后台异步将change buffer缓存写回NUSI的leaf page中，GC并推进LWN。

从这里可以看到，secondary index的变更的路径分为两个分支：

- 如果NUSI的leaf page在buffer pool中：这个和之前B+ tree中页的更新完全一样，不再详述，可以保证A、D，也可以保证index tree页操作的一致性（SMO）
- 如果不在，变更先缓存到change buffer中，然后再写回原处：这里要保证change buffer的A、D

这里更新NUSI的场景有：

- 用户线程的DML
- 后台purge线程

对于读取来说，直接通过物理读读取NUSI的leaf page后，需要判断change buffer中是否已缓存变更操作（缓存该页的change buffer record），如果进行了缓存，要先merge change buffer，然后再读取leaf page中的record。

那么接下来看一下InnoDB中的change buffer的具体实现细节。

# 实现

首先，change buffer也是一颗B+ tree，page大小也是16K，我们先从page和tree的组织谈起，然后再进入到page内部的记录，接着看merge是如何写回原处的，最后是gc，也就是purge线程推进LWM。

## change buffer tree

change buffer tree虽然也是一个B+ tree，但是其和之前的index tree有异同之处。

相同的地方：

1. 叶子和非叶节点存储的数据和B+ tree一样，也是branch node（node pointer，存储的是key + page_no）和leaf node（key+data）

差异的地方：

1. change buffer tree全局只有一颗，类型为DICT_CLUSTERED | DICT_IBUF，index_id = 0xFFFFFFFF00000000ULL
2. change buffer tree的index root page是固定的，位于系统表空间的（0, 4）位置
3. 空闲页通过链表管理，链表在index root page（PAGE_BTR_IBUF_FREE_LIST），复用了原来的leaf segment header（PAGE_BTR_SEG_LEAF），空闲页的页类型为FIL_PAGE_IBUF_FREE_LIST，空闲页通过链表节点串联前后位置（PAGE_BTR_IBUF_FREE_LIST_NODE），也复用了原来的leaf segment header
4. 因为#3复用了leaf segment header，所以用一个特殊的change buffer header page（0, 3）用于change buffer tree page的管理，其segment entry位于IBUF_TREE_SEG_HEADER（PAGE_BTR_SEG_LEAF PAGE_DATA + 0：38 + 0）

## change buffer record

change buffer record历史上有过多次版本演进，这里以MySQL 5.7.26中使用记录格式为例介绍。

chagne buffer tree中的leaf node（页）中的记录格式为：

![InnoDB_change_buffer_record](/InnoDB_change_buffer_record.png)

其中，space_id+page_no+count可以表示页的变更序（INSERT x, DEL MARK x, INSERT x），在构建change buffer record时：

1. 首先将count设为0xFFFF
2. 然后再change buffer tree的leaf node中通过page cursor LE模式查找定位（space_id, page_no, 0xFFFF）到最后一条已存在的记录
3. 取该记录的count+1作为此次插入记录的count

这三元组也是change buffer tree的key。

op有3种：insert、delete mark、delete

````
typedef enum {
    IBUF_OP_INSERT = 0,
    IBUF_OP_DELETE_MARK = 1,
    IBUF_OP_DELETE = 2,
 
    /* Number of different operation types. */
    IBUF_OP_COUNT = 3
} ibuf_op_t;
````

## change buffer bitmap page

change buffer会将更改的辅助索引记录缓存起来，但是其必须保证merge一定成功，即保证缓存的NUSI leaf page在merge change buffer record后不会引发页的合并或分裂。

为了保证这一点，change buffer就需要tracking NUSI leaf page上的空闲空间，存储这个信息的页被称为change buffer bitmap page。该页在每个XDE page后一页出现（每256M一个，第2个页），每个index page用4个bit来描述，整个占用8192字节。

ibuf bitmap page的页面布局如下：

![InnoDB_change_buffer_bitmap_page](/InnoDB_change_buffer_bitmap_page.png)

change buffer bitmap中保存的针对每个页的bitmap中位信息如下：

| 字段                 | 占用位 | 说明                                                         |
| :------------------- | :----- | :----------------------------------------------------------- |
| IBUF_BITMAP_FREE     | 2      | 记录UNSI leaf page的剩余空间                                 |
| IBUF_BITMAP_BUFFERED | 1      | 该页是否已被chagne buffer cache，用于物理读NUSI leaf page后判断是否需要merge |
| IBUF_BITMAP_IBUF     | 1      | 该页是否为change buffer page，<font color=red>主要用于异步IO（ibuf_thread）的读操作（ibuf_page_low）</font> |

其中IBUF_BITMAP_FREE表示的剩余空间含义为：

- 0：    0
- 1：  512 bytes
- 2：1024 bytes
- 3：2048 bytes

从上面可以看出，因为change buffer bitmap只能追踪一个UNSI leaf page最多2KB的空闲空间，因此change buffer一次最多缓存的记录总量为2KB。

IBUF_BITMAP_FREE的信息在每次UNSI leaf page更改时都会更新。

## 缓存

对于change buffer，可以缓存的操作有：

````
/** Allowed values of innodb_change_buffering */
static const char* innobase_change_buffering_values[IBUF_USE_COUNT] = {
    "none",     /* IBUF_USE_NONE */
    "inserts",  /* IBUF_USE_INSERT */
    "deletes",  /* IBUF_USE_DELETE_MARK */
    "changes",  /* IBUF_USE_INSERT_DELETE_MARK */
    "purges",   /* IBUF_USE_DELETE */
    "all"       /* IBUF_USE_ALL */
};
````

这些操作对应到DML为：

- insert
  - 乐观插入：inserts
- update
  - 非主键更新
    - 乐观更新（delete mark+insert）：changes
- delete
  - delete mark：deletes
  - purge：purges

这里前3种都为用户线程的DML触发，而最后一种为后台purge线程触发。

### 缓存条件

只有真正删除UNSI leaf page上的物理记录才会导致页的合并，而这只有purge线程才做这个动作。因此，只有purge线程准备插入op=IBUF_OP_DELETE的change buffer record时，才进行判定：先预估在merge完成后该page上的所有change buffer record后，还会剩下多少记录（ibuf_get_volume_buffered）。如果只剩下一条记录，为了避免成为empty page而触发页的合并，则不进行cache，而走普通的的修改流程（将UNSI leaf page读入buffer pool，merge change buffer record，然后再进行页的修改）。

在进行UNSI操作时，满足下面条件，才进行cache（ibuf_should_try）：

1. 开启了innodb_change_buffering且innodb_change_buffer_max_size不为0
2. 必须是UNSI leaf node（delete mark不要求unique）
3. 表没有进行flush操作（通过dict_table_t::quiesce 标识）：flush table xxx with read lock/for export

满足以上条件后，通过buf_page_get_gen尝试获取数据页：

- BUF_GET_IF_IN_POOL：只在buffer pool中查找，不在直接返回null
- BUF_GET_IF_IN_POOL_OR_WATCH：purge线程使用，只在buffer pool中查找（在：加LRU list+线性预读，不在：设置watch）

如果不在buffer pool中，则判断如果不会导致页的分裂、合并后，进行缓存。

为了避免页的分裂，则通过space_id+page_no获取对应的change buffer bitmap page，然后找到该数据页的空闲空间，如果超出，则进行异步merge。

ibuf_insert_low

````
if (op == IBUF_OP_INSERT) {
    ulint   bits = ibuf_bitmap_page_get_bits(
        bitmap_page, page_id, page_size, IBUF_BITMAP_FREE,
        &bitmap_mtr);
 
    if (buffered + entry_size + page_dir_calc_reserved_space(1)
        > ibuf_index_page_calc_free_from_bits(page_size, bits)) {
        /* Release the bitmap page latch early. */
        ibuf_mtr_commit(&bitmap_mtr);
 
        /* It may not fit */
        do_merge = TRUE;
 
        ibuf_get_merge_page_nos(FALSE,
                    btr_pcur_get_rec(&pcur), &mtr,
                    space_ids,
                    page_nos, &n_stored);
 
        goto fail_exit;
    }
}
````

## change buffer merge

正如前面介绍的，change buffer最多使用buffer pool的25%，所以变更的辅助索引记录不可能一直停留在change buffer索引树中，记录的变化最终还是要存储到UNSI leaf page中，这个操作称为merge。

根据处理方式的不同，merge可以分为主动和被动两类。

### 主动merge

主动merge是指由用户线程主动发起的UNSI leaf page页的读取操作，这时会将记录merge到leaf page上，并且，由于leaf page已经读取到buffer pool，那么之后对于该页的更改操作将不会再缓存到change buffer，为同步IO。

### 被动merge

- 已经为NUSI leaf page buffer了太多change buffer record，可能导致NUSI leaf page分裂，发起异步IO读取leaf page后merge，最多merge8个leaf page（IBUF_MERGE_AREA）
- change buffer tree size超过最大值10以上（ibuf→max_size + 10 IBUF_CONTRACT_DO_NOT_INSERT），进行同步IO merge（ibuf_contract），随机merge 8个leaf page
- 可能引起change buffer tree上page的分裂
  - ibuf→size < ibuf→max_size，no-op
  - ibuf→size >= ibuf→max_size +5（IBUF_CONTRACT_ON_INSERT_SYNC），进行同步IO merge，随机
  - 在二者之间，进行异步IO merge，随机
- 后台master thread周期性merge（ibuf_merge_in_background）
  - IDLE：100% io_capacity
  - ACTIVE：5% io_capacity
- slow shutdown，全部merge（srv_master_do_shutdown_tasks）
- flush table：flush table xxx for export/with read lock; 表级别merge 

# 死锁

change buffer的最大挑战是对死锁的处理。之前我们介绍过，当我们将辅助索引页从磁盘读取到buffer pool中是，需要进行change buffer的merge，而merge可能会引起ibuf tree的收缩。因此需要持有相应的latch进行并发控制：

![InnoDB_change_buffer_deadlock_ibuf_latch](/InnoDB_change_buffer_deadlock_ibuf_latch.png)

需要持有fsp x-latch是因为b+树索引发生合并时需要对文件存储模块进行整理。但在之前介绍过的B+树索引中，获得latch的顺序如下：

![InnoDB_change_buffer_deadlock_B+tree_latch](/InnoDB_change_buffer_deadlock_B+tree_latch.png)

latch的规定顺序如下：

~~~~
enum latch_level_t {
    ...
    SYNC_FSP_PAGE,
    SYNC_FSP,
    SYNC_TREE_NODE,
    SYNC_TREE_NODE_FROM_HASH,
    SYNC_TREE_NODE_NEW,
    SYNC_INDEX_TREE,
    ...
};
~~~~

如果将change buffer作为一颗普通的B+树来对待，将会导致死锁。

还有一种情况也会导致change buffer产生死锁：所有的I/O线程都在异步读取操作，读取到的都是辅助索引页并且都需要进行change Buffer的合并，那么这时候将没有空闲的I/O线程处理对于change Buffer树的读取操作，从而导致死锁。

通过上面的介绍可以发现，change buffer对于锁的层次设计才是其难点与技巧所在。为了避免上述死锁情况发生，InnoDB将页逻辑的分为三个层次（level）：

- 非change buffer页
- 除change buffer bitmap页之外的change buffer页，包括change buffer索引内存对象的latch，change buffer页的latch
- change buffer bitmap页

当持有某层的latch时不得持有其上层的latch。这就意味着，在处理change buffer对象时，fsp模块相关的latch的优先级要高于Insert Buffer索引树的latch！！！既然fsp模块相关的latch要比索引树的latch优先级要高，那么处理B+树索引的合并与扩展就要与普通索引树不同。

Insert Buffer索引拥有自己的存储管理，可以发现在Insert Buffer索引的root页中存在free list链表。这样的设计可以使得Insert Buffer在合并与扩展时不需要通过fsp相关模块的latch保护，从而使得fsp模块的latch优先级可以高于Insert Buffer的设计。图11-4显示了在InnoDB引擎中，普通B+树索引（也就是用户表）与Insert Buffer的B+树索引树在文件空间管理上的不同：

普通的B+树索引其root页保存有非叶子节点段的段头（segment header）信息和叶子节点段头。B+树索引操作是自上往下，自左往右进行加锁。当需要进行存储空间的扩展与收缩操作时，通过持有fsp模块的相关latch进行操作。这就是图11-3中显示的latch顺序。

Insert Buffer虽然也是B+树索引，但是其只有一个段，段头保存在独立的页中（Insert Buffer header page）。另外其有独立的文件空间管理，Insert Buffer段所申请到的页都保存在root页的free list链表中。当需要进行B+树的扩展或收缩时，首先通过判断Insert Buffer的root页中的freelist链表是否有足够的空闲页。这样的设计使得在操作Insert Buffer索引树时不需要持有fsp模块的相关latch，只需要向free list链表中添加或删除页即可。也就是原fsp模块的相关latch与Insert Buffer索引树的相关存储操作分离，即之前所说的fsp模块的latch优先级高于Insert Buffer相关对象的情况，从而避免死锁问题的产生。
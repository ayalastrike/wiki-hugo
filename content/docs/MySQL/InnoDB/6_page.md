---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

InnoDB使用的是索引组织表，因此聚簇索引的叶子节点中存放完整的数据记录，辅助索引页的叶子节点中存放指向聚簇索引页叶子节点的书签（bookmark），也可以称为路标。

主要是两部分：

- page layout
- scan rec with cursor, and then insert, update, delete

# 页

页是InnoDB存储数据的基本单位。页的大小可以设置为4K~64K（4K的倍数），默认为16K。页的大小设置要从IO性能考虑。

````
/* Define the Min, Max, Default page sizes. */
/** Minimum Page Size Shift (power of 2) */
#define UNIV_PAGE_SIZE_SHIFT_MIN    12
/** Maximum Page Size Shift (power of 2) */
#define UNIV_PAGE_SIZE_SHIFT_MAX    16
/** Default Page Size Shift (power of 2) */
#define UNIV_PAGE_SIZE_SHIFT_DEF    14
 
static MYSQL_SYSVAR_ULONG(page_size, srv_page_size,
  PLUGIN_VAR_OPCMDARG | PLUGIN_VAR_READONLY,
  "Page size to use for all InnoDB tablespaces.",
  NULL, NULL, UNIV_PAGE_SIZE_DEF,
  UNIV_PAGE_SIZE_MIN, UNIV_PAGE_SIZE_MAX, 0);
 
/** The universal page size of the database */
#define UNIV_PAGE_SIZE      ((ulint) srv_page_size)
````

页和记录一样，也存在两种形态：物理页和逻辑页。物理页存储在外部的存储设备上，逻辑页存在于缓冲池中。一般，用block表示物理页，用page表示逻辑页（注意这里的block和page并不是指的buf_block_t、buf_page_t）。

缓冲池中的逻辑页在进行读取操作时首先分配一个PCB，并将其FIX，然后将磁盘上的物理页读入内存中，形成逻辑页，然后对逻辑页进行操作（读取或者修改）；修改后，再异步刷入磁盘。这意味着，在某一段时间窗口内物理页和逻辑页可能是不一致的，最新修改的数据在内存中，而不一定体现在磁盘上。而通过事务的ACID特性，可以保证最终某一时间点两者的数据会达到最终一致。

# page layout

在InnoDB中，页根据用途分为多种类型。而使用最多的就是索引页，所以在这里我们focus在index page。

索引页（index page）的页面布局如下：

![InnoDB_page_layout_summary](/InnoDB_page_layout_summary.png)

从这里我们可以看出，index page包含3部分信息：

1. （表）空间相关的信息：FIL_HEADER、FIL_TRAIL
2. 页相关的信息：PAGE_HEADER、PAGE_DIRECTORY
3. 数据：记录

我们下面逐个拆解：PAGE_HEADER、PAGE_DIRECTORY和记录的组织。

{{< hint info >}}
FIL_HEADER、FIL_TRAIL存储的是表空间相关的信息，在此不展开，具体信息参见storage management一节。

{{</hint>}}

## page header

page header用于保存页的信息，占用56个字节，位于file header之后：

![InnoDB_page_detail](/InnoDB_page_detail.png)

page header中存放的是页中数据存储的信息：页维度的存储空间、使用和事务信息。

page header中的字段含义如下：

| 名称              | 大小 | 说明                                                         |
| :---------------- | :--- | :----------------------------------------------------------- |
| 名称              | 大小 | 说明                                                         |
| PAGE_N_DIR_SLOTS  | 2    | page directory中槽的数量                                     |
| PAGE_HEAP_TOP     | 2    | 堆顶，堆中空闲空间的位置（offset），空闲区                   |
| PAGE_N_HEAP       | 2    | page中的heap no（单调递增），最高位（第1个bit）表示行格式是new style还是old style |
| PAGE_FREE         | 2    | 指向页中空闲空间的位置（offset），碎片区，放的都是被purge线程回收的记录 |
| PAGE_GARBAGE      | 2    | 已删除记录的字节数，即行记录中delete flag=1的记录的总量      |
| PAGE_LAST_INSERT  | 2    | 最后插入记录的位置（offset）                                 |
| PAGE_DIRECTION    | 2    | 最后插入记录的方向 PAGE_LEFT          0x01 PAGE_RIGHT        0x02 PAGE_SAME_REC     0x03 PAGE_SAME_PAGE    0x04 PAGE_NO_DIRECTION 0x05 |
| PAGE_N_DIRECTION  | 2    | 一个方向上连续插入记录的数量                                 |
| PAGE_N_RECS       | 2    | 页中用户记录的数量                                           |
| PAGE_MAX_TRX_ID   | 8    | 修改当前页的TRX ID，仅在辅助索引和insert buffer中使用        |
| PAGE_LEVEL        | 2    | 当前页在索引树中的位置，0x00代表叶子节点，即叶子节点总是在第0层（不变性），page_is_leaf |
| PAGE_INDEX_ID     | 8    | 索引ID，标识当前页属于哪个索引                               |
| PAGE_BTR_SEG_LEAF | 10   | B+树叶子节点所在段的segment header，仅在B+树的root页中定义   |
| PAGE_BTR_SEG_TOP  | 10   | B+树非叶子节点所在段的segment header，仅在B+树的root页中定义 |

PAGE_MAX_TRX_ID及之前的所有字段在页创建的时候时设置，其后的字段在填充具体数据时才设置。

## page directory

page header中保存了页中记录的存储信息，如果要进行记录的查询，则需要通过page directory。并且，B+树的查找（node pointer.page_no）只能定位到记录所在的页，精确定位到记录（记录在页中的位置）则需要通过page directory。

这种数据结构可以保证比较高的插入删除和查找效率：是一种经典的读写分离模型：

page directory由槽（slot）组成，每个槽占用2个字节，指向记录在页中的偏移量。槽根据指向记录的主键顺序逆序存放，可以通过二叉查找算法快速定位到查询的记录。但是，为了提高存储以及插入的效率，InnoDB对于槽的设计采用了稀疏（sparse）方式，槽和记录不是1:1的，而是1:n的。每个槽对应一个记录，这个记录中记录n，即用n_owned属性记录所对应的槽中拥有的记录数量。槽所指向的记录被称为owner record，即从后往前找的n_owned不为0的那个record，表示其前面n个记录归其index。

- insert：heap no++, owned++, （optional） 第8个split page directory slot
- delete：link摘除, owned--, （optional） <=3, merge page directory slot
- owned record, special

{{< hint warning>}}
Because the overhead of inserts is so small, we may also increase the
page size from the projected default of 8 kB to 64 kB without too
much loss of efficiency in inserts. Bigger page becomes actual
when the disk transfer rate compared to seek and latency time rises.
On the present system, the page size is set so that the page transfer
time (3 ms) is 20 % of the disk random access time (15 ms).

{{</hint>}}

除了首尾两个槽外，每个槽总是包含4~8条记录（PAGE_DIR_SLOT_MIN_N_OWNED ~ PAGE_DIR_SLOT_MAX_N_OWNED），第一个槽（即infimum记录）仅包含一个记录（因为要临时保存锁信息，所以只描述自己），最后一个槽（即supremum记录）可以包含1~8个记录。

{{< hint warning>}}
Assuming a page size of 8 kB, a typical index page of a secondary index contains 300 index entries, and the size of the page directory is 50 x 4 bytes = 200 bytes.

{{</hint>}}

page_dir_split_slot和page_dir_balance_slot用来保证每个槽所包含的记录在上述的定义范围内。如果超出范围，则调用page_dir_split_slot来产生新的slot，如果低于这个范围，则调用page_dir_balance_slot来平衡slot中的记录，可能删除slot，或者对相邻的slot进行merge。

在查找记录时，如果是通过记录找下一条记录，可以直接根据next_record得出。否则，需要先定位记录所在的slot（从supremum slot开始从后往前查找slot，直至找到记录相应的slot），然后在两个slot之间在查找记录。

## record organization

在record中讲过，每个页都有2个伪记录，如图所示。在infimum记录和supremum记录之间的被称为用户记录（page_rec_is_user_rec_low）。因此在page header中用PAGE_N_HEAP和PAGE_N_RECS标识堆和页中的记录数量，PAGE_N_RECS表示一页上真实的记录数，PAGE_N_HEAP表示从HEAP上分配出去的记录数。

![InnoDB_page-infimum_supremum_record](/InnoDB_page-infimum_supremum_record-2683587.png)

在old-style和new-style行格式下，infimum和supremum记录稍有区别，如下图所示：

![InnoDB_page_view_with_row_format_layout](/InnoDB_page_view_with_row_format_layout.png)

在PAGE_HEADER中：

- 如果PAGE_N_RECS = 0，则认为是空页（page_is_empty）
- PAGE_GARBAGE != 0，则认为页中有碎片区，可以回收（page_has_garbage）

页中的可用空间可以分为两种：

- 空闲区：一块连续的未使用空间，用PAGE_HEAP_TOP表示
- 碎片区：已删除的记录构成的不连续的回收空间，用PAGE_FREE（指明位置，链表）和PAGE_GARBAGE（指明可用字节）表示

当记录被删除时，将其空间放入可用回收空间的首部，即PAGE_FREE指向最近删除的记录空间。接着通过记录中record header中的next record可以得到后续的已删除记录，这样就构成了一个可用回收空间链表。这个可用空间的大小用PAGE_GARBAGE表示。

当记录在页中申请空间时，首先尝试使用碎片区（PAGE_FREE），空间不足再尝试从堆顶的空闲区（PAGE_HEAP_TOP）中分配。

对于碎片区的空间检查，InnoDB只检查第一个可重用空间，而不是遍历整个可用回收空间链表。比如可用空间链表为100 -> 400 -> 300，当要插入的记录大于100字节时，就不会重用碎片区。这是因为InnoDB具有页整理（re-organize）的功能，即在页空间不足时会首先对页进行重新组织（btr_page_reorganize_low），按照页中的记录主键顺序重新整理，回收碎片空间。

![InnoDB_page_space_management](/InnoDB_page_space_management.png)

{{< hint info>}}
On top of it are the index records in a heap linked into a one way linear list according to alphabetic order.

{{</hint>}}

页中的记录按照索引键顺序排列的，这个排序是逻辑的，不是物理的，完全按照物理排序的代价非常巨大。因此，页只是一个存储记录的堆，即记录是按照堆号逻辑排序的。页中的记录的实际组织形式由record.next_record串联起来，如下图所示：

![InnoDB_page_record_organization](/InnoDB_page_record_organization.png)

PAGE_LAST_INSERT、PAGE_DIRECTION、PAGE_N_DIRECTION用于页的分裂（split）操作。在传统的B+树中，分裂操作都是向左进行的。但是InnoDB会根据以上三个值来判断插入的方向，是顺序升序插入、顺序降序插入、还是无序的随机插入，而采用不同的分裂策略。

为了获得更好的顺序存储性，InnoDB将叶子节点（leaf page）和非叶子节点（non leaf page，不包含root page）的数据通过聚簇的方式存放到两个不同的segment中。

{{< hint info>}}
MySQL 3.23.49中使用page_template作为初始化页的模板供page_create调用：当该模板初始化完成后，之后对于页的初始化仅需要复制此模板即可。后续的版本废弃了这种做法，将需要初始化的静态部分声明为字符串常量。

{{</hint>}}

page_move_rec_list_start（移动一个）和page_move_rec_list_end（移动一个及以后所有）用于将页中的记录迁移到新的页中。在记录的页迁移过程中，需要更新锁、page header中的PAGE_MAX_TRX_ID以及对应的自适应哈希索引。这两个函数主要用于B+树的分裂。

下面我们结合一个具体的例子来看一下InnoDB是如何保存page header、page directory和实际的record。

````
create table t (
    a   int     not null auto_increment primary key,
    b   char(3) default null
) enngine = InnoDB, row_format = dynamic;
 
insert into t (b) values ('aaa');
insert into t (b) values ('bbb');
insert into t (b) values ('ccc');
insert into t (b) values ('ddd');
insert into t (b) values ('eee');
insert into t (b) values ('fff');
insert into t (b) values ('ggg');
insert into t (b) values ('hhh');
insert into t (b) values ('iii');
insert into t (b) values ('jjj');
````

整理出page header如下：

````
PAGE_N_DIR_SLOTS    00 03                           3
PAGE_HEAP_TOP       01 86
PAGE_N_HEAP         80 0c                           32780
PAGE_FREE           00 00
PAGE_GARBAGE        00 00
PAGE_LAST_INSERT    01 72
PAGE_DIRECTION      00 02                           PAGE_RIGHT
PAGE_N_DIRECTION    00 09                           9
PAGE_N_RECS         00 0a                           11
PAGE_MAX_TRX_ID     00 00 00 00 00 00 00 00
PAGE_LEVEL          00 00
PAGE_INDEX_ID       00 00 00 00 00 00 00 2c
PAGE_BTR_SEG_LEAF   00 00 00 1a 00 00 00 02 00 f2
PAGE_BTR_SEG_TOP    00 00 00 1a 00 00 00 02 00|32
````

接着后面就是插入的10条用户记录

````
  |  |extra info    |  ROW ID   |     Trx ID      |    undo pointer    |  数据
     |01 00 02 00 1c|           |                 |                    |69 6e 66 69 6d 75 6d 00
     |07 00 0b 00 00|           |                 |                    |73 75 70 72 65 6d 75 6d
03|00|00 00 10 00 1b|80 00 00 01|00 00 00 00 05 33|a6 00 00 01 1a 01 10|61 61 61
03 00 00 00 18 00 1b 80 00 00 02 00 00 00 00 05 34 a7 00 00 01 1b 01 10 62 62 62
03 00 00 00 20 00 1b 80 00 00 03 00 00 00 00 05 38 aa 00 00 01 1e 01 10 63 63 63
03 00 04 00 28 00 1b 80 00 00 04 00 00 00 00 05 3a ab 00 00 01 1f 01 10 64 64 64
03 00 00 00 30 00 1b 80 00 00 05 00 00 00 00 05 3b ac 00 00 01 20 01 10 65 65 65
03 00 00 00 38 00 1b 80 00 00 06 00 00 00 00 05 3c ad 00 00 01 21 01 10 66 66 66
03 00 00 00 40 00 1b 80 00 00 07 00 00 00 00 05 3d ae 00 00 01 22 01 10 67 67 67
03 00 00 00 48 00 1b 80 00 00 08 00 00 00 00 05 3e af 00 00 01 23 01 10 68 68 68
03 00 00 00 50 00 1b 80 00 00 09 00 00 00 00 05 3f b0 00 00 01 24 01 10 69 69 69
03 00 00 00 58 fe fe 80 00 00 0a 00 00 00 00 05 40 b1 00 00 01 25 01 10 6a 6a 6a
````

该页上所有记录extra info的heap_no、n_owned和record type信息：

````
heap_no         n_owned     record type
0               1           010 infimum record
1               7           011 supremum record
2               0           000 conventional record
3               0           000 conventional record
4               0           000 conventional record
5               4           000 conventional record
6               0           000 conventional record
7               0           000 conventional record
8               0           000 conventional record
9               0           000 conventional record
10              0           000 conventional record
11              0           000 conventional record
````

从上面可以看出，一共有3个槽位（PAGE_N_DIR_SLOTS = 3），infimum记录的n_owned是1，第4个用户记录的n_owned是4，supremum记录的n_owned是7。

页面的最后部分是page directory + fil trailer（8个字节）：

````
Page Directory  00 70 00 d0 00 63
Fil Trailer     84 a2 0d 2f 00 14 b1 6f
````

fil trailer之前就是page directory，每个槽位占2个字节，这里可以看出一共有3个槽位，并且槽是逆序存放的，因此，整理后（正向排序）的page directory为：

````
(00 63), (00 d0), (00 70)
````

这里的槽位表示记录在页中的偏移量，比如通过（00 d0）可以直接定位到row id为（80 00 00 04）的记录。此外，infimum和supremum记录的位置不会改变，相对应的槽的位置总是（00 63）和（00 70），如下图所示：

![InnoDB_record_and_page_directory](/InnoDB_record_and_page_directory.png)

## 数据页的完整性

数据也的完整性通过checksum保证，在InnoDB中，checksum值的计算方法通过innodb_checksum_algorithm配置。

目前提供三种计算checksum的方法：

1. crc校验：这种是一种比较新的计算方法，可以使用cpu硬件指令来加速（buf_calc_page_crc32）
2. InnoDB校验：InnoDB自己开发的一种计算方法，有新老两种变体
3. none模式：使用一个指定的值填充checksum字段（等于没有...）

另外还有strict前缀的选项，这表示在读取数据页时必须通过指定的校验方式的校验值。提供strict选项是为了兼容老版本的MySQL，防止校验算法被修改而导致的数据不可用。

# page cursor

上面介绍了page中的数据组织，接下来介绍如何定位记录和对记录进行变更。

## 定位记录

page cursor更为准确的说是index page cursor（索引页的游标），即一个用来指向记录所在位置的游标，包括三元组：所属的索引（index）、定位的page（block+offsets）、记录（rec），定义如下：

````
/** Index page cursor */
struct page_cur_t{
    const dict_index_t* index;
    rec_t*          rec;    /*!< pointer to a record on page */
    ulint*          offsets;
    buf_block_t*    block;  /*!< pointer to the block containing rec */
};
````

在InnoDB中有btr cursor（B-tree cursor）和page cursor和rec，关系为btr cursor→page cursor→record，即搜索路径为：

- B+树层的搜索
- page层的搜索
- 在page中定位到record

B-tree cursor在index一章介绍。

page cursor用来在page内定位（lookup）记录，内部通过查询模式进行向前或者向后的记录扫描（scan）。

查询模式定义如下：

```
/* Page cursor search modes; the values must be in this order! */
enum page_cur_mode_t {
    PAGE_CUR_UNSUPP = 0,
    PAGE_CUR_G  = 1,        // 大于
    PAGE_CUR_GE = 2,        // 大于等于
    PAGE_CUR_L  = 3,        // 小于
    PAGE_CUR_LE = 4,        // 小于等于
 
/*      PAGE_CUR_LE_OR_EXTENDS = 5,*/ /* This is a search mode used in
                 "column LIKE 'abc%' ORDER BY column DESC";
                 we have to find strings which are <= 'abc' or
                 which extend it */
 
/* These search mode is for search R-tree index. */
    PAGE_CUR_CONTAIN            = 7,
    PAGE_CUR_INTERSECT          = 8,
    PAGE_CUR_WITHIN             = 9,
    PAGE_CUR_DISJOINT           = 10,
    PAGE_CUR_MBR_EQUAL          = 11,
    PAGE_CUR_RTREE_INSERT       = 12,
    PAGE_CUR_RTREE_LOCATE       = 13,
    PAGE_CUR_RTREE_GET_FATHER   = 14
};
```

我们这里只关注G\GE\L\LE。

查找的流程如下：

1. 通过二分查找定位记录所在的槽
2. 通过顺序查找扫描槽中的记录
3. 定位记录

结合查询模式，具体的查询如下图所示：

![InnoDB_page_cursor_search](/InnoDB_page_cursor_search.png)

比如，对于一个主键的范围查询，首先需要定位第一个记录，然后进行记录的顺序扫描。然而，根据不同的查询模式所定位的记录可能完全不同，例如：

```
1, 2, 2, 3, 3, 3, 4, 5, 5, 6, 7
```

如果查找=3, ≥3的记录，那么page cursor应该定位到第一个记录为3的位置。如果查找>3的记录，则应定位记录为4的位置（前2者都是从左往右）。如果查找<3，则定位到第二个记录为2的位置。如果查找≤3，则定位到最后一个记录为3的位置（后2者从右往左）。

page_cur_search_with_match通过查询的记录tuple来定位页中的记录，并返回page cursor来标定位置。与此同时，4个变量ilow_matched_fields、ilow_matched_bytes、iup_matched_fields、iup_matched_bytes用来返回使用二叉查找算法进行记录比较时，左右各已经匹配的字段数量和字节数。

查询记录与页中记录的比较通过函数cmp_dtuple_rec_with_match_low进行。

下面来看一个查询≥3或者<3的记录的实例：

![InnoDB_page_cursor_GE_L](/InnoDB_page_cursor_GE_L.png)

cmp_dtuple_rec_with_match_low返回的4个变量为iup_matched_fields = 0、iup_matched_bytes = 3、ilow_matched_fields = 1、ilow_matched_bytes = 0：

|      | 匹配的字段（fields） | 部分匹配的字节数（bytes） |
| :--- | :------------------- | :------------------------ |
| low  | 0                    | 3                         |
| up   | 1                    | 0                         |

在上面的例子中，比较的记录列的数量为1，如果记录由（a, b, c）3列组成，查询根据（a,b）定位，情况如何呢？

![InnoDB_page_cursor_LE_G](/InnoDB_page_cursor_LE_G.png)

cmp_dtuple_rec_with_match_low返回的4个变量为iup_matched_fields = 2、iup_matched_bytes = 0、ilow_matched_fields = 0、ilow_matched_bytes = 3。

|      | 匹配的字段（fields） | 部分匹配的字节数（bytes） |
| :--- | :------------------- | :------------------------ |
| low  | 2                    | 0                         |
| up   | 0                    | 3                         |

## 插入记录

page_cur_insert_rec_low用来插入记录，记录可以是物理记录或者逻辑记录，一般是逻辑记录。变量tuple为要插入的逻辑记录，插入时首先需要将tuple转化为物理记录，cursor指向插入记录之前的记录（一方面是next-key locking，一方面是保证加锁的有序）。

举例来说，在记录集合（1，3，6，7）中插入一条新纪录5，首先需要定位待插入的位置（page_cur_search_with_match），查询模式为PAGE_CUR_LE，也就是定位到值为3的记录的位置，然后再进行插入。插入完成后需要更新row id 3的next record，然后将插入的row id 5的next record指向row id 6的位置，并返回新插入记录的位置（记录的original offset）。

![InnoDB_page_cursor_record_insert](/InnoDB_page_cursor_record_insert.png)

具体流程如下：

![InnoDB_page_record_insert](/InnoDB_page_record_insert.png)

page_cur_insert_rec_low

1. 获得物理记录的长度
2. 获取空闲空间，从碎片区或者从空闲区（heap）分配
3. 将物理记录拷贝到空闲空间
4. 修改记录的双向链表前后指针，和page header中的PAGE_N_RECS
5. 设置record header（n_owned = 0, heap_no）
6. 更新page header（PAGE_LAST_INSERT、PAGE_DIRECTION、PAGE_N_DIRECTION）
7. 更新slot的n_owned
8. 如果需要，分裂slot
9. 写redo log

插入记录会对页进行修改，所以要产生redo log，redo log的类型是MTR_LOG_INSERT，在每次调用page_cur_insert_rec_low的同时，调用page_cur_insert_rec_write_log将变更写入redo log。MTR_LOG_INSERT的redo log日志格式如图所示：

| 字段                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| type                  | MTR_LOG_INSERT                                               |
| space                 | 记录插入到的表空间ID                                         |
| page no               | 记录插入的索引页在表空间中的偏移量                           |
| cur_rec_offset        | 插入前，page cursor指向的行记录在页中的偏移量                |
| lex & extra_info_flag | 保存redo log body长度和记录的extra_info_flag                 |
| info_bits             | extra_info_flag=1时，保存插入记录的info_bits                 |
| original_offset       | extra_info_flag=1时，保存插入记录的original offset           |
| mis_match_index       | extra_info_flag=1时，保存插入记录与cur_rec_offset所指向记录的第一个不同的字节（record header不用比较） |
| original rec body     | 插入记录的body                                               |

MTR_LOG_INSERTS格式说明：

![InnoDB_redo_log_MLOG_REC_INSERT](/InnoDB_redo_log_MLOG_REC_INSERT.png)

简单来看，MTR_LOG_INSERTS重做日志就是cursor定位的记录偏移量+插入的记录。为了使redo log尽可能的小（性能），对插入的记录进行优化，比较插入的记录和cursor定位的记录，找出第一个不同的字节位置。这里不需要对record header进行比较，因为record header是在记录插入完成后再进行初始化的。cursor定位的物理记录如图所示：

![InnoDB_redo_log_MLOG_REC_INSERT_example](/InnoDB_redo_log_MLOG_REC_INSERT_example.png)

上述的例子中cursor定位的记录为（1，'aaa'），那么当要插入的记录为（2，'bbb'）时，可以进行压缩。由于插入记录的长度与之前cursor定位的记录长度相同，因此不需要保存到重做日志。此外，列row id的前三个字节也都相同，都是80 00 00，因此同样不需要保存。故重做日志仅需保存插入记录的第13个字节之后（4+6+3）开始的内容。

在重做日志中，extra_info_flag为1的判断依据为cursor定位的记录与插入记录的info_bits、extra_size、记录大小是否相等。其在源码中为：

````c++

if ((rec_get_info_bits(insert_rec) != rec_get_info_bits(cursor_rec))
   || (extra_size != cur_extra_size)
   || (rec_size != cur_rec_size)) {
   extra_info_yes = 1;
} else {
   extra_info_yes = 0;
}
````

记录插入通常是将逻辑记录插入到页中，但在函数page_copy_rec_list_*中，插入的记录是物理记录。这是因为这些操作是对页中已存在的记录进行"整理"。而在函数page_copy_rec_list_中将物理记录插入的重做日志定义为MTR_LOG_SHORT_INSERTS。这可以理解为一种较为“快速”的插入。其和MTR_LOG _INSERT不同之处在于不需要在重做日志中保存cursor指向记录的偏移量，因为这是将页中的所有记录插入到新页中的。

在记录插入完成后，需要对page directory进行维护，看是否需要对槽进行平衡操作。

## 删除记录

page_cur_delete_rec将cursor所指向的物理记录进行"彻底"地删除，而并不是将记录的delete flag设置为1。具体流程如下

1. 记日志
2. PAGE_LAST_INSERT置空
3. 找到前后记录
4. 更新记录双向链表
5. 更新page directory
6. 更新n_owned
7. 释放记录（PAGE_FREE+，PAGE_GARBAGE+，PAGE_N_RECS-）
8. 如果slot的n_owned小于4，balance slot

相对于插入，删除记录的重做日志就显得简单多了，其只需额外地记录删除记录在页中的偏移量即可。重做日志的结构如图所示：

![InnoDB_redo_log_MLOG_REC_DELETE](/InnoDB_redo_log_MLOG_REC_DELETE.png)

## 并发控制

对于索引页的并发控制是在上层的调用中进行的，主要集中在btr模块中。在InnoDB中，btr模块负责对于B+树索引的控制，其中需要涉及索引页的并发控制以及锁信息的管理。

page模块是对索引页进行操作的最底层函数。InnoDB本身不直接调用该模块中的函数。而且page模块操作页的数据结构为page_t，也就是字节。因此在该模块中并不涉及并发的控制，在源码中也看不到任何对于索引页加s-latch或者x-latch的过程。函数page_copy_rec_list_*中有需要对页上的锁信息进行更新，但这并不是对页进行并发保护。

## 页面重组

我们知道，对于数据的插入，InnoDB只检查碎片链表的第一个可重用空间。另外，如果频繁的插入删除记录，则会加大碎片的产生，最终可能所有的碎片区空间加起来可以存储很多数据，但却无法将其使用起来。

页面的碎片过多，会造成以下问题：

- 页面比较多，但实际用于索引的数据较少
- 大量的碎片导致需要读取更多的页面，产生大量的无效IO
- 数据库整体性能变差

页面重组（re-organize）的功能，即在页空间不足时会首先对页进行重新组织（btr_page_reorganize_low），按照页中的记录主键顺序重新整理，回收碎片空间，详细流程如下：

1. 新建一个page
2. 将碎片页面的数据一条一条的插入到新的page
3. 将旧页面释放

这样新页面中的所有数据都是连续插入进去的，空间完全没有浪费，最后一条记录后面的剩余空间（可用空间）都在PAGE_HEAP_TOP下管理。
---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: record
---

# 设计

## 行存

MySQL主要面向的是OLTP场景，所以InnoDB中存储的数据采用行存（NSM - n-ary storage model）。

基于行进行存储有以下几个好处：

- 记录存放在一个页中，存储一条记录需要访问的页面较少
- 符合传统机械硬盘的访问方式
- 易于理解，数据的存取就像是对一张二维表进行访问

在整体上看，表中的数据是按照如下形式组织的：

![InnoDB record表中的行](/InnoDB_record表中的行.png)

## 记录

那我们如何来理解记录呢？

首先，在关系数据库系统理论中，通常用元组（tuple）描述记录，用字段（field）描述列，每个tuple由多个field组成，每个表由多个tuple组成。

行和tuple在意义上是相等的。但是更愿意将行（row）理解为物理记录，将元组（tuple）理解为逻辑记录。物理记录为行实际存放在物理存储中的格式，其内容由二进制字符串组成，可读性差。逻辑记录则容易理解的多，每张表中的多条记录就像是一个数组。由于其只是“逻辑”上的含义，因此逻辑记录只是物理记录在内存中的表现形式，实际并不占用任何的物理存储空间。

关系如下图所示：

![InnoDB record 记录的展现](/InnoDB_record记录的展现.png)

物理记录和逻辑记录的差异如下：

|          | 物理记录                        | 逻辑记录                             |
| :------- | :------------------------------ | :----------------------------------- |
| 可读性   | 差                              | 好                                   |
| 存储位置 | 磁盘                            | 内存                                 |
| 亲和性   | 对存储友好（更紧凑）            | 对查找友好（更易寻址）               |
| 存储内容 | 记录中各个列的数据+一些额外信息 | 元组（用于比较、展现而组织的列数据） |

这两种记录之间可以互相转换。比如，在插入一条记录时，原来没有数据，首先需要根据插入的记录构造一个逻辑记录，然后再转换成物理记录存放到磁盘上。对于读取，要从磁盘上seek出相应的数据页，再将页中的物理记录转换成逻辑记录展现给用户。

除此之外，在MySQL server层需要在binlog中记录数据的变化，这也是一种行格式（RBR-logging）。因此，在MySQL中，行格式一共有3种存储方式：

- Server层格式：与存储引擎无关，server层的binlog行格式（Row-Base Replication下的binlog格式）
- 逻辑记录格式：tuple，也称为索引元组格式（因为InnoDB是IOT）。在同一个表中，不同索引对应的元组是不同的
- 物理记录格式：record，也称为physical record

## 物理记录的设计

物理记录承载着数据的最终存储，因此，我们首先讨论物理记录。

磁盘上的物理记录需要面向计算机友好，更紧凑、强调IO性能，以及在此基础上支持事务语义。

目标：

- 描述行存的数据
- 适配存储引擎的结构
- 查找快
- DML快
- 事务语义
- 更少的资源占用（disk、buffer pool、update I/O）

列存采用directory+meta+data的形式组织：

![InnoDB record physical layout design](/InnoDB_record_physical_layout_design.png)

其中元信息（meta）按照以下维度的场景分类：

![InnoDB record行存的元信息](/InnoDB_record行存的元信息.png)

从上图中，我们按照从大到小的维度罗列了需要用到的数据，可以看出，有些元信息存储在了列上，有些则存储在了行、页上。

对于directory+meta+data来说，具体的行存的physical storage layout（这里以old style格式为例）：

- offset：变长，取决于列的数量（F），F*1/F*2，因此MySQL规定varchar的最大长度为65535（216=65536）
- extra info：定长，48 bit
- 数据：变长，取决于列的数量

列的表现形式：

- 物理：record
- 内存：tuple（没有事务信息，因为可以从物理中构建出来，并且集中存储在trx_sys中）

InnoDB在行的存储格式上进行过的优化：

- 存储更高效（节省20%）
  - NULL：not-NULL column list+ NULL bitmap（NULL char/varchar都不占用空间）
  - extern：列长超过127才需要extern marker
  - 无需n_fields，通过null bitmap+非空变长列表（2F）可以算出
- workload如果在cache & disk IO上，则更高效，如果是CPU-bound，则会慢
-  查找
  - next record：绝对位置→相对位置（减少record re-org的开销）
  - \+ record type
- 兼容性
  - 版本标识+启动检查+参数控制
- 版本
  - 数据页上进行标识

## 逻辑记录的设计

磁盘上的物理记录面向的是计算机友好的。同时，为了性能的考虑，数据也需要常驻内存（buffer pool），所以需要设计相应的内存态数据结构用于表述逻辑记录。

并且，考虑到InnoDB使用的是IOT（B+树），还需要考虑数据在B+树的表现形式：

- 非叶子节点：node pointer，由（key, page no）组成
- 叶子节点：data，或（key, PK）

从这里看出，tuple根据不同的index，所存储的数据也各不相同。

我们从磁盘上读取物理记录后，会将其转换为逻辑记录，用于比较。同时，用户的输入由于需要持久化，也需要转换为物理记录，并且，用户的输入有虚拟化（constant, as）的需求，需要额外的列来描述（虚拟化列）。

![InnoDB_physical_record_物理记录和逻辑记录的转换](/InnoDB_physical_record_物理记录和逻辑记录的转换.png)

因此，我们在tuple设计上，需要考虑：

- 可比较性
- 虚拟列
- 内存布局的友好性（layout、cacheline、大记录）

## 数据表示&比较

记录由列组成，列需要具有类型+属性。

数据表示：

**data representation**

- integer/bigint/smallint/tinyint
  - C/C++ representation
- float/real vs numeric/decimal
  - IEEE-754 standard / fixed-point decimals
- varchar/varbinary/text/blob
  - header with length, followed by data types (may add checksum)
- time/date/timestamp
  - 32/64-bit integer of (micro) seconds since Unix epoch (most)

数据比较则需要考虑：

- 二进制数据（是否可以按内存序比较）
- 字符串（字符集/校对字符集 bin/ci/cs）
- NULL（minimum）：被视为最小值，任何与NULL进行的等值比较返回的值都是NULL

## 并发控制

并发控制有两个层次：

- 内存物理结构的一致
- 事务语义的一致

首先看一下内存物理结构的一致：在B+树中，access path是从树顶到叶子节点的，所以concurrency control只需要对B+树（index tree + index page）以及叶子节点（data page）进行latch并发控制（SMO & FIX）即可。对于存储在data page内部的行，无需关心latch并发控制。

事务语义的一致，则是通过悲观（locking）或乐观（TSO、Multi-version）的方式保证的。在MySQL中，是通过MV2PL protocol的方式来实现的。MySQL为了控制锁资源，通过在锁对象内部的page bitmap的方式实现行锁以及type_mode实现谓词锁。

## MVCC

根据不同的隔离级别，可以读到不同版本的数据，即通过readview确定可视范围（trxID），并提供版本管理（version storage：delta/append only）、GC。

# 物理记录的实现

InnoDB中的表使用的是索引组织表（**I**ndex **O**rganized **T**able - IOT），这意味着表中的所有数据是按照B+树的方式进行存储的，行数据存在在B+树的叶子节点上，即使创建表时没有显式指定主键索引，也会自动创建一个6字节的隐藏列，用作主键索引（用于unique）。

此外，InnoDB为了支持事务和多版本并发控制（MVCC），所以每行记录还有一个回滚指针列和记录事务ID的列，这两列都是隐藏列，对用户不可见。回滚指针用来构造当前记录的上一个版本，实现事务回滚和MVCC，事务ID用于判断当前记录对于其他事务是否可见，用来实现事务的隔离性和MVCC。

以上三个系统隐藏列仅存在于聚簇索引的物理记录中，逻辑记录不含有这些信息，逻辑记录本身只保存每个元组中的实际内容。

<font color=red>物理记录存储在页中，但是每个页存放的行记录也是有硬性定义的，最多允许存放16KB/2-200行的记录，即7992行记录。</font>

{{< hint info >}}

<font color=red>每页最少、最多放多少行

每个数据页中的行数可以通过以下方式计算：

row storage layout中的heap_no：13bit，即213=8192，那么16KB的页中每行最少用16*1024/8192 = 2 bytes 存储。

每行最多有多少列</font>

{{</hint>}}

在实际的InnoDB工程实践中，为了存储格式的演进，需要对行格式进行修改。同时，考虑到兼容性（backward、foward），由此提出了named file format的概念。

InnoDB 1.0.x之前，提供了**Redundant**和**Compact**两种格式来存储行数据：5.0之前的格式是Redundant，5.0引入了Compact，二者也被称为old style和new style，同时又统称为Antelope文件格式。

InnoDB 1.0.x之后，采用了新的行存储格式（row storage layout），即Barracuda文件格式，新的文件格式兼容之前版本的格式，即包括Antelope文件格式，并加入了2种新的记录存储格式：**Dynamic**（更有效率的off-page column、larger index key prefix（767→ 3072））和支持记录压缩的**Compressed**。

{{< hint info >}}
InnoDB中的数据压缩分为三种：

- 页压缩
- 透明页压缩
- 行压缩

{{</hint>}}

这2种文件格式以及行格式的关系如下图所示：![InnoDB_physical_record_format](/InnoDB_physical_record_format.png)

页是数据库的基本存储单位，所以在数据页上会标识该页所使用的行格式，PageHeader.PAGE_N_HEAP的最高位（第一个bit）标识该页所使用的行格式为new style还是old style，判断函数为page_is_comp。

这几种行格式的官方说明：

**InnoDB Row Format Overview**

| Row Format   | Compact Storage Characteristics | Enhanced Variable-Length Column Storage | Large Index Key Prefix Support | Compression Support | Supported Tablespace Types      | Required File Format  |
| :----------- | :------------------------------ | :-------------------------------------- | :----------------------------- | :------------------ | :------------------------------ | :-------------------- |
| `REDUNDANT`  | No                              | No                                      | No                             | No                  | system, file-per-table, general | Antelope or Barracuda |
| `COMPACT`    | Yes                             | No                                      | No                             | No                  | system, file-per-table, general | Antelope or Barracuda |
| `DYNAMIC`    | Yes                             | Yes                                     | Yes                            | No                  | system, file-per-table, general | Barracuda             |
| `COMPRESSED` | Yes                             | Yes                                     | Yes                            | Yes                 | file-per-table, general         | Barracuda             |

从上图可以看出，行格式主要的区别是：

- REDUNDANT：中古版本
- COMPACT：更紧凑
- DYNAMIC：更有效率的off-page column（768->127）、index key prefix支持范围更大（767 → 3072，[innodb_large_prefix](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix)）
- COMPRESSED：更高的存储效率，对off-page column进行压缩（lz4）

这些版本的兼容性过渡（downgrade）通过以下参数提供，MySQL 5.7后会废弃这些参数（当时是为了5.1的兼容性，现在5.1生命周期已经结束，所以不再需要这些参数）：

- [innodb_file_format](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_format)
- [innodb_file_format_check](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_format_check)
- [innodb_file_format_max](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_file_format_max)
- [innodb_large_prefix](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix)

**Backward Compatibility**

The only way to “downgrade” an InnoDB tablespace to the earlier Antelope file format is to copy the data to a new table, in a tablespace that uses the earlier format.

Beginning with version InnoDB 1.0.1, the system tablespace records an identifier or tag for the “highest” file format used by any table in any of the tablespaces that is part of the ib-file set. Checks against this file format tag are controlled by the configuration parameter innodb_file_format_check, which is ON by default.
The ability to set innodb_file_format_check is useful (with future releases) if you manually “downgrade” all of the tables in an ib-file set. You can then rely on the file format check at startup if you subsequently use an older version of InnoDB to access the ib-file set.

首先来看Redundant格式。

## Redundant行格式

从上面我们可以得知，物理记录由3部分组成：

- record directory：通过column offset list存储每列的长度（len+data）
- extra info：行的元信息
- 行数据：按列存储

InnoDB依靠column offset list和extra info以二进制的形式从数据页中完整的读取一行物理记录，如果需要，还需要将其转化为逻辑记录。其中，实际存储的第一列的位置称为original offset，物理记录总指向这个位置，而非物理记录实际的开始位置。

首先是column offset list，其根据列的顺序存放，每列的offset都是之前offset的累加（从后往前累加），并且每列offset的前两个bit标识该列是否为空（NULL），以及是否为溢出列（EXTERN）。![InnoDB_physical_record_old-style](/InnoDB_physical_record_old-style.png)

extra info一共使用了48 bit（6个字节），字段含义见下：

| 字段                | 大小（bit） | 说明                                                         |
| :------------------ | :---------- | :----------------------------------------------------------- |
| info_bits           | 4           | info_bits信息                                                |
| n_owned             | 4           | page directory的slot所指向的记录所拥有的记录数量，即槽所代表的区间的记录数量 |
| heap no             | 13          | 记录在堆中的序号（伪记录 0 1，用户记录从2开始，顺序分配，保证其不变性） |
| n_fields            | 10          | 记录中列的数量                                               |
| offset 1byte or not | 1           | col offset list为1个字节还是2个字节，如果列长度小于127个字节，用1个字节表示，否则用2个字节表示 |
| next record         | 16          | 指向下一个记录的original offset位置                          |

除了info_bits，extra info中的其他信息都不存放在逻辑记录中（不需要，因为描述的是physical layout信息）。

info_bits内容如下：

| 字段         | 大小（bit） | 说明                           |
| :----------- | :---------- | :----------------------------- |
| 字段         | 大小（bit） | 说明                           |
| /            | 1           | 未使用                         |
| /            | 1           | 未使用                         |
| deleted_flag | 1           | 记录删除标记                   |
| min_rec_flag | 1           | B+树中非叶子节点的最小记录标记 |

这里稍微展开一下，InnoDB采用的是索引组织表的形式（B+树）组织数据，也就是说，记录是根据B+树的规则进行存放的，记录存放在页中。但是页中的记录并不是根据索引规则进行排序的，因为如果这样组织行，对行进行数据变更会因为要保持顺序性而不断的调整位置，这样的开销过于巨大。于是，页中将行组织成堆的形式，即页中记录的存放是无序的，heap no标识在页堆中记录的序号，记录和记录之间通过next_record进行逻辑顺序的串联。另外，heap no也用于实现行锁（即行锁以页的方式组织，内部通过heap no的bitmap来标识行锁信息），在锁的介绍中再进行详细说明。

在B+树的非叶子节点（非最下层）用最左节点（leftmost）来确定每层的左边界（predefined minimum record），即在非叶子层的第一条用户记录时设置该bit（REC_INFO_MIN_REC_FLAG）。

每层的左边界判断逻辑为：

```
PAGE_LEVEL != 0 && FIL_PAGE_PREV == FIL_NULL
```

使用场景为：

- B+树split/merge
- 收集index的统计信息（first n_prefix columns）
- GIS

下面我们看一个实际的例子，来直观的了解数据的物理存储：

```
create table t1 (
    c1 varchar(10),
    c2 varchar(10),
    c3 char(10),
    c4 varchar(10)
) ROW_FORMAT=REDUNDANT;
 
insert into t1 values ('a', 'bb', 'bb', 'ccc');
insert into t1 values ('d', 'ee', 'ee', 'fff');
insert into t1 values ('d', NULL, NULL, 'fff');
```

hexdump -c -V t1.ibd

根据物理记录的格式，可以将hexdump整理为：

```
Redundant
// ROW 1
37 34 16 14 13 0c 06            // col offset list （逆序） 3 | 30 | 2 | 1 | 7 | 6 | 6
00 00 10 0f 00 ce               // extra info（定长 6字节 48 bit）
00 00 00 00 02 03               // ROW ID（定长 6字节）
00 00 00 00 05 22               // Trx ID（定长 6字节）
ba 00 00 01 2e 01 10            // 回滚指针（定长 7字节）
61                              // 1字节
62 62                           // 2字节
62 62 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 // 30字节
63 63 63                        // 3字节
 
// ROW 2
37 34 16 14 13 0c 06            // col offset list （逆序） 3 | 30 | 2 | 1 | 7 | 6 | 6
00 00 18 0f 01 12               // extra info（定长 6字节 48 bit）
00 00 00 00 02 04               // ROW ID（定长 6字节）
00 00 00 00 05 23               // Trx ID（定长 6字节）
bb 00 00 01 2f 01 10            // 回滚指针（定长 7字节）
64                              // 1字节
65 65                           // 2字节
65 65 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 // 30字节
66 66 66                        // 3字节
 
// ROW 3
35 b2 94 14 13 0c 06            // col offset list （逆序） 3 | 30 | N | 1 | 7 | 6 | 6
00 00 20 0f 00 74               // extra info（定长 6字节 48 bit）
00 00 00 00 02 05               // ROW ID（定长 6字节）
00 00 00 00 05 28               // Trx ID（定长 6字节）
be 00 00 01 31 01 10            // 回滚指针（定长 7字节）
64                              // 1字节
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 // 30字节
66 66 66                        // 3字节
```

从中我们可以看出：

- varchar列如果为NULL，则不占用任何实际的存储空间
- char列不管是否为NULL都需要padding补齐全部空间（填充0）。这是为了避免char由NULL改为not-NULL而引发page re-org，只需要调整column offset list中的NULL bit。另外，根据字符集的不同，char类型的实际存储长度也不同：如果字符集为latin1，则每个字符占1个字节，如果字符集为utf8，第三列总长则为10*3 = 30字节。从这里可以看出，char类型会按照规定长度（定义）存储，而不是按照实际数据长度存储。

column offset list的layout布局如下：![InnoDB_physical_record_old-style part1](/InnoDB_physical_record_old-style_part1.png)

展开extra info：![InnoDB_physical_record_old-style_part2](/InnoDB_physical_record_old-style_part2.png)

从这里可以看到，3个记录的heap no依次为2、3、4，这是因为在每个页中都有两个虚拟的伪记录（REC_STATUS_INFIMUM和REC_STATUS_SUPREMUM），其heap no分别是0和1。因此用户记录总是从heap no = 2开始。这两个虚拟的伪记录也起到边界的作用，关于虚拟的伪记录在索引中有详细介绍。

n_fields标识有7列，由3列系统列和4列用户列组成。

next_record标识了本记录相对于下一个记录的偏移量，可以想象成一个小兔子，next_record标识了下一个记录从页头要跳跃的长度（相对于页的绝对偏移量）。

## Compact行格式

Compact行格式如下图所示：![InnoDB_physical_record_new-style](/InnoDB_physical_record_new-style.png)

Compact行格式和Redundant行格式的主要区别如下：

1. 不再存储系统列的元信息（ROW ID、Trx ID、回滚指针）
2. 列的NULL值单独提出来一个NULL标志位（CEILING(N/8) bytes），每列1bit表示，varchar/char的NULL值都不再占用存储空间
3. EXTERN flag只有当字段列表为2个字节时才标识，列长小于127字节没有意义；去掉了1bytes_off_flag，每个字段采用1个字节还是2个字节通过col的第一个bit可以表示
4. 增加了record_type：000=conventional（叶子节点，总在第0层）, 001=node pointer （B+树非叶子节点指针记录）, 010=infimum, 011=supremum, 1xx=reserved
5. 去掉了n_fields，其通过非NULL字段列表（2F）和NULL标记位（NULL bitmap）可以算出列的数量
6. next record采用相对偏移量代替绝对偏移量

在构建内存tuple的时候，tuple→info_bits会保存，同样，record_type中的node pointer也会在构建时设置到其上：

````c++
dict_index_build_node_ptr
    dtuple_set_info_bits(tuple, dtuple_get_info_bits(tuple)
                 | REC_STATUS_NODE_PTR);
````

那我们接着用之前例子，看一下在Compact行格式下的physical layout：

````
Compact
// ROW 1
03 0a 02 01                     // 非空变长字段列表（N=4, varchar index = [03 02 01] char len = 10）
00 00 00 10 00 2d               // extra info（定长 5字节 + NULL bitmap（8 bit））
00 00 00 00 02 00               // ROW ID（定长 6字节）
00 00 00 00 05 11               // Trx ID（定长 6字节）
ae 00 00 01 22 01 10            // 回滚指针（定长 7字节）
61                              // 1字节
62 62                           // 2字节
62 62 20 20 20 20 20 20 20 20   // 10字节
63 63 63                        // 3字节
 
// ROW 2
03 0a 02 01                     // 非空变长字段列表（N=4, varchar index = [03 02 01] char len = 10）
00 00 00 18 00 2b               // extra info（定长 5字节 + NULL bitmap（8 bit））
00 00 00 00 02 01               // ROW ID（定长 6字节）
00 00 00 00 05 12               // Trx ID（定长 6字节）
af 00 00 01 23 01 10            // 回滚指针（定长 7字节）
64                              // 1字节
65 65                           // 2字节
65 65 20 20 20 20 20 20 20 20   // 10字节
66 66 66                        // 3字节
 
// ROW 3
03 01                           // 非空变长字段列表（N=2, varchar index = [03 01]）
06 00 00 20 ff 96               // extra info（定长 5字节 + NULL bitmap（8 bit））
00 00 00 00 02 02               // ROW ID（定长 6字节）
00 00 00 00 05 17               // Trx ID（定长 6字节）
b2 00 00 01 26 01 10            // 回滚指针（定长 7字节）
64                              // 1字节
66 66 66                        // 3字节
````

首先看一下行数据的存储：

````
Redundant
// ROW 1
61
62 62
62 62 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20
63 63 63
 
// ROW 2
64
65 65
65 65 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20
66 66 66
 
// ROW 3
64
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
66 66 66
 
Compact
// ROW 1
61
62 62
62 62 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20
63 63 63
 
// ROW 2
64
65 65
65 65 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20
66 66 66
 
// ROW 3
64
66 66 66
````

从上面可以看出，第三行记录显著节省了空间。

首先是非空可变字段列表：![InnoDB_physical_record_new-style_part1](/InnoDB_physical_record_new-style_part1.png)

接着是extra info：![InnoDB_physical_record_new-style_part2](/InnoDB_physical_record_new-style_part2.png)

获取extra info采用位操作：![InnoDB_physical_record_new-style_get_extra_info](/InnoDB_physical_record_new-style_get_extra_info.png)

## 逻辑记录和物理记录的转换

逻辑记录和物理记录之间的转换，通过offsets[]数组来进行：![InnoDB_record_tuple_convert](/InnoDB_record_tuple_convert.png)

详细的函数注释如下：

````c++
/*********************************************************//**
Builds a new-style physical record out of a data tuple and
stores it beginning from the start of the given buffer.
@return pointer to the origin of physical record */
static
rec_t*
rec_convert_dtuple_to_rec_new(
/*==========================*/
    byte* buf, /*!< in: start address of the physical record */
    const dict_index_t* index, /*!< in: record descriptor */
    const dtuple_t* dtuple) /*!< in: data tuple */
{
    ulint extra_size;   // 计算extra_size（not-null dynamic col list + extra info）
    ulint status;
    rec_t* rec;         // 指向实际的行数据（original offset）
 
    status = dtuple_get_info_bits(dtuple) & REC_NEW_STATUS_MASK;
    rec_get_converted_size_comp(index, status, dtuple->fields, dtuple->n_fields, &extra_size);
    rec = buf + extra_size;
 
    rec_convert_dtuple_to_rec_comp(rec, index, dtuple->fields, dtuple->n_fields, NULL,status, false); // 逻辑记录转换为物理记录
 
    /* Set the info bits of the record */
    rec_set_info_and_status_bits(rec, dtuple_get_info_bits(dtuple));
 
    return(rec);
}
 
/*********************************************************//**
Builds a ROW_FORMAT=COMPACT record out of a data tuple. */
UNIV_INLINE
void
rec_convert_dtuple_to_rec_comp(
/*===========================*/
    rec_t* rec, /*!< in: origin of record */
    const dict_index_t* index, /*!< in: record descriptor */
    const dfield_t* fields, /*!< in: array of data fields */
    ulint n_fields, /*!< in: number of data fields */
    const dtuple_t* v_entry, /*!< in: dtuple contains virtual column data */
    ulint status, /*!< in: status bits of the record */
    bool temp) /*!< in: whether to use the format for temporary files in index creation */
{
    const dfield_t* field;
    const dtype_t* type;
    byte* end;
    byte* nulls;
    byte* lens;
    ulint len;
    ulint i;
    ulint n_node_ptr_field;
    ulint fixed_len;
    ulint null_mask = 1;
    ulint n_null;
    ulint num_v = v_entry ? dtuple_get_n_v_fields(v_entry) : 0;
 
    ut_ad(temp || dict_table_is_comp(index->table));
 
    if (temp) {
        ut_ad(status == REC_STATUS_ORDINARY);
        ut_ad(n_fields <= dict_index_get_n_fields(index));
        n_node_ptr_field = ULINT_UNDEFINED;
        nulls = rec - 1;
        if (dict_table_is_comp(index->table)) {
            /* No need to do adjust fixed_len=0. We only
            need to adjust it for ROW_FORMAT=REDUNDANT. */
            temp = false;
        }
    } else {
        ut_ad(v_entry == NULL);
        ut_ad(num_v == 0);
        nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);
 
        switch (UNIV_EXPECT(status, REC_STATUS_ORDINARY)) {
        case REC_STATUS_ORDINARY:
            ut_ad(n_fields <= dict_index_get_n_fields(index));
            n_node_ptr_field = ULINT_UNDEFINED;
            break;
        case REC_STATUS_NODE_PTR:
            ut_ad(n_fields
                  == dict_index_get_n_unique_in_tree_nonleaf(index)
                 + 1);
            n_node_ptr_field = n_fields - 1;
            break;
        case REC_STATUS_INFIMUM:
        case REC_STATUS_SUPREMUM:
            ut_ad(n_fields == 1);
            n_node_ptr_field = ULINT_UNDEFINED;
            break;
        default:
            ut_error;
            return;
        }
    }
 
    end = rec;
 
    if (n_fields != 0) {
        n_null = index->n_nullable;
        lens = nulls - UT_BITS_IN_BYTES(n_null);
        /* clear the SQL-null flags */
        memset(lens + 1, 0, nulls - lens);
    }
 
    /* Store the data and the offsets */
    for (i = 0; i < n_fields; i++) {             // 将每一列的len、NULL、数据存储到相应的位置
        const dict_field_t* ifield;
        dict_col_t* col = NULL;
 
        field = &fields[i];
 
        type = dfield_get_type(field);
        len = dfield_get_len(field);
 
        // 如果是索引指针，则只用4个字节存储（处理后退出循环），即data = fixed 4 bytes
        if (UNIV_UNLIKELY(i == n_node_ptr_field)) {
            ut_ad(dtype_get_prtype(type) & DATA_NOT_NULL);
            ut_ad(len == REC_NODE_PTR_SIZE);
            memcpy(end, dfield_get_data(field), len);
            end += REC_NODE_PTR_SIZE;
            break;
        }
 
        // 处理null bits
        if (!(dtype_get_prtype(type) & DATA_NOT_NULL)) {
            /* nullable field */
            ut_ad(n_null--);
 
            if (UNIV_UNLIKELY(!(byte) null_mask)) {
                nulls--;
                null_mask = 1;
            }
 
            ut_ad(*nulls < null_mask);
 
            /* set the null flag if necessary */
            if (dfield_is_null(field)) {
                *nulls |= null_mask;
                null_mask <<= 1;
                continue;
            }
 
            null_mask <<= 1;
        }
        /* only nullable fields can be null */
        ut_ad(!dfield_is_null(field));
 
        ifield = dict_index_get_nth_field(index, i);
        fixed_len = ifield->fixed_len;
        col = ifield->col;
        if (temp && fixed_len
            && !dict_col_get_fixed_size(col, temp)) {
            fixed_len = 0;
        }
 
        /* If the maximum length of a variable-length field
        is up to 255 bytes, the actual length is always stored
        in one byte. If the maximum length is more than 255
        bytes, the actual length is stored in one byte for
        0..127. The length will be encoded in two bytes when
        it is 128 or more, or when the field is stored externally. */
        if (fixed_len) {                            // 定长，则不存储长度
        } else if (dfield_is_ext(field)) {          // off-page column，需要2个字节存储长度
            ut_ad(DATA_BIG_COL(col));
            ut_ad(len <= REC_ANTELOPE_MAX_INDEX_COL_LEN
                  + BTR_EXTERN_FIELD_REF_SIZE);
            *lens-- = (byte) (len >> 8) | 0xc0;
            *lens-- = (byte) len;
        } else {
            /* DATA_POINT would have a fixed_len */
            ut_ad(dtype_get_mtype(type) != DATA_POINT);
            ut_ad(len <= dtype_get_len(type)
                  || DATA_LARGE_MTYPE(dtype_get_mtype(type))
                  || !strcmp(index->name,
                     FTS_INDEX_TABLE_IND_NAME));
            if (len < 128 || !DATA_BIG_LEN_MTYPE(    // 变长列，长度<125，使用1个字节存储长度
                dtype_get_len(type), dtype_get_mtype(type))) {
 
                *lens-- = (byte) len;
            } else {                                // 变长列，长度>=125，使用2个字节存储长度
                ut_ad(len < 16384);
                *lens-- = (byte) (len >> 8) | 0x80;
                *lens-- = (byte) len;
            }
        }
 
        memcpy(end, dfield_get_data(field), len);
        end += len;
    }
 
    if (!num_v) {
        return;
    }
 
    /* reserve 2 bytes for writing length */
    byte* ptr = end;
    ptr += 2;
 
    /* Now log information on indexed virtual columns */
    for (ulint col_no = 0; col_no < num_v; col_no++) {
        dfield_t* vfield;
        ulint flen;
 
        const dict_v_col_t* col
            = dict_table_get_nth_v_col(index->table, col_no);
 
        if (col->m_col.ord_part) {
            ulint pos = col_no;
 
            pos += REC_MAX_N_FIELDS;
 
            ptr += mach_write_compressed(ptr, pos);
 
            vfield = dtuple_get_nth_v_field(
                v_entry, col->v_pos);
 
            flen = vfield->len;
 
            if (flen != UNIV_SQL_NULL) {
                /* The virtual column can only be in sec
                index, and index key length is bound by
                DICT_MAX_FIELD_LEN_BY_FORMAT */
                flen = ut_min(
                    flen,
                    static_cast<ulint>(
                    DICT_MAX_FIELD_LEN_BY_FORMAT(
                        index->table)));
            }
 
            ptr += mach_write_compressed(ptr, flen);
 
            if (flen != UNIV_SQL_NULL) {
                ut_memcpy(ptr, dfield_get_data(vfield), flen);
                ptr += flen;
            }
        }
    }
 
    mach_write_to_2(end, ptr - end);
}
````

物理记录（new style）的构建，如图所示：![InnoDB_physical_record_new-style_encapsulate](/InnoDB_physical_record_new-style_encapsulate.png)

## 大记录格式

如果列过大，一个数据页会存储不下，这样的记录称之为大记录（big record），大列存储的页称为溢出页（overflow page），列称为off-page columns。对于InnoDB来说，只有类型为BLOB和TEXT才可能是off-page column。如果实际的列值少于1000字节，则放在当前页，否则才采用溢出页存储（多个溢出页组成单向链表）。

{{< hint danger>}}

能不能不用page存，而直接采用外部的存储系统存储：这样就脱离了事务控制的范围，无法保证ACID

{{</hint>}}

对于InnoDB表，转为大记录的前提是：

- 当前记录的总字节数 > 空白页可用空间的一半 > 1/2 * page_get_free_space_of_empty()，即8132字节
- 列大于整页（REC_MAX_DATA_SIZE），即16384字节

可以发现，只要满足第一个条件即可。因此在进行插入操作时将大记录中的off-page column的部分数据存放到溢出页中，直到记录占用的字节数小于一半页的大小（8132字节），这种列的属性为extern，在column offset列的偏移量列表中通过REC_2BYTE_EXTERN_MASK标记。

![InnoDB_physical_record_overflow_page_and_off-page_column](/InnoDB_physical_record_overflow_page_and_off-page_column.png)

{{< hint info>}}

一个overflow page只能给一个off-page column使用。这样的设计减少了实现的复杂度，清晰明了。

{{</hint>}}

从上图中可以看到，col 2的数据并不完全存放在记录所在的索引页中。对于extern属性列的存放进行下列“格式化”：仅在当前记录所在的页中存放前127字节，另外20个字节指向溢出页的信息。extern列的格式如下：

| 名称                | 大小（字节） | 说明                                                         |
| :------------------ | :----------- | :----------------------------------------------------------- |
| column prefix       | 127          | extern列前127字节的数据                                      |
| BTR_EXTERN_SPACE_ID | 4            | off-page的SPACE ID                                           |
| BTR_EXTERN_PAGE_NO  | 4            | off-page的page no                                            |
| BTR_EXTERN_OFFSET   | 4            | off-page中记录存放的开始位置                                 |
| BTR_EXTERN_LEN      | 8            | 存放在off-page的字节数（前2个bit保留：BTR_EXTERN_OWNER_FLAG + BTR_EXTERN_INHERITED_FLAG） |

虽然BTR_EXTERN_LEN的长度为8个字节，但是实际存储列长度仅占用后4个字节，所以BLOB字段最大支持4GB（2 * 231），前4个字节（只用到第一个字节）用于存放其他的extern属性信息，通过下面的宏进行提取：

````
** The most significant bit of BTR_EXTERN_LEN (i.e., the most
significant bit of the byte at smallest address) is set to 1 if this
field does not 'own' the externally stored field; only the owner field
is allowed to free the field in purge! */
#define BTR_EXTERN_OWNER_FLAG       128
/** If the second most significant bit of BTR_EXTERN_LEN (i.e., the
second most significant bit of the byte at smallest address) is 1 then
it means that the externally stored field was inherited from an
earlier version of the row.  In rollback we are not allowed to free an
inherited external field. */
#define BTR_EXTERN_INHERITED_FLAG   64
````

extern的其他属性包括：

- purge：BTR_EXTERN_OWNER_FLAG：标记对属性为extern的列进行了purge操作
- （除extern列外的其他列更新）删除后回滚：BTR_EXTERN_INHERITED_FLAG：标记当前extern列继承（inherited）一个早先的记录版本。若回滚，则不需要对其进行删除。

{{< hint info>}}

为什么需要设置一个继承标记位呢？这是因为在对大记录进行更新的时候，可能会遇到主键值更新了，但是extern列却没有更新的情况。在这种情况下，InnoDB首先将会对原主键记录进行伪删除（即delete flag设为1），然后插入一条含有新主键的记录。这时，由于extern列没有更新，则只需要将BTR_EXTERN_*（space+page+offset+len）——指向原来的位置即可。但是当事务发生回滚时会删除新插入的记录，将伪删除记录的delete flag重置为0。而由于extern列是继承前一个较早版本的记录，其实并不需要删除，只需要通过BTR_EXTERN_INHERITED_FLAG进行标识。

{{</hint>}}

当需要将记录转化为大记录对象时（dtuple_convert_big_rec），InnoDB首先选择最大长度的列进行extern格式化。如果格式化后当前页中该记录所占的字节数已经小于1/2 * page_get_free_space_of_empty()时，那么完成转化，否则重新再选择一个列进行extern格式化，直到满足条件。

下面结合一个实际的例子来看一下溢出页：

````
create table big_rec_t (
    a int not null,
    b BLOB,
    primary key(a)
)engine=InnoDB;
 
insert into big_rec_t values (1, 'David');
insert into big_rec_t values (2, repeat('y',1000));
insert into big_rec_t values (3, repeat('z',10000));
````

前2条记录的长度没有超过1/2 * page_get_free_space_of_empty()，因此记录还是存储在当前数据页中，第3条记录在当前数据页中仅存放前127个字节，其余9873个字节存放在off-page中。

````
// 第3条记录
40 a4 00 11 00 0a 00 04     // column offset list 4列 逆序（变长，因为列的总长度超过127字节，所以每列offset采用2个字节存储）
00 00 20 08 00 74           // extra info（定长 6字节）
80 00 00 03                 // ROW ID（定长 4字节）
00 00 00 00 27 09           // Trx ID（定长 6字节）
80 00 00 00 2d 00 84        // 回滚指针（定长 7字节）
7a 7a 7a .. 7a              // 第4列的前127个字节
00 00 00 00                 // BTR_EXTERN_SPACE_ID
00 00 00 35                 // BTR_EXTERN_PAGE_NO
00 00 00 26                 // BTR_EXTERN_OFFSET
00 00 00 00 00 00 00 26 91  // BTR_EXTERN_LEN
````

4列的长度按照顺序排列后依次是0x0004、0x000a、0x0011、0x40a4，最后一列的属性是extern，随意最后一列的offset实际上是 0x00a4 & REC_2BYTE_EXTERN_MASK得到的值。通过0x00a4 - 0x0011可以计算出第4列的长度为147（127 + 20），其中前127个字节为第4列的前缀，后20个字节为off-page的信息。通过BTR_EXTERN_LEN可以计算出存放在off-page中的数据为9873 bytes（0x2691）。通过BTR_EXTERN_PAGE_NO和BTR_EXTERN_OFFSET可以定位到溢出页：

````
00 00 26 91                 // BTR_BLOB_HDR_PART_LEN
ff ff ff ff                 // BTR_BLOB_HDR_NEXT_PAGE_NO
7a 7a 7a ... 7a             // 第4列的后面9873个字节
````

由于仅用1个off-page就可以存下第4列的溢出数据，因此这里BTR_BLOB_HDR_PART_LEN = 之前BTR_EXTERN_LEN的值，即9873，并且BTR_BLOB_HDR_NEXT_PAGE_NO的值为0xffffffff（EOF）。

大记录对InnoDB的性能会产生影响，例如对于更新、删除操作，其只能进行悲观操作，而这意味着并发性能的下降。这部分在B+树的章节中进行具体的介绍。总之，除非必要，尽量使得表中的记录紧凑，同时尽可能的符合数据库理论的范式要求。

## 伪记录

在索引页中存在两个伪记录，分别为infimum记录（REC_STATUS_INFIMUM）和supremum记录（REC_STATUS_SUPREMUM）。可以将其理解为页中最小和最大的记录，起一个“边界”的作用，并且infimum记录可以优化锁的性能（诣在更新时用于临时保存更新记录上的锁信息。这样做的好处是可以提高更新锁的效率，否则需要先删除老的锁对象然后再创建新的锁对象）。![InnoDB_page-infimum_supremum_record](/InnoDB_page-infimum_supremum_record.png)

# 逻辑记录的实现

## 逻辑记录的内存布局

上面已经提到，逻辑记录（dtuple_t）存放在内存中，每个逻辑记录内部又包含多个字段（dfield_t），逻辑记录在内存中的布局如下图所示：![InnoDB_record_tuple](/InnoDB_record_tuple.png)

dtuple_t对象：

| 变量         | 说明                                                         |
| :----------- | :----------------------------------------------------------- |
| 变量         | 说明                                                         |
| info_bits    | 同物理记录（delete flag、leftmost）                          |
| n_fields     | 记录中的列数量                                               |
| n_fields_cmp | 在记录进行比较时，仅比较这些数量的列在索引中进行搜索时，仅判断这些数量的列，默认和n_fields相同 |
| fields       | 记录中的列，array of <d_field_t>                             |
| n_v_fields   | 记录中的虚拟列数量                                           |
| v_fields     | 记录中的虚拟列，array of <d_field_t>                         |
| tuple_list   | 多个记录的链表节点                                           |
| magic_n      | magic number                                                 |

dfield_t对象：

| 变量             | 说明                  |
| :--------------- | :-------------------- |
| data             | 列的值                |
| ext:1            | 是否为off-page column |
| spatial_status:2 | GIS索引使用           |
| len              | 列的长度              |
| type             | 列的类型（dtype_t）   |

## 构建逻辑记录

dtuple_t从mem_heap_t上alloc（dtuple_t + n * dfield_t）个字节，所以内存布局上是连续的，其中dtuple_t位于&tuple[0]，field地址位于&tuple[1]，如图所示：

![InnoDB_record_tuple_layout](/InnoDB_record_tuple_layout.png)

列的类型(（dfield_t.type）dtype是对两个记录进行比较的判断基础。

对于大记录，不用dtuple_t表示，而是用big_rec_t表示：

![InnoDB_record_big_logical_record](/InnoDB_record_big_logical_record.png)

big_rec_t比dtuple_t简单很多，只是多了一个变量heap，该变量的作用为申请一个内存堆之后对big_rec_field_t对象中的data都从该内存堆中进行分配。其余变量和之前介绍相同。

dtuple_convert_big_rec用于将逻辑记录转化为大记录的格式，当转化完成时，之前的dfield_t只记录列的前127个字节，其余的数据放到big_rec_field_t.data中，即函数调用会首先对dtuple_t的列进行修改，同时又生成了新的big_rec_t对象。同样，可以将大记录对象重新转化为普通逻辑记录，由dtuple_convert_back_big_rec完成。

# 记录之间的比较

InnoDB在通过B+树搜索时，通过B+树索引只能定位到记录所在的页，不能直接定位到具体的记录（行）。在找到页后，还需要通过二叉查找算法进行搜索，最终定位到查询的记录。因此，在查找时需要对记录进行比较。

记录的比较可以分为逻辑记录与物理记录之间的比较，以及物理记录之间的比较。通常来说，一般进行的都是逻辑记录和物理记录的比较，这是因为，对于插入操作，本身就不存在物理记录，所以需要构造一个逻辑记录，而对于UPDATE、DELETE操作，首先需要通过SELECT进行定位，这时就会转化为逻辑记录。

物理记录之间的比较通常用于对索引caridinality（索引散列度）的统计，即统计索引中唯一记录的数量。InnoDB会随机选择一些页，然后对页中的记录进行逐个比较后进行统计，这时需要物理记录之间进行比较。

即比较矩阵为：

|          | 逻辑记录      | 物理记录                 |
| :------- | :------------ | :----------------------- |
| 逻辑记录 | update/delete | insert                   |
| 物理记录 | insert        | 索引统计（caridinality） |

记录在进行比较时，实际上是根据列进行比较，而根据列的不同类型（dtype_t），比较的方式也不相同。

dtype_t的定义如下：

```
struct` `dtype_t {
  ``ulint mtype:8;     ``// main data type
  ``ulint prtype:32;    ``// precise type; MySQL data type
  ``ulint len:16;      ``// maximum byte length of the string data
  ``ulint mbminmaxlen:5;  ``// 字符集存储字符的最大/最小长度
};
```

{{< hint info >}}

mb是multi-byte character的缩写，不同的字符集在存储字符的时候使用的字节数不同。mbminlen和mbmaxlen代表这个字符集存储字符的最大/最小长度

{{</hint>}}

从上述数据结构可以看到，类的类型又分为了mtype和ptype。可以把mtype理解为列的类型，ptype理解为列的属性。

![InnoDB_record_tuple.field.type](/InnoDB_record_tuple.field.type.png)

mtype的类型如下，其中字符串有string和binary string之分：

| 类型           | 值   | 说明                                                | string类型 | binary string类型                     | 最小长度                                                     | 最大长度       |
| :------------- | :--- | :-------------------------------------------------- | :--------- | :------------------------------------ | :----------------------------------------------------------- | :------------- |
| DATA_VARCHAR   | 1    | latin1字符集VARCHAR类型                             | Y          |                                       | 0                                                            | index_col->len |
| DATA_CHAR      | 2    | latin1字符集CHAR类型                                | Y          |                                       | index_col->len                                               | index_col->len |
| DATA_FIXBINARY | 3    | BINARY类型                                          | Y          | Y                                     | index_col->len                                               | index_col->len |
| DATA_BINARY    | 4    | VARBINARY类型                                       | Y          | Y                                     | 0                                                            | index_col->len |
| DATA_BLOB      | 5    | 大记录类型                                          | Y          | Y & prtype == BINARY bit （11th-bit） | 0                                                            | ULINT_MAX      |
| DATA_INT       | 6    | INT类型                                             |            |                                       | index_col->len                                               | index_col->len |
| DATA_SYS_CHILD | 7    | B+树索引中的node pointer中指向的child page no       |            |                                       | /                                                            | /              |
| DATA_SYS       | 8    | 系统列类型，比如隐藏的TrxID列、ROW ID列、回滚指针列 |            |                                       | index_col->len                                               | index_col->len |
| DATA_FLOAT     | 9    | FLOAT类型                                           |            |                                       | index_col->len                                               | index_col->len |
| DATA_DOUBLE    | 10   | DOUBLE类型                                          |            |                                       | index_col->len                                               | index_col->len |
| DATA_DECIMAL   | 11   | DECIMAL类型                                         |            |                                       | 0                                                            | index_col->len |
| DATA_VARMYSQL  | 12   | 非latin1字符集VARCHAR类型                           | Y          |                                       | 0                                                            | index_col->len |
| DATA_MYSQL     | 13   | 非latin1字符集的CHAR类型                            | Y          |                                       | prtype == BINARY bit （11th-bit）   index_col→len else   mbminmaxlen | index_col->len |
| DATA_GEOMETRY  | 14   | geometry类型                                        |            |                                       | 0                                                            | ULINT_MAX      |
| DATA_POINT     | 15   | geo                                                 |            |                                       | index_col->len                                               | index_col->len |
| DATA_VAR_POINT | 16   | geo                                                 |            |                                       | 0                                                            | ULINT_MAX      |
| DATA_MTYPE_MAX | 63   | 上限                                                |            |                                       |                                                              |                |
| DATA_ERROR     | 111  | 错误类型                                            |            |                                       |                                                              |                |

记录之间的比较用mtype类型，而不是MySQL提供给用户定义的数据列类型（上层的列类型）：比如没有诸如SMALLINT、TINYINT、ENUM这些，对于VARCHAR类型，mtype有DATA_VARCHAR和DATA_VARMYSQL两种。get_innobase_type_from_mysql_type负责将各种MySQL上层的列类型转化为dtype的mtype类型。

对于mtype大于DATA_FLOAT的列，对整个列的字节（padding后）进行比较。而对于非latin1字符集的VARCHAR和CHAR类型，则调用cmp_whole_field进行比较。为什么之前的列类型不需要进行整个字节的比较呢？这是因为可以允许下面的情况发生：

select 'a' = ('a   ') => 1

在latin1字符集下对填充字段不进行判断，视为两个字符串是相等的（字符串'a'的十六进制为0x61，字符串'a  '的十六进制为0x61202020）.

在之前的MySQL版本中，有一个小问题，对于类型VARBINARY和BINARY，其存储是按照字节进行存储的，而排序是根据latin1字符集进行排序的，好在后续已经解决了这个问题，即属于了下面的string类型，带有了字符集和校对字符集信息。

区别参见官方文档：[The BINARY and VARBINARY Types](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html)，简单来说就是一个是带字符集和校对字符集的字符串，一个是单纯的二进制字符串，即文档中提到的“contain byte strings rather than character strings. This means they have the binary character set and collation, and comparison and sorting are based on the numeric values of the bytes in the values.”，即熟悉的utf8_bin、utf8_general_ci、utf8_general_cs的区别。

ptype表示对于列更为精确的表述，如果为系统列，还可以标识这个系统列的类型，此外，还定义了列的属性。

通过mtype和ptype就可以对列进行比较了。逻辑记录与物理记录的比较通过cmp_dtuple_rec_*，物理记录之间的比较通过cmp_rec_rec_*。返回值有-1、0、1，分别表示比较的记录1小于、等于、大于记录2。

对于返回值不等于0的情况（即大于、小于），还通过matched_fields、matched_bytes这两个变量来返回额外的信息。matched_fields标识匹配的记录不相等，但是已经匹配相等的列的个数，matched_bytes标识对于最后一个不匹配的列中，已经匹配的字节数。

NULL值在InnoDB中被视为最小的值，任何与NULL值进行的等值比较返回的值都是NULL

1 = NULL => NULL

NULL = NULL => NULL

然而在InnoDB内部的记录比较中，也就是cmp_dtuple_rec_with_match、cmp_rec_rec_with_match的比较中，两个NULL值被视为是相等的。这也好理解，因为这里所阐述的记录比较是用于索引的排序、记录的查询，因此NULL值相同表示其在索引中所处的位置。
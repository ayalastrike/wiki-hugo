---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
title: B+ tree
---

在InnoDB中，数据的存储组织为IoT，采用B+ tree的数据结构提供高效访问数据的方式（存取路径）。

在B+ tree实现中，两块内容最为重要，一块是concurrency control，一块是SMO（Struct Modification Operations）。

# 演进

[B+ tree](https://en.wikipedia.org/wiki/B%2B_tree)是由二叉查找树、平衡二叉树、[B-tree](https://en.wikipedia.org/wiki/B-tree)演进而来的。

{{< hint info >}}

**二叉查找树 BST（Binary Search Tree）**

**平衡二叉树 AVL（Balanced Binary Tree）**

在计算机科学中，AVL树是最早被发明的自平衡二叉查找树。在AVL树中，任一节点对应的两棵子树的最大高度差为1，因此它也被称为高度平衡树。查找、插入和删除在平均和最坏情况下的时间复杂度都是O(logn)。增加和删除元素的操作则可能需要借由一次或多次树旋转，以实现树的重新平衡。AVL 树得名于它的发明者 G. M. Adelson-Velsky 和 Evgenii Landis，他们在1962年的论文《An algorithm for the organization of information》中公开了这一数据结构。

{{</hint>}}

## 二叉查找树 BST（Binary Search Tree）

我们在介绍B+ tree之前，先了解一下binary search tree。在BST中，左子树的键值总是小于根的键值，右子树的键值总是大于根的键值。因此可以通过中序遍历得到键值的排序输出。

中序遍历（LDR - Inorder Traversal）是二叉树遍历的一种，也叫做中根遍历、中序周游。在二叉树中，中序遍历首先遍历左子树，然后访问根结点，最后遍历右子树。

比如有如下二叉查找树：

![InnoDB_b+tree_BST_1](/InnoDB_b+tree_BST_1.png)

中序遍历后输出：2、3、5、6、7、8。

二叉查找树的平均查找速度比顺序查找来得更快：顺序查找的平均查找次数为(1+2+3+4+5+6)/6=3.3次，二叉查找树的平均查找次数为(3+3+3+2+2+1)/6=2.3次。

但是，二叉查找树可以任意地构造，如果构造成下面的二叉查找树：

![InnoDB_b+tree_BST_2](/InnoDB_b+tree_BST_2.png)

则平均查找次数为(1+2+3+4+5+5)/6=3.16次，和顺序查找差不多。而二叉查找树的查找效率取决于树的高度，因此若想最大性能地构造一个二叉查找树，需要这棵二叉查找树是平衡的（即树的高度最小），因此引出了平衡二叉树，或称为AVL tree。

## 平衡二叉树 AVL tree（Balanced Binary Tree）

平衡二叉树的定义如下：首先符合二叉查找树的定义，其次必须满足任何节点的两个儿子子树的高度最大差为1。显然，上图不满足平衡二叉树的定义，而下图是一棵平衡二叉树。平衡二叉树对于查找的性能是比较高的，但不是最高的，只是接近最高性能。最好的性能需要建立一棵最优二叉树，但是最优二叉树的建议和维护需要大量的操作，因此，用户一般只需建立一棵平衡二叉树即可。

平衡二叉树对于查询速度的确很快，但是维护一棵平衡二叉树的代价是需要付出代价的。通常来说，需要1次或多次左旋和右旋来得到插入或更新后树的平衡性。

比如插入9，需要左旋以保证平衡：

![InnoDB_b+tree_AVL_1](/InnoDB_b+tree_AVL_1.png)

有的情况可能需要多次：

![InnoDB_b+tree_AVL_2](/InnoDB_b+tree_AVL_2.png)

上面2个图都是插入的例子，更新和删除操作同理，也是通过左旋或者右旋来完成的。因此对于一棵平衡树的维护是有一定开销的，不过平衡二叉树多用于内存结构对象中，因此维护的开销相对较小。

平衡因子 = 左子树深度/高度 - 右子树深度/高度

以下的动图更直观一些：

![InnoDB_b+tree_AVL_rotate_E_left](/InnoDB_b+tree_AVL_rotate_E_left.gif)

![InnoDB_b+tree_AVL_rotate_S_right](/InnoDB_b+tree_AVL_rotate_S_right.gif)

## 平衡多路查找树 B-tree

B-tree是为磁盘等外存储设备而设计的一种平衡查找树。

B-tree需要满足：

- the nodes in a B-tree of order ![m](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-fdc40b8ad1cdad0aab9d632215459d28_l3.svg) can have a maximum of ![m](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-fdc40b8ad1cdad0aab9d632215459d28_l3.svg) children
- each internal node (non-leaf and non-root) can have at least (![m](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-fdc40b8ad1cdad0aab9d632215459d28_l3.svg)/2) children (rounded up)
- the root should have at least two children – unless it’s a leaf
- a non-leaf node with ![k](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-d42bc2203d6f76ad01b27ac9acc0bee1_l3.svg) children should have ![k-1](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-7dfca2445cd362ac42fb9032c9cf2367_l3.svg) keys
- all leaves must appear on the same level

使用B-tree可以高效的找到数据所在的磁盘块，这是因为B-tree的每个节点包含了大量的key信息和分支，相对于AVL tree减少了节点个数，查询效率更高（通过磁盘IO将数据加载到内存）。

## B+ tree

B+ tree由B-tree和索引顺序访问方法（ISAM - Indexed Sequential Access Method，这也是MyISAM引擎最初参考的数据结构）演化而来。B+ tree相对于B-tree主要有两个区别：

1. 所有叶子节点（记录）都由双向链表顺序串联起来。
2. 非叶子节点只持有key，承担router到叶子节点的作用。

解读：

1. 叶子节点的有序可以保证data access graph是从上到下的，这比B-tree的data分布在各层节点上，对于数据的加锁顺序更简单和友好
2. 这可以保证非叶子节点可以最大化的存储key，而减少了树的高度。

比如下面这颗B+ tree，其高度为2，每页可存放4条记录，扇出（fan out）为5，如下图所示：

![InnoDB_b+tree](/InnoDB_b+tree.png)

所有记录都在叶子节点，并且是顺序存放的，如果用户从最左边的叶节点开始顺序遍历，可以得到所有键值的顺序排序：5、10、15、20、25、30、50、55、60、65、75、80、85、90。

B+ tree和B-tree的不同之处有以下3点：

1. B-tree的数据只会出现一次，存储在叶子节点或者非叶子节点。B+ tree的数据存储在叶子节点，非叶子节点作为router（可能出现多次）。
2. 因为#1，B-tree的非叶子节点存储的数据量变小，扇出率低，所以B-tree的层高会更多，导致维护代价变大，并且搜索修改的性能变低。而B+ tree的非叶子节点只存储键值，所以是一个矮胖子，而B-tree是一个瘦高个子。
3. B-tree的查询效率和其数据在树中的位置相关。最大时间复杂度是root到leaf page（和B+树相同），最小时间复杂度为1（root）。B+ tree的查询复杂度是固定的。

在InnoDB中，每个页的大小是16KB，假设平均每行记录的大小为100个字节，则每个页能存放的记录数量为160。那么B+ tree高度和总共可以存放的记录之间的关系为：总记录数 = 记录数^树高^ log~d~(N)

|      | 树高(d) = 2 | 树高(d) = 3 | 树高(d) = 4 |
| :--- | :---------- | :---------- | :---------- |
| N    | 1602        | 409 6000    | 6 5536 0000 |

需要注意的是B+ tree只能找到记录所在的页，但是并不能定位到记录在页中的具体位置（偏移量），还需要通过page directory的二分查找才能找到记录。为了提高这部分效率，InnoDB会将页加载到内存中，加快查找效率。

在B+ tree中，为了充分利用磁盘的顺序特性，InnoDB还会根据不同插入情况考虑不同的分裂点记录以及分裂的方向。

# index tree

在数据库中，每个表可能会创建多个索引。其中，存储行数据的B+ tree称为聚簇索引（clustered index），其他索引称为辅助索引（secondary index）。

## clustered index

聚簇索引是用表的主键作为key来构造B+ tree的。如果没有显式创建主键（或唯一键），InnoDB会自动创建一个隐藏的ROWID作为主键。

聚簇索引中的记录是根据主键顺序存放的，这里的顺序指的是逻辑上顺序，并不是物理存储上的顺序。因为物理存储要保证顺序的开销很大，另外数据库本身也做不到，这其中还牵涉到文件系统在磁盘上的布局。

聚簇索引的非叶子节点存放的是<primary_key, page_no>，其中page_no指向下一层节点的页地址，称为index page；叶子节点存放的是实际的数据，称为data page。

下面来看一个具体的例子：

~~~~
create table t1 (
    a int not null auto_increment,
    b blob,
    c int not null,
    primary key (a),
    index idx_c (c)
) engine=InnoDB;
 
insert into t1 values (1, repeat('a', 7000), -1);
insert into t1 values (2, repeat('a', 7000), -2);
insert into t1 values (3, repeat('a', 7000), -3);
insert into t1 values (4, repeat('a', 7000), -4);
~~~~

创建好表和记录后，我们先看一下文件的大小：144K，即9个页。

通过hexdump查看二进制，我们将这9个页详细拆解一下。

这9个页的关键信息如下图所示：

![InnoDB_b+tree_page_data](/InnoDB_b+tree_page_data.png)

{{< hint info >}}

对于page链表来说，空节点用FF表示，从上图可以看到：非叶子节点，prev+next都是FF，对于叶子节点，最左和最右叶子的prev和next都是FF

{{</hint>}}

我们将数据页组织一下：

![InnoDB_b+tree_page_organization](/InnoDB_b+tree_page_organization.png)

{{< hint warning>}}

注意：在聚簇索引root page中的*，第一个记录上加*表示该记录是页中的最小记录，即record header中min rec位置1（标记为蓝色）。这是InnoDB的B+ tree index的另一个不同之处，这种方式可以降低最小记录发生变更时须对页所进行的操作。

page 3（聚簇索引 root page）

BTR_SEG_TOP  0000 001b 0000 0002 0032

BTR_SEG_LEAF 0000 001b 0000 0002 00f2

​         <   PK   ,  page_no  >

00 **1**0  00 11 00 0e <80 00 00 01,00 00 00 05>

00 00 00 19 00 0e <80 00 00 02,00 00 00 06>

00 00 00 21 ff  d6 <80 00 00 04,00 00 00 07>

{{</hint>}}

原始的ibd文件数据：

~~~~
page 0 FSP_HDR page
00000000  d3 e1 4c 85 00 00 00 00  00 00 00 00 00 00 00 00  |..L.............|
00000010  00 00 00 00 f4 78 5b 23  00 08 00 00 00 00 00 00  |.....x[#........|
00000020  00 00 00 00 00 1b 00 00  00 1b 00 00 00 00 00 00  |................|
00000030  00 09 00 00 00 40 00 00  00 21 00 00 00 08 00 00  |.....@...!......|
00000040  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 00 00  |................|
00000050  00 01 00 00 00 00 00 9e  00 00 00 00 00 9e 00 00  |................|
00000060  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 00 00  |................|
00000070  00 00 00 00 00 05 00 00  00 00 ff ff ff ff 00 00  |................|
00000080  ff ff ff ff 00 00 00 00  00 01 00 00 00 02 00 26  |...............&|
00000090  00 00 00 02 00 26 00 00  00 00 00 00 00 00 ff ff  |.....&..........|
000000a0  ff ff 00 00 ff ff ff ff  00 00 00 00 00 02 aa aa  |................|
000000b0  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff 00 00  |................|
000000c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00003ff0  00 00 00 00 00 00 00 00  d3 e1 4c 85 f4 78 5b 23  |..........L..x[#|
page 1 IBUF_BITMAP page
00004000  f0 c4 15 62 00 00 00 01  00 00 00 00 00 00 00 00  |...b............|
00004010  00 00 00 00 f4 77 6a 64  00 05 00 00 00 00 00 00  |.....wjd........|
00004020  00 00 00 00 00 1b 00 00  00 00 00 00 00 00 00 00  |................|
00004030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00007ff0  00 00 00 00 00 00 00 00  f0 c4 15 62 f4 77 6a 64  |...........b.wjd|
page 2 INODE page
00008000  ea ab f8 5c 00 00 00 02  00 00 00 00 00 00 00 00  |...\............|
00008010  00 00 00 00 f4 78 5b 23  00 03 00 00 00 00 00 00  |.....x[#........|
00008020  00 00 00 00 00 1b ff ff  ff ff 00 00 ff ff ff ff  |................|
00008030  00 00 00 00 00 00 00 00  00 01 00 00 00 00 00 00  |................|
00008040  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 00 00  |................|
*
00008060  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 05 d6  |................|
00008070  69 d2 00 00 00 03 ff ff  ff ff ff ff ff ff ff ff  |i...............|
00008080  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
000080f0  ff ff 00 00 00 00 00 00  00 02 00 00 00 00 00 00  |................|
00008100  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 00 00  |................|
*
00008120  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 05 d6  |................|
00008130  69 d2 00 00 00 05 00 00  00 06 00 00 00 07 ff ff  |i...............|
00008140  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
000081b0  ff ff 00 00 00 00 00 00  00 03 00 00 00 00 00 00  |................|
000081c0  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 00 00  |................|
*
000081e0  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 05 d6  |................|
000081f0  69 d2 00 00 00 04 ff ff  ff ff ff ff ff ff ff ff  |i...............|
00008200  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00008270  ff ff 00 00 00 00 00 00  00 04 00 00 00 00 00 00  |................|
00008280  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 00 00  |................|
*
000082a0  00 00 ff ff ff ff 00 00  ff ff ff ff 00 00 05 d6  |................|
000082b0  69 d2 ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |i...............|
000082c0  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00008330  ff ff 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00008340  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
0000bff0  00 00 00 00 00 00 00 00  ea ab f8 5c f4 78 5b 23  |...........\.x[#|
page 3 INDEX page
0000c000  17 8d 80 b2|00 00 00 03 |ff ff ff ff|ff ff ff ff  |................|
                    FIL_PAGE_OFFSET FIL_PAGE_PREV FIL_PAGE_NEXT
0000c010  00 00 00 00 f4 78 5b 23  45 bf 00 00 00 00|00 00  |.....x[#E.......|
0000c020  00 00 00 00 00 1b|00 02  00 a2 80 05 00 00 00 00  |................|
                    SPACE_ID
0000c030  00 9a 00 02 00 02|00 03 |00 00 00 00 00 00 00 00  |................|
                            PAGE_N_RECS    PAGE_MAX_TRX_ID
0000c040 |00 01|00 00 00 00 00 00  00 34|00 00 00 1b 00 00  |.........4......|
         PAGE_LEVEL INDEX_ID             PAGE_BTR_SEG_LEAF
0000c050  00 02 00 f2|00 00 00 1b  00 00 00 02 00 32|01 00  |.............2..|
                       PAGE_BTR_SEG_TOP              record_header (5)
0000c060  02 00 1b 69 6e 66 69 6d  75 6d 00 04 00 0b 00 00  |...infimum......|
0000c070  73 75 70 72 65 6d 75 6d  00 10 00 11 00 0e 80 00  |supremum........|
0000c080  00 01 00 00 00 05 00 00  00 19 00 0e 80 00 00 02  |................|
0000c090  00 00 00 06 00 00 00 21  ff d6 80 00 00 04 00 00  |.......!........|
0000c0a0  00 07 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0000c0b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
0000fff0  00 00 00 00 00 70 00 63  17 8d 80 b2 f4 78 5b 23  |.....p.c.....x[#|
page 4 INDEX page
00010000  9c d3 03 9b|00 00 00 04 |ff ff ff ff|ff ff ff ff  |................|
                    FIL_PAGE_OFFSET FIL_PAGE_PREV FIL_PAGE_NEXT
00010010  00 00 00 00 f4 78 5b 47  45 bf 00 00 00 00|00 00  |.....x[GE.......|
00010020  00 00 00 00 00 1b|00 02  00 ac 80 06 00 00 00 00  |................|
                    SPACE_ID
00010030  00 a4 00 01 00 03|00 04| 00 00 00 00 00 00 0d 5c  |...............\|
                            PAGE_N_RECS    PAGE_MAX_TRX_ID
00010040 |00 00|00 00 00 00 00 00  00 35|00 00 00 1b 00 00  |.........5......|
         PAGE_LEVEL INDEX_ID             PAGE_BTR_SEG_LEAF
00010050  00 02 02 72|00 00 00 1b  00 00 00 02 01 b2|01 00  |...r............|
                       PAGE_BTR_SEG_TOP              record_header (5)
00010060  02 00 41 69 6e 66 69 6d  75 6d 00 05 00 0b 00 00  |..Ainfimum......|
00010070  73 75 70 72 65 6d 75 6d| 00 00 10 ff f3 7f ff ff  |supremum........|
00010080  ff 80 00 00 01 00 00 18  ff f3 7f ff ff fe 80 00  |................|
00010090  00 02 00 00 20 ff f3 7f  ff ff fd 80 00 00 03 00  |.... ...........|
000100a0  00 28 ff f3 7f ff ff fc  80 00 00 04 00 00 00 00  |.(..............|
000100b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00013ff0  00 00 00 00 00 70 00 63  9c d3 03 9b f4 78 5b 47  |.....p.c.....x[G|
page 5 INDEX page
00014000  10 90 9a 69|00 00 00 05 |ff ff ff ff|00 00 00 06  |...i............|
                    FIL_PAGE_OFFSET FIL_PAGE_PREV FIL_PAGE_NEXT
00014010  00 00 00 00 f4 78 5b 23  45 bf 00 00 00 00|00 00  |.....x[#E.......|
00014020  00 00 00 00 00 1b|00 02  37 62 80 04 1b f5 1b 75  |........7b.....u|
                    SPACE_ID
00014030  00 00 00 05 00 00|00 01| 00 00 00 00 00 00 00 00  |................|
                            PAGE_N_RECS    PAGE_MAX_TRX_ID
00014040 |00 00|00 00 00 00 00 00  00 34|00 00 00 00 00 00  |.........4......|
         PAGE_LEVEL INDEX_ID             PAGE_BTR_SEG_LEAF
00014050  00 00 00 00|00 00 00 00  00 00 00 00 00 00|01 00  |................|
                       PAGE_BTR_SEG_TOP              record_header (5)
00014060  02 00 1d 69 6e 66 69 6d  75 6d 00 02 00 0b 00 00  |...infimum......|
00014070  73 75 70 72 65 6d 75 6d| 58 9b 00 00 00 10 ff f0  |supremumX.......|
00014080  80 00 00 01 00 00 00 00  0d 55 b6 00 00 01 57 01  |.........U....W.|
00014090  10 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |.aaaaaaaaaaaaaaa|
000140a0  61 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |aaaaaaaaaaaaaaaa|
*
00015be0  61 61 61 61 61 61 61 61  61 7f ff ff ff 58 9b 00  |aaaaaaaaa....X..|
00015bf0  00 00 18 00 00 80 00 00  02 00 00 00 00 0d 56 b7  |..............V.|
00015c00  00 00 01 58 01 10 61 61  61 61 61 61 61 61 61 61  |...X..aaaaaaaaaa|
00015c10  61 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |aaaaaaaaaaaaaaaa|
*
00017750  61 61 61 61 61 61 61 61  61 61 61 61 61 61 7f ff  |aaaaaaaaaaaaaa..|
00017760  ff fe 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00017770  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00017ff0  00 00 00 00 00 70 00 63  10 90 9a 69 f4 78 5b 23  |.....p.c...i.x[#|
page 6 INDEX page
00018000  ed 83 1a c5|00 00 00 06 |00 00 00 05|00 00 00 07| |................|
                    FIL_PAGE_OFFSET FIL_PAGE_PREV FIL_PAGE_NEXT
00018010  00 00 00 00 f4 78 5b 23  45 bf 00 00 00 00|00 00  |.....x[#E.......|
00018020  00 00 00 00 00 1b|00 02  37 62 80 04 00 00 00 00  |........7b......|
                    SPACE_ID
00018030  1b f5 00 05 00 00|00 02| 00 00 00 00 00 00 00 00| |................|
                            PAGE_N_RECS    PAGE_MAX_TRX_ID
00018040 |00 00|00 00 00 00 00 00  00 34|00 00 00 00 00 00  |.........4......|
         PAGE_LEVEL INDEX_ID             PAGE_BTR_SEG_LEAF
00018050  00 00 00 00|00 00 00 00  00 00 00 00 00 00|01 00  |................|
                       PAGE_BTR_SEG_TOP              record_header (5)
00018060  02 00 1d 69 6e 66 69 6d  75 6d 00 03 00 0b 00 00  |...infimum......|
00018070  73 75 70 72 65 6d 75 6d| 58 9b 00 00 00 10 1b 75  |supremumX......u|
00018080  80 00 00 02 00 00 00 00  0d 56 b7 00 00 01 58 01  |.........V....X.|
00018090  10 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |.aaaaaaaaaaaaaaa|
000180a0  61 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |aaaaaaaaaaaaaaaa|
*
00019be0  61 61 61 61 61 61 61 61  61 7f ff ff fe 58 9b 00  |aaaaaaaaa....X..|
00019bf0  00 00 18 e4 7b 80 00 00  03 00 00 00 00 0d 5a ba  |....{.........Z.|
00019c00  00 00 01 59 01 10 61 61  61 61 61 61 61 61 61 61  |...Y..aaaaaaaaaa|
00019c10  61 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |aaaaaaaaaaaaaaaa|
*
0001b750  61 61 61 61 61 61 61 61  61 61 61 61 61 61 7f ff  |aaaaaaaaaaaaaa..|
0001b760  ff fd 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0001b770  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
0001bff0  00 00 00 00 00 70 00 63  ed 83 1a c5 f4 78 5b 23  |.....p.c.....x[#|
page 7 INDEX page
0001c000  af a6 e2 6c|00 00 00 07 |00 00 00 06|ff ff ff ff| |...l............|
                    FIL_PAGE_OFFSET FIL_PAGE_PREV FIL_PAGE_NEXT
0001c010  00 00 00 00 f4 78 5b 23  45 bf 00 00 00 00|00 00  |.....x[#E.......|
0001c020  00 00 00 00 00 1b|00 02  1b ed|80 03 00 00 00 00  |................|
                    SPACE_ID
0001c030  00 80 00 05 00 00|00 01| 00 00 00 00 00 00 00 00| |................|
                            PAGE_N_RECS    PAGE_MAX_TRX_ID
0001c040 |00 00|00 00 00 00 00 00  00 34|00 00 00 00 00 00  |.........4......|
         PAGE_LEVEL INDEX_ID             PAGE_BTR_SEG_LEAF
0001c050  00 00 00 00|00 00 00 00  00 00 00 00 00 00|01 00  |................|
                       PAGE_BTR_SEG_TOP              record_header (5)
0001c060  02 00 1d 69 6e 66 69 6d  75 6d 00 02 00 0b 00 00  |...infimum......|
                infimum record\0          record_header(5)
0001c070  73 75 70 72 65 6d 75 6d| 58 9b 00 00 00 10 ff f0  |supremumX.......|
                supremum
0001c080  80 00 00 04 00 00 00 00  0d 5c bb 00 00 01 5a 01  |.........\....Z.|
0001c090  10 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |.aaaaaaaaaaaaaaa|
0001c0a0  61 61 61 61 61 61 61 61  61 61 61 61 61 61 61 61  |aaaaaaaaaaaaaaaa|
*
0001dbe0  61 61 61 61 61 61 61 61  61 7f ff ff fc 00 00 00  |aaaaaaaaa.......|
0001dbf0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
0001fff0  00 00 00 00 00 70 00 63  af a6 e2 6c f4 78 5b 23  |.....p.c...l.x[#|
page 8 fresh page
00020000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00024000
~~~~

详细拆解后：

~~~~
segment inode page (page 2)
BTR_SEG_TOP     0000 001b 0000 0002 0032
segment inode entry
00 00 00 00 00 00 00 01                             FSEG_ID (01)
00 00 00 00                                         FSEG_NOT_FULL_N_USED
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FREE
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    FSEG_NOT_FULL
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FULL
05 d6 69 d2                                         FSEG_MAGIC_N
00 00 00 03                                         FSEG_FRAG_ARR 0..31 (03) (未用使用FF填充)
ff ... ff
 
BTR_SEG_LEAF    0000 001b 0000 0002 00f2
segment inode entry
00 00 00 00 00 00 00 02                             FSEG_ID (02)
00 00 00 00                                         FSEG_NOT_FULL_N_USED
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FREE
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    FSEG_NOT_FULL
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FULL
05 d6 69 d2                                         FSEG_MAGIC_N
00 00 00 05 00 00 00 06 00 00 00 07                 FSEG_FRAG_ARR 0..31 (05 06 07)
ff ... ff
 
BTR_SEG_TOP     0000 001b 0000 0002 01b2
segment inode entry
00 00 00 00 00 00 00 03                             FSEG_ID (03)
00 00 00 00                                         FSEG_NOT_FULL_N_USED
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FREE
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    FSEG_NOT_FULL
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FULL
05 d6 69 d2                                         FSEG_MAGIC_N
00 00 00 04                                         FSEG_FRAG_ARR 0..31 (04)
ff ... ff
 
BTR_SEG_LEAF    0000 001b 0000 0002 0272
segment inode entry
00 00 00 00 00 00 00 04                             FSEG_ID (04)
00 00 00 00                                         FSEG_NOT_FULL_N_USED
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FREE
00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00    FSEG_NOT_FULL
00 00 00 00 ff ff ff ff  00 00 ff ff ff ff 00 00    FSEG_FULL
05 d6 69 d2                                         FSEG_MAGIC_N
ff ... ff                                           FSEG_FRAG_ARR 0..31 (FF)
 
page 3(聚簇索引 root page)
BTR_SEG_TOP     0000 001b 0000 0002 0032
BTR_SEG_LEAF    0000 001b 0000 0002 00f2
                  <    PK     ,  page_no  >
00 10 00 11 00 0e <80 00 00 01,00 00 00 05>
00 00 00 19 00 0e <80 00 00 02,00 00 00 06>
00 00 00 21 ff d6 <80 00 00 04,00 00 00 07>
 
page 4(辅助索引 root page)
BTR_SEG_TOP     0000 001b 0000 0002 01b2
BTR_SEG_LEAF    0000 001b 0000 0002 0272
               <     key   ,    PK     >
00 00 10 ff f3 <7f ff ff ff,80 00 00 01>
00 00 18 ff f3 <7f ff ff fe,80 00 00 02>
00 00 20 ff f3 <7f ff ff fd,80 00 00 03>
00 00 28 ff f3 <7f ff ff fc,80 00 00 04>
 
             record header (5) row id (4)     trx id (6)           rollback pointer (7)
page 5
58 9b 00     00 00 10 ff f0    80 00 00 01    00 00 00 00 0d 55    b6 00 00 01 57 01 10 | 61 ... 61 | 7f ff ff ff
58 9b 00     00 00 18 00 00    80 00 00 02    00 00 00 00 0d 56    b7 00 00 01 58 01 10 | 61 ... 61 | 7f ff ff fe
page 6
58 9b 00     00 00 10 1b 75    80 00 00 02    00 00 00 00 0d 56    b7 00 00 01 58 01 10 | 61 ... 61 | 7f ff ff fe
58 9b 00     00 00 18 e4 7b    80 00 00 03    00 00 00 00 0d 5a    ba 00 00 01 59 01 10 | 61 ... 61 | 7f ff ff fd
page 7
58 9b 00     00 00 10 ff f0    80 00 00 04    00 00 00 00 0d 5c    bb 00 00 01 5a 01 10 | 61 ... 61 | 7f ff ff fc
~~~~

## secondary index

辅助索引的非叶子节点存储的是记录格式为<index_key，primary_key，page_no>，这里注意辅助索引的非叶子节点存放了主键值信息。另外，辅助索引节点的记录不保存系统列 trx_id和rollback point。

{{< hint danger>}}

这里给读者留2个问题：

1. 为什么非叶子节点的记录中要包含primary key？
2. 辅助索引记录不包吃trx id和rollback point，那么MVCC正常吗？

{{</hint>}}

辅助索引又称为二级索引或者非聚簇索引，其叶子节点保存的是<index_key，primary_key/primary记录的地址>。从这里可以看出，value可以有两种形式：

- 记录的主键值
- 记录的物理地址

MyISAM采用第一种方式存储，即采用堆表（heap table）的方式组织数据，即完整的记录存放在堆表中，主键和其他索引都存放的是记录的物理地址，这些索引的区别只是是否唯一、非空。所以MyISAM的索引和记录的关系，和InnoDB的对比如下图所示：

![InnoDB_b+tree_MyISAM_index_organization](/InnoDB_b+tree_MyISAM_index_organization.png)

![InnoDB_b+tree_InnoDB_index_organization](/InnoDB_b+tree_InnoDB_index_organization.png)

InnoDB和MyISAM的辅助索引对比见以下表格：

|          | InnoDB                                                       | MyISAM                            |
| :------- | :----------------------------------------------------------- | :-------------------------------- |
| 读       | 通过辅助索引仅能得到记录的主键值，查询完整记录还需要通过聚簇索引的B+ tree回表查询（bookmark lookup） | 只需要一次B+ tree即可访问完整记录 |
| 写       | 仅主键变更需要更新辅助索引                                   | 记录发生改变，需要更新所有索引    |
| B+树高度 | 聚簇索引通常比辅助索引的高度要高                             | 低                                |
| 范围查询 | 聚簇索引高效（顺序IO），辅助索引相对低效                     | 低效（离散IO）                    |

## node pointer

在clustered index和secondary index中，leaf page存储的是实际的数据，non-leaf page（n_level != 0）——存储的是node pointer（也称为router，指示牌...），除了key之外，有一个指针用于执行下层的页地址。

{{< hint info>}}

InnoDB中的B+ tree是一个ordinary B+ tree，而不是B-link tree。

{{</hint>}}

node pointer也是由key-value构成，其中key需要可以唯一标识index record，value为指向下层的page no，（意味着下层的所有页<key，且non-leaf中下层的最左节点的key也是该key，leaf node是实际数据），对于key，不同的索引类型key有所不同：

- clusterd index：PK + < 系统列>
- non-unique secondary index：index_column, PK
- unique seconary index：index_column

另外，在每层的node pointer，最左节点都会设置一个特殊的bit来标记minimum record，即left most node of this level，数据表示为<min, page no>。这是为了更改好的组织树形结构的指针，对于非叶子节点，其page no指向下一层最左边的记录，即指向存储比本页面所有key都要小的记录页面。

## fill factor

在InnoDB中，聚簇索引与辅助索引中叶子节点实际存放的记录数量还和填充因子（fill factor）有关。当填充因子小于1/2时需要进行页的合并，因此常态下填充因子肯定总是大于1/2的。此外，填充因子还受到插入的影响，如果是顺序插入，那么页的填充因子比较高，可以达到90%甚至更高。如果插入是无序的，那么按照Jim Gray所提到的，填充因子一般为69%。

对于聚集索引，如果主键是自增ID，那么插入数据是顺序IO；如果副主索引是时间维度的key，那么也是顺序的，都可以有较高的填充因子。而如果主键为散列ID，比如UUID，那么插入就会无序，插入性能会有明显的退化，所以在线上场景中，尽量避免使用UUID类似的值作为主键。。

# concurrency control

B+ tree的concurrency control包括两个方面：

- 事务并发访问数据内容的并发控制，也被称为concurrency control protocol，比如MV2PL、MVTO、MVOCC
- 线程并发访问内存数据结构的并发控制，也被称为multi-threaded index concurrency control

在DBMS中，这两个目标用两种不同的机制来实现（lock和latch）。

这二者的区别在此不介绍。

以下内容buf_block->lock代表frame latch，简称为block->lock，也称为page latch。

## InnoDB中的B+ tree concurrency control

在本章我们只讨论第二种情况，即如何用latch对B+ tree提供并发控制机制。在之前的介绍中我们知道，在B+ tree有非叶子节点和叶子节点，这两者都可以通过page表示，而page通过PCB保护。同时，InnoDB还要保证每一个index（B+树）的一致性，因此需要维护两种rw lock：

- index级别：保护B+ tree结构的一致性（非叶子节点），通过dict_index_t→lock保护整个B+ tree（不包含叶子节点），并且每个branch node通过block→lock保护。
- block级别：保护叶子节点的一致性，通过block→lock保护叶子节点，并且上面的branch node也通过block→lock保护，所以block→lock统一用于保护PCB.frame（即数据页）的内部结构（页）的一致性

另外，rw lock保护的对象级别越高，冲突的可能性就越大，并发瓶颈也就越容易出现。

B+ tree的latch并发控制策略为：

1. 搜索：在B+ tree的遍历中，首先通过index→lock的方式遍历非叶子节点，当达到叶子节点时，加上block→lock后，释放index→lock。同时，在MySQL 5.7之前，为了节省CPU时间，只对index（非叶子节点）加一个s-latch，并不对中间遍历的每一个branch node加任何latch。
2. 如果需要重组B+ tree（悲观插入记录），则需要对index加x-latch，并对页进行分裂
   1. 在叶子节点上确定split point
   2. 分配一个新的page
   3. 将新生成的node pointer更新到上一层的no-leaf page
   4. 释放index x-latch
   5. 将split point的记录移动到新的page

因此，InnoDB对B+树的并发控制的优化历史过程如下图所示：

![InnoDB_b+tree_concurrency_control](/InnoDB_b+tree_concurrency_control.png)

## 传统的B+ tree concurrency control

**单个节点并发控制**

如果要修改B+ tree的内容或者结构，必须先把该B+ tree节点读取到内存中，修改后再写回磁盘（Read-Modify-Write）。因此，在多线程并发的场景下，该节点在内存中被一个线程读取时，不能被另一个线程修改，这种场景就是典型的多线程编程中共享数据的临界区问题。数据库中使用latch来控制单个B+ tree节点的访问，从而保持 B+ tree物理结构的一致性，通常的做法是在每个节点（索引页）的控制块（PCB）中嵌入一个latch。

**latch coupling**

当一个线程沿着某个路径把指针从一个节点移动到另一个节点时，比如从父节点移动到子节点，或者在叶子节点前后移动，在这期间不允许其它线程去改变这个指针。这个时候需要持有后续节点的latch后，才能释放当前节点，这种方法通常称为 ”latch crabbing”，也被称为蟹行。

如果后续节点不在内存中时，还需要一次磁盘 IO 来获取该节点，因为latch的持有时间必须很短，在等待IO时，不应该长时间占据当前节点的latch，而应该释放。在后续节点的IO操作结束之后，再重新进行一次从root到leaf的遍历来获得之前当前节点的latch，以避免在等待IO的时间窗口内其他线程对B+树结构的修改导致的不一致问题。这个重新遍历的代价并不大，因为可以通过检查上一次遍历时保存的路径是否还有效，来重用之前的路径，当需要的节点已经从磁盘读取到内存池中时，它的祖先节点可能还没有被其它线程修改过。

**反向遍历 level list**

latch coupling中除了上面提到的从父节点到子节点遍历的情况，还有一种是同层相邻节点遍历的场景，比如在范围查询时需要沿着leaf链表正向或反向遍历，而asc+desc会由于遍历的方向不同导致死锁。为了避免相向遍历产生的死锁，一种方法是让latch立即重试，即当一个 latch 被其它线程占有而获取不到时，立即返回失败而不是继续等待latch被其它线程释放，从而让冲突的对方能继续执行下去，而自己进行一次从root到leaf的重试。这里要考虑的一个问题是，如何避免两个冲突的线程同时重试的情况，因为同时重试后有可能还在相同的地方发生冲突，可以规定一个遍历方向的优先级，这样可以保证冲突时只有一个线程会重试，另一个线程会继续执行。

**递归向上更新**

另外，在对B+ tree的节点更改时可能会导致页的合并和分裂。最简单的解决办法是对整个B+ tree加一个互斥锁，但是这样太影响多线程并发，最好的方法应该是只对B+ tree的某些节点加锁（范围），可以有如下几种策略：

- 在从上到下查找目标节点时，就把整个路径节点加X-latch，这样在从下到上的节点变更时就不再需要加锁。很明显的这种方法每次都会锁住root节点，跟锁住整个B+树没有本质区别，严重影响 B-tree 的并发性。
- 在从上到下遍历时给查找路径加S-latch，在必要时再将S-latch升级成S-latch，升级过程中需要检查死锁的风险。由于这种方法是可能失败的，因此需要有额外的备选方案，增加了逻辑的复杂度。
- 引入一种共享互斥锁（SX-latch），从上到下遍历时给路径上的节点都加 SX-latch，SX-latch可以与S-latch相容，从而不会阻塞其它线程的读请求。但是SX-latch无法与自身相容，因此对于并发更新来说B+树的root节点依然是一个瓶颈，只允许一个线程进行修改B+树结构的操作。

这三种方法都需要一直持有遍历路径上节点的latch，直到一个节点不再会触发向上更新才释放路径上的所有 latch。实际情况是大多数节点都不是满的，因此大多数插入操作都不会触发节点分裂并向上变更，锁住整个路径的节点是没有必要的。如果在插入操作的从上到下遍历时主动进行节点分裂，就能避免了根节点的瓶颈问题，也没有升级 latch 时失败的问题。缺点是需要在实际分裂之前预先分配空间，造成一定的空间浪费，并且在可变长记录更新时无法准确地判断是否需要预先分裂。

为了解决不必要的对节点加互斥锁，可以在第一次从上到下遍历时加共享锁，直到一个节点需要分裂时，重新回到 root 做一次遍历，这次给要分裂的节点加互斥锁，并进行实际的分裂操作。第二次遍历时可以通过检查第一次遍历保存的路径来进行重用，而不必从根节点重新遍历。

以上参考了以下文章

[MySQL · 引擎特性 · InnoDB index lock前世今生](http://mysql.taobao.org/monthly/2015/07/05/)

[MySQL · 内核特性 · InnoDB btree latch 优化历程](http://mysql.taobao.org/monthly/2020/06/02/)

[Database · 理论基础 · B-tree 物理结构的并发控制](http://mysql.taobao.org/monthly/2020/11/02/)

# B+ tree SMO

## 页的分裂

页的分裂会对性能和成本都会有所影响，所以InnoDB对B+ tree的分裂进行了优化，使其更符合磁盘的特性：并不总是将中间记录作为分裂点，而是需要根据插入的情况进行判断，从而更为有效地利用磁盘空间。

比如，如果记录是严格按照递增顺序进行插入的，页P1中有记录1、2、3、4、5、6、7、8、9、10。如果按照中间节点进行分裂的策略，那么分裂完成后页P1中有记录1，2，3，4，页P2中有记录5，6，7，8，9，10。如果顺序插入记录，这意味着之后不会再有记录向页P1中进行插入，会导致页P1的利用率很低，如果后续还有不断的顺序插入+分裂，每个页的填充率基本都在50%左右，这也会影响查询效率。

因此，当页进行分裂操作时，首先需要通过页头（PAGE_HEADER）上的插入信息判断是否为连续插入模式，插入信息如下：

- PAGE_LAST_INSERT：最后插入记录的位置（offset）
- PAGE_DIRECTION：最后插入记录的方向
- PAGE_N_DIRECTION：一个方向上连续插入记录的数量

页的分裂由btr_page_split_and_insert实现，分裂方式有两种：

- 50% - 50%算法：将旧页50%的数据量移动到新页
- 0% - 100%算法：不移动旧页任何的数据，只将引起分裂的记录插入到新页

如下图所示：

![InnoDB_b+tree_page_split](/InnoDB_b+tree_page_split.png)

在页的分裂插入，首先通过查找定位cursor（即insert point），然后确定分裂点（split rec），再进行页的分裂，最后插入新的记录（insert rec）。

向右分裂和向左分裂都是有序插入（即连续插入），如果插入是无序的，则采用50% - 50%算法，页中的中间记录（middle record）作为分裂点记录（page_get_middle_rec）。

{{< hint info>}}

在选择split rec时，不判断该记录是否为标记删除，即不对记录的delete flag进行判断。

{{</hint>}}

分裂操作步骤如下：

1. 确定split rec
2. 从索引的数据段（LEAF段/TOP段）分配一个page，并对该页加x-latch
3. 确定需要移动到新页中的第一条记录（first_rec）以及源页中的最后一条记录（move_limit）
4. 进行B+树的分裂，在父节点添加node pointer记录，如果父节点空间不足，继续触发分裂操作
5. 将记录移动到新的page中
6. 确定insert rec待插入的page
7. 将insert rec插入到page中
8. 如果上述操作失败，对page reorganize，然后重新进行插入操作；如果还失败，回到步骤1再次尝试

这里需要注意，一次分裂操作可能会再次引起分裂。这是因为，待插入的记录非常大，如果第一次分裂完成后，左页占用了6000字节空间，右页占用10000字节，但需要往右页插入的记录一共占用8000字节，则右页还需要再进行一次分裂操作。这时分裂点记录的判断由btr_page_get_sure_split_rec实现。

操作完成后，叶子节点的x-latch由mtr.commit释放，而index的x-latch在步骤4结束之后就进行释放，以减少对于index的竞争。

### 连续向右插入后的分裂

通过启发式算法（eager heuristics）策略决定分裂点，当前insert point之后是否有两条记录，如果有，则采用50%-50%算法分裂；反之采用0%-100%算法分裂，如下图所示：

![InnoDB_b+tree_page_split-right](/InnoDB_b+tree_page_split-right.png)

下面是具体示例：

![InnoDB_b+tree_page_split-right_1_2](/InnoDB_b+tree_page_split-right_1_2.png)

###  连续向左插入后的分裂

向左分裂插入一共会有3种情况，如下图所示：

![InnoDB_b+tree_page_split-left](/InnoDB_b+tree_page_split-left.png)

这三种情况，第一种split rec为insert point，后两种split rec为insert point的下一个记录待分裂的记录，即

- 如果insert point为infimum或者infimum后的第一条记录，则split rec为insert point下一个
- 否则同值

~~~~
		if (infimum != insert_point
		    && page_rec_get_next(infimum) != insert_point) {
			*split_rec = insert_point;
		} else {
			*split_rec = page_rec_get_next(insert_point);
		}
~~~~

向左分裂的的这三种情况如下图所示：

![InnoDB_b+tree_page_split-left-1](/InnoDB_b+tree_page_split-left-1.png)

![InnoDB_b+tree_page_split-left-2](/InnoDB_b+tree_page_split-left-2.png)

在构建B+ tree插入的例子之前，我们先做如下假设：

- 页面扇出为3，即每个页面（non-leaf page和leaf page）最多可以插入3条记录，插入更多的记录，会产生页的分裂
- 插入的数据序列为10，20，5，8，23，22，50，21，53，40，9

第一步，我们首先插入10，20，5这3条记录，因为在插入时该B+树只有一个空的root page（位于non-leaf segment），所以在插入后，页面如下图所示：

![InnoDB_b+tree_insert_1](/InnoDB_b+tree_insert_1.png)

第二步，插入8，这时root page已满，需要进行页的分裂。我们之前已经知道，non-leaf page和leaf page存储在不同的segment中，所以我们需要从leaf segment中申请一个新的page，然后将root page的数据搬过来。从这里我们可以看到，在B+ tree的分裂中，root page始终是不会变的，无论变成多大的树，root page的page no始终如一。具体过程如下：

1. 新创建一个leaf page（page 101）
2. 将root page中的全部记录复制到leaf page中
3. 清除原root page的全部记录，并构建minimum record的node pointer，其中的page no指向子节点页面page 101

![InnoDB_b+tree_insert_2](/InnoDB_b+tree_insert_2.png)

第三步，虽然完成了root page的分裂，但page 101仍然没有空间可以存储接着待插入的8，所以还需要继续分裂，只不过这次是leaf page的分裂：

1. 新创建一个leaf page（page 102）
2. 将page 101的一部分记录（记录20）移到page 102中
3. 将page 101和102组成双向链表（结成兄弟关系）
4. 将page 102中的20构建父节点的node pointer，即<20, page 102>，并更新到父节点（page 100）（结成父子关系）

![InnoDB_b+tree_insert_3](/InnoDB_b+tree_insert_3.png)

第四步，插入8，这时通过查找定位可以知道，insert point位于page 101的<10, data>。同理，插入23，22。

![InnoDB_b+tree_insert_4](/InnoDB_b+tree_insert_4.png)

第五步，插入50，这时首先定位到page 102，发现页已满，需要首先进行页的分裂，过程和第三步一样：

1. 新创建一个leaf page（page 103）
2. 将page 102的一部分记录（记录23）移到page 103中
3. 将page 102和103组成双向链表（结成兄弟关系）
4. 将page 103中的23构建父节点的node pointer，即<23, page 103>，并更新到父节点（page 100）（结成父子关系）

![InnoDB_b+tree_insert_5](/InnoDB_b+tree_insert_5.png)

第六步，从root page开始重新搜索，定位到page 103，插入记录50，以同样的方式在page 102插入记录21，在page 103插入记录53

![InnoDB_b+tree_insert_6](/InnoDB_b+tree_insert_6.png)

第七步，插入记录40，定位到page 103，发现page 103已满，同样需要分裂，这里不再详述页的分裂步骤了，和前面相同。

![InnoDB_b+tree_insert_7](/InnoDB_b+tree_insert_7.png)

第八步，插入记录9，定位到page 101，发现page 101已满，同样进行页的分裂，此时的B+树处在变化中，不是完整的：

![InnoDB_b+tree_insert_8.1](/InnoDB_b+tree_insert_8.1.png)

但是...... root page已满，需要进行root page的分裂了。而我们同时也知道root page必须始终是root page，所以我们需要在non-leaf segment中申请一个新的page 106，并将page 100的全部数据移动到page 106中（相当于leaf page 101 105 102 103 104的父节点都从page 100变为page 106），并且page 100的minimum record也会指向page 106。

![InnoDB_b+tree_insert_8.2](/InnoDB_b+tree_insert_8.2.png)

这时我们尝试将之前pending的<10, page 105>插入到page 106中，但是page 106已满，我们需要页的分裂，此时分裂是non-leaf page的分裂，其分裂方式和leaf page的分裂一样：

1. 新创建一个non-leaf page（page 107）
2. 将page 106的一部分记录（记录53）移到page 107中
3. 将page 106和107组成双向链表（结成兄弟关系）
4. 将page 107中的记录53构建父节点（root page）的node pointer，即<53, page 107>，并更新到父节点（page 100）（结成父子关系）

![InnoDB_b+tree_insert_8.3](/InnoDB_b+tree_insert_8.3.png)

这时，我们有了足够的空间用于容纳<10, page 105>，我们将其插入到page 106，让page 105"找到了爸爸"。

![InnoDB_b+tree_insert_8.4](/InnoDB_b+tree_insert_8.4.png)

第九步，最终我们要在page 1010中插入记录9，所有的步骤完成。

![InnoDB_b+tree_insert_9](/InnoDB_b+tree_insert_9.png)

我们将leaf page中的记录按照从左到右的顺序读入：5 8 9 10 20 21 22 23 40 50 53，即索引序。

## 页的合并

当对页中的记录进行删除或者更新后，页的填充率可能会低于50%（BTR_CUR_PAGE_COMPRESS_LIMIT），这时会尝试进行页的合并，即页中的记录合并到页的左兄弟/右兄弟页中。如果待被合并的页为当前层的最后一个页，则需要减少B+树的高度（btr_lift_page_up）。与传统的B+ tree SMO不同的是，InnoDB并不要求合并操作一定发生，如果左右兄弟页没有空间进行合并，则放弃合并操作。

合并页（btr_compress）总是判断先判断左兄弟页，如果左兄弟页存在，则将记录合并到左兄弟页中。如果左兄弟页没有足够空间，则尝试进行右兄弟页合并。

如果合并的是叶子节点，则开始合并前（函数调用前）要保证已经对其左右兄弟节点加x-latch。

## 页的回收

从上面我们可以得知，随着数据的插入，B+ tree会不断的分裂，从而导致B+ tree成为一个矮胖子。如果数据不断的删除，传统的B+ tree都会有一个fill factor来控制page中数据的删除比例，如果达到阈值，则进行页的合并。

在InnoDB中的记录删除，分为以下3种情况：

1. 在删除记录时，如果该page只剩下一条记录，就直接将该page回收（也只有在这种情况下，InnoDB才会回收page）。从这里我们可以看到InnoDB和传统的B+ tree的删除算法的不同，也可以说，InnoDB除了此情况外，根本不存在页的合并这么一个说法。
   当然，这个page不能是root page
2. 如果要删除的记录是non-leaf page，并且为该page的最左边记录（infimum的下一条记录），则说明此时的记录删除还会影响上一层的父节点node pointer。所以此时还要更新其父节点node pointer（可能出现继续向上递归）。
3. 此时，可以放心的将本page要删除的记录删除掉，记录删除的操作就完成了。

对于#1，从B+ tree中摘除一个page的流程如下：

1. 首先判断该page的同层兄弟节点，如果没有任何兄弟，则认定该层只有一个节点，并且为空节点。这样可以递归的将整个B+ tree释放掉，最后只剩下一个空的root page
2. 找到该page上一层的父节点node pointer，删除该node pointer记录的流程参照上面的#2
3. 将该page从前后兄弟的双向链表中摘除
4. 归还到本表空间的碎片管理链表中

对于#2，更新父节点node pointer的流程如下：

1. 根据当前的待删除记录找到父节点中的node pointer，通过递归删除的方法将其删除
2. 根据待删除记录的下一条记录，构造一个新的node pointer，其中指向子节点的page no不变
3. 将#2构造的node pointer插入到当前节点的父节点中，即更新父节点中的node pointer

# 页的查找

通过B+ tree查找是InnoDB中最为常见的操作，无论是查询，还是增删改，都需要首先定位到所需操作的记录。在InnoDB中，通过btr_cur_search_to_nth_level来查找指定的记录。

btr_cur_search_to_nth_level的函数原型为：

````
/********************************************************************//**
Searches an index tree and positions a tree cursor on a given level.
NOTE: n_fields_cmp in tuple must be set so that it cannot be compared
to node pointer page number fields on the upper levels of the tree!
Note that if mode is PAGE_CUR_LE, which is used in inserts, then
cursor->up_match and cursor->low_match both will have sensible values.
If mode is PAGE_CUR_GE, then up_match will a have a sensible value. */
void
btr_cur_search_to_nth_level(
/*========================*/
	dict_index_t*	index,	/*!< in: index */
	ulint		level,	/*!< in: the tree level of search */
	const dtuple_t*	tuple,	/*!< in: data tuple; NOTE: n_fields_cmp in
				tuple must be set so that it cannot get
				compared to the node ptr page number field! */
	page_cur_mode_t	mode,	/*!< in: PAGE_CUR_L, ...;
				NOTE that if the search is made using a unique
				prefix of a record, mode should be PAGE_CUR_LE,
				not PAGE_CUR_GE, as the latter may end up on
				the previous page of the record! Inserts
				should always be made using PAGE_CUR_LE to
				search the position! */
	ulint		latch_mode, /*!< in: BTR_SEARCH_LEAF, ..., ORed with
				at most one of BTR_INSERT, BTR_DELETE_MARK,
				BTR_DELETE, or BTR_ESTIMATE;
				cursor->left_block is used to store a pointer
				to the left neighbor page, in the cases
				BTR_SEARCH_PREV and BTR_MODIFY_PREV;
				NOTE that if has_search_latch
				is != 0, we maybe do not have a latch set
				on the cursor page, we assume
				the caller uses his search latch
				to protect the record! */
	btr_cur_t*	cursor, /*!< in/out: tree cursor; the cursor page is
				s- or x-latched, but see also above! */
	ulint		has_search_latch,
				/*!< in: latch mode the caller
				currently has on search system:
				RW_S_LATCH, or 0 */
	const char*	file,	/*!< in: file name */
	ulint		line,	/*!< in: line where called */
	mtr_t*		mtr);	/*!< in: mtr */
````

btr_cur_search_to_nth_level中的mode表示查询模式，但在具体的查询场景上有不同的使用限制：

- 非叶子节点上进行查询时，其mode只能为LE/L，GE→L G→ LE
- 对于insert，其在叶子节点上总是通过LE进行定位待插入记录的前一条记录
- 对主键和唯一索引进行查询时，只能为GE，而不能为LE，这是因为InnoDB支持MVCC，因此即使有唯一性约束，但在实际的页中仍然可能包含多个键值相同的记录，只是其中仅有一个记录的delete flag=0，并且该记录总是在最后一个

{{< hint info>}}

比如如下的叶子节点的主键记录，对于用户来说，看到的主键记录只有1, 2, 3, 4, 5：(1, 'A'), *(2, 'B'), (2, 'C'), *(3, 'D'), (3, 'E'), (4, 'F'), (5, 'G')如果根据LE查询为3的记录，则会得到记录的前一个页，很显然这会引起并发错误。

{{</hint>}}

![InnoDB_b+tree_search_mode](/InnoDB_b+tree_search_mode.png)

{{< hint danger>}}

这里留给读者一个问题：为什么删除的记录在左边，delete=flag=0的记录一定在右边呢？

{{</hint>}}

## 非叶子节点的mode

对于非叶子节点，其mode只能为LE/L，即GE→L，G→ LE。

````c++
	/* We use these modified search modes on non-leaf levels of the
	B-tree. These let us end up in the right B-tree leaf. In that leaf
	we use the original search mode. */

	switch (mode) {
	case PAGE_CUR_GE:
		page_mode = PAGE_CUR_L;
		break;
	case PAGE_CUR_G:
		page_mode = PAGE_CUR_LE;
		break;
	default:
		page_mode = mode;
		break;
	}
````

这是由B+ tree的特点所决定的，即依左搜索（上面图中的灰色框），然后通过取下一个记录的方式实现mode翻转，恢复正常（L→GE G→LE ）。这样会遇到一个问题，即通过mode L、LE定位到的low为supremum，需要再搜索下一页的第一条用户记录才能得到最终的记录（high）。

````c++
	/* PHASE 4: Look for matching records in a loop */

	if (page_rec_is_supremum(rec)) {

		/* A page supremum record cannot be in the result set: skip
		it now that we have placed a possible lock on it */

		prev_rec = NULL;
		goto next_rec;
	}
````

{{< hint danger>}}

这里留给读者一个问题：B+ tree的特点为什么决定了要依左搜索？

{{</hint>}}

## latch_mode

latch_mode表示在搜索过程中需要对page和index加的latch种类：

| latch_mode           | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| BTR_SEARCH_LEAF      | 查找leaf page，index加s-latch，leaf page加s-latch后释放index s-latch |
| BTR_MODIFY_LEAF      | 修改leaf page，index加s-latch，leaf page加x-latch后释放index s-latch |
| BTR_NO_LATCHES       | index加s-latch，leaf page不加任何latch                       |
| BTR_MODIFY_TREE      | 修改B+树，index加x-latch，leaf page加x-latch                 |
| BTR_CONT_MODIFY_TREE | 修改B+树（继续），函数开始前对index已加x-latch，对leaf page已加x-latch |
| BTR_SEARCH_PREV      | 查找leaf page→prev，对leaf page加s-latch                     |
| BTR_MODIFY_PREV      | 修改leaf page→prev，对leaf page加x-latch                     |
| BTR_SEARCH_TREE      | 搜索B+树                                                     |
| BTR_CONT_SEARCH_TREE | 搜索B+树（继续）                                             |

如果btr_cur_search_to_nth_level最终定位到的页不是叶子节点，即level大于0，那么对页加x-latch，同时不释放索引内存对象上的latch保护；否则根据latch_mode对叶子节点加上latch保护，并根据latch_mode选择是否释放索引对象上的latch保护。

从上面的latch_mode中我们可以发现，InnoDB总是先对index加上s-latch保护，随后进行页的操作，如果insert/update/djelete操作不会引起非叶子节点发生变化，即不会发生分裂、合并、树的高度变化，则在定位到leaf page后立即释放index的s-latch，这种方式称为乐观方式；否则，立即释放index及leaf page上的latch，并通过悲观方式对index和leaf page加x-latch保护。

{{< hint info>}}

这里的latch_mode指定是朴素的latch方式，也就是优化前的index concurrency control

{{</hint>}}

另外，latch_mode中还包含以下的flag：

- BTR_INSERT
- BTR_ESTIMATE
- BTR_IGNORE_SEC_UNIQUE
- BTR_DELETE_MARK
- BTR_DELETE
- BTR_ALREADY_S_LATCHED
- BTR_LATCH_FOR_INSERT
- BTR_LATCH_FOR_DELETE
- BTR_RTREE_UNDO_INS
- BTR_MODIFY_EXTERNAL
- BTR_RTREE_DELETE_MARK

BTR_ESTIMATE表示需要对返回的结果集数量进行预估，仅在RANGE查询开始前需要进行预估操作。

## btr cursor

btr cursor用于表示B+ tree的查找，其数据结构如下：

````
struct btr_cur_t {
	dict_index_t*	index;		此次查询所使用的的索引
	page_cur_t	page_cur;		page cursor
	purge_node_t*	purge_node;	/*!< purge node, for BTR_DELETE */
	buf_block_t*	left_block;	左兄弟页，用于BTR_SEARCH_PREV/BTR_MODIFY_PREV
	que_thr_t*	thr;			/*!< this field is only used when btr_cur_search_to_nth_level is called for an index entry insertion: the calling query thread is passed here to be used in the insert buffer */

	// 以下通过btr_cur_search_to_nth_level赋值
	enum btr_cur_method	flag;	使用何种方式查询得到的记录结果
								BTR_CUR_HASH = 1		AHI
								BTR_CUR_HASH_FAIL		AHI->B+ tree
								BTR_CUR_BINARY,			B+ tree
								BTR_CUR_INSERT_TO_IBUF	/*!< performed the intended insert to the insert buffer */
								BTR_CUR_DEL_MARK_IBUF	/*!< performed the intended delete mark in the insert/delete buffer */
								BTR_CUR_DELETE_IBUF		/*!< performed the intended delete in the insert/delete buffer */
								BTR_CUR_DELETE_REF	/*!< row_purge_poss_sec() failed */
	ulint		tree_height;	查询所在的层
	ulint		up_match;		page_cur_search_with_match设置
	ulint		up_bytes;		page_cur_search_with_match设置
	ulint		low_match;		page_cur_search_with_match设置
	ulint		low_bytes;		page_cur_search_with_match设置
	ulint		n_fields;		AHI介绍
	ulint		n_bytes;		AHI介绍
	ulint		fold;			fold value，用于BTR_CUR_HASH
	btr_path_t*	path_arr;		查询得到记录所走过的路径（保存每个路径上的查询信息）
};
````

其中，btr_path_t用于描述B+ tree自上而下的查询路径，每一层一个slot表示（即一个btr_path）

````
struct btr_path_t {
	ulint	nth_rec;			查询到的记录是页中的第几个（包括伪记录）
	ulint	n_recs;				页中的记录总数（不包括伪记录）
	ulint	page_no;			page no
	ulint	page_level;			page level，通过page level的变化来感知是否发生页的重组
};
````

比如在一个page中的记录信息如下：

````
	records:             (inf, a, b, c, d, sup)
	index of the record:    0, 1, 2, 3, 4, 5
````

如果查找的是记录c，则nth_rec=3，n_recs=4。

我们通过nth_rec、n_recs可以预估一个range查询返回的记录数量（btr_estimate_n_rows_in_range）：

1. 统计range查询时两次定位之间的记录数量，即n_rows
2. 统计下层共有多少记录：n_rows = n_rows * 每个page的平均记录数
3. 当查询到leaf page后，预估range的记录数量：如果树的层数<=2，则预估值 = n_rows；否则预估值 = 2 * n_rows

在B+ tree中一个索引页至少有2个记录，所以预估值最小为n_rows * 2。

我们接下来看一个例子：

![InnoDB_b+tree_btr_cursor_btr_path](/InnoDB_b+tree_btr_cursor_btr_path.png)

比如range查询为[40, 120)，首先通过btr_cur_search_to_nth_level(PAGE_CUR_GE 40)定位到记录40，此时查询路径为：

````
path1[0].nth_recs = 1, path1[0].n_rec = 4 // root page
path1[1].nth_recs = 1, path1[1].n_rec = 4 // leaf page
````

如果通过btr_cur_search_to_nth_level(PAGE_CUR_LE 120)定位到记录120，则查询路径为：

````
path2[0].nth_recs = 3, path2[0].n_rec = 4 // root page
path2[1].nth_recs = 1, path2[1].n_rec = 3 // leaf page
````

预估的流程为：

1. 统计root page页的两次记录之间的距离：n_rows = path2[0].nth_recs - path2[0].nth_recs = 3 - 1 = 2
2. 计算leaf page的平均数量：(path1[1].n_rec + path2[1].n_rec) / 2 = (4 + 3) / 2 = 7 / 2 = 3
3. 统计下层（leaf page）共有多少记录：n_rows = n_rows * (leaf page的平均数量) = 2 * 3 = 6
4. 因为已到leaf page，且树的层数<=2，预估值 = n_rows = 6

可以看到range查询实际需要遍历的记录数为7，这里预估值为6，相差不多。

更进一步，InnoDB还考虑了两次查询之间树的变化情况（高度、页）。

## 持久游标

btr0cur模块用于对B+树索引进行search、insert、update、delete操作，并产生对应的undo log和redo log，并处理可能出现的page split和merge。但是，在大多数情况下，上层并不直接调用btr0cur中的这些相关函数，比如btr_cur_search_to_nth_level，而是通过持久游标（persistent cursor）的对象来处理select、update、delete，并且，查询到的记录也会保存在持久游标中。

在进行select、update、delete操作时，首先需要定位到第一条记录，然后开始扫描下一个记录（fetch next record），直到扫描到不符合条件的记录为止。持久游标用于保存（btr_pcur_store_position）每次查询到的记录（保存在old_rec中），待查询下一条记录时，首先恢复上一次查询的记录（如果页没有发生改变，直接使用old_rec来定位下一条记录，否则，要根据old_rec中的索引键值重新定位记录再进行查询（btr_pcur_restore_position）），然后再获取下一条记录。这样设计的原因是当用户扫描记录时，页中的记录可能会发生变化。这时若按照之前的记录进行扫描，可能得到错误的情况。比如页中的记录发生了变化，或者页发生了split/merge。

# B+ tree中变更记录

## 插入记录

在InnoDB中，往B+ tree中插入记录采用乐观插入和悲观插入两个阶段，首先采用乐观插入（btr_cur_optimistic_insert），如果发现此次插入操作会导致页的分裂，则对页进行一次整理，再尝试一次，如果还是不行，再采用悲观插入（btr_cur_pessimistic_insert）。

### 乐观插入

首先来看一下乐观插入的参数：

````
btr_cur_optimistic_insert(
    ulint       flags,  0
                        BTR_NO_UNDO_LOG_FLAG    不需要记undo log，比如回滚时不需要再次产生undo log
                        BTR_NO_LOCKING_FLAG     表示插入后不需要对查询的记录加锁。比如对于change buffer，不会对其进行并发的读取，因此不需要进行锁保护
                        BTR_KEEP_SYS_FLAG       不含有该bit时，更新记录的隐藏列rollback pointer。比如如果插入的是辅助索引或者非叶子节点，记录不含有隐藏列，需要设置该位
                        BTR_KEEP_POS_FLAG       btr_cur_pessimistic_update() must keep cursor position when moving columns to big_rec
                        BTR_CREATE_FLAG         the caller is creating the index or wants to bypass the index->info.online creation log
                        BTR_KEEP_IBUF_BITMAP    the caller of btr_cur_optimistic_update() or btr_cur_update_in_place() will take care of updating IBUF_BITMAP_FREE
    btr_cur_t*  cursor, cursor定位到insert point（即待插入记录的前一条记录），模式为PAGE_CUR_LE | BTR_INSERT
    ulint**     offsets,插入后的物理记录的original offset
    mem_heap_t** heap,
    dtuple_t*   entry,  待插入的记录（逻辑记录）
    rec_t**     rec,    插入后的记录（物理记录）
    big_rec_t** big_rec,如果插入的是大记录，返回插入后的大记录（extern指向的big rec）
    ulint       n_ext,  大记录中存储的列数
    que_thr_t*  thr,    query thread
    mtr_t*      mtr)
````

乐观插入的执行流程如下：

1. 将待插入的记录（逻辑记录）转换为物理记录，计算页是否有足够的空间来容纳。这里需要注意的是，InnoDB要求插入操作保留1/32页的大小作为预留空间（fill factor）。其目的是降低update操作时页的分裂概率。
2. 如果插入的tuple需要转换为大记录格式，则首先将over-flow page的数据保存在big_rec中，待btr_cur_optimistic_insert返回DB_SUCCESS后，再向over-flow page插入big_rec。从这里可以看出，大记录的插入是分为两个阶段进行的。
3. 进行碎片整理后，再次尝试插入
4. 如果还没有足够的空间，则返回DB_FAIL
5. 检查锁的信息，并生成对应的undo日志（btr_cur_ins_lock_and_undo）。若锁检测到下一个记录已经被其他事务持有锁，则等待，返回DB_WAIT_LOCK。
6. 向页中插入记录（page_cur_tuple_insert）
7. 更新AHI
8. 不包含BTR_NO_LOCKING_FLAG，更新锁信息（lock_update_insert）
9. 最后若插入的对象是插入缓冲时（插入缓冲本身也是一棵B+树），更新对应插入缓冲位图页中的信息。
10. 函数执行完成后，会释放持有页的x-latch，如果还有行溢出数据（big_rec），则还要对index和leaf page加x-latch，再将溢出的列数据存放到over-flow page中（btr_store_big_rec_extern_fields）。

### 悲观插入

悲观插入的参数和乐观插入完全相同。

但有以下几点不同：

- 持有了index的x-latch，影响了B+ tree的并发
- 为了避免死锁，除了持有page自身的x-latch外，还需要持有前后兄弟节点的x-latch
- 确定悲观插入后，需要预留一些区的空间，以保证B+ tree的结构变化一定有足够的磁盘空间（3个区）可以保证这次操作完成。插入操作完成后会释放这部分预留空间。
- 在InnoDB中，对于非叶子节点的插入都是悲观插入（btr_insert_on_non_leaf_level）

悲观插入的执行流程如下：

1. 检查锁的信息，并生成对应的undo日志（btr_cur_ins_lock_and_undo）。若锁检测到下一个记录已经被其他事务持有锁，则函数返回DB_WAIT_LOCK。
2. 将待插入的记录（逻辑记录）转换为物理记录，然后可以计算页是否有足够的空间来容纳，如果没有，返回DB_FAIL。
3. 如果插入的tuple需要转换为大记录格式，则首先将over-flow page的数据保存在big_rec中，待btr_cur_optimistic_insert返回DB_SUCCESS后，再向over-flow page插入big_rec。
4. 经过之前的检查，这一步确保记录是一定能够插入页中的：
   root page插入（btr_root_raise_and_insert）
   非root page插入（btr_page_split_and_insert）
5. 更新AHI（btr_search_update_hash_on_insert）
6. 不包含BTR_NO_LOCKING_FLAG，更新锁信息（lock_update_insert）
7. 最后若插入的对象是插入缓冲时（插入缓冲本身也是一棵B+树），更新对应插入缓冲位图页中的信息。
8. 函数执行完成后，会释放持有页的x-latch，如果还有行溢出数据（big_rec），则还要对index和leaf page加x-latch，再将溢出的列数据存放到over-flow page中（btr_store_big_rec_extern_fields）。

## 更新记录

和插入操作一样，更新分为非主键更新和主键更新两个场景。这是因为，node pointer在clustered index和secondary index所存储的key是不同的。

### 非主键更新

非主键更新指的是对非PK进行update，比如非PK列，或者辅助索引列。

非主键更新分为乐观和悲观两种。其中，乐观更新又分为原地更新（update in place）和普通的乐观更新。

#### 原地更新

原地更新（row_upd_changes_field_size_or_external）是指更新记录中的各个列大小在更新过程中没有发生改变（注意，并不是记录的整体大小没有发生变化）。另外，含有extern属性的列不能原地更新。

更新操作中变化列的新值保存在upd_t（update vector）中：

````
/* Update vector structure */
struct upd_t{
    mem_heap_t* heap;      
    ulint       info_bits;  // info_bits
    dtuple_t*   old_vrow;   /*!< pointer to old row, used for virtual column update now */
    ulint       n_fields;   // 新列的数量
    upd_field_t*    fields; // 新列
};
 
/* Update vector field */
struct upd_field_t{
    unsigned    field_no:16;// 列号
    dfield_t    new_val;    // 新值
    dfield_t*   old_v_val;  /*!< old value for the virtual column */
};
````

对于辅助索引来说，一般都不是原地更新，这是因为如果更新了辅助索引的列，一般都会引起索引的位置发生变化（'AAA' → 'BBB'），只有一种情况比较特殊，即只有字符集发生改变，且排序规则没有改变，这时才会原地更新。

change buffer tree也不能是原地更新。

原地更新（btr_cur_update_in_place）的具体流程如下：

1. 如果是压缩页，检查是否有足够的空间（btr_cur_update_alloc_zip）
2. 检查lock，生成undo log
3. 更新系统列（trx ID + rollback pointer）
4. 更新AHI
5. 将update vector更新到物理记录中
6. 处理extern列（之前设置了delete mark，现在需要把extern列设为record owned）
7. 更新insert buffer

#### 乐观更新

普通的乐观更新就是delete mark+insert，具体流程如下图所示：

![InnoDB_b+tree_btr_cur_optimistic_update](/InnoDB_b+tree_btr_cur_optimistic_update.png)

其中，乐观更新（btr_cur_optimistic_update）的过程中仅需要持有记录所在页的x-latch。

从上面的流程我们可以看出，乐观更新首先判断是否可以原地更新，如果不可以，再进行delete mark + insert。而在delete mark+insert中，因为记录还在同一个页中，并且主键并没有改变，只是新旧记录的heap no发生了变化，所以只需要更新锁信息即可（不需要创建锁）：将原来的锁信息移动到伪记录infimum上，待记录更新完成后再将伪记录上的锁移动到新记录上。记录的锁通过移动完成，不需要先删除后新建的方式来完成，提高了锁管理的效率。

#### 悲观更新

在悲观更新中，首先检查是否可以乐观更新，这是因为更新记录所在的page可能已经split，乐观更新有可能成功。

对于锁信息的处理方式和乐观更新一样，即首先将更新记录的锁信息移动到伪记录infimum上，待更新完成后将锁信息移动回原记录上。但是，因为悲观更新会引起页的分裂操作，因此在某些情况下，需要对伪记录supremum锁进行修正。发生这种修正的条件为：

- 更新的记录不是页中的第一个用户记录
- 更新操作会引起页向右进行分裂
- 分裂点记录就是更新记录

那么这时分裂后原记录所在页的伪记录supremum会继承分裂点记录后的锁信息，并设置为gap类型。但是由于更新时记录本身已经有锁，因此伪记录supremum需要继承的是更新的记录，也就是分裂点记录锁的信息，而在分裂时，其锁信息被移动到了伪记录infimum上。

### 主键更新

在真正的生产环境中，应该尽量避免更新主键。

主键更新的流程：

- 原主键记录 delete mark
- insert新主键记录
- purge线程判断是否有其他事务引用已删除的主键记录，如果没有，则彻底删除

这里我们对第2步中的extern列进行详细说明：extern列会把大部分数据存放在over-flow page中。如果更新不涉及extern列，那么新插入的主键记录需要继承（inherit）over-flow page的数据。同样，在purge线程清理已删除的主键记录时，如果其不拥有（owned）over-flow page，则不删除over-flow page。类似的，如果回滚时被删除的记录列继承over-flow page，那么同样不能删除over-flow page。继承和拥有通过extern列上的BTR_EXTERN_OWNER_FLAG和BTR_EXTERN_INHERITED_FLAG进行标记：BTR_EXTERN_OWNER_FLAG为0表示记录拥有该over-flow page，purge线程可以进行删除。回滚时，如果BTR_EXTERN_INHERITED_FLAG为1，则不能删除over-flow page。

## 删除记录

因为InnoDB支持MVCC，所以对于删除操作，分为2个步骤：

1. delete mark
2. purge

其中，步骤#1在用户线程（事务）中完成；步骤#2通过后台的purge线程完成，purge线程检查是否还有其他事务还在使用该记录，如果没有，则将记录彻底删除，记录所占用的空间放回到page的PAGE_FREE链表。

### delete mark

其中，根据删除对象分为clustered index和secondary index：

| clustered index的delete markbtr_cur_del_mark_set_clust_rec   | secondary index的delete markbtr_cur_del_mark_set_sec_rec     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1. 加锁（lock_clust_rec_modify_check_and_lock），<font color="blue">可能存在的隐式锁转化为显式锁</font>，加显示锁（LOCK_X \| LOCK_REC_NOT_GAP） <br />2. <font color="blue">生成undo log</font><br />3. delete mark<br />4. <font color="blue">更新系统列</font><br />5. 生成redo log | 1. 加锁（btr_cur_del_mark_set_sec_rec），加显示锁（LOCK_X \| LOCK_REC_NOT_GAP） <br />2. delete mark<br />3. 生成redo log |

从上面可以看出来，两种索引的delete mark区别主要有两点：锁转换和支持数据访问的MVCC。

产生的redo log如下：

![InnoDB_b+tree_index_delete_mark_mtr_log](/InnoDB_b+tree_index_delete_mark_mtr_log.png)

### purge deleted index record

purge线程对delete mark记录进行gc，具体的策略是：先删除辅助索引，再删除聚簇索引中的deleted index record，调用链如下：

````
row_purge
    row_purge_record
        row_purge_del_mark
            row_purge_remove_sec_if_poss            // 删除辅助索引中的deleted index record
                row_purge_remove_sec_if_poss_leaf   // 只删除index record
                    btr_cur_optimistic_delete       // 乐观删除
                row_purge_remove_sec_if_poss_tree   // 需要修改index tree
                    btr_cur_pessimistic_delete      // 悲观删除
            row_purge_remove_clust_if_poss          // 删除聚簇索引中的deleted index record
                row_purge_remove_clust_if_poss_low
                    BTR_MODIFY_LEAF：
                        btr_cur_optimistic_delete   // 乐观删除
                    BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE：
                        btr_cur_pessimistic_delete  // 悲观删除
````

从上面的调用我们可以得知：

如果是叶子节点（BTR_MODIFY_LEAF），则采用乐观删除（btr_cur_optimistic_delete），仅对记录所在的页加x-latch，调用page_cur_delete_rec，并且有如下前提：

- 删除的记录不包含extern属性的列
- 删除操作不会引起页发生合并操作

否则（BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE）采用悲观删除（btr_cur_pessimistic_delete），不仅对记录所在的页加x-latch，还对其兄弟节点和index持有x-latch。

悲观删除具体还需要区分该记录的位置：位于叶子节点还是非叶子节点，删除叶子节点中的记录，不需要更新上层叶子的键值，这样的设计可以减少对于上层节点更新的开销，如下图所示：

![InnoDB_b+tree_btr_delete_leaf_node](/InnoDB_b+tree_btr_delete_leaf_node.png)

从上面的图中可以看到，删除leaf node上的记录25，不需要更新上层的node pointer，因为这并没有破坏查询得到的结果。

而删除非叶子节点记录，则需要根据该记录的不同情况进行不同的处理：

- 删除的非叶子节点记录含有REC_INFO_MIN_REC_FLAG标识，并且记录所在page   是B+ tree当前层的第一个page，删除记录，并对下一个记录设置REC_INFO_MIN_REC_FLAG标识
- 删除的非叶子节点记录含有REC_INFO_MIN_REC_FLAG标识，并且记录所在page不是B+ tree当前层的第一个page，删除记录，并更新上一层的node pointer
- 直接删除记录

具体示例如下图所示：

![InnoDB_b+tree_btr_delete_branch_node](/InnoDB_b+tree_btr_delete_branch_node.png)

{{< hint danger>}}

留给读者一个小问题，InnoDB是如何保证B+ tree中叶子节点不会出现latch死锁？

{{</hint>}}

# 自适应哈希索引（转）

本节AHI转自阿里内核月报

自适应哈希索引，简称AHI（Adaptive Hash Index），目的是为了可以使InnoDB在有足够的内存和特定的工作负载下，看起来更像一个内存数据库，并且不会牺牲任何事务特征和稳定性，即对热数据的访问提供加速，也可以称之为一种特定的page table。

我们从前面可以看到，InnoDB是基于索引组织表来构建数据的，因此查询都是基于B+ tree来进行的，而B+ tree的查询效率和树的高度成正比，tree的高度代表了I/O次数。而如果采用哈希查询，则复杂度为O(1)。InnoDB通过观察B+ tree的搜索模式，在特定条件下使用index key的前缀（前缀长度任意）自动建立哈希索引。

被称为”自适应“的原因，一是其根据观察搜索模式自动建立索引的机制，二是在访问时首先查询哈希索引，对用户是透明的。从某种意义上来说，AHI可以理解为建立在B+ tree索引上的索引。

另外，在某些场景下，AHI并不适用，比如高并发的join操作，模糊或者范围查询。在这种场景下，维护AHI的成本比其带来的收益更大，所以需要根据具体的访问方式来决定是否开启AHI。

AHI和B+ tree索引的区别如下：

|                | B+ tree索引                | AHI              |
| :------------- | :------------------------- | :--------------- |
| 查询时间复杂度 | O(logd(N))                 | O(1)             |
| 是否持久化     | 是（并通过日志保证完整性） | 否（仅在内存中） |
| 索引对象       | 页                         | 记录             |

InnoDB在进行B+ tree查询搜索时，为了减少了重新寻路的次数，采用多种策略来提高性能：

1. 对于连续记录扫描，InnoDB在满足比较严格的条件时采用row cache的方式连续读取8条记录（并将记录格式转换成Server层的MySQL Format），存储在用户线程私有的row_prebuilt_t::fetch_cache中。这样一次寻路就可以获取多条记录，在server层处理完一条记录后，可以直接从cache中取数据而无需再次寻路，直到cache中数据取完，再进行下一轮。
2. 另一种方式是，当一次进入InnoDB层获得数据后，在返回server层前，当前的btr cursor会暂存到持久游标（row_prebuilt_t::pcur）中，当再次返回InnoDB层捞数据时，如果对应的block没有发生任何修改，则可以继续沿用之前存储的cursor，无需重新定位。
3. 通过观察记录的搜索模式，创建AHI，其后的查询可以直接定位记录。

## 创建AHI

AHI就是一个哈希表（btr_search_sys::hash_tables），在InnoDB的buffer pool初始化过程中创建，大小为buffer pool的1/64。

从这里可以看出：

- AHI占用buffer pool的空间
- 在MySQL 5.7.8之前，AHI使用btr_search_latch latch来控制并发访问，在高负载的情况下这可能会成为性能的瓶颈点。于是，在MySQL 5.7.8中，将AHI分为多个partition（默认为8），每个partition有自己的latch来控制并发访问（btr_search_latches），来降低latch的争用。
- MySQL从5.7.5可以运行时调整buffer pool size，当buffer_pool动态调整的大小超过一定程度时（扩大2倍或者缩小2倍以上），重新分配AHI（btr_search_sys_resize()）

{{< hint info>}}

可以通过观察show engine innodb status中的SEMAPHORES使用情况来判断latch的争用情况，比如看到大量线程在btr0sea.cc上争用latch，这种情况下，可能需要考虑关闭AHI

Percona有一篇文章讲AHI的btr_search_latch和index lock的性能问题：[Index lock and adaptive search – next two biggest InnoDB problems](https://www.percona.com/blog/2010/02/25/index-lock-and-adaptive-search-next-two-biggest-innodb-problems/)，MySQL 5.7通过对AHI拆分为多个partition（commit id: [ab17ab91](https://github.com/mysql/mysql-server/commit/ab17ab91ce18a47bb6c5c49e4dc0505ad488a448)）以及引入更细粒度的索引锁协议（[WL#6326 fix index→lock contention](https://dev.mysql.com/worklog/task/?id=6326)）来解决这两个问题

{{</hint>}}

函数调用链如下：

````
innobase_init()
    innobase_start_or_create_for_mysql()
        buf_pool_init(srv_buf_pool_size, srv_buf_pool_instances)                    // 2147483648(2G) 8
            btr_search_sys_create(buf_pool_get_curr_size() / sizeof(void*) / 64)    // 2147221504/8/64=4193792
                ib_create((hash_size / btr_ahi_parts, ...)                          // 4193792/8=524224
                     hash_create(n);                                                // 524224 -> 553193
````

数据结构如下：

![InnoDB_b+tree_AHI_btr_search_sys](/InnoDB_b+tree_AHI_btr_search_sys.png)

## 维护AHI的查询信息

在每个索引对象上（dict_index_t->search_info），维护了记录的查询信息（btr_search_t），用来启发当前索引对AHI的使用。

当索引对象添加到数据字典缓存时，创建该索引的查询信息。随后，搜索从B+ tree的root page开始，定位到leaf page时，更新查询信息（btr_search_info_update）。代码如下：

btr_cur_search_to_nth_level

````
level == 0
#ifdef BTR_CUR_ADAPT
        /* We do a dirty read of btr_search_enabled here.  We
        will properly check btr_search_enabled again in
        btr_search_build_page_hash_index() before building a
        page hash index, while holding search latch. */
        if (btr_search_enabled && !index->disable_ahi) {
            btr_search_info_update(index, cursor);
        }
#endif
````

当info->hash_analysis值超过BTR_SEARCH_HASH_ANALYSIS（17）时，也就是说对该索引寻路到叶子节点17次后，才会去做AHI分析，这是为了避免频繁的索引查询分析产生的过多CPU开销。

InnoDB通过索引条件构建一个可用于查询的tuple，而AHI需要根据tuple定位到叶子节点上记录的位置，既然AHI是建立在B+树索引上的索引，其的键值就是通过B+树索引的前N列的值计算得来的，所有的查询信息（search info）统计都是为了确定一个合适的"Ｎ" ，这个值是动态的，会跟随应用的负载（查询模式）自适应调整并触发block上的AHI rebuild。

维护AHI的查询信息（btr_search_info_update_slow）分为三部分，我们依次进行介绍：

1. 更新index的查询信息（index search info）
2. 更新block的查询信息（block search info）
3. build AHI

### 更新索引的查询信息

通过之前的介绍我们知道，查询首先将结果保存在btr cursor中，其中，通过二叉查找比较得到的匹配结果放在变量up_match、up_bytes、low_match、low_bytes中。AHI根据这4个变量来更新索引的查询信息（btr_search_info_update_hash），即更新search info的left_side、n_fields、n_bytes。其中left_size表示在相同索引前缀时采用最左/最右记录构建AHI，n_fields表示推荐的AHI列数，n_bytes为索引中的字符串前缀大小。这样，通过left_size+n_fields就可以确定选择哪些列来作为索引前缀构建AHI的哈希记录（fold, rec）。当用户的SQL的索引前缀列的个数大于等于构建AHI时的前缀索引，就可以用上AHI。

两种情况需要build建议的前缀索引列（set a new recommendation）：

- 当前是第一次为该索引做AHI分析，btr_search_t::n_hash_potential = 0，需要构建建议的前缀索引列
- 新的记录匹配模式发生了变化(info->left_side == (info->n_fields <=cursor->low_match))，需要重新设置前缀索引列

代码如下：

````c++
cmp = ut_pair_cmp(cursor->up_match, cursor->up_bytes,
          cursor->low_match, cursor->low_bytes);
if (cmp == 0) {
    info->n_hash_potential = 0;
 
    /* For extra safety, we set some sensible values here */
 
    info->n_fields = 1;
    info->n_bytes = 0;
 
    info->left_side = TRUE;
 
} else if (cmp > 0) {
    info->n_hash_potential = 1;
 
    if (cursor->up_match >= n_unique) {
 
        info->n_fields = n_unique;
        info->n_bytes = 0;
 
    } else if (cursor->low_match < cursor->up_match) {
 
        info->n_fields = cursor->low_match + 1;
        info->n_bytes = 0;
    } else {
        info->n_fields = cursor->low_match;
        info->n_bytes = cursor->low_bytes + 1;
    }
 
    info->left_side = TRUE;
} else {
    info->n_hash_potential = 1;
 
    if (cursor->low_match >= n_unique) {
 
        info->n_fields = n_unique;
        info->n_bytes = 0;
    } else if (cursor->low_match > cursor->up_match) {
 
        info->n_fields = cursor->up_match + 1;
        info->n_bytes = 0;
    } else {
        info->n_fields = cursor->up_match;
        info->n_bytes = cursor->up_bytes + 1;
    }
 
    info->left_side = FALSE;
}
````

从上述代码可以看到，在low_match和up_match之间，选择小一点match的索引列数的来进行设置，但不超过唯一确定索引记录值的列的个数：

1. 当low_match小于up_match时，left_side设置为true，表示相同前缀索引的记录只缓存最左记录
2. 当low_match大于up_match时，left_side设置为false，表示相同前缀索引的记录只缓存最右记录

另外，如果不是第一次进入，有两种情况需要递增n_hash_potential（如果使用AHI构建索引，潜在的可能成功的次数）：

- 本次查询的up_match和当前推荐的前缀索引n_fields都能唯一决定一条索引记录（比如unique index），则根据search_info推荐的前缀索引列构建AHI肯定能命中

  ````c++
      /* Test if the search would have succeeded using the recommended
      hash prefix */
   
      if (info->n_fields >= n_unique && cursor->up_match >= n_unique) {
  increment_potential:
          info->n_hash_potential++;
   
          return;
      }
  ````

- 本次查询的tuple可以通过建议的前缀索引列构建的AHI定位到

  ````c++
  cmp = ut_pair_cmp(info->n_fields, info->n_bytes,
            cursor->up_match, cursor->up_bytes);
   
  if (info->left_side ? cmp <= 0 : cmp > 0) {
      goto increment_potential;
  }
  ````


从上面我们可以得知，AHI对于同一个索引使用相同的查询模式进行查询的场景，收益最大。

### 更新block的查询信息

我们更新完index的查询信息后，接着需要更新数据页block的查询信息（btr_search_update_block_hash_info）。

需要更新的查询信息有：

- btr_search_info::last_hash_succ：是否最近一次通过AHI查找成功
- buf_block_t::n_hash_helps：如果使用当前推荐的前缀索引列构建AHI，可能命中的次数
- buf_block_t::left_size：同index search info
- buf_block_t::n_fields ：同index search info
- buf_block_t::n_fields ：同index search info

处理流程：

1. 首先将btr_search_info::last_hash_succ设为false。这意味着每次分析一个新的block，都会导致AHI短暂不可用。
2. 比较index search info和block search info的三元组，计算n_hash_helps
3. 判断是否为整个page构建AHI

这里展开说一下#3，即当满足下面3个条件时，就会为整个page构建AHI：

- block可能命中的次数（n_hash_helps）大于page总记录数的1/16
- index（连续）潜在的可能成功的次数（n_hash_potential）大于100
- 还没有为当前block构造过索引（!block->index），或者当前block上已经构建了AHI索引并且其可能命中的次数（n_hash_helps）大于page总记录数的2倍，或者当前block上推荐的前缀索引列和当前已构建的AHI索引发生了变化。

### build AHI

如果在上一步中判断可以为当前的page构建AHI（build_index = TRUE），则根据当前推荐的索引前缀build AHI。

build分为3个阶段：

1. 检查阶段：检查是否已开启AHI，加btr_search_latch s-latch，如果block上已经构建了AHI（block->index）但是和当前推荐的不同，则清空block的AHI对应项（btr_search_drop_page_hash_index），释放btr_search_latch s-latch
2. 搜集阶段：根据推荐的索引列计算记录的fold值，将page中对应记录的内存地址放入（fold, rec*）数组中
   left_side不同，相同的前缀索引列值不一样，比如page上记录为 (2, 1), (2, 2), (5, 3), (5, 4), (7, 5), (8, 6)，n_fields＝１
   如果left_side = true （最左），则hash存储的记录为(2, 1),  (5, 3), (7, 5), (8, 6)
   如果left_side = false（最右），则hash存储的记录为(2, 2), (5, 4), (7, 5), (8, 6)
3. 插入阶段：加btr_search_latch x-latch，将搜集到的（fold, rec*）插入到AHI中

### index search info

我们整体看一下index search info：

````
/** The search info struct in an index */
struct btr_search_t{
    ulint   ref_count;          // AHI中的block数（即block->index的个数）
 
    buf_block_t* root_guess;    /*!< the root page frame when it was last time fetched, or NULL */
    ulint   withdraw_clock;     /*!< the withdraw clock value of the buffer pool when root_guess was stored */
    ulint   hash_analysis;      // 在B+树索引中进入leaf page的次数，超过BTR_SEARCH_HASH_ANALYSIS（17），则更新index search info
    ibool   last_hash_succ;     // 是否最近一次通过AHI查找成功（该值不一定准确）
    ulint   n_hash_potential;   // 如果使用AHI构建索引，潜在的可能成功的次数
 
    ulint   n_fields;           // 推荐的AHI列数
    ulint   n_bytes;            // 索引中的字符串前缀大小
    ibool   left_side;          // 在相同索引前缀时采用最左/最右记录构建AHI
 
    ulint   n_hash_succ;        // AHI查询命中的次数
    ulint   n_hash_fail;        // AHI查询miss的次数
    ulint   n_patt_succ;        // 未使用
    ulint   n_searches;         // B+树的查询次数
};
````

## 使用AHI

在B+ tree搜索中（btr_cur_search_to_nth_level）使用AHI，代码如下：

````c++
#ifndef BTR_CUR_ADAPT
    guess = NULL;
#else
    info = btr_search_get_info(index);
 
    if (!buf_pool_is_obsolete(info->withdraw_clock)) {
        guess = info->root_guess;
    } else {
        guess = NULL;
    }
 
#ifdef BTR_CUR_HASH_ADAPT
 
# ifdef UNIV_SEARCH_PERF_STAT
    info->n_searches++;
# endif
    /* Use of AHI is disabled for intrinsic table as these tables re-use
    the index-id and AHI validation is based on index-id. */
    if (rw_lock_get_writer(btr_get_search_latch(index))
        == RW_LOCK_NOT_LOCKED
        && latch_mode <= BTR_MODIFY_LEAF
        && info->last_hash_succ
        && !index->disable_ahi
        && !estimate
# ifdef PAGE_CUR_LE_OR_EXTENDS
        && mode != PAGE_CUR_LE_OR_EXTENDS
# endif /* PAGE_CUR_LE_OR_EXTENDS */
        && !dict_index_is_spatial(index)
        /* If !has_search_latch, we do a dirty read of
        btr_search_enabled below, and btr_search_guess_on_hash()
        will have to check it again. */
        && UNIV_LIKELY(btr_search_enabled)
        && !modify_external
        && btr_search_guess_on_hash(index, info, tuple, mode,
                    latch_mode, cursor,
                    has_search_latch, mtr)) {
        btr_cur_n_sea++;
 
        DBUG_VOID_RETURN;
    }
# endif /* BTR_CUR_HASH_ADAPT */
#endif /* BTR_CUR_ADAPT */
````

我们从中可以看到，需要满足如下条件才能够使用AHI：

- 没有加btr_search_latch写锁。如果加了写锁，可能操作时间比较耗时，走AHI检索记录得不偿失
- 本次查询或者DML不改变B+树的结构（latch_mode <= BTR_MODIFY_LEAF）
- 最近一次使用AHI（可能）成功（info→last_hash_succ）
- 已开启AHI
- 查询优化阶段的估值操作，例如计算range范围等，典型的堆栈包括：handler::multi_range_read_info_const　–> ha_innobase::records_in_range –> btr_estimate_n_rows_in_range –> btr_cur_search_to_nth_level
- 不是空间索引
- 调用者无需分配外部存储页(BTR_MODIFY_EXTERNAL，主要用于辅助写入大的blob数据，参考struct btr_blob_log_check_t)。

当满足上述所有条件时，才根据当前的查询tuple对象计算fold，并查询AHI（btr_search_guess_on_hash）；只有当前检索使用的tuple列的个数大于等于构建AHI的列的个数时，才能够使用AHI索引。

**btr_search_guess_on_hash**

- 首先用户提供的前缀索引查询条件必须大于等于构建AHI时的前缀索引列数，这里存在一种可能性：索引上的search_info的n_fields 和block上构建AHI时的cur_n_fields值已经不相同了，但是我们并不知道本次查询到底落在哪个block上，这里一致以search_info上的n_fields为准来计算fold，去查询AHI
- 在检索AHI时需要加&btr_search_latch的S锁
- 如果本次无法命中AHI，就会将btr_search_info::last_hash_succ设置为false，这意味着随后的查询都不会去使用AHI了，只能等待下一路查询信息分析后才可能再次启动（btr_search_failure）
- 对于从ahi中获得的记录指针，还需要根据当前的查询模式检查是否是正确的记录位置（btr_search_check_guess）

如果本次查询使用了AHI，但查询失败了（cursor->flag == BTR_CUR_HASH_FAIL），并且当前block构建AHI索引的curr_n_fields等字段和btr_search_info上的相符合，则根据当前cursor定位到的记录插入AHI。参考函数：btr_search_update_hash_ref。

从上述分析可见，AHI如其名，完全是自适应的，如果检索模式不固定，很容易就出现无法用上AHI或者AHI失效的情况。

## shortcut查询模式

在row_search_mvcc函数中，首先会去判断在满足一定条件时，使用shortcut模式，利用AHI索引来进行检索。

只有满足严苛的条件时（例如需要唯一键查询、使用聚集索引、长度不超过八分之一的page size、隔离级别在RC及RC之上、活跃的Read view等等条件，具体的参阅代码），才能使用shortcut：

- 加btr_search_latch的S锁
- 通过row_sel_try_search_shortcut_for_mysql检索记录；如果找到满足条件的记录，本次查询可以不释放 btr_search_latch，这意味着InnoDB/server层交互期间可能持有AHI锁，但最多在10000次（BTR_SEA_TIMEOUT）交互后释放AHI latch。一旦发现有别的线程在等待AHI X 锁，也会主动释放其拥有的S锁。

然而， Percona的开发Alexey Kopytov认为这种长时间拥有的btr_search_latch的方式是没有必要的，这种设计方式出现在很久之前加锁、解锁非常昂贵的时代，然而现在的CPU已经很先进了，完全没有必要，在Percona的版本中，一次shortcut的查询操作后都直接释放掉btr_search_latch（参阅[bug#1218347](https://bugs.launchpad.net/percona-server/+bug/1218347)）。

## AHI运维和监控项

### AHI参数

InnoDB提供了如下AHI参数

[innodb_adaptive_hash_index](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_adaptive_hash_index)：开启/关闭AHI
[innodb_adaptive_hash_index_parts](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_adaptive_hash_index_parts)：AHI分区数

### AHI监控项

打开AHI监控项

````
set global innodb_monitor_enable = module_adaptive_hash;
````

查看AHI的运行状态

**通过show engine innodb status查看AHI各个partition的使用情况**

````
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
...
Hash table size 553193, node heap has 108 buffer(s)
Hash table size 553193, node heap has 122 buffer(s)
Hash table size 553193, node heap has 121 buffer(s)
Hash table size 553193, node heap has 159 buffer(s)
Hash table size 553193, node heap has 154 buffer(s)
Hash table size 553193, node heap has 118 buffer(s)
Hash table size 553193, node heap has 154 buffer(s)
Hash table size 553193, node heap has 89 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
````

**通过INFORMATION_SCHEMA.INNODB_METRICS查看运行时信息**

````
mysql> select status, name, subsystem,count, max_count, min_count, avg_count, time_enabled, time_disabled from INFORMATION_SCHEMA.INNODB_METRICS where subsystem like '%adaptive_hash%';
+----------+------------------------------------------+---------------------+----------+-----------+-----------+-------------------+---------------------+---------------+
| status   | name                                     | subsystem           | count    | max_count | min_count | avg_count         | time_enabled        | time_disabled |
+----------+------------------------------------------+---------------------+----------+-----------+-----------+-------------------+---------------------+---------------+
| enabled  | adaptive_hash_searches                   | adaptive_hash_index |  8038491 |   8038491 |      NULL |  1.22195712800762 | 2019-03-27 11:42:05 | NULL          |
| enabled  | adaptive_hash_searches_btree             | adaptive_hash_index | 58550686 |  58550686 |      NULL | 8.900479966630051 | 2019-03-27 11:42:05 | NULL          |
| disabled | adaptive_hash_pages_added                | adaptive_hash_index |        0 |      NULL |      NULL |              NULL | NULL                | NULL          |
| disabled | adaptive_hash_pages_removed              | adaptive_hash_index |        0 |      NULL |      NULL |              NULL | NULL                | NULL          |
| disabled | adaptive_hash_rows_added                 | adaptive_hash_index |        0 |      NULL |      NULL |              NULL | NULL                | NULL          |
| disabled | adaptive_hash_rows_removed               | adaptive_hash_index |        0 |      NULL |      NULL |              NULL | NULL                | NULL          |
| disabled | adaptive_hash_rows_deleted_no_hash_entry | adaptive_hash_index |        0 |      NULL |      NULL |              NULL | NULL                | NULL          |
| disabled | adaptive_hash_rows_updated               | adaptive_hash_index |        0 |      NULL |      NULL |              NULL | NULL                | NULL          |
+----------+------------------------------------------+---------------------+----------+-----------+-----------+-------------------+---------------------+---------------+
8 rows in set (0.00 sec)
````


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
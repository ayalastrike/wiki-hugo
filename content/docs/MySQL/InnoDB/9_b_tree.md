---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---


在InnoDB中，数据的存储组织为IoT，采用B+ tree的数据结构提供高效访问数据的方式（存取路径）。

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

## fill factor

在InnoDB中，聚簇索引与辅助索引中叶子节点实际存放的记录数量还和填充因子（fill factor）有关。当填充因子小于1/2时需要进行页的合并，因此常态下填充因子肯定总是大于1/2的。此外，填充因子还受到插入的影响，如果是顺序插入，那么页的填充因子比较高，可以达到90%甚至更高。如果插入是无序的，那么按照Jim Gray所提到的，填充因子一般为69%。

对于聚集索引，如果主键是自增ID，那么插入数据是顺序IO；如果副主索引是时间维度的key，那么也是顺序的，都可以有较高的填充因子。而如果主键为散列ID，比如UUID，那么插入就会无序，插入性能会有明显的退化，所以在线上场景中，尽量避免使用UUID类似的值作为主键。。
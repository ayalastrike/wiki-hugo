---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

typora-root-url: ../../../../static

在MySQL源代码中，每个模块都有自己单独的目录存放，里面按照模块名0子模块名.cc来组织。所有头文件都放在include目录下，同时include目录下还有*.ic的文件，这个文件中存放定义的内联函数。

如果要在*.ic中使用宏UNIV_INLINE定义内联，需要#include "univ.i"，即

```c++
#include "univ.i"
UNIV_INLINE
return_value
function foo(param1, ...) ``// 函数声明或实现
{
  ``...
}
```

{{< hint info >}}
注意，这种风格只是适用于c函数，对于ic文件中的类成员函数定义，还是需要手动写inline
{{</hint>}}

**univ.i中UNIV_INLINE的宏定义**

```c++
#ifndef UNIV_MUST_NOT_INLINE
/* Definition for inline version */
#define UNIV_INLINE static inline
#else /* !UNIV_MUST_NOT_INLINE */
/* If we want to compile a noninlined version we use the following macro definitions: */
#define UNIV_NONINL
#define UNIV_INLINE
#endif /* !UNIV_MUST_NOT_INLINE */
```

阅读源码层次

推荐从下至上进行逐层阅读

![InnoDB存储引擎代码模块划分](/InnoDB存储引擎代码模块划分.png)

最下一层是基础管理模块：

- File Manager主要封装了InnoDB对于文件的各类操作，如读、写、异步I/O等。
- Concurrency Manager模块主要封装了引擎内部使用的各类mutex和latch。
- Common Utility模块用于一些基本数据结构与算法的定义，如链表、哈希表等。

图中间虚线标注的部分为InnoDB的内核实现，也就是InnoDB存储引擎中的事务、锁、缓冲区、日志、存储管理、资源管理、索引、change buffer模块，这部分是整个存储引擎的核心。

图最上面的两层是接口层，通过这些接口实现server层与存储引擎的交互。InnoDB存储引擎可以不依赖MySQL数据库，而作为一个嵌入式数据库存在，因此还存在嵌入式的API接口。

详细的目录（模块）说明如下：

| 目录    | 说明               | 文件                                                         |                                                              |
| ------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ut      | 基本数据结构和算法 | ut0byte<br />ut0crc32<br />ut0dbg<br /><br />ut0listut0lstut0memut0newut0rbtut0rndut0vecut0wqueue | 内存双向链表 ib_list内存双向链表 ut_list                     |
| mem     | 内存管理           | mem0mem                                                      | 内存管理                                                     |
| os      | 进程控制           | univos0atomicos0eventos0fileos0procos0thread                 | 定义了POD、编译器hint、代码段宏Atomic Read-Modify-WriteOS Event getpid、大页分配内存PSI key、进程控制原语 |
| sync    | 同步机制           | ut0mutexsync0arrsync0debugsync0rwsync0syncsync0policy        | 定义了mutex的宏、mutex_init()和mutex_destroy()、MutexMonitor   MutexDebug |
| log     | 日志及恢复         | log0loglog0recv                                              | 重做日志恢复                                                 |
| mtr     | mini-transaction   | mtr0typemtr0mtrmtr0logdyn0buf                                | mtr相关定义mtr基本操作mtr日志操作mtr_buf_t                   |
| fsp     | 存储管理           | fsp0filefsp0fspfsp0spacefsp0sysspace                         | 数据文件物理文件结构与实现表空间系统表空间                   |
| fil     | 文件管理           | fil0filos0file                                               | 文件内存数据结构及相关文件操作底层文件操作实现               |
| fut     |                    | fut0lstfut0fut                                               | 磁盘双向链表 flst                                            |
| data    | 逻辑记录           | data0datadata0typedata0types                                 | 逻辑记录逻辑记录的操作逻辑记录数据结构                       |
| rem     | 物理记录           | rem0recrem0cmprem0types                                      | 物理记录物理记录的比较物理记录数据结构                       |
| page    | 索引页             | page0curpage0pagepage0typespage0sizepage0zip                 | 索引页中记录的定位、插入、删除索引页的维护类型定义           |
| lock    | 锁                 | lock0locklock0iterlock0prdtlock0waitlock0typeslock0priv      | 锁模式                                                       |
| btr     | B+树               | btr0btrbtr0bulkbtr0curbtr0pcurbtr0sea                        |                                                              |
| buf     | 缓冲池             | buf0buddybuf0bufbuf0checksumbuf0dblwrbuf0dumpbuf0flubuf0lrubuf0rea |                                                              |
| dict    | 数据字典           | dict0bootdict0creadict0dictdict0loaddict0memdict0statsdict0stats_bg |                                                              |
| ibuf    | change buffer      | ibuf0ibuf                                                    |                                                              |
| row     |                    | row0extrow0ftsortrow0importrow0insrow0logrow0mergerow0mysqlrow0purgerow0quiescerow0rowrow0selrow0truncrow0uninsrow0umodrow0undorow0updrow0vars |                                                              |
| trx     | 事务               | trx0i_strx0purgetrx0rectrx0rolltrx0rsegtrx0systrx0trxtrx0undo |                                                              |
| handler |                    | ha_innodbha_innoparthandler0alteri_s                         |                                                              |
| read    |                    | read0read                                                    |                                                              |
| api     |                    | api0apiapi0misc                                              |                                                              |
| eval    |                    | eval0evaleval0misc                                           |                                                              |
| ha      |                    | ha0haha0storagehash0hash                                     |                                                              |
| mach    |                    | mach0data                                                    |                                                              |
| pars    |                    | lexyypars0grmpars0lexpars0optpars0parspars0sym               |                                                              |
| que     |                    | que0que                                                      |                                                              |
| srv     |                    | srv0concsrv0monsrv0srvsrv0start                              |                                                              |
| usr     |                    | usr0sess                                                     |                                                              |
| gis     |                    | gis0geogis0rtreegis0sea                                      |                                                              |
| fts     |                    | fts0astfts0blexfts0configfts0ftsfts0optfts0parsfts0pluginfts0quefts0sqlfts0tlex |                                                              |



| 目录       | 文件                               | 说明                                |
| ---------- | ---------------------------------- | ----------------------------------- |
| fut        | fut0fut                            | File-based utilities                |
| fu0lst     | File-based list utilities          |                                     |
| ha         | ha0ha                              | The hash table with external chains |
| ha0storage | Hash storage                       |                                     |
| hash0hash  | The simple hash table utility      |                                     |
| mem        | mem0mem                            | The memory management               |
| ut         | ut0byte                            | Byte utilities                      |
|            |                                    |                                     |
| ut0crc32   | CRC32                              |                                     |
| ut0dbg     | Debug utilities for Innobase.      |                                     |
| ut0list    | A double-linked list               |                                     |
| ut0mem     | Memory primitives                  |                                     |
| ut0new     | Instrumented memory allocator.     |                                     |
| ut0rbt     | Red-Black tree implementation      |                                     |
| ut0rnd     | Random numbers and hashing         |                                     |
| ut0ut      | Various utilities for Innobase.    |                                     |
| ut0vec     | A vector of pointers to data items |                                     |
| ut0wqueue  | work queue.                        |                                     |
|            |                                    |                                     |

db0err Error Codes

eval

```
File Name   What Name Stands For  Size    Comment Inside File ---------   --------------------  ------  ------------------- eval0eval.c Evaluating/Evaluating 17,061  SQL evaluator eval0proc.c Evaluating/Procedures  5,001  Executes SQL procedures
```

The evaluating step is a late part of the process of interpreting an SQL statement --- parsing has already occurred during \pars (PARSING).

The ability to execute SQL stored procedures is an InnoDB feature, but MySQL handles stored procedures in its own way, so the eval0proc.c program is unimportant.

# 代码风格

InnoDB的代码缩进风格更接近于K&R风格：所有的非函数语句块（if、switch、for、while、do），起始大括号放在行尾，而把结束大括号放在行首。函数开头的左花括号放到最左边。

此外，每个文件都包含一段简短说明其功能的注释开头，同时每个函数也注释说明函数的功能，需要哪些种类的参数，参数可能值的含义以及用途。最后，对于变量的声明，使用下画线以分隔单词，坚持使用小写，并把大写字母留给宏和枚举常量。

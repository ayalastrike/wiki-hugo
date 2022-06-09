---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

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

# 代码风格

InnoDB的代码缩进风格更接近于K&R风格：所有的非函数语句块（if、switch、for、while、do），起始大括号放在行尾，而把结束大括号放在行首。函数开头的左花括号放到最左边。

此外，每个文件都包含一段简短说明其功能的注释开头，同时每个函数也注释说明函数的功能，需要哪些种类的参数，参数可能值的含义以及用途。最后，对于变量的声明，使用下画线以分隔单词，坚持使用小写，并把大写字母留给宏和枚举常量。

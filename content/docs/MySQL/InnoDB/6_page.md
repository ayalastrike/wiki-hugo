
---
---





# 索引页

InnoDB存储引擎是索引组织表，因此聚簇索引页的叶子节点中存放完整的数据记录，辅助索引页的叶子节点中存放指向聚簇索引页叶子节点的书签（bookmark），也可以称为路标。

主要是两部分：

- page layout
- scan rec with cursor, and then insert, update, delete

## 1. 页

页是InnoDB存储引擎的最小存储单位。页的大小可以设置为4K、8K、16K、32K、64K，默认为16K，页的大小设置要考虑IO性能，也会影响到区的分配大小和重做日志缓冲的大小，详见[innodb_page_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_page_size)。




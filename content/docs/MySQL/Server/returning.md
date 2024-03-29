（转载自阿里内核月报）

# 背景

MySQL对于statement执行结果报文通常分为两类：Resultset和OK/ERR，针对 DML语句则返回OK/ERR报文，其中包括几个影响记录，扫描记录等属性。但在很多业务场景下，通常 INSERT/UPDATE/DELETE 这样的DML语句后，都会跟随SELECT查询当前记录内容，以进行接下来的业务处理， 为了减少一次 Client <-> DB Server 交互，类似 PostgreSQL / Oracle 都提供了 returning clause 支持 DML 返回 Resultset。

AliSQL 为了减少对 MySQL 语法兼容性的侵入，并支持 returning 功能， 采用了 native procedure 的方式，使用DBMS_TRANS package，统一使用 returning procedure 来支持 DML 语句返回 Resultset。

## 语法

````
DBMS_TRANS.returning(Field_list=>, Statement=>);
````

其中: 

- Field list : 代表期望的返回字段，以 “,” 进行分割，支持 * 号表达；
- Statement ：表示要执行的DML 语句， 支持 INSERT / UPDATE / DELETE；

# INSERT Returning

针对 insert 语句， returning proc 返回插入到表中的记录内容；

````
mysql> CREATE TABLE `t` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `col1` int(11) NOT NULL DEFAULT '1',
    `col2` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
    ) ENGINE=InnoDB;
 
mysql> call dbms_trans.returning("*", "insert into t(id) values(NULL),(NULL)");
+----+------+---------------------+
| id | col1 | col2                |
+----+------+---------------------+
|  1 |    1 | 2019-09-03 10:39:05 |
|  2 |    1 | 2019-09-03 10:39:05 |
+----+------+---------------------+
2 rows in set (0.01 sec)
````

如果没有填入任何 Fields, returning 将退化成 OK/ERR 报文： 

````
mysql> call dbms_trans.returning("", "insert into t(id) values(NULL),(NULL)");
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0
 
mysql> select * from t;
+----+------+---------------------+
| id | col1 | col2                |
+----+------+---------------------+
|  1 |    1 | 2019-09-03 10:40:55 |
|  2 |    1 | 2019-09-03 10:40:55 |
|  3 |    1 | 2019-09-03 10:41:06 |
|  4 |    1 | 2019-09-03 10:41:06 |
+----+------+---------------------+
4 rows in set (0.00 sec)
````

注意：INSERT returning 只支持 insert values 形式的语法，类似create as， insert select 不支持：

````
mysql> call dbms_trans.returning("", "insert into t select * from t");
ERROR 7527 (HY000): Statement didn't support RETURNING clause
````

# UPDATE Returning

针对 update 语句， returning 返回更新后的记录：

````
mysql> call dbms_trans.returning("id, col1, col2", "update t set col1 = 2 where id >2");
+----+------+---------------------+
| id | col1 | col2                |
+----+------+---------------------+
|  3 |    2 | 2019-09-03 10:41:06 |
|  4 |    2 | 2019-09-03 10:41:06 |
+----+------+---------------------+
2 rows in set (0.01 sec)
````

**注意:** UPDATE returning 不支持多表 update 语句。

# DELETE Returning

针对 delete 语句， returning 返回删除的记录前映像：

````
mysql> call dbms_trans.returning("id, col1, col2", "delete from t where id < 3");
+----+------+---------------------+
| id | col1 | col2                |
+----+------+---------------------+
|  1 |    1 | 2019-09-03 10:40:55 |
|  2 |    1 | 2019-09-03 10:40:55 |
+----+------+---------------------+
2 rows in set (0.00 sec)
````

### 注意

**1. 事务上下文** DBMS_TRANS.returning() 不是事务性语句，根据 DML statement 来继承 事务上下文，
结束事务需要显式的 COMMIT 或者 ROLLBACK。

**2. 字段不支持计算**   Field list 中，只支持表中原生的字段，或者 * 号， 不支持进行计算或者聚合等操作。
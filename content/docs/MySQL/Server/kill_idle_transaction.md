# 1. 背景

我们在线上遇到[5.7.26的锁问题]({{< ref  "../InnoDB/10.1_mysql5.7.26_lock_issue.md" >}})，需要解决idle事务长时间挂起的问题。同时也调研了现有的[mysql timeout机制]({{< ref  "timeout" >}})，以确保其和现有的timeout机制可以吻合。

Percona从5.1.59-13.0引入了innodb_kill_idle_transaction，用于解决长事务场景，即对idle事务设定一个超时时间，对超过该时间的事务所在的用户连接进行断开。

引入该参数也可以防止purge线程的长时间阻塞（长事务会一直保持在活跃状态，则会导致purge长时间的等待，从而导致undo无法清理从而造成磁盘空间的不断增加）。

在实现上，开始是通过扫描InnoDB事务列表来进行判断的，在Percona Server 5.6.35-80.0则改为判断connection socket read timeout。这样优化的好处是，巡检可能会造成CPU空跑，而基于socket select超时则发生超时才会触发，使代码的运行更有效率。

另外，percona现在提供了两个参数：innodb_kill_idle_transaction（后者的alias，5.7中已标记为deprecated）和kill_idle_transaction。我们在port时只保留kill_idle_transaction。

# 2. 设计

复用net_wait_timeout：

- 处于事务中（SERVER_STATUS_IN_TRANS）& min(kill_idle_transaction, net_wait_timeout)
- 其他场景还是返回net_wait_timeout

kill_idle_transaction配置项

| 配置项                | 默认值      | 值范围     |
| :-------------------- | :---------- | :--------- |
| kill_idle_transaction | 0（不开启） | 0 ~ 1 year |

# 3. 示例

~~~~
1. 验证idle事务的连接会自动kill掉
mysql> show global variables like '%idle%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| kill_idle_transaction | 10    |
+-----------------------+-------+
  
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
  
mysql> select * from t1;
+------+
| id   |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.01 sec)
  
// 等待15s
  
mysql> select * from t1;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    8
Current database: tjw
  
+------+
| id   |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.01 sec)
  
mysql> show processlist;
+----+------+-----------+------+---------+------+----------+------------------+
| Id | User | Host      | db   | Command | Time | State    | Info             |
+----+------+-----------+------+---------+------+----------+------------------+
|  5 | root | localhost | tjw  | Sleep   |   10 |          | NULL             |
|  6 | root | localhost | NULL | Query   |    0 | starting | show processlist |
|  7 | root | localhost | NULL | Sleep   |   40 |          | NULL             |
+----+------+-----------+------+---------+------+----------+------------------+
3 rows in set (0.00 sec)
  
15s后连接断开
  
mysql> show processlist;
+----+------+-----------+------+---------+------+----------+------------------+
| Id | User | Host      | db   | Command | Time | State    | Info             |
+----+------+-----------+------+---------+------+----------+------------------+
|  6 | root | localhost | NULL | Query   |    0 | starting | show processlist |
|  7 | root | localhost | NULL | Sleep   |   41 |          | NULL             |
+----+------+-----------+------+---------+------+----------+------------------+
2 rows in set (0.00 sec)
  
2. 变量是global级别，不支持设置session级别
mysql> set session kill_idle_transaction = 20;
ERROR 1229 (HY000): Variable 'kill_idle_transaction' is a GLOBAL variable and should be set with SET GLOBAL
  
3. auto_commit=1的事务不受影响，不会断开
mysql> show global variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
  
mysql> select * from t1;
+------+
| id   |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.00 sec)
  
// 等待30s
  
mysql> select * from t1;
+------+
| id   |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.00 sec)
  
mysql> show processlist;
+----+------+-----------+------+---------+------+----------+------------------+
| Id | User | Host      | db   | Command | Time | State    | Info             |
+----+------+-----------+------+---------+------+----------+------------------+
|  6 | root | localhost | NULL | Query   |    0 | starting | show processlist |
|  7 | root | localhost | NULL | Sleep   | 1905 |          | NULL             |
| 10 | root | localhost | tjw  | Sleep   |   29 |          | NULL             |
+----+------+-----------+------+---------+------+----------+------------------+
~~~~

5.7.26 锁问题

|                    kill_idle_transaction                     |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ |
|                          session A                           | session B                                                    |
| begin;insert into t values (3, 'A');ERROR 1062 (23000): Duplicate entry 'A' for key 'name' |                                                              |
|                                                              | insert into t values (3, 'Z');waiting...                     |
| util kill_idle_transactiontransaction is killed by MySQL事务自动中止ERROR 2006 (HY000): MySQL server has gone away No connection. Trying to reconnect... Connection id: 16 Current database: tjwERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY' |                                                              |
|                                                              | Query OK, 1 row affected (5.44 sec) mysql> select * from t; +----+------+ \| id \| name \| +----+------+ \| 1 \| A \| \| 2 \| B \| \| 3 \| Z \| +----+------+ 3 rows in set (0.00 sec) |

~~~~
mysql> show processlist;
+----+------+-----------+------+---------+------+----------+-------------------------------+
| Id | User | Host      | db   | Command | Time | State    | Info                          |
+----+------+-----------+------+---------+------+----------+-------------------------------+
| 11 | root | localhost | NULL | Query   |    0 | starting | show processlist              |
| 14 | root | localhost | tjw  | Query   |   14 | update   | insert into t values (3, 'Z') |
| 15 | root | localhost | tjw  | Sleep   |   20 |          | NULL                          |
+----+------+-----------+------+---------+------+----------+-------------------------------+
3 rows in set (0.00 sec)
 
mysql> show processlist;
+----+------+-----------+------+---------+------+----------+------------------+
| Id | User | Host      | db   | Command | Time | State    | Info             |
+----+------+-----------+------+---------+------+----------+------------------+
| 11 | root | localhost | NULL | Query   |    0 | starting | show processlist |
| 14 | root | localhost | tjw  | Sleep   |   14 |          | NULL             |
+----+------+-----------+------+---------+------+----------+------------------+
2 rows in set (0.00 sec)
~~~~


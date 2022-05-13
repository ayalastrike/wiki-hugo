MySQL client-server protocol：https://dev.mysql.com/doc/internals/en/client-server-protocol.html

MySQL 5.7重构了Protocol模块

我们在这里主要聚焦在command phase，即MySQL server和client如何处理query的交互。

我们从MySQL协议可以知道，在query的交互上，一般采用ping-pong模型，即：

- client->server：发query
- server->client：回包

所以我们在下面详细拆解这两个阶段。

# 读取query

当client发送一条query后，server对query进行以下处理：

- 读包
- 解析包体
- 根据命令指派执行

数据报文包括包头+包体，包头已在读包内部验证后丢弃（read_packet），然后包体返回给Protocol_classic::get_command封装为raw_packet。从raw_packet[0]中判断query的命令号，进行报文解析（parse_packet），拆解为COM_DATA（根据命令号封装了不同的struct）。

````
do_command                              // conn_handler
    Protocol_classic::get_command       // 读包
        Protocol_classic::read_packet
            my_net_read                 // 处理多包、压缩包
                net_read_packet         // 将读到的数据填充到NET中
        Protocol_classic::parse_packet  // 解析包体
    dispatch_command                    // 根据命令指派执行
````

# 回包

返回的报文类型有：OK Packet，Error Packet和结果集包（Data Packet，EOF Packet）。

## OK Packet

````
do_command
    dispatch_command
        THD::send_statement_status          // 根据这条statement执行的情况确定回包类型
            Protocol_classic::send_ok       // 回OK包
````

## Error Packet

````
do_command
    dispatch_command
        THD::send_statement_status          // 根据这条statement执行的情况确定回包类型
            Protocol_classic::send_error    // 回错误包
````

## 结果集包

结果集包的结构如下:

| 数据             | 说明       |
| :--------------- | :--------- |
| ResultSet Header | 列数量     |
| Field            | 列（多个） |
| EOF              |            |
| Row              | 行（多个） |
| EOF              |            |

Server层的SQL执行器拼装结果集：

````c++
// sql_executor
Query_result_send::send_result_set_metadata
    thd->send_result_metadata
        Protocol_classic::start_result_metadata()   // 列数量
        Protocol_classic::send_field_metadata()     // 列（多个）
        Protocol_classic::end_result_metadata()     // EOF
Query_result_send::send_data(List<Item> &items)     // 行（多个）
    protocol->start_row();
    thd->send_result_set_row(&items)
    for loop : items
        protocl->store()
    thd->inc_sent_row_count(1);
    protocol->end_row()
Query_result_send::send_eof                         // EOF
    net_send_ok
````

注意最后两行：send_eof调用了send_ok？？？

````c++

bool Protocol_classic::send_eof(uint server_status, uint statement_warn_count)
{
  DBUG_ENTER("Protocol_classic::send_eof");
  bool retval;
  /*
    Normally end of statement reply is signaled by OK packet, but in case
    of binlog dump request an EOF packet is sent instead. Also, old clients
    expect EOF packet instead of OK
  */
#ifndef EMBEDDED_LIBRARY
  if (has_client_capability(CLIENT_DEPRECATE_EOF) &&
      (m_thd->get_command() != COM_BINLOG_DUMP &&
       m_thd->get_command() != COM_BINLOG_DUMP_GTID))
    retval= net_send_ok(m_thd, server_status, statement_warn_count, 0, 0, NULL,
                        true);
  else
#endif
    retval= net_send_eof(m_thd, server_status, statement_warn_count);
  DBUG_RETURN(retval);
}
````

这是因为在MySQL 5.7.5中，有一个worklog：[WL#7766: Deprecate the EOF packet](https://dev.mysql.com/worklog/task/?id=7766)，认为EOF and OK packets serve the same purpose (to mark the end of a query execution result. 

将ok和eof报文的发送放在了同一块逻辑中，client/server支持一个flag：[CLIENT_DEPRECATE_EOF](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_basic_ok_packet.html)，[EOF包说明](https://dev.mysql.com/doc/internals/en/packet-EOF_Packet.html)里也有提到。

如果我们要自己组装结果集包，则按照如下API组装即可：

````
thd->send_result_metadata
protocol->start_row();
protocol->store();
protocol->end_row();
my_eof(thd);
````

下面是一个实际的组装示例：

````c++
mysql> show sql_filters;
+--------+---------+----------+----------+---------+-------------+
| type   | item_id | cur_conc | max_conc | key_num | key_str     |
+--------+---------+----------+----------+---------+-------------+
| SELECT |       4 |        0 |        1 |       2 | +,1,a=1~a=2 |
+--------+---------+----------+----------+---------+-------------+
 
void mysqld_list_sql_filters(THD *thd)
{
  List<Item> field_list;
 
  field_list.push_back(new Item_empty_string("type", 21));
  field_list.push_back(new Item_return_int("item_id", 21, MYSQL_TYPE_LONGLONG));
  field_list.push_back(new Item_return_int("cur_conc", 21, MYSQL_TYPE_LONGLONG));
  field_list.push_back(new Item_return_int("max_conc", 21, MYSQL_TYPE_LONGLONG));
  field_list.push_back(new Item_return_int("key_num", 21, MYSQL_TYPE_LONGLONG));
  field_list.push_back(new Item_empty_string("key_str", SQL_FILTER_STR_LEN));
 
  if (thd->send_result_metadata(&field_list, Protocol::SEND_NUM_ROWS | Protocol::SEND_EOF))
        return;
 
  mysql_rwlock_rdlock(&LOCK_filter_list);
  if (list_one_sql_filter(thd, select_filter_list, "SELECT")
          || list_one_sql_filter(thd, update_filter_list, "UPDATE")
          || list_one_sql_filter(thd, delete_filter_list, "DELETE"))
  {
        ;
  }
 
  mysql_rwlock_unlock(&LOCK_filter_list);
  my_eof(thd);
}
 
  int list_one_sql_filter(THD *thd, LIST *filter_list, const char *type)
{
  CHARSET_INFO *cs= system_charset_info;
  filter_item *item= NULL;
  Protocol *protocol= thd->get_protocol();
 
  while (filter_list)
  {
        protocol->start_row();
        item= (filter_item*)filter_list->data;
        protocol->store(type, cs);
        protocol->store((longlong)item->id);
        protocol->store(__sync_fetch_and_add(&(item->cur_conc), 0));
        protocol->store(item->max_conc);
        protocol->store((longlong)item->key_num);
        protocol->store(item->orig_str, cs);
 
        if (protocol->end_row())
                return 1; //no cover line
 
        filter_list= filter_list->next;
  }
 
  return 0;
}
````

# MySQL 5.7的协议重构

MySQL 5.7大幅重构了Protocol模块代码, 采用了OO的设计方式：[WL#7126: Refactoring of protocol class](https://dev.mysql.com/worklog/task/?id=7126)：

{{< hint info >}}

```
New Protocol ``class` `hierarchy
============================
The ``new` `hierarchy consists of 4 classes:
   Protocol
      |
   Protocol_classic
      |
      |---Protocol_text
      |---Protocol_binary
Protocol is an abstract ``class` `that defines the ``new` `API. Protocol_classic is
ex-Protocol ``class``, implements core of both classic protocols - text and
binary. Protocol_text and Protocol_binary are implementations of appropriate
classic protocols.
```

{{</hint>}}

Protocol作为一个注释丰满且只有纯虚函数的抽象类, 非常容易理顺protocol模块能够提供的API。细节实现主要在Protocol_classic中（所以上文的调用栈可以看到, 实际逻辑是走到Protocol_classic中的）, 而逻辑上还划分出的两个类:

- Protocol_binary是Prepared Statements使用的协议
- Protocol_text[场景](https://dev.mysql.com/doc/internals/en/text-protocol.html)

上面提到了MySQL 5.7.5引入的Deprecate EOF，实际上MySQL 5.7上对OK/EOF报文做了大量修改，使得client可以通过报文拿到更多的会话状态信息。方便中间层会话保持，主要涉及几个worklog：

[WL#4797: Extending protocol’s OK packet](https://dev.mysql.com/worklog/task/?id=4797)

[WL#6885: Flag to indicate session state](https://dev.mysql.com/worklog/task/?id=6885)

[WL#6128: Session Tracker: Add GTIDs context to the OK packet](https://dev.mysql.com/worklog/task/?id=6128)

[WL#6972: Collect GTIDs to include in the protocol’s OK packet](https://dev.mysql.com/worklog/task/?id=6972)

[WL#7766: Deprecate the EOF packet](https://dev.mysql.com/worklog/task/?id=7766)

[WL#6631: Detect transaction boundaries](https://dev.mysql.com/worklog/task/?id=6631)

同时新增变量控制报文行为:

- session_track_schema = [ON | OFF]
  ON时, 如果session中变更了当前database, OK报文中回返回新的database

- session_track_state_change = [ON | OFF]
  ON时, 当发生会话环境改变时, 会给CLIENT返回一个FLAG(1)，会话环境变化包括：

  {{< hint info >}}

  当前database
  
  系统变量
  
  User-defined 变量
  
  临时表的变更
  
  prepare xxx

  {{</hint>}}

  但是只通知变更发生，具体值是多少，还需要配合session_track_schema、session_track_system_variables使用，所以限制还是很多…
  
- session_track_system_variables = [“list of string, seperated bt ‘,’”]
  这个参数用来追踪的变量, 目前只有time_zone, autocommit, character_set_client, character_set_results, character_set_connection可选。当这些变量的值变动时，client可以收到variable_name: new_value的键值对

- session_track_gtids = [OFF | OWN_GTID | ALL_GTIDS]
  OWN_GTID：在会话中产生新GTIDs（当然只读操作不会后推GTID位点）时，以字符串形式返回新增的GTIDs
  ALL_GTIDS：在每个包中返回当前的executed_gtid值，但是这样报文的payload很高，不推荐

- session_track_transaction_info = [ON | OFF] 打开后, 通过标志位表示当前会话状态，有8bit可以表示状态信息（其中使用字符’_‘表示FALSE）：

  - T: 显示开启事务; I: 隐式开启事务（autocommit = 0）

  - r: 有非事务表读

  - R: 有事务表读

  - w: 非事务表写

  - W: 事务表写

  - s: 不安全函数（比如 select uuid()）

  - S: server返回结果集

  - L: 显示锁表(LOCK TABLES) 一个事务内，返回的状态值是累加的

    示例 表t1是InnoDB，表t2是MyISAM

    ````
    START TRANSACTION;               // T_______
    INSERT INTO t1 VALUES (1);       // T___W___
    INSERT INTO t2 VALUES (1);       // T__wW___
    SELECT f1 FROM t1;               // T_RwW_S_
    ...
    COMMIT/ROLLBACK;
    ````
    

OK和EOF报文在MySQL 5.6上是走不同的逻辑构造报文，但实际上都是返回一些执行状态。MySQL 5.7中的Deprecated EOF报文，实际上是复用了OK报文中新增的状态，但是实际上这两个报文还是不同的：

OK Packet：  header = 0 and length of packet > 7

EOF Packet：header = 0xfe and length of packet < 9

只是复用了在`net_send_ok`里的扩充逻辑。

有了以上这些信息，我们可以做很多中间层的开发工作，比如读写分离就用状态追踪对外提供透明的读写分离。

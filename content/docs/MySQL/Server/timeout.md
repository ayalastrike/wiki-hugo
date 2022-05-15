# 背景

在分布式环境下，异步网络是一个挑战，当遇到网络问题时，提供超时机制可以提升系统的可用性。并且，对于事务系统，锁机制也需要超时机制来保证资源可以在有限时间内释放，避免饥饿现象的产生。

为此，MySQL在多种场景下提供了timeout机制。

{{< hint warning>}}

简单一句话，MySQL Protocol ping-pong模型各个节点都有超时检测

{{</hint>}}

# MySQL timeout配置

MySQL内有多种timeout，我们先看一下有多少：

````
mysql> show global variables like '%timeout%';
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| connect_timeout             | 5        |
| net_read_timeout            | 30       |
| net_write_timeout           | 60       |
| wait_timeout                | 28800    |
| interactive_timeout         | 28800    |
| lock_wait_timeout           | 31536000 |
| innodb_lock_wait_timeout    | 3        |
| innodb_rollback_on_timeout  | OFF      |
+-----------------------------+----------+
````

通过阅读官方文档，结合我们在下面对于timeout实现的论证，这里先放上结论：

网络超时

- [connect_timeout](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_connect_timeout)：在connection phase阶段超时
- [net_read_timeout](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_net_read_timeout)：在comamnd phase阶段网络读超时
- [net_write_timeout](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_net_write_timeout)：在comamnd phase阶段网络写超时
- [wait_timeout](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_wait_timeout)、[interactive_timeout](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_interactive_timeout)：在connection phase结束后，在command phase节点，多长时间没有收到命令包。这里的wait_timeout和interactive_timeout的区别只是连接的种类不同，wait_timeout是对noninteractive的连接空闲超时，interactive_timeout是对interactive的连接空闲超时（客户端连接时设置了CLIENT_INTERACTIVE）

总结一下：

- connection phase应用建链期间，没有收到client回复，connect_timeout
- command phase期间没有收到请求时，net_wait_timeout/net_interactive_timeout，当收到请求后，接受一次完整的请求间，如果出现网络读包超时，net_read_timeout，回包时写socket超时，net_write_timeout

锁超时

- [lock_wait_timeout](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_lock_wait_timeout)：获取mdl锁的超时时间
- [innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)：InnoDB中行锁等待的超时时间（对表锁无效）
- [innodb_rollback_on_timeout](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_rollback_on_timeout)：当InnoDB锁等待超时后，OFF只rollback事务中的最后一条语句，ON则rollback整个事务

timeout配置项的值范围

| 配置项                     | 默认值 | 值范围         | 线上默认配置 |
| :------------------------- | :----- | :------------- | :----------- |
| connect_timeout            | 10     | 2 ~ 1 year     | 10           |
| net_read_timeout           | 30     | 1 ~ 1 year     | 30           |
| net_write_timeout          | 60     | 1 ~ 1 year     | 60           |
| wait_timeout               | 8 hour | 1 ~ 1 year     | 28800        |
| interactive_timeout        | 8 hour | 1 ~ 1 year     | 28800        |
| lock_wait_timeout          | 1 year | 1 ~ 1 year     | 31536000     |
| innodb_lock_wait_timeout   | 50     | 1 ~ 1073741824 | 5            |
| innodb_rollback_on_timeout | OFF    | ON/OFF         | OFF          |

# 实现细节

下面结合代码看一下这些timeout的具体含义。

## connect_timeout net_read_timeout net_write_timeout

````
两处设置网络超时
1. 网络初始化
Protocol_classic::init_net
mysql client
    my_net_init
        my_net_local_init
            my_net_set_read_timeout
            my_net_set_write_timeout
                net->read_timeout = ...
                net->write_timeout = ...
2. 网络读写
mysql client
    vio_socket_connect  // 根据等待的连接事件（VIO_IO_EVENT_CONNECT）
vio_read
vio_write
    vio_socket_io_wait  // 根据等待的读写事件（VIO_IO_EVENT_READ/VIO_IO_EVENT_WRITE）将select超时设置为vio->read_timeout或vio->write_timeout
        vio_io_wait     // timeout:-1
            select
````

即

- 在connection phase，将connect_timeout设置为net->vio->read_timeout/write_timeout
- 在command phase，将net_read_timeout设置为net->vio->read_timeout，将net_write_timeoutnet->vio->write_timeout

其实底层都是搞的select超时，然后通过以下调用链返回

````
vio->read
    vio_read_buff
    vio_read
        mysql_socket_recv
        vio_socket_io_wait
            vio_io_wait
                select
vio->write
    vio_write
        mysql_socket_send
        vio_socket_io_wait
            vio_io_wait
                select
 
Protocol_classic::write
    my_net_write
    net_write_command
    net_flush
    net_write_buff
        net_write_packet
            net_write_raw_loop
                vio->write
````

当select超时时，返回-1，然后设置net->error = 2

````c++
  /* On failure, propagate the error code. */
  if (count)
  {
    /* Socket should be closed. */
    net->error= 2;
 
    /* Interrupted by a timeout? */
    if (vio_was_timeout(net->vio))
      net->last_errno= ER_NET_WRITE_INTERRUPTED;
    else
      net->last_errno= ER_NET_ERROR_ON_WRITE;
 
#ifdef MYSQL_SERVER
    my_error(net->last_errno, MYF(0));
#endif
  }
````

判断net->error是否为非零，非零意味着有错误，然后关闭连接。

- connection phase：close_connection
- command phase ：end_connection

## net_wait_timeout net_interactive_timeout

在从网络读命令包之前设置net_wait_timeout，然后在读完之后设置为net_read_timeout

````c++
bool do_command(THD *thd) {
  ...
  if (classic)
  {
    /*
      This thread will do a blocking read from the client which
      will be interrupted when the next command is received from
      the client, the connection is closed or "net_wait_timeout"
      number of seconds has passed.
    */
    net= thd->get_protocol_classic()->get_net();
    my_net_set_read_timeout(net, thd->variables.net_wait_timeout);
    net_new_transaction(net);
  }
 
  rc= thd->get_protocol()->get_command(&com_data, &command);
  thd->m_server_idle= false;
  if (classic)
    my_net_set_read_timeout(net, thd->variables.net_read_timeout);
 
  return_value= dispatch_command(thd, &com_data, command);
  ...
}
````

## lock_wait_timeout

在获取mdl锁时设置等待时长

````
thd->mdl_context.acquire_locks(&mdl_requests, thd->variables.lock_wait_timeout)
````

# 实践

需要调整wait_timeout/interactive_timeout：

````
[Warning] Aborted connection 6 to db: 'unconnected' user: 'root' host: 'localhost' (Got timeout reading communication packets)
````

需要调整net_write_timeout：

````
[Warning] Aborted connection 12 to db: 'test' user: 'root' host: 'localhost' (Got timeout writing communication packets)
````

需要注意的是，MySQL的关于网络的错误，除了超时以外都认为是error，没有做进一步的细分，比如可能会看到下面这种日志，有可能是客户端异常退出了，也有可能是网络链路异常。

````
[Warning] Aborted connection 8 to db: 'unconnected' user: 'root' host: 'localhost' (Got an error reading communication packets)
[Warning] Aborted connection 13 to db: 'test' user: 'root' host: 'localhost' (Got an error writing communication packets)
````
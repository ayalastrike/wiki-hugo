（转载自阿里内核月报）

# 建立连接过程

MySQL建立连接过程如下

````c++
// 初始化网络
network_init()
    set_ports(); // 设置port
 
    Mysqld_socket_listener *mysqld_socket_listener=
        new (std::nothrow) Mysqld_socket_listener(bind_addr_str, mysqld_port, back_log, mysqld_port_timeout, unix_sock_name);
 
    Connection_acceptor<Mysqld_socket_listener> *mysqld_socket_acceptor=
        new (std::nothrow) Connection_acceptor<Mysqld_socket_listener>(mysqld_socket_listener);
 
    mysqld_socket_acceptor->init_connection_acceptor();
 
...
 
// 监听socket事件
mysqld_socket_acceptor->connection_event_loop() {
    Connection_handler_manager *mgr= Connection_handler_manager::get_instance();
 
    while (!abort_loop)
    {
      Channel_info *channel_info= m_listener->listen_for_connection_event();
      if (channel_info != NULL)
        mgr->process_new_connection(channel_info);
    }
}
 
Channel_info* Mysqld_socket_listener::listen_for_connection_event() {
    int retval= poll(&m_poll_info.m_fds[0], m_socket_map.size(), -1); // POLL
    for (uint i= 0; i < m_socket_map.size(); ++i) {
        if (m_poll_info.m_fds[i].revents & POLLIN)
        {
          listen_sock= m_poll_info.m_pfs_fds[i];
          is_unix_socket= m_socket_map[listen_sock];
          break;
        }
    }
    MYSQL_SOCKET connect_sock= mysql_socket_accept(key_socket_client_connection, listen_sock, (struct sockaddr *)(&cAddr), &length);
    Channel_info* channel_info= new (std::nothrow) Channel_info_tcpip_socket(connect_sock);
    return channel_info;
}
 
void Connection_handler_manager::process_new_connection(Channel_info* channel_info) {
    check_and_incr_conn_count(); // 检查max_connections
    m_connection_handler->add_connection(channel_info);
}
 
// One_thread_connection_handler 一个线程处理所有连接
// Per_thread_connection_handler 一个线程处理一个连接
bool Per_thread_connection_handler::add_connection(Channel_info* channel_info) {
    // 检查thread cache是否有空闲
    check_idle_thread_and_enqueue_connection(channel_info);
    // 没有空闲，创建用户线程
    mysql_thread_create(key_thread_one_connection, &id, &connection_attrib, handle_connection, (void*) channel_info);
}
 
extern "C" void *handle_connection(void *arg) {
    my_thread_init(); // 线程初始化
    for (;;) {
        THD *thd= init_new_thd(channel_info); // 初始化THD对象
        thd_manager->add_thd(thd);
 
        if (thd_prepare_connection(thd)) { // 请求第一次进入
            lex_start(thd); // 初始化sqlparser
            rc= login_connection(thd);
                check_connection(thd);
                    acl_authenticate(thd, COM_CONNECT); // auth认证
                thd->send_statement_status();
            prepare_new_connection_state(thd); // 准备接受QUERY
        } else {
            while (thd_connection_alive(thd)) // 判活
            {
                if (do_command(thd)) // 处理query sql/sql_parser.c
                  break;
            }
            end_connection(thd);
        }
        close_connection(thd, 0, false, false);
        thd->release_resources();
        // 进入thread cache，等待新连接复用
        channel_info= Per_thread_connection_handler::block_until_new_connection();
    }
    my_thread_end();
    my_thread_exit(0);
}
````

具体SQL处理流程：

````c++
bool do_command(THD *thd) {
    // 新建连接，或者连接没有请求时，会block在这里等待网络读包
    NET *net= thd->get_protocol_classic()->get_net(); 
    my_net_set_read_timeout(net, thd->variables.net_wait_timeout);
    net_new_transaction(net);
 
    rc= thd->get_protocol()->get_command(&com_data, &command);
    dispatch_command(thd, &com_data, command);
}
 
int Protocol_classic::get_command(COM_DATA *com_data, enum_server_command *cmd) {
    read_packet(); // 网络读包
        my_net_read(&m_thd->net);
        raw_packet= m_thd->net.read_pos;
 
    *cmd= (enum enum_server_command) raw_packet[0]; // 获取命令号
    parse_packet(com_data, *cmd);
}
 
bool dispatch_command(THD *thd, const COM_DATA *com_data, enum enum_server_command command) {
    switch (command) {
        case COM_QUERY:
            alloc_query(thd, com_data->com_query.query, com_data->com_query.length); // 从网络读Query并存入thd->query
            mysql_parse(thd, &parser_state); // 解析
    }
}
 
// sql/sql_parse.cc
void mysql_parse(THD *thd, Parser_state *parser_state) {
    mysql_reset_thd_for_next_command(thd);
 
    lex_start(thd);
    parse_sql(thd, parser_state, NULL); // 解析SQL语句
 
    mysql_execute_command(thd, true); // 执行SQL语句
        LEX  *const lex= thd->lex;
        TABLE_LIST *all_tables= lex->query_tables;
         
        // 隐式提交 sql/transaction.cc
        trans_commit_implicit(thd);
        switch (lex->sql_command) {
            case SQLCOM_INSERT:
            {
                res= lex->m_sql_cmd->execute(thd);
                break;
            }
            case SQLCOM_DELETE:
            {
                res= lex->m_sql_cmd->execute(thd);
                break;
            }
            case SQLCOM_UPDATE:
            {
                res= lex->m_sql_cmd->execute(thd);
                break;
            }
            case SQLCOM_SELECT:
            {
                res= select_precheck(thd, lex, all_tables, first_table); // 检查privileges
                res= execute_sqlcom_select(thd, all_tables);
            }
            // 显式提交
            case SQLCOM_COMMIT:
            {
                trans_commit(thd);
                    ha_commit_trans(thd, TRUE);
                        Transaction_ctx *trn_ctx= thd->get_transaction();
                        tc_log->commit(thd, all)); // MYSQL_BIN_LOG::commit sql/binlog.cc
                            ordered_commit(thd, all, skip_commit);
            }
            ...
        }
}
````

# thread cache

连接复用受到[thread_cache_size](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_thread_cache_size)连接池配置大小的影响，0为关闭连接池；默认值为

````
8 + (max_connections / 100)
````

运行时查看连接情况：

- Threads_cached：缓存的 thread数量，新连接建立时，优先使用cache中的thread
- Threads_connected：已连接的thread数量
- Threads_created：建立的thread数量
- Threads_running：running状态的 thread 数量

Threads_created = Threads_cached + Threads_connected

Threads_running <= Threads_connected

MySQL 建立新连接非常消耗资源，频繁使用短连接，又没有其他组件实现连接池时，可以适当提高 thread_cache_size，降低新建连接的开销

````
mysql> show status like 'Thread%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 1     |
| Threads_connected | 1     |
| Threads_created   | 2     |
| Threads_running   | 1     |
+-------------------+-------+
4 rows in set (0.00 sec)
````

## 源码分析

| 变量                      | 类型                     | 说明                                                         |
| :------------------------ | :----------------------- | :----------------------------------------------------------- |
| channel_info              | class                    | 连接信息                                                     |
| waiting_channel_info_list | std::list<Channel_info*> | 空闲连接链表                                                 |
| wake_pthread              | uint                     | 空闲连接链表的长度                                           |
| LOCK_thread_cache         | mysql_mutex_t            | 连接池锁                                                     |
| COND_thread_cache         | mysql_cond_t             | cond_wait 释放连接 block_until_new_connection()signal 新连接建立 check_idle_thread_and_enqueue_connection()broadcast 杀掉thread cache中的连接 kill_blocked_pthreads() |
| COND_flush_thread_cache   | mysql_cond_t             | cond_wait 杀掉thread cache中的连接 kill_blocked_pthreads()signal 释放连接 block_until_new_connection() |
| blocked_pthread_count     | ulong                    | 被block的线程数                                              |
| slow_launch_threads       | ulong                    | 连接建立慢的线程数（> slow_launch_time）                     |
| max_blocked_pthreads      | ulong                    | 被block的最大线程数，也就是thread_cache_size                 |

### block_until_new_connection

handle_connection 线程结束之前，会执行 block_until_new_connection，尝试进入thread cache等待其他连接复用

如果 blocked_pthread_count < max_blocked_pthreads，blocked_pthread_count++，然后等待被 COND_thread_cache 唤醒，唤醒之后 blocked_pthread_count– , 返回 waiting_channel_info_list 中的一个 channel_info ，进行 handle_connections 的下一个循环

### check_idle_thread_and_enqueue_connection

检查是否 blocked_pthread_count > wake_pthread （有足够的block状态线程用来唤醒） 如有 插入 channel_info 进入 waiting_channel_info_list，并发出 COND_thread_cache 信号量

# auth连接限制

除了参数 max_user_connections 限制每个用户的最大连接数，还可以对每个用户制定更细致的限制。以下四个限制保存在mysql.user表中：

- MAX_QUERIES_PER_HOUR 每小时最大请求数（语句数量）
- MAX_UPDATES_PER_HOUR 每小时最大更新数（更新语句的数量）
- MAX_CONNECTIONS_PER_HOUR 每小时最大连接数
- MAX_USER_CONNECTIONS 这个用户的最大连接数

````
GRANT
    priv_type [(column_list)]
      [, priv_type [(column_list)]] ...
    ON [object_type] priv_level
    TO user [auth_option] [, user [auth_option]] ...
    [REQUIRE {NONE | tls_option [[AND] tls_option] ...}]
    [WITH {GRANT OPTION | resource_option} ...]
 
resource_option: {
  | MAX_QUERIES_PER_HOUR count
  | MAX_UPDATES_PER_HOUR count
  | MAX_CONNECTIONS_PER_HOUR count
  | MAX_USER_CONNECTIONS count
}
 
ALTER USER 'jeffrey'@'localhost' WITH MAX_QUERIES_PER_HOUR 90;
````

## 源码分析

````
typedef struct user_resources {
 
  uint questions;       /* MAX_QUERIES_PER_HOUR */
  uint updates;         /* MAX_UPDATES_PER_HOUR */
  uint conn_per_hour;   /* MAX_CONNECTIONS_PER_HOUR */
  uint user_conn;       /* MAX_USER_CONNECTIONS */
 
  /*
     Values of this enum and specified_limits member are used by the
     parser to store which user limits were specified in GRANT statement.
  */
  enum {QUERIES_PER_HOUR= 1, UPDATES_PER_HOUR= 2, CONNECTIONS_PER_HOUR= 4,
        USER_CONNECTIONS= 8};
  uint specified_limits;
} USER_RESOURCES;
````

## ACL_USER

ACL_USER 是保存用户认证相关信息的类 USER_RESOURCES 是它的成员变量

````
class ACL_USER :public ACL_ACCESS
{
public:
  USER_RESOURCES user_resource;
...
}
````

ACl_USER 对象保存在数组 acl_users 中，每次mysqld启动时，从mysql.user表中读取数据，初始化 acl_users，初始化过程在函数 acl_load 中

调用栈如下：

````
main()
	mysqld_main()
		acl_init(opt_noacl);
    		acl_reload(thd);
        		acl_load(thd, tables);
````

## USER_CONN

保存用户资源使用的结构体，建立连接时，调用 get_or_create_user_conn 为 THD 绑定 USER_CONN 对象：

````c++
// 请求第一次处理时
acl_authenticate()
    if ((acl_user->user_resource.questions || acl_user->user_resource.updates ||
         acl_user->user_resource.conn_per_hour ||
         acl_user->user_resource.user_conn ||
         global_system_variables.max_user_connections) &&
        get_or_create_user_conn(thd,
          (opt_old_style_user_limits ? sctx->user().str :
                                       sctx->priv_user().str),
          (opt_old_style_user_limits ? sctx->host_or_ip().str :
                                       sctx->priv_host().str),
          &acl_user->user_resource))
    ------->
			thd->set_user_connect(uc);
````

每个用户第一个连接创建时，建立一个新对象，存入 hash_user_connections。

第二个连接开始，从 hash_user_connections 取出 USER_CONN 对象和 THD 绑定。

同一个用户的连接，THD 都和同一个 USER_CONN 对象绑定。

````
typedef struct user_conn {
  /* hash_user_connections hash key: user+host key */
  char *user;
  char *host;
 
  /* Total length of the key. */
  size_t len;

  ulonglong reset_utime;

  uint connections;
  uint conn_per_hour, updates, questions;

  USER_RESOURCES user_resources;
} USER_CONN;
````

资源限制在源码中的位置

| 资源名称                 | 函数                             |
| :----------------------- | :------------------------------- |
| MAX_USER_CONNECTIONS     | check_for_max_user_connections() |
| MAX_CONNECTIONS_PER_HOUR | check_for_max_user_connections() |
| MAX_QUERIES_PER_HOUR     | check_mqh()                      |
| MAX_UPDATES_PER_HOUR     | check_mqh()                      |

调用链

````
handle_connection
	thd_prepare_connection(thd)
		login_connection
			check_connection
				acl_authenticate
					check_for_max_user_connections
	do_command
		dispatch_command
			mysql_parse
				check_mqh
````
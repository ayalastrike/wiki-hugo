---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

# 概述

在MySQL中，对于client发来的请求，其处理流程分为建链和请求处理两部分，这两个阶段分别称为connection phase和command phase。

MySQL的server-client protocol交互如下：

![MySQL_conn_client-server_protocol](/MySQL_conn_client-server_protocol.png)

从上图中可以看出，connection phase负责连接的建立，而日常的query处理，则称为command phase，command phase的结束，以COM_QUIT query的到来作为标志。

一般典型的交互过程是connect，query，query，query... quit，其中query可以是dml、ddl、multi-statement或是prepared statement。

下面我们先看一下connection phase。

# 建链

connection phase用于在client-server间建立连接，而建链分为TCP建链和应用建链。

TCP建链是指TCP socket的listen、accept。

应用建链是在TCP建链的基础上，通过应用层协议进行认证：server发送handshake（initial handshake）、客户端回username/pwd（handshake response），server回应是否通过认证（OK/Error）。

我们接下来首先看一下connection phase，即在MySQL中如何处理TCP建链和应用建链的。

## TCP建链

TCP连接处理分为两步：

1. 初始化，创建conn_mgr和conn_handler，acceptor和listener
2. 监听建链，由acceptor+listener负责
3. 对已建链连接进行线程分发处理，由conn_mgr+conn_handler负责

整体流程如下图所示：

![connection_overview](/MySQL_connection_overview.png)

代码实现

```c++
mysqld_main
	init_common_variables
	Connection_handler_manager::init() // 初始化conn_mgr和conn_handler

	network_init()		// 初始化网络
		set_ports();	// 设置port
                        // 初始化acceptor、listener
	    Mysqld_socket_listener *mysqld_socket_listener= new (std::nothrow) Mysqld_socket_listener(bind_addr_str, mysqld_port, back_log, mysqld_port_timeout, unix_sock_name);
		Connection_acceptor<Mysqld_socket_listener> *mysqld_socket_acceptor= new (std::nothrow) Connection_acceptor<Mysqld_socket_listener>(mysqld_socket_listener);
		mysqld_socket_acceptor->init_connection_acceptor();
		...

	mysqld_socket_acceptor->connection_event_loop();	// 监听、接受、处理连接
```

### 监听建链

MySQL的连接方式支持多种方式，常见的有：socket、TCP/IP、named_pipe和shared_memory。因为我们一般都在Unix-like系统上编程，所以这里只展开讨论socket，其余连接方式的处理类似。

连接处理分为分为监听（listen）和接受（accept）两部分：

1. listener：Mysqld_socket_listener
2. acceptor：Connection_acceptor

我们先看一下listener

#### listener

Mysqld_socket_listener用于处理监听（listen）和建链（accept），包括：

- 监听信息 ：ip、port、backlog、socket、socket_map
- 处理 ：POLL（tcp socket、unix sock file）
- 状态信息 ：错误（select、accept、tcpwrap产生的状态）

Mysqld_socket_listener

| 方法                        | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| ctor/dtor                   | /                                                            |
| setup_listener              | 建链准备（初始化listen socket，POLL、socket_map）            |
| listen_for_connection_event | 建链处理（处理poll/select，accept，创建channel_info）        |
| close_listener              | 关闭连接（socket_shutdown、socket_close、unlink_socket_file） |

#### acceptor

Connection_acceptor是一个模板类，根据type展开不同的listener（Mysqld_socket_listener、Named_pipe_listener、Shared_mem_listener），负责将listener TCP监听、建链的连接（channel_info）交给conn_mgr处理。

核心函数如下：

`````c++
  /**
    Connection acceptor loop to accept connections from clients.
  */
  void connection_event_loop()
  {
    Connection_handler_manager *mgr= Connection_handler_manager::get_instance();
    while (!abort_loop)
    {
      Channel_info *channel_info= m_listener->listen_for_connection_event();
      if (channel_info != NULL)
        mgr->process_new_connection(channel_info);
    }
  }
`````

Connection_acceptor

| 方法                     | 说明                                           |
| ------------------------ | ---------------------------------------------- |
| ctor/dtor                | 传入listener                                   |
| init_connection_acceptor | 调用listener->setup_listener()                 |
| connection_event_loop    | 调用listener监听建链，然后交给conn_mgr处理连接 |
| close_listener           | 调用listener->close_listener                   |

![connection_acceptor_listener](/MySQL_connection_acceptor_listener.png)

### LibWrap

TCP Wrappers作为服务程序安全增强工具，提供 IP 层存取过滤控制，扩展了 inetd (xinetd ) 对服务程序的控制能力，其作用相当于给 xinetd 增加了一道防火墙。最常用的场景如下：通过配置/etc/hosts.allow和/etc/hosts.deny ，以允许或阻止指定客户端对指定服务的访问。

## 处理连接

经过上面的处理后，用户的建链请求已经经过listen+accept，下面交给conn_mgr+conn_handler。

在MySQL中，为了支持多种的连接处理方式（单线程only-once、多线程1:1、线程池m:n），通过Connection_handler基类来定义连接处理所需要的函数，具体的处理方式则由子类实现。

conn_mgr和conn_handler的关系：

![MySQL_connection_conn_mgr-conn_handler](/MySQL_connection_conn_mgr-conn_handler.png)

### connection manager

conn_mgr采用单例模式，进行全局的连接资源管理，这些资源包括：

- conn_handler的创建
- 将lisenter创建的连接（channel_info）转交给conn_handler
- 连接相关计数：当前、历史、中止、错误
- 提供callback接口用于在相关连接线程等待时进行回调

Connection_handler_manager

| 方法                                             | 说明                                                         |
| ------------------------------------------------ | ------------------------------------------------------------ |
| ctor/dtor                                        | 由init调用                                                   |
| init                                             | 根据thread_handler的配置，实例化conn_handler,调用conn_mgr ctor实例化conn_mgr，注册callbacks |
| destroy_instance                                 | 销毁conn_handler和conn_mgr                                   |
| get_instance                                     | 返回conn_mgr instance                                        |
| process_new_connection                           | 移交连接                                                     |
| wait_till_no_connection                          | 关闭MySQL时等待连接清零                                      |
| load_connection_handlerunload_connection_handler | 为企业版线程池（plugin）准备的钩子                           |

这里的核心函数为process_new_connection。需要注意一点：channel_info连接信息的所有权转移，由listener转移给conn_handler。

````c++
void
Connection_handler_manager::process_new_connection(Channel_info* channel_info)
{
  // 连接控制
  if (abort_loop || !check_and_incr_conn_count())
  {
    channel_info->send_error_and_close_channel(ER_CON_COUNT_ERROR, 0, true);
    delete channel_info;
    return;
  }

  // 转交连接
  if (m_connection_handler->add_connection(channel_info))
  {
    inc_aborted_connects();
    delete channel_info;
  }
}
````

conn_mgr的生命周期：初始化和销毁的时机分别为MySQL启动和停止，代码如下：

````c++
// 初始化
mysqld_main
init_embedded_erver
	init_common_variables
		get_options
			Connection_handler_manager::init()
// 停止
lib_sql.cc
	end_embedded_server
	unireg_clear
		cleanup
			Connection_handler_manager::destroy_instance();
mysqld.cc
	mysqld.cc
		mysqld_main
		unireg_abort
			clean_up
				Connection_handler_manager::destroy_instance();
````

### connection handler

Connection_handler

| 方法            | 说明                                 |
| --------------- | ------------------------------------ |
| ctor/dtor       | /                                    |
| add_connection  | 处理连接                             |
| get_max_threads | 获取conn_handler可以创建的最大线程数 |

从上面的conn_handler类图可以看到，Connection_handler一共有三个子类，这里主要看Per_thread_connection_handler。

Per_thread_connection_handler的功能是新起一个线程（handle_connection）1:1处理连接，即进行应用协议的处理。

# 请求处理

## 请求分发

MySQL请求处理的详细过程如下：

````c++
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
            int retval= select((int) m_select_info.m_max_used_connection, &m_select_info.m_read_fds, 0, 0, 0); // 或者SELECT

            for (uint i= 0; i < m_socket_map.size(); ++i) {
                if (m_poll_info.m_fds[i].revents & POLLIN)
                {
                  listen_sock= m_poll_info.m_pfs_fds[i];
                  is_unix_socket= m_socket_map[listen_sock];
                  break;
                }
            }
            MYSQL_SOCKET connect_sock;
            connect_socket= mysql_socket_accept(key_socket_client_connection, listen_sock, (struct sockaddr *)(&cAddr), &length);
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
                THD *thd= init_new_thd(channel_info);           // 初始化THD对象
                thd_manager->add_thd(thd);

                if (thd_prepare_connection(thd)) {              // connection phase
                    lex_start(thd); // 初始化sqlparser
                    rc= login_connection(thd);
                        check_connection(thd);
                            acl_authenticate(thd, COM_CONNECT); // auth认证
                        thd->send_statement_status();
                    prepare_new_connection_state(thd);          // 准备接受QUERY
                } else {										// command phase
                    while (thd_connection_alive(thd))           // 判活
                    {
                        if (do_command(thd))                    // 处理query sql/sql_parser.c
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

## command phase

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
            mysql_parse(thd, &parser_state);        // 解析
    }
}

// sql/sql_parse.cc
void mysql_parse(THD *thd, Parser_state *parser_state) {
    mysql_reset_thd_for_next_command(thd);

    lex_start(thd);
    parse_sql(thd, parser_state, NULL);             // 解析SQL语句

    mysql_execute_command(thd, true);               // 执行SQL语句
        LEX  *const lex= thd->lex;
        TABLE_LIST *all_tables= lex->query_tables;
        
        trans_commit_implicit(thd);                 // 隐式提交 sql/transaction.cc
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
            
            case SQLCOM_COMMIT:                     // 显式提交
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

# 网络模型

MySQL对于网络处理模型做了非常好的抽象分层。

## 网络处理模型

MySQL对于网络通信的封装层次如下：

````
| Channel_info		连接
| THD			线程
| Protocol		应用协议
| NET			网络缓冲
| VIO			网络I/O
| SOCKET		socket fd
````

Channel_info封装了连接信息。

THD封装了线程相关的数据结构。

Protocol封装了应用协议，一共有5种，其中2种最常用，统称为classic protocol：

- PROTOCOL_TEXT：用于plain SQL
- PROTOCOL_BINARY：用于prepared statement，也称为[prepared statement protocol](https://dev.mysql.com/doc/internals/en/prepared-statements.html)。

NET封装了网络缓冲区，包括buffer、packet、read/write点位。

VIO封装了网络I/O，包括sockaddr等待。

SOCKET封装了socket fd，里面只有fd信息。

这些对象的创建时机如下：

![MySQL_network_layout](/MySQL_network_layout.png)

创建对象的函数调用链如下：

````c++
mysqld_socket_acceptor->connection_event_loop()
    Mysqld_socket_listener::listen_for_connection_event()
        mysql_socket_accept
        new Channel_info_tcpip_socket                       // accept & 封装 Channel_Info
    Connection_handler_manager::process_new_connection()
        Per_thread_connection_handler::add_connection       // 交给具体的conn_handler处理连接
            create pthread(handle_connection)
                for loop
                    init_new_thd                            // 初始化VIO, THD, Protocol和VIO
                        channel_info->create_thd()
                        delete channel_info
                    thd_manager->add_thd                    // 加thd
                    thd_prepare_connection                  // connection phase
                        login_connection
                    while thd_connection_alive              // command phase
                        do_command
                            dispatch_command
 
create_thd()
    create_and_init_vio
        mysql_socket_vio_new                    // malloc，初始化VIO
            VIO malloc
            vio_init
        THD malloc                              // malloc，初始化THD，Protocol
        thd->get_protocol_classic()->init_net   // 初始化NET
            my_net_init
                my_net_local_init               // 设置net_read_timeout, net_write_timeout
````

从上面我们可以看出，TCP建链后主线程只封装了Channel_nfo用于存放连接的信息，后续的THD、NET、VIO等信息的创建和初始化都（connections and disconnects）是在用户线程完成的。通过这种方式，主线程可以更高效的accept新的连接请求，从而优化在短连接场景下的性能。

参见[Improving connect/disconnect performance](http://mysqlserverteam.com/improving-connectdisconnect-performance/)和[WL#6606: Offload THD initialization and network initialization to worker thread](https://dev.mysql.com/worklog/task/?id=6606)。

短连接的性能优化效果如下：

![MySQL_5.7_connect_disconnect_performance](/MySQL_5.7_connect_disconnect_performance.jpeg)

我们先看一下Channel_info。

## Channel_info 连接

Channel_info对象封装了连接信息，以区分处理不同的连接方式：local、TCP/IP、named pipes和shared memory，并负责整个网络模型层次中各个对象的初始化。类和类关系图如下：

Channel_info_local_socket

Channel_info_shared_mem

Channel_info_named_pipe

Channel_info_tcpip_socket

![MySQL_Channel_info](/MySQL_Channel_info.png)

属性

| 属性                   | 说明           |
| ---------------------- | -------------- |
| prior_thr_create_utime | 连接的创建时间 |

方法

| 方法                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| ctor/dtor                    | /                                                            |
| create_thd                   | 创建THD                                                      |
| create_and_init_vio          | 创建并初始化VIO（只针对local、TCP/IP）                       |
| send_error_and_close_channel | 发送错误包并关闭socket                                       |
| prior_thr_create_utime       | getter/setter Per_thread_connection_handler::add_connection时设置 |

## THD 线程

[THD]({{< ref  "thd.md" >}} "THD")

## Protocol 应用协议

[Protocol]({{< ref  "Protocol.md" >}} "Protocol")

## NET 网络缓冲

[NET]({{< ref  "net.md" >}} "NET")

## VIO 网络

[VIO]({{< ref  "vio.md" >}} "VIO")

## SOCKET socket fd

SOCKET最简单，直接代码说话。

````c++
/** An instrumented socket. */
struct st_mysql_socket
{
  /** The real socket descriptor. */
  my_socket fd;

  /**
    The instrumentation hook.
    Note that this hook is not conditionally defined,
    for binary compatibility of the @c MYSQL_SOCKET interface.
  */
  struct PSI_socket *m_psi;
};

/**
  An instrumented socket.
  @c MYSQL_SOCKET is a replacement for @c my_socket.
*/
typedef struct st_mysql_socket MYSQL_SOCKET;
````

## thread cache

pthread复用可以通过[thread_cache_size](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_thread_cache_size)配置：默认值为8 + (max_connections / 100)。

MySQL提供如下status可以查看thread的数量信息：

- Threads_cached：缓存的 thread数量
- Threads_connected：已连接的thread数量
- Threads_created：建立的thread数量
- Threads_running：running状态的 thread 数量

Threads_created = Threads_cached + Threads_connected

Threads_running <= Threads_connected

创建pthread新连接非常消耗资源，特别是在短连接频繁场景下，如果又没有其他组件实现连接池，通过观察Connections/Threads_created的比例，适当提高 thread_cache_size，可以降低新建连接的开销。

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

![MySQL_pthread_cache](/MySQL_pthread_cache.png)

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

````c++
typedef struct user_resources {

  uint questions;   	/* MAX_QUERIES_PER_HOUR */
  uint updates; 		/* MAX_UPDATES_PER_HOUR */
  uint conn_per_hour; 	/* MAX_CONNECTIONS_PER_HOUR */
  uint user_conn; 		/* MAX_USER_CONNECTIONS */

  /*
     Values of this enum and specified_limits member are used by the
     parser to store which user limits were specified in GRANT statement.
  */
  enum {QUERIES_PER_HOUR= 1, UPDATES_PER_HOUR= 2, CONNECTIONS_PER_HOUR= 4,
        USER_CONNECTIONS= 8};
  uint specified_limits;
} USER_RESOURCES;
````

### ACL_USER

ACL_USER 是保存用户认证相关信息的类 USER_RESOURCES 是它的成员属性

````c++
class ACL_USER :public ACL_ACCESS
{
public:
  USER_RESOURCES user_resource;
...
}
````

ACl_USER 对象保存在数组 acl_users 中，每次mysqld启动时，从mysql.user表中读取数据，初始化 acl_users，初始化过程在函数 acl_load 中

调用栈如下：

````c++
main()
	mysqld_main()
		acl_init(opt_noacl);
    		acl_reload(thd);
        		acl_load(thd, tables);
````

### USER_CONN

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

````c++
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
| ------------------------ | -------------------------------- |
| MAX_USER_CONNECTIONS     | check_for_max_user_connections() |
| MAX_CONNECTIONS_PER_HOUR | check_for_max_user_connections() |
| MAX_QUERIES_PER_HOUR     | check_mqh()                      |
| MAX_UPDATES_PER_HOUR     | check_mqh()                      |

调用链

````c++
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

# 权限存储与管理

MySQL用户权限信息都存储在以下系统表中，用户权限的创建、修改和回收都会同步更新到系统表中。

| 系统表             | 存储的权限信息          |
| ------------------ | ----------------------- |
| mysql.user         | 用户权限                |
| mysql.db           | 库权限                  |
| mysql.tables_priv  | 表权限                  |
| mysql.columns_priv | 列权限                  |
| mysql.procs_priv   | 存储过程和UDF的权限信息 |
| mysql.proxies_priv | proxy权限               |

{{< hint info >}}
mysql.db存储是库的权限信息，并不存储实例有哪些库（ls查找目录）。
{{</hint>}}

information_schema表的查询接口：

- USER_PRIVILEGES
- SCHEMA_PRIVILEGES
- TABLE_PRIVILEGES
- COLUMN_PRIVILEGES

## 权限缓存

用户在连接数据库的过程中，为了加快权限的验证过程，系统表中的权限会缓存到内存中。

- mysql.user → acl_users
- mysql.db → acl_dbs
- mysql.tables_priv和mysql.columns_priv → column_priv_hash
- mysql.procs_priv → proc_priv_hash和func_priv_hash

另外acl_cache缓存db级别的权限信息。例如执行use db时，会尝试从acl_cache中查找并更新当前数据库权限（thd->security_ct→db_access）。

## 权限更新

以grant select on test.t1为例:

1. 更新系统表mysql.user，mysql.db，mysql.tables_priv
2. 更新缓存acl_users，acl_dbs，column_priv_hash
3. 清空acl_cache

## flush privileges

重新从系统表中加载权限信息来构建缓存。

## MariaDB Role体系

从MairaDB 10.0.5开始，MariaDB开始提供Role（角色）的功能，补全了大家一直吐槽的MySQL不能像 Oracle 一样支持角色定义的功能。

一个角色就是把一堆的权限捆绑在一起授权，这个功能对于有很多用户拥有相同权限的情况可以显著提高管理效率。在有角色之前，这种情况只能为每个用户都做一大堆的授权操作，或者是给很多个需要相同权限的用户提供同一个账号去使用，这又会导致你要分析用户行为的时候不知道哪个操作是哪个具体用户发起的。

有了角色，这样的管理就太容易了。例如，可以把权限需求相同的用户赋予同一个角色，只要定义好这个角色的权限就行，要更改这类用户的权限，只需要更改这个角色的权限就可以了，变化会影响到所有这个角色的用户。

### 使用方法

创建角色需要使用CREATE ROLE语句，删除角色使用DROP ROLE语句。然后再通过GRANT语句给角色增加授权，也可以把角色授权给用户，然后这个角色的权限就会分配给这个用户。同样，REVOKE语句也可以用来移除角色的授权，或者把一个用户移除某个角色。

一旦用户连接上来，他可以执行SET ROLE语句来把自己切换到某个被授权的角色下，从而使用这个角色的权限。通过CURRENT_ROLE函数可以显示当前用户执行在哪个角色下，没有就是NULL。

只有直接被授予用户的角色才可以使用SET ROLE语句，间接授予的角色并不能被SET ROLE设置。例如角色B被授予角色A，而角色A被授予用户A，那么用户A只能 SET ROLE 角色A，而不能设置角色B。（角色B->角色A->用户A）

从MariaDB 10.1.1开始，可以利用SET DEFAULT ROLE语句来给某个用户设置默认的角色。当用户链接的时候，会默认使用这个角色，其实就是连接后自动做了一个SET ROLE语句。

创建一个角色并给他赋权:

````
CREATE ROLE journalist;
GRANT SHOW DATABASES ON *.* TO journalist;
GRANT journalist to hulda;
````

这里hulda并不马上拥有SHOW DATABASES权限，他还需要先执行一个SET ROLE语句启用这个角色：

````
// 一开始只能看到IS库
SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
+--------------------+

// 当前用户没有对应的角色
SELECT CURRENT_ROLE;
+--------------+
| CURRENT_ROLE |
+--------------+
| NULL         |
+--------------+

// 启用角色
SET ROLE journalist;

SELECT CURRENT_ROLE;
+--------------+
| CURRENT_ROLE |
+--------------+
| journalist   |
+--------------+

SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| ...                |
| information_schema |
| mysql              |
| performance_schema |
| test               |
| ...                |
+--------------------+.

SET ROLE NONE;
````

角色也可以授权给另一个角色（角色累加）：

````
CREATE ROLE writer;
GRANT SELECT ON db1.* TO writer;
GRANT writer TO journalist;
````

但是只能SET ROLE直接给用户的角色。像这里hulda只能SET ROLE journalist，而不能SET ROLE writer，并且只要启用了journalist角色，hulda也自动获得了writer角色的权限：

````
SELECT CURRENT_ROLE;
+--------------+
| CURRENT_ROLE |
+--------------+
| NULL         |
+--------------+
SHOW TABLES FROM data;
Empty set (0.01 sec)

// 启用角色
SET ROLE journalist;
SELECT CURRENT_ROLE;
+--------------+
| CURRENT_ROLE |
+--------------+
| journalist   |
+--------------+

// 叠加了wirter角色，可以访问db1中的表
SHOW TABLES FROM db1;
+------------------------------+
| Tables_in_db1                |
+------------------------------+
| set1                         |
| ...                          |
+------------------------------+
````

### 限制

角色和视图、存储过程

当用户设置启用了一个角色，从某种意义上说他有两个身份的权限集合（用户本身和他的角色）但是一个视图或者存储过程只能有一个定义者。所以，当一个视图或者存储过程通过SQL SECURITY DEFINER创建时，只能指定CURRENT_USER或者CURRENT_ROLE中的一个。所以有些情况下，你创建了一个视图，但是你却可能没法使用它。

````
CREATE ROLE r1;
GRANT ALL ON db1.* TO r1;
GRANT r1 TO foo@localhost;
GRANT ALL ON db.* TO foo@localhost;

SELECT CURRENT_USER;
+---------------+
| current_user  |
+---------------+
| foo@localhost |
+---------------+

SET ROLE r1;
CREATE TABLE db1.t1 (i int);
CREATE VIEW db.v1 AS SELECT * FROM db1.t1;

SHOW CREATE VIEW db.v1;
+------+------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
| View | Create View                                                                                                                              | character_set_client | collation_connection |
+------+------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
| v1   | CREATE ALGORITHM=UNDEFINED DEFINER=`foo`@`localhost` SQL SECURITY DEFINER VIEW `db`.`v1` AS SELECT `db1`.`t1`.`i` AS `i` from `db1`.`t1` | utf8                 | utf8_general_ci      |
+------+------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+

CREATE DEFINER=CURRENT_ROLE VIEW db.v2 AS SELECT * FROM db1.t1;

SHOW CREATE VIEW db.b2;
+------+-----------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
| View | Create View                                                                                                                 | character_set_client | collation_connection |
+------+-----------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
| v2   | CREATE ALGORITHM=UNDEFINED DEFINER=`r1` SQL SECURITY DEFINER VIEW `db`.`v2` AS select `db1`.`t1`.`a` AS `a` from `db1`.`t1` | utf8                 | utf8_general_ci      |
+------+-----------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
````

# 相关文件

源代码文件按照文章顺序整理：

````
源代码文件按照文章顺序整理：
sql/conn_handler/named_pipe_connection.h			Named_pipe_listener
sql/conn_handler/named_pipe_connection.cc			Channel_info_named_pipe
sql/conn_handler/shared_memory_connection.h			Shared_mem_listener
sql/conn_handler/shared_memory_connection.cc			Channel_info_shared_mem
sql/conn_handler/socket_connection.h				Mysqld_socket_listener
sql/conn_handler/socket_connection.cc				Channel_info_local_socket
								Channel_info_tcpip_socket
								TCP_socket
								Unix_socket
sql/conn_handler/channel_info.h				Channel_info
sql/conn_handler/channel_info.cc
sql/conn_handler/connection_acceptor.h				Connection_acceptor
sql/conn_handler/connection_handler_manager.h			Connection_handler_manager
sql/conn_handler/connection_handler_manager.cc
sql/conn_handler/connection_handler.h				Connection_handler
sql/conn_handler/connection_handler_impl.h			Per_thread_connection_handler
								One_thread_connection_handler
sql/conn_handler/plugin_connection_handler.h			Plugin_connection_handler
sql/conn_handler/connection_handler_one_thread.cc
sql/conn_handler/connection_handler_per_thread.cc
````


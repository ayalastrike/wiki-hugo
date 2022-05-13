MySQL启动过程

```
main() // 入口 sql/main.cc
	mysqld_main() // sql/mysqld.cc

		// 记录入参
		my_progname = argv[0];
		orig_argc = argc;
		orig_argv = argv;

		// 处理配置文件my.cnf及启动参数
		load_defaults(MYSQL_CONFIG_NAME, load_default_groups, &argc, &argv, &argv_alloc);

		// 继续处理启动参数，为初始化系统表做准备 sys_var::m_parse_flag == PARSE_EARLY
		handle_early_options();

		// 为status统计计数做准备
		init_sql_statement_names();

		// 初始化system variables哈希表,链表sys_var_chain sys_var,遍历链表后,加入到system_variable_hash哈希表
		// sys_var_chain链表已经通过sys_vars.cc的sys_var()构造函数static初始化
		sys_var_init();

		// 计算打开文件数并初始化table cache
		adjust_related_options(&requested_open_files);

		// init error log global variables
		init_error_log();

		// init audit global variables
		mysql_audit_initialize();

		// 初始化query log和slow log
		query_logger.init();

		// 初始化system variables
		init_common_variables();
            default_storage_engine= const_cast<char *>("InnoDB"); // 设置默认storage engine
            add_status_vars(status_vars); // 初始化status变量(show status), status_vars为全局变量
			set_server_version();
			get_options(&remaining_argc, &remaining_argv); // sys_var::m_parse_flag == PARSE_NORMAL
			// 设置thread_cache_size
			init_client_errs(); // 读出给client返回出错信息的文件
            lex_init(); // 初始化词法分析
			item_create_init(); // 初始化函数列表 func_array为全局变量
			// 设置默认字符集和校验字符集
		    global_system_variables.collation_connection= default_charset_info;
		    global_system_variables.character_set_results= default_charset_info;
		    global_system_variables.character_set_client= default_charset_info;
			// 设置默认storage engine
			lex_init();

		// 初始化信号量
        my_init_signals();
		// 启动核心模块
		init_server_components();
			mdl_init();					// mdl元数据锁
			table_def_init(); 			// 表定义缓存
			init_server_query_cache();	// Query Cache
			init binlog relaylog		// Binlog Relaylog
			gtid_server_init();			// GTID
			plugin_register_builtin_and_init_core_se(); // Load builtin plugins, initialize MyISAM, CSV and InnoDB

		// 初始化并创建GTID
		init_server_auto_options();

		// 初始化SSL
		init_ssl();

		// 初始化网络
		network_init();

		// 创建PID文件
		create_pid_file();

		// 初始化status variables
		init_status_vars();

		// binlog相关检查初始化
		check_binlog_cache_size(NULL);
		check_binlog_stmt_cache_size(NULL);
		binlog_unsafe_map_init();

		// 初始化Slave
		init_slave();

		// 创建线程处理信号量
		start_signal_handler();

		// 如果是安装初始化，创建handle_bootstrap线程进行初始化datadir,系统表
		if (opt_bootstrap)
			bootstrap(mysql_stdin);

		// 创建manager线程
		start_handle_manager();

		// 执行DDL crash recovery
		execute_ddl_log_recovery();

		// 监听socket事件
		mysqld_socket_acceptor->connection_event_loop();

		if (signal_thread_id.thread != 0)
			ret= my_thread_join(&signal_thread_id, NULL);
		clean_up(1);
		mysqld_exit(MYSQLD_SUCCESS_EXIT);
```

其中，load_defaults()会寻找my.cnf，并根据load_default_groups，使用search_default_file_with_ext()解析每一行配置，并进行标准化（配置项名称前加上--）

```
// sql/mysqld.cc
2325 const char *load_default_groups[]= {
...
2329 "mysqld","server", MYSQL_BASE_VERSION, 0, 0};
 
// mysys_ssl/my_default.cc
my_load_defaults
	my_search_option_files
		my_search_option_files
		search_default_file_with_ext
```

配置项会被handle_default_option()缓存在内存中

```
// mysys_ssl/my_default.cc handle_default_option()
struct handle_option_ctx
{
   MEM_ROOT *alloc;
   My_args *m_args;
   TYPELIB *group;
};
```
---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

# 日志

MySQL server层日志有：

- general log
- slow log
- error log

日志的存储方式（log_output）有：

- 文件
- 表

所有日志的配置参数如下：

````
log_output                              general log，slow log存放在何处（文件/表）
log_timestamps                          general log, slow log, error log写入文件中的时间戳所用时区（UTC/SYSTEM），写入日志表中的不在此列。
 
general_log                             是否开启general log
sql_log_off                             当前session是否关闭general log
general_log_file                        general log文件名
 
slow_query_log                          是否开启slow log
slow_query_log_file                     slow log文件名
long_query_time                         slow query的判断时间（秒）
log_queries_not_using_indexes           是否记录没有使用index的query
log_throttle_queries_not_using_indexes  log_queries_not_using_indexes配额（每分钟）
log_slow_admin_statements               是否记录慢管理语句（ALTER TABLE, ANALYZE TABLE, CHECK TABLE, CREATE INDEX, DROP INDEX, OPTIMIZE TABLE, and REPAIR TABLE）
 
log_error                               error log文件名
log_error_verbosity                     error, warning, and note messages
log_warnings                            deprecated，用log_error_verbosity替代
 
log_syslog
log_syslog_facility
log_syslog_include_pid
log_syslog_tag
 
log-slow-slave-statements               记录从库上回放时的慢SQL（SBR/MIXED）
 
mysqld startup options
--log-isam[=file_name]                  记录所有MyISAM的变化（debug）
--log-raw                               记录query rewrite plugin改写前的SQL
````

# 日志实现机制

## general log和slow log

### 实现

如下图所示：

![MySQL_logger_implemenation](/MySQL_logger_implemenation.png)

### logger生命周期

````
// mysqld启动
query_logger.set_handlers(LOG_NONE | LOG_FILE | LOG_NONE);  // 设置log event handler list
query_logger.init();                                        // new log event handler
 
// 运行时
query_logger.general_log_write                              // 写general log
log_slow_statement                                          // 写slow log
    log_slow_do                                             // 记录query rewrite plugin改写前的SQL
        query_logger.slow_log_write
 
// mysqld关闭
query_logger.cleanup();
````

### 日志格式

general log日志格式

````
thd->current_utime()    thd->thread_id()    enum_server_command     thd->query().str
````

slow log日志格式

````
# Time: thd->current_utime()
# User@Host: sctx->priv_user().str [thd->security_context()->user()] @ thd->security_context()->host() [thd->security_context()->ip()]  Id:     thd->thread_id()
# Query_time: current_utime - thd->start_utime  Lock_time: thd->utime_after_lock - thd->start_utime Rows_sent: thd->get_sent_row_count()  Rows_examined: thd->get_examined_row_count()
use thd->db().str
SET timestamp=thd->current_utime()
thd->query().str
````

{{< hint info >}}

thd->utime_after_lock由thd_storage_lock_wait设置

{{</hint>}}

示例

````
general log
2021-05-27T10:16:16.864244+08:00          154 Query     select now()
 
slow log
# Time: 2021-03-23T10:41:29.249524+08:00
# User@Host: root[root] @ localhost []  Id:    34
# Query_time: 11.409186  Lock_time: 0.000117 Rows_sent: 11  Rows_examined: 7499929
use device_manager;
SET timestamp=1616467289;
select distinct cmd_param_key from  device_task_info;
````

### slow log细节

slow_log 是在语句执行完后记录的，因为加锁时间和返回记录数这些信息，在执行之后才知道，general_log 记录是在语句解析完执行前。

判断query是否记入slow_log的函数为log_slow_applicable()：

1. warn_no_index：没有用到索引，并且log_queries_not_using_indexes打开（这种情况下还可能触发 throttle 导致不记录）
2. log_this_query：被标记为慢SQL：(thd->server_status & SERVER_QUERY_WAS_SLOW) 或者 warn_no_index 或者扫描行数大于min_examined_row_limit

**log_slow_applicable()**

````c++
bool warn_no_index= ((thd->server_status &
                      (SERVER_QUERY_NO_INDEX_USED |
                       SERVER_QUERY_NO_GOOD_INDEX_USED)) &&
                     opt_log_queries_not_using_indexes &&
                     !(sql_command_flags[thd->lex->sql_command] &
                       CF_STATUS_COMMAND));
bool log_this_query=  ((thd->server_status & SERVER_QUERY_WAS_SLOW) ||
                       warn_no_index) &&
                      (thd->get_examined_row_count() >=
                       thd->variables.min_examined_row_limit);
````

在SQL执行结束时，会先调用thd->update_server_status()判断是否是慢SQL，如果判断为真则将server_status |= SERVER_QUERY_WAS_SLOW，逻辑如下：

````c++
/**
 Update server status after execution of a top level statement.
 
 Currently only checks if a query was slow, and assigns
 the status accordingly.
 Evaluate the current time, and if it exceeds the long-query-time
 setting, mark the query as slow.
*/
void update_server_status()
{
  ulonglong end_utime_of_query= current_utime();
  if (end_utime_of_query > utime_after_lock + variables.long_query_time)
    server_status|= SERVER_QUERY_WAS_SLOW;
}
````

utime_after_lock表示的是拿到锁的时间点，server层通过THD::set_time_after_lock()设置，引擎层（InnoDB）如果发生锁等待，会通过thd_storage_lock_wait()累加。因此一个SQL如果一直处于等待中而没有执行完，是不会记入慢SQL的。

query_time和lock_time的计算逻辑如下：

````
Query_logger::slow_log_write
 
 if (thd->start_utime)
 {
   query_utime= (current_utime - thd->start_utime);
   lock_utime=  (thd->utime_after_lock - thd->start_utime);
 }
````

同理，rows_sent通过THD::inc_sent_row_count()一直累加。而rows_examined会在累加后由JOIN::exec()重置。

```
JOIN::exec() {
 ``...
 ``/*
  ``Initialize examined rows here because the values from all join parts
  ``must be accumulated in examined_row_count. Hence every join
  ``iteration must count from zero.
 ``*/
 ``examined_rows= 0;
 ``...
}
```

我们go through了一遍，从上面我们可以得知，参与判断为慢SQL的变量有：

- thd->server_status
  - thd->utime_after_lock - thd->start_utime
- SQL类型为CF_STATUS_COMMAND
- m_examined_row_count

在用户SQL的执行中，会在每个SQL语句执行前，把server_status, start_utime, m_sent_rows_count重置，调用链如下：

````
mysql_parse()
    THD::reset_for_next_command()
````

## error log

### 实现

````
// mysqld使用
sql_print_error
sql_print_warning
sql_print_information
{
  va_list args;
  DBUG_ENTER("sql_print_warning");
 
  va_start(args, format);
  error_log_print(INFORMATION_LEVEL, format, args);
  va_end(args);
 
  DBUG_VOID_RETURN;
}
 
// plugin使用
my_plugin_log_message
{
  va_start(args, format);
  my_snprintf(format2, sizeof (format2) - 1, "Plugin %.*s reported: '%s'",
              (int) plugin->name.length, plugin->name.str, format);
  error_log_print(lvl, format2, args);
  va_end(args);
}
````

二者都是调用error_log_print，然后通过buffer IO将错误信息写入文件或者syslog。

{{< hint info >}}

从上面可以看出，mysql的所有日志都是buffer IO，其中general log，slow log通过IO_CACHE缓冲，error log通过std::ostringstream缓冲，然后在某些时刻调用flush_error_log_messages()进行flush。

{{</hint>}}
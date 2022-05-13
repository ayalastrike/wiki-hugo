# THD对象

THD封装了线程相关的数据，可以视作一个处理单元。

The size of the THD is ~10K and its definition is found in sql_class.h.

The THD is a large data structure which is used to keep track of various aspects of execution state. Memory rooted in the THD will grow significantly during query execution, but exactly how much it grows will depend upon the query. For memory planning purposes we recommend to plan for ~10MB per connection on average.

````c++
For each client connection we create a separate thread with THD serving as a thread/connection descriptor
 
class THD
{
  NET     net;                      // client connection descriptor
  Vio* active_vio;
  Protocol *m_protocol;             // Current protocol
 
  Protocol_text   protocol_text;    // Normal protocol
  Protocol_binary protocol_binary;  // Binary protocol
  ...
}
````

# Global_THD_manager

Global_THD_manager作为thd管理器，统一提供thd资源的管控，提供thd的全局查找、计数、执行操作。

## 生命周期

````
create_instance()       // 创建thd_mgr，mysqld启动时
destroy_instance()      // 销毁thd_mgr，mysqld关闭时
get_instance()          // 在mysqld运行时获取thd_mgr，以调用其api
````

## API

````
add_thd()               // 添加thd
remove_thd()            // 移除thd
wait_till_no_thd        // 移除所有thd
````

## thd操作

### 查找

提供Find_THD_Impl函数模板，并由以下方法调用：

- find_thd

函数模板

````
Find_THD_variable   在PS查看thd信息时提供并发控制（thd->LOCK_thd_data）
Find_thd_user_var   在PS查看thd信息时提供并发控制（thd->LOCK_thd_data）
...
````

### 计数

num_thread_running
thread_created

### 执行操作

提供Do_THD_Impl函数模板，并由以下方法调用：

- do_for_all_thd_copy ：提供LOCK_thd_remove对remove thd进行并发控制
- do_for_all_thd ：不做并发控制

函数模板

````
Set_kill_conn       // kill thd
List_process_list   // 列出所有thd
...
````

# kill process_id

kill命令格式

````
KILL [CONNECTION | QUERY] processlist_id
````

其中：

- CONNECTION：中止processlist_id对应的查询中止，连接退出
- QUERY ：中止processlist_id对应的查询中止，连接保持
- Ctrl+C ：MySQL client Ctrl+C会新建立一个临时的connection，将kill query的命令发送给MySQL，停止之前的命令，再回收掉临时的connection

## 处理kill

kill的执行是异步的，分为标记kill和中止执行两阶段。

### 标记kill

当MySQL Server收到kill命令时，会根据命令中指定的process_id，查找到对应的THD，设置kill标记

````c++
static uint kill_one_thread(THD *thd, my_thread_id id, bool only_kill_query)
{
  THD *tmp= NULL;
  uint error=ER_NO_SUCH_THREAD;
  Find_thd_with_id find_thd_with_id(id);
 
  DBUG_ENTER("kill_one_thread");
  DBUG_PRINT("enter", ("id=%u only_kill=%d", id, only_kill_query));
  tmp= Global_THD_manager::get_instance()->find_thd(&find_thd_with_id);
  ...
  tmp->awake(only_kill_query ? THD::KILL_QUERY : THD::KILL_CONNECTION);
  ...
}
````

### 中止执行

1. 空闲的process_id立即退出

````c++
void THD::awake(THD::killed_state state_to_set) {
  ...
  if (this->m_server_idle && state_to_set == KILL_QUERY)
  { /* nothing */ }
  else
  {
    killed= state_to_set;
  }
  ...
}
````

2. 在SQL的执行过程中，会在各种位置检测THD上的kill标记，中止执行，清理后退出

3. 等待中响应

   如果查询此时等待在某个condition_variable上，那么短时间内可能很难唤醒，如果出现了死锁的情况，那么就更不可能唤醒了。因此，kill实现了针对等待的特殊响应，其主要思路是：在某个查询进入等待状态之前，在THD上记录下当前查询等待的condition_variable对象及其对应的mutex。

   ````c++
   void enter_cond(mysql_cond_t *cond, mysql_mutex_t* mutex,
                   const PSI_stage_info *stage, PSI_stage_info *old_stage,
                   const char *src_function, const char *src_file,
                   int src_line)
   {
     DBUG_ENTER("THD::enter_cond");
     mysql_mutex_assert_owner(mutex);
     /*
       Sic: We don't lock LOCK_current_cond here.
       If we did, we could end up in deadlock with THD::awake()
       which locks current_mutex while LOCK_current_cond is locked.
     */
     current_mutex= mutex;
     current_cond= cond;
     enter_stage(stage, old_stage, src_function, src_file, src_line);
     DBUG_VOID_RETURN;
   }
   ````

   在等待的条件上增加对thd->killed状态的判断，即检测到killed时退出等待

   ````c++
   longlong Item_func_sleep::val_int()
   {
     THD *thd= current_thd;
     Interruptible_wait timed_cond(thd);
     mysql_cond_t cond;
    
     timeout= args[0]->val_real();
    
     mysql_cond_init(key_item_func_sleep_cond, &cond);
     mysql_mutex_lock(&LOCK_item_func_sleep);
    
     thd->ENTER_COND(&cond, &LOCK_item_func_sleep, &stage_user_sleep, NULL);
    
     error= 0;
     thd_wait_begin(thd, THD_WAIT_SLEEP);
     while (!thd->killed)
     {
       error= timed_cond.wait(&cond, &LOCK_item_func_sleep);
       if (error == ETIMEDOUT || error == ETIME)
         break;
       error= 0;
     }
     thd_wait_end(thd);
     mysql_mutex_unlock(&LOCK_item_func_sleep);
     thd->EXIT_COND(NULL);
    
     mysql_cond_destroy(&cond);
    
     return MY_TEST(!error);       // Return 1 killed
   }
   ````
   
   kill发生时, 使用THD记录的condition_variable进行pthread_cond_signal，进行唤醒，等待的线程醒来检测kill标记，发现已被标记kill快速退出。
   
   ````c++
   void THD::awake(THD::killed_state state_to_set)
   {
     ...
     /* Broadcast a condition to kick the target if it is waiting on it. */
     if (is_killable)
     {
       mysql_mutex_lock(&LOCK_current_cond);
       /*
         This broadcast could be up in the air if the victim thread
         exits the cond in the time between read and broadcast, but that is
         ok since all we want to do is to make the victim thread get out
         of waiting on current_cond.
         If we see a non-zero current_cond: it cannot be an old value (because
         then exit_cond() should have run and it can't because we have mutex); so
         it is the true value but maybe current_mutex is not yet non-zero (we're
         in the middle of enter_cond() and there is a "memory order
         inversion"). So we test the mutex too to not lock 0.
    
         Note that there is a small chance we fail to kill. If victim has locked
         current_mutex, but hasn't yet entered enter_cond() (which means that
         current_cond and current_mutex are 0), then the victim will not get
         a signal and it may wait "forever" on the cond (until
         we issue a second KILL or the status it's waiting for happens).
         It's true that we have set its thd->killed but it may not
         see it immediately and so may have time to reach the cond_wait().
    
         However, where possible, we test for killed once again after
         enter_cond(). This should make the signaling as safe as possible.
         However, there is still a small chance of failure on platforms with
         instruction or memory write reordering.
       */
       if (current_cond && current_mutex)
       {
         mysql_mutex_lock(current_mutex);
         mysql_cond_broadcast(current_cond);
         mysql_mutex_unlock(current_mutex);
       }
       mysql_mutex_unlock(&LOCK_current_cond);
     }
     DBUG_VOID_RETURN;
   }

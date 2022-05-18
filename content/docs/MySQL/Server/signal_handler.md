# 引出问题

在之前MySQL发版测试后，QA发现一个问题，用mysql.server stop脚本无法使mysqld进程退出，只会无限等待。

通过查看mysql.server脚本，发现使用stop选项时执行了如下操作：

1. 检查mysql/var/mysql.pid文件是否存在，若不存在说明mysqld进程未启动；若存在，读出文件中记录的mysqld进程号pid；
2. 发送kill -0 pid，检查是否真的存在进程号为pid的进程。若进程不存在则无需stop，直接删除mysql/var/mysql.pid文件；
3. 若pid进程存在，发送kill pid（即kill -15 pid），随后循环等待直到mysql/var/mysql.pid文件被删除（意味着mysqld已经退出）。

由此逻辑可知，mysql.server stop脚本无限等待大概率是因为第3步发送kill pid后，mysqld并未走退出流程，mysql/var/mysql.pid迟迟不被删所致。通过观察mysql/log/mysql.err文件中的日志，发现执行mysql.server stop后只有这么一条记录：

````
Got signal 15 from thread 0Got signal 15 from thread 0
````

说明mysqld进程确实有收到SIGTERM信号，但是并未发起shutdown流程，因此需要了解Linux信号处理机制及MySQL中信号处理逻辑才好判断是哪里出了问题。

# Linux信号处理机制

在Linux系统中，我们可以通过kill -信号名 进程ID的方式向指定进程发送各种信号。比如常用的kill pid其实等价于kill -**15** pid，会向pid进程发送**SIGTERM**信号；在一个shell前台程序运行中按Ctrl+C，等价于kill -2 pid，会向pid进程发送SIGINT信号；而kill -9 pid则会向pid进程发送SIGKILL信号。

当进程收到信号时如何处理？通常有几种处理方式：

1. 默认：如果不自定义处理函数，则每个信号都有默认的处理方式。很多信号的默认处理方式都是进程中断+直接退出，极其暴力，这是我们不希望的，因此经常针对某些信号设置自定义处理函数；
2. 捕获：利用signal函数或sigaction函数可以注册用户自定义的信号处理函数（进程级生效）。比如在程序中使用signal(SIGINT, handle_shutdown)，可以注册进程收到SIGINT信号时调用用户自定义函数handle_shutdown，然后用户就可以在此函数中实现一些优雅退出的代码，比如保存工作进度、打印日志等等，不至于退出得过于暴力。**需要注意的是，并不是所有信号都允许被捕获并进入自定义处理函数，比如SIGKILL（kill -9）就不允许；**
3. 忽略：将某信号的回调函数设置为SIG_IGN(ignore)。进程收到此信号后直接丢弃，不作任何处理。一般涉及网络通信的服务端程序至少都会忽略坑爹信号SIGPIPE（当试图向对方发送数据时发现对方已断连就会触发，默认处理方式是进程退出，极其坑爹……）。**需要注意的是，并不是所有信号都允许被忽略，比如SIGKILL（kill -9）就不允许；**
4. 屏蔽：通过pthread_sigmask函数可以针对某一线程设置一个信号屏蔽集合，比如选择屏蔽SIGTERM，那么该线程在收到此信号是只是将信号丢入一个pending队列，并不做处理。在此要强调，屏蔽和忽略是有区别的，忽略是直接将信号丢弃，而屏蔽是可以通过pthread_sigmask函数解除的，当解除以后pending队列里的信号就会再次弹出。每个线程可以设置不同的屏蔽规则，但是线程创建时默认继承父线程的屏蔽规则。

在多线程程序中，向进程发送的信号除硬件故障导致的信号外，是可能被随机转发到任何一个线程的。不仅如此，连续发送多个信号，可能在多个子线程中被并发触发，这使得信号处理逻辑变得异常复杂。试想一下，假如我们在进程中对SIGTERM信号注册了优雅退出的处理函数，当用户对进程执行了kill命令，你的线程1捕获到SIGTERM正在优雅退出时，用户等得不耐烦又发了一遍kill命令，此时线程2就有可能捕获到SIGTERM，和线程1并发进行退出，这可能会导致各种非预期行为发生。

因此，**大部分多线程程序常用的信号处理模式是使用单独的信号处理线程，将可能在多个线程中并发出现的信号，转化为只在信号处理线程中排队依次出现，同步处理。**具体实现方式为：对于程序中绝大多数不关注信号的线程，都通过设置信号屏蔽规则在整个线程生命周期中将某些信号屏蔽；建立一个单独的信号处理线程，用一个无限循环调用sigwait函数等待所有关注的信号，阻塞式等待在别的线程“碰壁”后被加入pending队列的信号，每次取出一个信号，进行对应处理，再调用sigwait取下一个信号。这样就将多线程异步信号转为了单线程同步处理，使编写多线程程序的信号处理逻辑变得非常简单。当采用单独的信号处理线程后，不管哪个线程收到SIGTERM，都会在信号处理线程的sigwait函数中返回，这样同时就只可能由信号处理线程执行优雅退出，此时就算向进程发送别的信号，只要信号处理线程不将前一个信号处理完后再次调用sigwait，都是感知不到的。在下文中我们看到，MySQL正是使用了这种单独信号处理线程的模式。

# MySQL信号处理逻辑

在sql/mysqld.cc的主入口函数mysqld_main中靠前的位置，我们可以找到设置MySQL信号处理规则的函数my_init_signals：

````c++
// sql/mysqld.cc mysqld_main()
if (init_common_variables())
   unireg_abort(MYSQLD_ABORT_EXIT);        // Will do exit

my_init_signals();
````

在my_init_signals函数中首先设置了进程级的信号处理规则：

- 将SIGSEGV、SIGABRT、SIGBUS、SIGILL、SIGFPE等严重错误信号的自定义处理函数设置为handle_fatal_signal（此函数主要负责在mysql.err日志中记录进程挂掉之前的状态信息、函数调用栈，以及生成core文件等）；
- 将SIGPIPE、SIGALRM信号忽略（几乎没有哪个网络程序的服务端不忽略这个信号……否则客户端不close连接的情况下断线，服务端再尝试给客户端发送数据的时候就会触发SIGPIPE，直接中断进程）；
- 将SIGTERM、SIGHUP信号的自定义处理函数设置为print_signal_warning（此函数只负责在mysql.err里打印一条Got signal xx from thread yy，也就是之前调用mysql.server stop时日志里打印Got signal 15 from thread 0的原因；在MySQL 5.7中已经改为SIG_DFL默认动作）；

my_init_signals函数中还设置了线程级的信号屏蔽

- 将SIGTERM、SIGHUP、SIGQUIT信号用pthread_sigmask函数屏蔽。

由于线程创建时默认继承父线程的屏蔽规则，在my_init_signals函数调用结束之后，mysqld_main产生的所有子进程都继承了以上的信号处理逻辑。

之后，mysqld_main调用start_signal_handler函数，启动了独立的信号处理线程signal_thread，该线程执行函数signal_hand：

````c++
// sql/mysqld.cc mysqld_main()
#ifndef _WIN32
   //  Start signal handler thread.
   start_signal_handler();
#endif
  
// sql/mysqld.cc start_signal_handler()
mysql_mutex_lock(&LOCK_start_signal_handler);
if ((error=
      mysql_thread_create(key_thread_signal_hand,
                          &signal_thread_id, &thr_attr, signal_hand, 0)))
{
    sql_print_error("Can't create interrupt-thread (error %d, errno: %d)",
                    error, errno);
    flush_error_log_messages();
    exit(MYSQLD_ABORT_EXIT);
}
mysql_cond_wait(&COND_start_signal_handler, &LOCK_start_signal_handler);
mysql_mutex_unlock(&LOCK_start_signal_handler);
````

在signal_hand函数中，信号处理线程signal_thread使用无限循环调用sigwait等待SIGTERM、SIGQUIT和SIGHUP三种信号，其中当收到SIGTERM、SIGQUIT时都会调用pthread_kill(main_thread_id)启动shutdown流程。

因此可以知道，当收到SIGTERM信号时：

1. 凡是在my_init_signals函数后启动的线程，除signal_thread线程外，都会屏蔽SIGTERM信号，不做处理；
2. signal_thread线程收到SIGTERM信号，发起shutdown流程。

那么问题来了，如果有线程在my_init_signals函数前启动会如何？答案是该线程不屏蔽SIGTERM信号，如果进程将SIGTERM信号转发给该线程，则调用print_signal_warning打印一行日志。这样问题定位的思路就清晰了，kill mysqld不退出，一定是发给进程的SIGTERM被某个启动时机甚早，早于my_init_signals函数的线程拦截到了。

# 问题定位

有了先前的知识，问题排查就简单多了，只要揪出这个早早启动的线程就好，通过如下步骤很容易找到：

1. 通过pstack检查这个版本MySQL和普通MySQL的线程栈信息差异，发现这个版本引入了一个独有线程sampling_thread：

   ````
   Thread 70 (Thread 0x7f162b266700 (LWP 19637)):#0  0x00007f162b31d9cd in nanosleep () from /opt/compiler/gcc-4.8.2/lib/libc.so.6
   #1  0x00007f162b3476f4 in usleep () from /opt/compiler/gcc-4.8.2/lib/libc.so.6
   #2  0x00000000010c71ba in bvar::detail::SamplerCollector::run() () at public/bvar/bvar/detail/sampler.cpp:134
   #3  0x00000000010c8b89 in bvar::detail::SamplerCollector::sampling_thread(void*) () at public/bvar/bvar/detail/sampler.cpp:84
   #4  0x00007f162cef51c3 in start_thread () from /opt/compiler/gcc-4.8.2/lib/libpthread.so.0
   #5  0x00007f162b34e12d in clone () from /opt/compiler/gcc-4.8.2/lib/libc.so.6
   ````

2. 在源代码中搜索，发现sampling_thread线程为另一基础库libbvar所建立，该线程是一个静态全局变量的一部分，在程序启动后即建立，因此创建时机自然比mysqld_main调用my_init_signals时间更早，并未屏蔽掉SIGTERM信号，导致信号被此线程捕获，通过print_signal_warning打印日志处理；而MySQL自身的signal_thread线程无法再获得SIGTERM信号，自然不会发起shutdown流程；

3. 再通过pstack检查该版本MySQL的线程栈，找到signal_thread线程的轻量级进程号19908（Linux系统中线程使用进程实现，因此也有自己的进程号）：

4. 使用kill 19908，将SIGTERM直接发给signal_thread线程，发现mysqld开始进入shutdown流程，说明如果signal_thread线程捕获了SIGTERM信号是能够正常发起退出流程的，只是因为主进程收到信号后大概率发给sampling_thread线程，才导致mysqld不退出。

# 问题解决

解决方案有两种，第一种是修改libbvar中相关代码，令sampling_thread线程屏蔽掉需要暴露给signal_thread线程的信号即可；第二种是在程序中控制sampling_thread线程的初始化时间晚于mysqld_main中调用my_init_signals函数的时间，以便继承全局的信号屏蔽策略。由于sampling_thread线程是静态全局变量创建，几乎不可能将其创建时间延后至my_init_signals函数之后，因此只能采用第一种方法。经验证，修改后的mysqld进程在收到SIGTERM信号后可以正确退出了。

# 附录：mysql.server stop和mysqladmin shutdown的区别

有同学发现mysql.server stop无法另mysqld退出时，可以使用mysqladmin shutdown的方式是可以使mysqld进入退出流程的。这两种shutdown方式究竟有什么区别？

通过浏览mysqladmin源代码，可以看出执行shutdown命令时实际是调用libmysql/libmysql.c中的mysql_shutdown函数，向MySQL服务端发送了命令类型为COM_SHUTDOWN的请求。服务端接收到请求后调用了kill_server函数，直接用pthread_kill(signal_thread, SIGTERM)指名道姓的给signal_thread线程发送了SIGTERM信号，这样信号就不可能被别的线程提前拦截处理了。

````c++
// client/mysqladmin.cc execute_commands()
case ADMIN_SHUTDOWN:
{
...

      /* Issue COM_SHUTDOWN if server version is older then 5.7*/
      int resShutdown= 1;
      if(mysql_get_server_version(mysql) < 50709)
        resShutdown= mysql_shutdown(mysql, SHUTDOWN_DEFAULT);
      else
        resShutdown= mysql_query(mysql, "shutdown");
  
// libmysql/libmysql.c mysql_shutdown()
int STDCALL
mysql_shutdown(MYSQL *mysql, enum mysql_enum_shutdown_level shutdown_level)
{
  DBUG_ENTER("mysql_shutdown");
...
  DBUG_RETURN(mysql_real_query(mysql, C_STRING_WITH_LEN("shutdown")));
}
````

PS.

也可以使用下列命令停MySQL

````
kill -USR1 `pid of mysqld`
````


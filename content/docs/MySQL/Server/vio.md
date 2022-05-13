---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

# VIO模块

为不同的protocol提供network I/O wrapper，类似于winsock。

{{< hint info >}}
Virtual I/O Library.

The VIO routines are wrappers for the various network I/O calls that happen with different protocols. The idea is that in the main modules one won't have to write separate bits of code for each protocol. Thus vio's purpose is somewhat like the purpose of Microsoft's winsock library.
https://dev.mysql.com/doc/internals/en/vio-directory.html
{{</hint>}}

SOCKET封装了socket fd，里面只有fd信息。

目前支持的protocol：

- TCP/IP
- Unix domain socket
- Named Pipes（Windows only）
- Shared Memory（Windows only）
- Secure Sockets（SSL）

{{< hint warning>}}

Unix domain socket

用于实现同一主机上的进程间通信，即使用socket文件来进行通信。socket原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX domain socket。虽然网络socket也可用于同一台主机的进程间通讯(通过loopback地址127.0.0.1)，但是UNIX domain socket用于IPC 更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。

使用方式：

socket(AF_UNIX, ...){{</hint>}}

# 文件组织

````
include/violite.h       VIO lite（封装了VIO的struct和对外的function）
vio_priv.h              VIO模块内部头文件
vio.c                   Declarations + open/close functions
viosocket.c             Send/retrieve functions
viossl.c                SSL variations
viosslfactories.c       Certification / Verification
viopipe.c               named pipe implemenation（windows only）
vioshm.c                shared memory implementation（windows only）
````

文件的层次关系和用途如下图所示：

![MySQL_VIO_file_layout](/MySQL_VIO_file_layout.png)

# 设计

## 3.1 I/O事件

网络I/O事件分为读、写、连接：

````
  VIO_IO_EVENT_CONNECT    IN
  VIO_IO_EVENT_READ       IN
  VIO_IO_EVENT_WRITE      OUT
````

## VIO对象

对于建链后的连接，两端都分别分配一个VIO对象，用于描述和处理网络I/O。

client和server支持的连接方式和属性：

````
client
Unix domain socket                VIO_TYPE_SOCKET         VIO_LOCALHOST | VIO_BUFFERED_READ
TCP/IP                            VIO_TYPE_TCPIP          VIO_BUFFERED_READ
Shared Memory                     VIO_TYPE_SHARED_MEMORY  VIO_LOCALHOST
Named Pipes                       VIO_TYPE_NAMEDPIPE      VIO_LOCALHOST

server
Channel_info_local_socket         VIO_TYPE_SOCKET         VIO_LOCALHOST
Channel_info_tcpip_socket         VIO_TYPE_TCPIP
````

VIO对象的数据结构

````c++
struct st_vio
{
  MYSQL_SOCKET  mysql_socket;           // socket fd (TCP/IP and Unix domain socket)
  my_bool       localhost;              // VIO_LOCALHOST
  struct sockaddr_storage   local;      // local internet address
  struct sockaddr_storage   remote;     // remote internet address
  size_t addrLen;                       // remote address len
  enum enum_vio_type    type;           // protocol type 
  my_bool               inactive;       // 连接不再活跃（已关闭）

  char                  desc[VIO_DESCRIPTION_SIZE]; // debug
  char                  *read_buffer;   // buffer for vio_read_buff 16k
  char                  *read_pos;      // start of unfetched data in the read buffer
  char                  *read_end;      // end of unfetched data
  int                   read_timeout;   // 读超时
  int                   write_timeout;  // 写超时
  
  // 通过vtable interface来对不同协议下的实现进行解耦（多态）
  /* 
     viodelete is responsible for cleaning up the VIO object by freeing 
     internal buffers, closing descriptors, handles. 
  */
  void    (*viodelete)(Vio*);
  int     (*vioerrno)(Vio*);
  size_t  (*read)(Vio*, uchar *, size_t);
  size_t  (*write)(Vio*, const uchar *, size_t);
  int     (*timeout)(Vio*, uint, my_bool);
  int     (*viokeepalive)(Vio*, my_bool);
  int     (*fastsend)(Vio*);
  my_bool (*peer_addr)(Vio*, char *, uint16*, size_t);
  void    (*in_addr)(Vio*, struct sockaddr_storage*);
  my_bool (*should_retry)(Vio*);
  my_bool (*was_timeout)(Vio*);
  /* 
     vioshutdown is resposnible to shutdown/close the channel, so that no 
     further communications can take place, however any related buffers,
     descriptors, handles can remain valid after a shutdown.
  */
  int     (*vioshutdown)(Vio*);
  my_bool (*is_connected)(Vio*);
  my_bool (*has_data) (Vio*);
  int (*io_wait)(Vio*, enum enum_vio_io_event, int);
  my_bool (*connect)(Vio*, struct sockaddr *, socklen_t, int);

};
````

VIO对象的创建、初始化和销毁：

![MySQL_VIO_connection_shutdown](/MySQL_VIO_connection_shutdown.png)

## 常用函数

VIO模块中的常用函数如下：

````
int vio_io_wait       // 等待网络I/O时间通知（poll/select）

vio_read              // io读
vio_read_buff         // io缓冲读
vio_write             // io写

vio_fastsend          // 尽可能设为TCP_NODELAY
vio_keepalive         // 尽可能设置TCP保活 SO_KEEPALIVE
vio_timeout           // 设置超时
vio_should_retry      // 是否需要重试read/write（SOCKET_EINTR）
vio_was_timeout       // 是否超时（SOCKET_ETIMEDOUT）
vio_is_connected      // 是否处于连接中，通过vio_io_ait尝试read/write，或者通过socket_peek_read看是否有可读的数据
vio_pending           // 获得缓冲区还有多少数据可以读 ioctl(FIONREAD) debug使用
vio_description       // 打印socket/TCP信息 vio->desc debug使用
vio_type              // 获得连接的protocol类型
vio_errno             // 获取socket error
vio_fd                // 获得socket fd
vio_socket_connect    // client发起connect
vio_set_blocking      // 设置socket fd为阻塞/非阻塞
vio_getnameinfo       // 获得socket可读信息
vio_peer_addr         // 获得远程socket的可读地址
vio_get_normalized_ip_string  // 获得socket可读信息
vio_is_no_name_error  // 是否错误为EAI_NONAME - Neither nodename nor servname provided, or not known. 即Name resolution error. Your hostname is invalid. It's not resolving through DNS to any network location.
````

## PSI观测

MySQL通过PSI机制观测一些关键路径和节点，在处理网络I/O时，其中的观测点是等待网络读写事件。

入口是

````
int vio_io_wait       // 等待网络I/O时间通知（poll/select）
````

通过PSI提供状态的监测。

````
宏
1. 定义变量
  #define MYSQL_SOCKET_WAIT_VARIABLES(LOCKER, STATE) \
    struct PSI_socket_locker* LOCKER; \
    PSI_socket_locker_state STATE;

2. 进入socket等待
  #define MYSQL_START_SOCKET_WAIT(LOCKER, STATE, SOCKET, OP, COUNT) \
    LOCKER= inline_mysql_start_socket_wait(STATE, SOCKET, OP, COUNT,\
                                           __FILE__, __LINE__)
3. 结束socket等待
  #define MYSQL_END_SOCKET_WAIT(LOCKER, COUNT) \
    inline_mysql_end_socket_wait(LOCKER, COUNT)
````

使用

````
MYSQL_SOCKET_WAIT_VARIABLES(locker, state)    // 定义socket locker和locker state
MYSQL_START_SOCKET_WAIT(locker, &state, vio->mysql_socket, PSI_SOCKET_SELECT, 0);
poll/select...
MYSQL_END_SOCKET_WAIT(locker, 0);
````

其中函数指针的管理如下：

````
start_socket_wait_v1_t start_socket_wait;
在pfs.cc中的PFS_v1注册
pfs_start_socket_wait_v1

end_socket_wait_v1_t end_socket_wait;
在pfs.cc中的PFS_v1注册
pfs_end_socket_wait_v1
````

# 参考链接

https://dev.mysql.com/doc/internals/en/vio-directory.html
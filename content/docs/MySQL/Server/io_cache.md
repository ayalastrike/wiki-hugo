设计理念：

1. 对于碎片化的IO会减少实际文件IO读写
2. 写缓冲：对于写，可以合并IO操作，并在写缓冲满时自动调用write写入文件
3. 读缓冲：对于读，多余读出来的数据可以给下次读使用
4. 对于文件设备支持字符设备和块设备
5. 支持多种IO模式，比如缓冲IO/direct IO/aio
6. 对文件操作进行了抽象：4k对齐（按照块粒度读取IO效率最高），缓冲封装在文件操作内部

**IO_CACHE初始化**

````c++
// 1. seek 如果文件不支持seek，则seek_offset要求为0
// 2. 设置最小对齐块min_cache（16k/8k）
// 3. 缩小cachesize：READ_CACHE/SEQ_READ_APPEND下，如果读取范围足够小，则重置cachesize为该范围，并关闭aio
// 4. cachesize按照min_cache对齐，如果为SEQ_READ_APPEND，则申请2块，malloc如果失败则缩小3/4重试(128k->96k)，少于等于min_cache则报错返回
  
int init_io_cache_ext(IO_CACHE *info, File file, size_t cachesize,
                      enum cache_type type, my_off_t seek_offset,
                      pbool use_async_io, myf cache_myflags,
                      PSI_file_key file_key)
{
  size_t min_cache;
  my_off_t pos;
  my_off_t end_of_file= ~(my_off_t) 0;
  
  info->file= file;
  info->file_key= file_key;
  info->type= TYPE_NOT_SET;                               // 直到创建append_buffer_lock后才设置type
  info->pos_in_file= seek_offset;
  info->pre_close = info->pre_read = info->post_read = 0;
  info->arg = 0;
  info->alloced_buffer = 0;
  info->buffer=0;
  info->seek_not_done= 0;
  
  if (file >= 0)
  {
    pos= mysql_file_tell(file, MYF(0));                   // 定位文件位置
    if ((pos == (my_off_t) -1) && (my_errno() == ESPIPE)) // 文件不支持seek，再试一次，如果还不行就失败，这种情况下传入的seek_offset要求为0
    {
      info->seek_not_done= 0;
      DBUG_ASSERT(seek_offset == 0);
    }
    else
      info->seek_not_done= MY_TEST(seek_offset != pos);
  }
  
  info->disk_writes= 0;
  info->share=0;
  
  min_cache=use_async_io ? IO_SIZE*4 : IO_SIZE*2;         // 设置最小对齐块（aio使用16k，否则使用8k）
  
  if (type == READ_CACHE || type == SEQ_READ_APPEND)
  {
    // 如果读取范围足够小（end-seek），则将cachesize设小一点，并且不使用aio
    if (!(cache_myflags & MY_DONT_CHECK_FILESIZE))        // 假设文件不会增长
    {
      end_of_file= mysql_file_seek(file, 0L, MY_SEEK_END, MYF(0));// 计算文件大小(end)
      /* Need to reset seek_not_done now that we just did a seek. */
      info->seek_not_done= end_of_file == seek_offset ? 0 : 1;    // 文件大小和查找位置不相同，设置为seek_not_done
      if (end_of_file < seek_offset)                              // 查找位置 > 文件大小，end = seek
        end_of_file=seek_offset;
      if ((my_off_t) cachesize > end_of_file-seek_offset+IO_SIZE*2-1)
      {
        cachesize= (size_t) (end_of_file-seek_offset)+IO_SIZE*2-1;
        use_async_io=0;    /* No need to use async */
      }
    }
  }
  cache_myflags &= ~MY_DONT_CHECK_FILESIZE;
  if (type != READ_NET && type != WRITE_NET)
  {
    cachesize= ((cachesize + min_cache-1) & ~(min_cache-1));       // cachesize按照min_cache对齐
    for (;;)
    {
      size_t buffer_block;
      /*
        Unset MY_WAIT_IF_FULL bit if it is set, to prevent conflict with
        MY_ZEROFILL.
      */
      myf flags= (myf) (cache_myflags & ~(MY_WME | MY_WAIT_IF_FULL));
  
      if (cachesize < min_cache)
        cachesize = min_cache;                                      // 最小为一个min_cache
      buffer_block= cachesize;
      if (type == SEQ_READ_APPEND)                                  // 申请2块
        buffer_block *= 2;
      if (cachesize == min_cache)
        flags|= (myf) MY_WME;
  
      if ((info->buffer= (uchar*) my_malloc(key_memory_IO_CACHE, buffer_block, flags)) != 0)
      {
        info->write_buffer=info->buffer;
        if (type == SEQ_READ_APPEND)                                // write_buffer指向第二块
          info->write_buffer = info->buffer + cachesize;
        info->alloced_buffer=1;                                     // 标记已malloc
        break;
      }
      if (cachesize == min_cache)
        DBUG_RETURN(2);                                             // 最小申请容量为min_cache，如果还失败就报错返回
  
      cachesize= (cachesize*3/4 & ~(min_cache-1));                  // 如果malloc失败，缩小cachesize到原大小的3/4，直到申请成功
    }
  }
  
  info->read_length=info->buffer_length=cachesize;
  info->myflags=cache_myflags & ~(MY_NABP | MY_FNABP);
  info->request_pos= info->read_pos= info->write_pos = info->buffer;
  if (type == SEQ_READ_APPEND)
  {
    info->append_read_pos = info->write_pos = info->write_buffer;
    info->write_end = info->write_buffer + info->buffer_length;
    mysql_mutex_init(key_IO_CACHE_append_buffer_lock, &info->append_buffer_lock, MY_MUTEX_INIT_FAST);
  }
  
  if (type == WRITE_CACHE)
    info->write_end= info->buffer+info->buffer_length- (seek_offset & (IO_SIZE-1));
  else
    info->read_end=info->buffer;                                     // 初始状态下, 读缓冲为空
  
  /* End_of_file may be changed by user later */
  
  info->end_of_file= end_of_file;
  info->error=0;
  info->type= type;
  init_functions(info); // 初始化读写函数read_func/write_func
  DBUG_RETURN(0);
}
  
static void init_functions(IO_CACHE* info)
{
  enum cache_type type= info->type;
  switch (type) {
  case READ_NET:
    break;
  case SEQ_READ_APPEND:
    info->read_function = _my_b_seq_read;
    info->write_function = 0;         // 若使用出core
    break;
  default:
    info->read_function = info->share ? _my_b_read_r : _my_b_read;
    info->write_function = _my_b_write;
  }
  
  setup_io_cache(info);
}
````

**IO_CACHE读取**

````c++
读取流程：
1. 现从读缓冲中读取所有缓冲，拷贝到用户缓冲区
2. 计算文件读写位置：实际读取位置为 pos_infile + buffer，并计算该位置相对于4k对齐的余量（余数diff）
3. 如果读取大于8k，从实际文件中读取（只读取到4k对齐的位置，余量交由#4进行），并直接拷贝到用户缓冲区
4. 从文件读取<=缓冲区大小的块（1. 未4k对齐的，读取4k-diff大小 2. 如果文件剩余大小小于缓冲区，一次性读取全部内容），放入读缓冲中
5. 从读缓冲中拷贝用户需要的剩余字节给用户缓冲区，并推进read_pos和pos_in_file
int _my_b_read[IO_CACHE *info, uchar *Buffer, size_t Count)
{
  size_t length;
  size_t diff_length;
  size_t left_length;
  size_t max_length;
  my_off_t pos_in_file;
  
  /* If the buffer is not empty yet, copy what is available. */
  if ((left_length= (size_t) (info->read_end-info->read_pos)))  // 如果读缓冲有数据可读（end-pos），则将现有缓冲区的数据拷贝给用户
  {
    memcpy(Buffer,info->read_pos, left_length);
    Buffer+=left_length;
    Count-=left_length;
  }
  
  pos_in_file=info->pos_in_file+ (size_t) (info->read_end - info->buffer); 计算读取的位置 pos = 文件pos - 读缓冲长度，以保证文件pos始终和读缓冲起始点对齐
  
  // 通过设置seek_not_done设为1暗示在IO_CACHE上进行过针对磁盘的flush/write操作，所以需要重新定位
  if (info->seek_not_done)
  {
    if ((mysql_file_seek(info->file, pos_in_file, MY_SEEK_SET, MYF(0)) != MY_FILEPOS_ERROR))
    {
      info->seek_not_done= 0; //如果没有错误，设置为seek已完成
    }
    else
    {
      // 如果seek失败，且错误号为ESPIPE，则意味着文件为pipe/socket/FIFO，不再重试，直接返回错误
      DBUG_ASSERT(my_errno() != ESPIPE);
      info->error= -1;
      DBUG_RETURN(1);
    }
  }
  
  diff_length= (size_t) (pos_in_file & (IO_SIZE-1));  // 如果之前读取到一半中断过，则找到IO_SIZE对齐后的多余字节(diff)
  
  // 如果要读取的大小(Count+diff)超过2个IO_SIZE，读取文件，并且不填充读缓冲
  if (Count >= (size_t) (IO_SIZE+(IO_SIZE-diff_length)))
  {         /* Fill first intern buffer */
    size_t read_length;
    if (info->end_of_file <= pos_in_file) // 如果读取点在文件尾部之外，则返回错误，并通过error告知读取到的部分内容
    {
      info->error= (int) left_length;
      DBUG_RETURN(1);
    }
    // 计算按照IO_SIZE对齐后需要读取的大小（减去多余字节），进行实际的文件读取，并进行4k对齐
    length=(Count & (size_t) ~(IO_SIZE-1))-diff_length;
    if ((read_length= mysql_file_read(info->file, Buffer, length, info->myflags)) != length)
    {
      /*
        If we didn't get, what we wanted, we either return -1 for a read
        error, or (it's end of file), how much we got in total.
      */
      // 返回-1或者部分读到的数据
      info->error= (read_length == (size_t) -1 ? -1 : (int) (read_length+left_length));
      DBUG_RETURN(1);
    }
    Count-=length;
    Buffer+=length;
    pos_in_file+=length;
    left_length+=length;
    diff_length=0;
  }
  
  // 填充读缓冲
  // 计算除4k对齐外还需要读取的字节数 = 读缓冲长度-对齐外剩余字节（diff）
  max_length= info->read_length-diff_length;
  // 如果文件剩余长度小于读缓冲，则读取文件所有剩余长度
  if (info->type != READ_FIFO && max_length > (info->end_of_file - pos_in_file))
    max_length= (size_t) (info->end_of_file - pos_in_file);
  
  // 如果文件无剩余内容可读，则返回错误
  if (!max_length)
  {
    if (Count)
    {
      /* We couldn't fulfil the request. Return, how much we got. */
      info->error= (int)left_length;
      DBUG_RETURN(1);
    }
    length=0;
  } else if ((length= mysql_file_read(info->file, info->buffer, max_length, info->myflags)) < Count ||
     length == (size_t) -1) // 遇到读取错误，没有读到所需的大小（Count）或者读到EOF（-1）
  {
    /*
      We got an read error, or less than requested (end of file).
      If not a read error, copy, what we got.
    */
    if (length != (size_t) -1)          // 遇到读取错误，将已读到的内容拷贝给用户Buffer
      memcpy(Buffer, info->buffer, length);
    info->pos_in_file= pos_in_file;     // 设置文件偏移量
    info->error= length == (size_t) -1 ? -1 : (int) (length+left_length); // error设置为-1（EOF）或已读到的字节数
    info->read_pos=info->read_end=info->buffer; // 读缓冲重置为空
    DBUG_RETURN(1);
  }
  
  info->read_pos=info->buffer+Count;
  info->read_end=info->buffer+length;
  info->pos_in_file=pos_in_file;
  memcpy(Buffer, info->buffer, Count);
  DBUG_RETURN(0);
}
````
MySQL通过5个步骤来设置read_only：

1. 获取global read lock，阻塞所有的写入请求
2. flush opened table cache，阻塞所有的显式读写锁
3. 获取commit lock，阻塞commit写入binlog
4. 设置read_only=1
5. 释放global read lock和commit lock

设置read_only阻塞用户写入，但只能阻塞普通用户的更新，MySQL为了最大可能的保护数据一致性，增强了read_only功能，通过设置super read only，阻塞除了slave线程以外以为的所有用户的写入，包括super用户。
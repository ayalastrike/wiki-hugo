是否使用索引对性能影响很大。在数据库运维过程中，索引的删除总是慎重的，一方面是因为实际的DDL变更代价，一方面是评估对性能的影响。

为了减少删除索引的代价，MySQL 8.0引入了invisible index，用于解决这个问题。

即在索引上增加一个invisible的属性，用于optimizer可以将其忽略掉。

从这里也可以看到，引入的带动在server层，不涉及到存储引擎，所以所有的存储引擎都适用。

MySQL的实现基本参考的是Oracle 12c。

{{< hint info >}}

**MySQL 8.0.0 Release Notes**

- Optimizer Notes
  - MySQL now supports invisible indexes. An invisible index is not used by the optimizer at all, but is otherwise maintained normally. Indexes are visible by default. Invisible indexes make it possible to test the effect of removing an index on query performance, without making a destructive change that must be undone should the index turn out to be required. This feature applies to `InnoDB` tables, for indexes other than primary keys.
  - To control whether an index is invisible explicitly for a new index, use a `VISIBLE` or`INVISIBLE` keyword as part of the index definition for [`CREATE TABLE`](https://dev.mysql.com/doc/refman/8.0/en/create-table.html), [`CREATE INDEX`](https://dev.mysql.com/doc/refman/8.0/en/create-index.html), or [`ALTER TABLE`](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html). To alter the invisibility of an existing index, use a `VISIBLE` or `INVISIBLE` keyword with the `ALTER TABLE ... ALTER INDEX` operation. For more information, see [Invisible Indexes](https://dev.mysql.com/doc/refman/8.0/en/invisible-indexes.html).

{{</hint>}}

{{< hint info >}}

**MySQL 8.0.1 Release Notes**

- Optimizer Notes
  - Previously, invisible indexes were supported only for the `InnoDB` storage engine. Invisible indexes are now storage engine neutral (supported for any engine). (Bug #23541244)
- Bugs Fixed
  - Index hints applied to invisible indexes produced no error. (Bug #24660093, [Bug #82960](https://bugs.mysql.com/bug.php?id=82960))

{{</hint>}}

{{< hint info >}}

**MySQL 8.0.11 Release Notes**

- Bugs Fixed
  - Row-based replication used the wrong set of indexes on the slave. ([Bug #88847](https://bugs.mysql.com/bug.php?id=88847), Bug #27244826)

{{</hint>}}

WorkLog:

- [WL#8697 Support for INVISIBLE indexes](https://dev.mysql.com/worklog/task/?id=8697)
- [WL#10891: optimizer_switch to see invisible indexes](https://dev.mysql.com/worklog/task/?id=10891)

MySQL Server团队、PERCONA和阿里数据库团队的解读

- [MySQL 8.0: Invisible Indexes](http://mysqlserverteam.com/mysql-8-0-invisible-indexes/)
- [Thoughts on MySQL 8.0 Invisible Indexes](https://www.percona.com/blog/2016/10/27/thoughts-mysql-8-invisible-indexes/)
- [AliSQL · 特性介绍 · 支持 Invisible Indexes](http://mysql.taobao.org/monthly/2017/07/03/)

其中背后的思想是以下两点：

- soft delete
- staged rollout

应用场景主要有：

- 线上的产品库数据规模上不同于线下库，数据访问方式可能不同，这对于索引评估至关重要
- 可以立即看到效果（ALTER TABLE algorithm = replace）
- 只有某个会话想用到某个索引
- 使用FORCE INDEX的风险：如果没有索引命中，可能会导致全表扫描的高昂代价

在MySQL 8.0中元信息不再存储在frm里，而在backport到5.7时需要处理frm，阿里的做法是复用了一个不存在frm里的flag HA_SORT_ALLOWS_SAME，我们是采用了一个D flag来标记。

backport changes：

````
1. 增加配置项 reset_frm_visibility_enabled
sql/mysqld.h
sql/mysqld.cc
sql/sys_vars.cc
在开启enable下扫描解析frm
sql/mysqld.cc
 
 
2. frm操作
增加
sql/sql_reset_index_visibility.h
sql/sql_reset_index_visibility.cc
修改
sql/CMakeLists.txt
 
扫描：find_all_frms
reset（兼容性）：reset_frm_index  [128] == D set to d
 
只扫描除了mysql/performance_schema/sys之外的目录
系统目录全集：https://dev.mysql.com/doc/refman/5.7/en/data-directory.html
 
3. 词法
sql/lex.h
VISIBLE
INVISIBLE
 
4. 语法
sql/share/
ER_PK_INDEX_CANT_BE_INVISIBLE
sql/sql_yacc.yy
sql/sql_alter.h
sql/sql_alter.cc
sql/sql_table.cc
 
          alter_commands
          {
            THD *thd= YYTHD;
            LEX *lex= thd->lex;
            if (!lex->m_sql_cmd)
            {
              /* Create a generic ALTER TABLE statment. */
              lex->m_sql_cmd= new (thd->mem_root) Sql_cmd_alter_table();
              if (lex->m_sql_cmd == NULL)
                MYSQL_YYABORT;
            }
          }
 
alter_commands
alter_command_list
alter_list
alter_list_item
 
keywords
    VISIBLE_SYM
    INVISIBLE_SYM
 
key_visibility
 
%type <NONE>
       key_visibility
END_OF_INPUT
 
key_visibility:
          VISIBLE_SYM { Lex->key_create_info.is_visible= true; }
        | INVISIBLE_SYM { Lex->key_create_info.is_visible= false; }
 
key algo
 
normal_key_options:
          /* empty */ {}
        | normal_key_opts
        | key_visibility
        ;
 
fulltext_key_options:
          /* empty */ {}
        | fulltext_key_opts
        | key_visibility
        ;
 
spatial_key_options:
          /* empty */ {}
        | spatial_key_opts
        | key_visibility
        ;
 
ALTER TABLE ALTER INDEX VISIBLE/INVISIBLE
        | ALTER INDEX_SYM ident key_visibility
          {
            LEX *lex= Lex;
            Alter_index_visibility *aiv= new Alter_index_visibility($3.str, lex->key_create_info.is_visible);
            if (aiv == NULL)
              MYSQL_YYABORT;
            lex->alter_info.alter_index_visibility_list.push_back(aiv);
            lex->alter_info.flags|= Alter_info::ALTER_INDEX_VISIBILITY;
          }
 
增加Alter_index_visibility类，并在Alloc_Info中配置为成员变量, ctor reset()
并实现ctor
 
PK不能设置invisible
create index
    add_create_index()
 
5. key_create_info增加is_visible属性
sql/handler.h   bool is_visible
sql/handler.cc  default_key_create_info
 
6. 在alter table时设置new_key.is_visible，把old_key，new_key放入index_altered_visibility_buffer的KEY_PAIR
+ Alter_inplace_info.add_altered_index_visibility()
+uint index_altered_visibility_count
+KEY_PAIR *index_altered_visibility_buffer
 
mysql_alter_table
    fill_alter_inplace_info
 
7. 读取frm文件中的key visible flag
sql/unireg.cc
pack_keys_visib
mysql_create_frm
sql/table.cc
open_binary_frm
 
8. 在表元信息中增加变量和方法
在TABLE_SHARE中增加key_map visible_indexes bit(1)
当开启thd.OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES，返回交集
+TABLE_SHARE::usable_indexes
sql/table.h
sql/sql_tmp_table.cc
sql/table.cc
 
9. SQL优化器调用usable_indexes
TABLE_LIST::process_index_hints
赋值、判断
setup_ftfuncs
    Item_func_match::fix_index()
Rows_log_event::decide_row_lookup_algorithm_and_key()
    search_key_in_table
sql/table.cc
sql/item_func.h
sql/item_func.cc
sql/log_event.cc
storage/myisam/ha_myisam.cc
更新setup_ftfuncs的调用，增加thd参数
 
sql_command接入
sql/sql_base.h
sql/sql_base.cc
sql/sql_update.cc
sql/sql_delete.cc
sql/sql_resolver.cc
setup_ftfuncs
    fix_index
 
10. key增加is_visible属性
标记是否对query optimizer可见      OPTIMIZER_SWITCH_USE_INVISIBLE_INDEXES
show create table
show table status
mysql_prepare_create_table
+check_promoted_index
sql/sql_const.h
sql/key.h
sql/sql_show.cc
sql/sql_table.cc
 
11. DDL
mysql_alter_Table
    fill_alter_inplace_info
分配ha_alter_info->index_altered_visibility_buffer
主键不允许设置invisible index
mysql_prepare_alter_table
 
12. 增加优化器优化开关
optimizer_switch_names
https://github.com/xiaoboluo768/qianjinliangfang/wiki/optimizer_switch
sql/sys_vars.cc
````


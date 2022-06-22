# 1、数据的导入导出

前言
PostgreSQL 使用 pg_dump 和 pg_dumpall 进行数据库的逻辑备份，使用 pg_restore 导入数据，pg_dumpall 是对整个数据库集群进行备份，pg_dump 可以选择一个数据库或者部分表进行备份。

## 关于 pg_dump

pg_dump 将表结构及数据以 SQL 语句的形式导出到 sql 文件或其他格式文件，恢复数据时，将导出的文件作为输入，执行其中的 SQL 语句，即可恢复数据。

pg_dump 能够对正在使用的 PostgreSQL 数据库进行备份，并且不影响正常业务的读写。

pg_dump 是一个客户端工具，可以远程或本地导出逻辑数据，恢复数据至导出时间点。

pg_dump 一次只转储一个数据库，并不会转储有关角色或表空间的信息 (因为那些是群集范围而不是每个数据库)。

## 关于 pg_dumpall

如果要备份 Cluster 中数据库共有的全局对象，例如角色和表空间，需要使用 pg_dumpall。

备份文件以文本或存档文件格式输出。

Script dumps 是一个普通文本文件，包含将数据库重构到保存时的状态所需的 SQL 命令。

pg_dumpall 在给定的群集中备份每个数据库, 并保留群集范围内的数据, 如角色和表空间定义。

## pg_dump 常用参数

```shell
-h host，指定数据库主机名，或者IP
-p port，指定端口号
-U user，指定连接使用的用户名
-W，按提示输入密码
dbname，指定连接的数据库名称，实际上也是要备份的数据库名称。
-a，–data-only，只导出数据，不导出表结构
-c，–clean，是否生成清理该数据库对象的语句，比如drop table
-C，–create，是否输出一条创建数据库语句
-f file，–file=file，输出到指定文件中
-n schema，–schema=schema，只转存匹配schema的模式内容
-N schema，–exclude-schema=schema，不转存匹配schema的模式内容
-O，–no-owner，不设置导出对象的所有权
-s，–schema-only，只导致对象定义模式，不导出数据
-t table，–table=table，只转存匹配到的表，视图，序列，可以使用多个-t匹配多个表
-T table，–exclude-table=table，不转存匹配到的表。
--inserts，使用insert命令形式导出数据，这种方式比默认的copy方式慢很多，但是可用于将数据导入到非PostgreSQL数据库。
–column-inserts，导出的数据，有显式列名
```

例如：

导出一张表：

```shell
pg_dump -h 127.0.0.1 -U postgres -p 5432 -W htmesjk -t collect_data --inserts > collect_data.sql
```

导出多张表：

```shell
pg_dump -h 127.0.0.1 -U admin -p 5432 -W db -t t1 -t t2 –inserts > bak.sql
```

导出整个数据库：

```shell
pg_dump -h 127.0.0.1 -U admin -p 5432 -W db –inserts > bak.sql
```

只导出表结构，不导出数据：

```shell
pg_dump -h 127.0.0.1 -U admin -p 5432 -W db -s > bak.sql
```

只导出数据，不导出表结构：

```shell
pg_dump -h 127.0.0.1 -U admin -p 5432 -W db –inserts -a > bak.sql
```

关于导入：

```shell
psql -h 0.0.0.0  -d one -U postgres -p 5432 -f dump2022.sql 
```

# 2、数据库基础运维操作

## 2.1、权限管理

在表'dm.dmd_ps_mat_acctg_eval_class_data_his'上给presto角色授予select权限

```sql
GRANT SELECT ON TABLE dm.dmd_ps_mat_acctg_eval_class_data_his TO presto;
```

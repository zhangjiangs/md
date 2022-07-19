# MySQL 表数据的导入导出

database下的table的数据导出`mysqldump -uroot -p****** database table > /bak/insert.sql`

```sql
mysql -uroot -p******
use database;
source /bak/insert.sql
#将上面导出的表的数据导入到另外一个表里
#q
```

建立名为xuelang的账号，密码Xuel_2022，并可以远程登陆
`create user'xuelang'@'%' identified by'Xuel_2022'`
给账号xuelang赋予远程连接rddw数据中所有表的查询权限
`grant select on rddw.* to xuelang@'%';`


# 删除有外键约束的数据

```sql
select @@foreign_key_checks;
set foreign_key_checks = 1;

foreign_key_checks的值为1时不能删除数据，值为0时可以删除
```

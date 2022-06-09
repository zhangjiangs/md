# MySQL 表数据的导入导出

```
mysqldump -uroot -p****** database table > /bak/insert.sql
#database下的table的数据导出
```

```
mysql -uroot -p******
use database;
source /bak/insert.sql
#将上面导出的表的数据导入到另外一个表里
#q
```


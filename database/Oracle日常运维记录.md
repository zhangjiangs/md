# RMAN备份于还原

## 1、备份

备份需要将数据库至于归档模式

1。关闭数据库

```sql
SQL> shutdown immediate;
```

启动数据库到mount状态

```sql
SQL> startup mount;
```

3。启用归档模式

```sql
SQL> alter database archivelog;
```

开启数据库

```sql
SQL> alter database open;
```

备份脚本：

```sh
rman target / msglog /home/oracle/script/full_bak.log append<<EOF
run
{
allocate channel c00 type disk;
allocate channel c01 type disk;
#在run块中，前两个命令ALLOCATE CHANNEL，分配一个通道，会启动一个服务器进程。每个通道都需要名称（只是一个随意的字符串，本例是d1和d2），必须指定是使用磁带还是磁盘作为备份目标。启动多个通道，会启用备份的并行性。RMAN会把工作负载分布到通道上。
backup as compressed backupset incremental level 0 database format '/oracle/backup/fulldb_%d_%T_%s.bak';
#compressed backupset是RMAN的备份压缩技术
#incremental level 0是增量备份的基础
sql 'alter system archive log current';
sql 'alter system archive log current';
sql 'alter system archive log current';
sql 'alter system archive log current';
#将所有的归档都备份出来，这样做是为了保证数据的完整和一致。
backup as compressed backupset archivelog all format '/oracle/backup/arch_%d_%T_%s.bak' ;
backup current controlfile format '/oracle/backup/ctl_%d_%T_%s.bak';
crosscheck backup;
#allocate channel for maintenance type disk;
delete noprompt obsolete device type disk;
delete noprompt expired backup;
delete force noprompt archivelog until time 'sysdate-3';
crosscheck archivelog all;
release channel c00;
release channel c01;
}
EOF
```



## 2、还原

还原步骤：

1、关闭数据库，并启动到nomount状态

```
SQL> shutdown immediate; 
SQL> startup nomount;
```

2、还原控制文件

```
RMAN> restore controlfile from '/oracle/backup/ctl_dbname_20190616_17329.bak';
```

3、将数据库开启到mount状态

```
RMAN> alter database mount;
```

4、还原数据库

```
RMAN> restore database; 
```

5、recover database（恢复日志文件）

```
RMAN> recover database; 
```

6、开启数据库

```
RMAN> alter database open resetlogs; 

database opened
```

还原完成

# 表空间相关

## 1、查表空间使用率情况（含临时表空间）

```sql
SELECT d.tablespace_name "Name", d.status "Status", 
       TO_CHAR (NVL (a.BYTES / 1024 / 1024, 0), '99,999,990.90') "Size (M)",
          TO_CHAR (NVL (a.BYTES - NVL (f.BYTES, 0), 0) / 1024 / 1024,
                   '99999999.99'
                  ) USE,
       TO_CHAR (NVL ((a.BYTES - NVL (f.BYTES, 0)) / a.BYTES * 100, 0),
                '990.00'
               ) "Used %"
  FROM SYS.dba_tablespaces d,
       (SELECT   tablespace_name, SUM (BYTES) BYTES
            FROM dba_data_files
        GROUP BY tablespace_name) a,
       (SELECT   tablespace_name, SUM (BYTES) BYTES
            FROM dba_free_space
        GROUP BY tablespace_name) f
 WHERE d.tablespace_name = a.tablespace_name(+)
   AND d.tablespace_name = f.tablespace_name(+)
   AND NOT (d.extent_management LIKE 'LOCAL' AND d.CONTENTS LIKE 'TEMPORARY')
UNION ALL
SELECT d.tablespace_name "Name", d.status "Status", 
       TO_CHAR (NVL (a.BYTES / 1024 / 1024, 0), '99,999,990.90') "Size (M)",
          TO_CHAR (NVL (t.BYTES, 0) / 1024 / 1024, '99999999.99') USE,
       TO_CHAR (NVL (t.BYTES / a.BYTES * 100, 0), '990.00') "Used %"
  FROM SYS.dba_tablespaces d,
       (SELECT   tablespace_name, SUM (BYTES) BYTES
            FROM dba_temp_files
        GROUP BY tablespace_name) a,
       (SELECT   tablespace_name, SUM (bytes_cached) BYTES
            FROM v$temp_extent_pool
        GROUP BY tablespace_name) t
 WHERE d.tablespace_name = a.tablespace_name(+)
   AND d.tablespace_name = t.tablespace_name(+)
   AND d.extent_management LIKE 'LOCAL'
   AND d.CONTENTS LIKE 'TEMPORARY';
```

## 2、为临时表空间增加一个文件到相应目录：

```sql
SQL> ALTER TABLESPACE &tablespace_name ADD TEMPFILE '/oracle/oradata/dbname/temp02.dbf' SIZE 10G;
```

```
Enter value for tablespace_name: TEMP
old   1: ALTER TABLESPACE &tablespace_name ADD TEMPFILE '/oracle/oradata/dbname/temp02.dbf' SIZE 10G
new   1: ALTER TABLESPACE TEMP ADD TEMPFILE '/oracle/oradata/dbname/temp02.dbf' SIZE 10G

Tablespace altered.
```

```sql
SQL> select * from dba_temp_files;
```

```
FILE_NAME					   FILE_ID TABLESPACE_NAME		       BYTES	 BLOCKS STATUS		      RELATIVE_FNO AUTOEXTEN   MAXBYTES  MAXBLOCKS INCREMENT_BY USER_BYTES USER_BLOCKS
-------------------------------------------------- ------- ------------------------------ ---------- ---------- --------------------- ------------ --------- ---------- ---------- ------------ ---------- -----------
/oracle/oradata/dbname/temp01.dbf			 1 TEMP 			    30408704	   3712 ONLINE				 1 YES	     3.4360E+10    4194302	     80   29360128	  3584
/oracle/oradata/dbname/temp02.dbf			 2 TEMP 	
```

## 3、查看临时表文件存放位置：

```sql
SQL> select * from dba_temp_files;
```

```
FILE_NAME					   FILE_ID TABLESPACE_NAME		       BYTES	 BLOCKS STATUS		      RELATIVE_FNO AUTOEXTEN   MAXBYTES  MAXBLOCKS INCREMENT_BY USER_BYTES USER_BLOCKS
-------------------------------------------------- ------- ------------------------------ ---------- ---------- --------------------- ------------ --------- ---------- ---------- ------------ ---------- -----------
/oracle/oradata/dbname/temp01.dbf			 1 TEMP 			    30408704	   3712 ONLINE				 1 YES	     3.4360E+10    4194302	     80   29360128	  3584
/oracle/oradata/dbname/temp02.dbf			 2 TEMP 	
```

## 4、查看表空间是否有自动扩展能力

```sql
SELECT T.TABLESPACE_NAME,D.FILE_NAME,     
D.AUTOEXTENSIBLE,D.BYTES,D.MAXBYTES,D.STATUS     
FROM DBA_TABLESPACES T,DBA_DATA_FILES D     
WHERE T.TABLESPACE_NAME =D.TABLESPACE_NAME     
 ORDER BY TABLESPACE_NAME,FILE_NAME; 
```

## 5、查看表空间使用情况

```sql
SELECT UPPER(F.TABLESPACE_NAME) "表空间名",     
D.TOT_GROOTTE_MB "表空间大小(M)",     
D.TOT_GROOTTE_MB - F.TOTAL_BYTES "已使用空间(M)",     
TO_CHAR(ROUND((D.TOT_GROOTTE_MB - F.TOTAL_BYTES) / D.TOT_GROOTTE_MB * 100,2),'990.99') "使用比",     
F.TOTAL_BYTES "空闲空间(M)",     
F.MAX_BYTES "最大块(M)"    
FROM (SELECT TABLESPACE_NAME,     
ROUND(SUM(BYTES) / (1024 * 1024), 2) TOTAL_BYTES,     
ROUND(MAX(BYTES) / (1024 * 1024), 2) MAX_BYTES     
FROM SYS.DBA_FREE_SPACE     
GROUP BY TABLESPACE_NAME) F,     
(SELECT DD.TABLESPACE_NAME,     
ROUND(SUM(DD.BYTES) / (1024 * 1024), 2) TOT_GROOTTE_MB     
FROM SYS.DBA_DATA_FILES DD     
GROUP BY DD.TABLESPACE_NAME) D     
WHERE D.TABLESPACE_NAME = F.TABLESPACE_NAME     
ORDER BY 4 DESC;  
```

## 6、增加一个临时表文件并设置为可以自动增长

```sql
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/dbname/temp02.dbf' SIZE 20G AUTOEXTEND ON NEXT 1G MAXSIZE 30G;
```

## 7、移动表空间文件从一个文件夹到另外一个文件夹

```sql
alter database rename file  '/opt/oracle/oradata/orcl/temp01.dbf' to '/data2/dbf/temp01.dbf';
```

## 8、删除临时表空间文件

```sql
alter database tempfile 2 drop including datafiles;     //2为FILE_ID
```

## 9、查询表空间文件路径：

```sql
select * from dba_data_files;
```

## 10、新建表空间文件：

```sql
create tablespace htmescs datafile '/oracle/oradata/htmescs/htmescs.dbf' size 2000M;
```

## 11. 查询表空间剩余字节大小

```sql
SELECT TABLESPACE_NAME, SUM(BYTES)/1024/1024 AS "FREE SPACE(M)"  FROM DBA_FREE_SPACE WHERE TABLESPACE_NAME = '&tablespace_name' GROUP BY TABLESPACE_NAME;

#注：如果是临时表空间，请查询DBA_TEMP_FREE_SPACE

SELECT TABLESPACE_NAME, FREE_SPACE/1024/1024 AS "FREE SPACE(M)"  FROM DBA_TEMP_FREE_SPACE WHERE TABLESPACE_NAME = '&tablespace_name';
```

## 12、查询表空间所有数据文件路径

```sql
SELECT TABLESPACE_NAME, FILE_ID, FILE_NAME, BYTES/1024/1024 AS "BYTES(M)"  FROM DBA_DATA_FILES WHERE TABLESPACE_NAME = '&tablespace_name';

#注：如果是临时表空间，请查询DBA_TEMP_FILES

SELECT TABLESPACE_NAME, FILE_ID, FILE_NAME, BYTES/1024/1024 AS "SPACE(M)"  FROM DBA_TEMP_FILES WHERE TABLESPACE_NAME = '&tablespace_name';
```

## 13、为空间不足的表空间增加数据文件

```sql
#查看表空间文件位置：
select  * from dba_data_files

#增加数据文件
ALTER TABLESPACE &tablespace_name ADD DATAFILE '&datafile_name' SIZE 2G;

#将users表空间文件设置为自动增长
alter database datafile '/usr/local/oracle/oradata/orcl/users03.dbf' autoextend on next 1G maxsize unlimited;

#查看该表中最大的表是什么：
select segment_name, sum(bytes/1024/1024/1024)"GB" from dba_segments where tablespace_name='USERS' group by segment_name order by 2 desc;

#注：如果要为临时表空间扩容，使用下面的语句
ALTER TABLESPACE &tablespace_name ADD TEMPFILE '&datafile_name' SIZE 2G;

#查看临时表空间的大小 和 数据文件路径
SELECT TABLESPACE_NAME, FILE_ID, FILE_NAME, BYTES/1024/1024 AS "SPACE(M)"  FROM DBA_TEMP_FILES WHERE TABLESPACE_NAME = 'TEMP';
#或者
select name, bytes/1024/1024 as "大小(M)" from v$tempfile order by bytes;
```

## 14、重建并修改默认临时表空间办法：

```sql
#查询当前数据库默认临时表空间名 
select * from database_properties where property_name='DEFAULT_TEMP_TABLESPACE'; 
#创建新的临时表空间 
create temporary tablespace temp02  tempfile 'E:\oracle\oradata\lims\TEMP02.DBF' size 1024M autoextend on; 
#修改默认表空间为刚刚建立的临时表空间 
alter database default temporary tablespace temp02; 
#查看用户所用临时表空间的情况 
SELECT USERNAME,TEMPORARY_TABLESPACE FROM DBA_USERS; 
#删除原来的临时表空间 
drop tablespace temp including contents and datafiles; 
#查看所有表空间名确认临时表空间是否已删除 
select tablespace_name from dba_tablespaces; 
```



# 索引

## 1、创建索引

```sql
create index 索引名 on 表名(列名);
例如：
CREATE INDEX TIMESTAMP ON SAPSR3.FAGLFLEXA (TIMESTAMP)
```

## 2、删除索引

```sql
drop index 索引名
```

## 3、创建组合索引

```sql
create index 索引名 on 表名(列名1,,列名2);
```

## 4、查看已经有的索引

```sql
--在数据库中查找表名
select * from user_tables where table_name like 'tablename%';
```

```sql
--查看该表的所有索引
select * from all_indexes where table_name = 'tablename';
```

```sql
--查看该表的所有索引列
select * from all_ind_columns where table_name = 'tablename';
```

# 用户管理运维

## 1、创建用户并赋权

```sql
SQL> create user htmescs identified by htmescs;

User created.

SQL> grant dba, resource to htmescs;

Grant succeeded.
```

## 2、删除用户

```sql
#查询需要删除的用户的连接信息，比如K3CLOUD_HTJT用户
SQL> select username, sid, serial# from v$session where username='K3CLOUD_HTJT';

USERNAME			            SID        SERIAL#
------------------------------ ---------- ----------
K3CLOUD_HTJT			        62	        2531
K3CLOUD_HTJT			        64          42627
K3CLOUD_HTJT			        77          12637
K3CLOUD_HTJT			        78	        8939
K3CLOUD_HTJT			        109         20477
K3CLOUD_HTJT			        154         33619
K3CLOUD_HTJT			        166         22385
K3CLOUD_HTJT			        184         24641
K3CLOUD_HTJT			        379	        5151
K3CLOUD_HTJT			        393	        875
K3CLOUD_HTJT			        453         26041

#删除用户：其中'17,759'为SID和SERIAL的组合
alter system kill session'17,759';
alter system kill session'451,20185';

#查询用户状态：
select saddr,sid,serial#,paddr,username,status from v$session where username is not null;
0000000C58F87308	32	4161	0000000C58DE6650	KINGDEE	INACTIVE
0000000C58F84228	33	5693	0000000C58DC4F50	KINGDEE	INACTIVE
0000000C58FB1F48	48	4067	0000000C58DC6008	SYS	INACTIVE
0000000C59010A68	77	12637	0000000C58EF05E0	K3CLOUD_HTJT	KILLED
0000000C590693C8	108	7337	0000000C58DEB9E8	KINGDEE	INACTIVE
0000000C59126A08	166	22423	0000000C58DCE5C8	KINGDEE	INACTIVE
0000000C59151648	182	17841	0000000C58DF0D80	ORCL	ACTIVE
0000000C5914B488	184	24685	0000000C58DCF680	KINGDEE	INACTIVE
0000000C5917C288	198	1043	0000000C58DD0738	KINGDEE	INACTIVE
0000000C591A9FA8	213	47873	0000000C58EF05E0	K3CLOUD_HTJT	KILLED
0000000C591D4BE8	229	1021	0000000C58DD28A8	KINGDEE	INACTIVE

#当用户都是kill状态时就可以删除了
drop user K3CLOUD_HTJT cascade;

#如果出现不断的有新的用户连接信息，可以采用先将用户禁用，再逐个kill的方法：
#禁用用户：
alter user K3CLOUD_HTJT account lock;
```

## 3、出现用户锁定的解决方法

### 3.1 查明所有的用户哪些被锁了

```sql
SQL> select username,account_status,lock_date from dba_users;
```

### 3.2 查看某个用户(ABCuser这个用户)是否被锁：

```sql
select LOCK_DATE,username from dba_users where username='ABCecuser';
```

LOCK_DATE为空说明没有锁定，非空为锁定。

### 3.3 解锁方法

```sql
ALTER USER USER_NAME ACCOUNT UNLOCK;
```

一般数据库默认是10次尝试失败后锁住用户

查看FAILED_LOGIN_ATTEMPTS的值

```sql
SQL> select * from dba_profiles where resource_name like 'FAILED_LOGIN_ATTEMPTS%';
PROFILE RESOURCE_NAME RESOURCE
------------------------------ -------------------------------- --------
LIMIT
----------------------------------------
DEFAULT FAILED_LOGIN_ATTEMPTS PASSWORD
10
MONITORING_PROFILE FAILED_LOGIN_ATTEMPTS PASSWORD
UNLIMITED
```

修改为30次：

```sql
alter profile default limit FAILED_LOGIN_ATTEMPTS 30;
```

修改为无限次：（不建议）

```sql
alter profile default limit FAILED_LOGIN_ATTEMPTS unlimited;
```

### 3.4 密码过期

先查看数据库密码规则，一般默认是180天过期，defult为无限制

```sql
select * from dba_profiles where profile='DEFAULT' and RESOURCE_NAME like 'PASSWORD%';
```

如果有时间限制，修改 成无限制：

```sql
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

如果用户密码已经过期，则需要重置密码：（密码可以为原来的密码）

```sql
alter user userXXX identified by xxx; 
```

# 其他

## 1、格式化字符串

```sql
col MEMBER format a20
SET linesize 300
```

遇到字段显示#####的是因为：出错的这个字段本身是number数据类型，而之前做查询时候格式化的同名字段是varchar2，所以，做了列格式化后必然要出错。

执行

```
col GROUP# format 9999
```

## 2、导出AWR报告

```
@?/rdbms/admin/awrrpt.sql
```

```sql
SQL> @/oracle/sqlhc.sql

Parameter 1:
Oracle Pack license (Tuning or Diagnostics) [Y|N] (required)

Enter value for 1: y
PL/SQL procedure successfully completed.
Parameter 2:
SQL_ID of the SQL to be analyzed (required)

Enter value for 2: 086ghv3armk7v
```


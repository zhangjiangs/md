# 给用户授权

```grant ALL ON SCHEMA htzn_ext_dm to znkj_read;```

```grant select ON  ALL TABLES IN SCHEMA htzn_ext_dm to znkj_read;```

```ALTER SCHEMA "htzn_ext_dm" OWNER to znkj_read;```

```alter role znkj_read with createexttable;```

授权一个用户访问PXF
要使用PXF读取外部数据，请使用CREATE EXTERNAL TABLE命令创建一个外部表，该命令指定pxf协议。您必须向所有需要此类访问权限的非SUPERUSER的Greenplum数据库角色明确授予对pxf协议的SELECT权限。

要授予特定角色对pxf协议的访问权限，请使用GRANT命令。例如，要授予名为bill的角色对使用pxf协议创建的外部表引用的数据的读取访问权限，请执行以下操作：

```GRANT SELECT ON PROTOCOL pxf TO bill;```
要使用PXF将数据写入外部数据存储，请使用CREATE WRITABLE EXTERNAL TABLE命令创建一个外部表，该命令指定pxf协议。您必须向需要此类访问的所有非SUPERUSER的Greenplum数据库角色明确授予对pxf协议的INSERT权限。 例如：

```GRANT INSERT ON PROTOCOL pxf TO bill;```

```GRANT select,INSERT ON PROTOCOL pxf TO znkj_read;```

创建连接pxf的外部表

```sql
create external table htzn_ext_dm.dmd_flg_parse_c030
(
 mat_lot_no text,
 mat_cd text,
 mat_name text,
 fac_name text,
 uom text,
 equip_name text,
 mantn_tm timestamp,
 ppn_tmstamp text
)
LOCATION ('pxf://htzn_dm.dmd_flg_parse_c030?PROFILE=Hive&SERVER=hdp3')FORMAT 'CUSTOM' ( FORMATTER='pxfwritable_import' );
```
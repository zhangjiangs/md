# hive如何连接

## 1、过Kerberos认证

ssh 连接到hdp的client节点，进入到keytab目录下通过Kerberos认证
/home/dpadmin/keytabs

kinit -kt dpgroup.dpadmin.keytab dpadmin/dpgroup@PROD.HT.COM

## 2、进入hive

```beeline``` 进入hive命令行

```show databases;```

```use database;```

```show tables;```

```create database htzn_dm;```
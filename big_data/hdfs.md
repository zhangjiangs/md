# 1、连接hdfs

## 1.1 连接client节点

## 1.2 通过Kerberos认证

进入到keytab所在目录

正式环境：```/home/dpadmin/keytabs```

测试环境：```/etc/security/keytabs```

过认证：
```kinit -kt dpgroup.dpadmin.keytab dpadmin/dpgroup@TEST.HT.COM```
```kinit -kt dpgroup.dpadmin.keytab dpadmin/dpgroup@PROD.HT.COM```

## 1.3 使用hdfs命令操作

查看hdfs根目录下的文件：

```shell
[root@ht-hdp-client-1 keytabs]# hdfs dfs -ls /
Found 15 items
drwxrwxrwt   - yarn   hadoop          0 2021-05-27 16:43 /app-logs
drwxr-xr-x   - hdfs   hdfs            0 2021-03-08 18:07 /apps
drwxr-xr-x   - yarn   hadoop          0 2021-03-08 17:38 /ats
drwxr-xr-x   - hdfs   hdfs            0 2021-03-08 17:39 /atsv2
drwxr-xr-x   - hdfs   hdfs            0 2021-03-08 17:38 /hdp
drwx------   - livy   hdfs            0 2021-05-25 08:41 /livy2-recovery
drwxr-xr-x   - mapred hdfs            0 2021-03-08 17:38 /mapred
drwxrwxrwx   - mapred hadoop          0 2021-03-08 17:39 /mr-history
drwxr-xr-x   - hdfs   hdfs            0 2021-03-10 13:20 /ranger
drwxr-xr-x   - hdfs   hdfs            0 2021-03-08 17:39 /services
drwxrwxrwx   - spark  hadoop          0 2022-07-14 08:23 /spark2-history
drwxr-xr-x   - hdfs   hdfs            0 2021-06-09 15:04 /system
drwxrwxrwx   - hdfs   hdfs            0 2021-03-17 13:16 /tmp
drwxr-xr-x   - hdfs   hdfs            0 2022-07-13 09:55 /user
drwxr-xr-x   - hdfs   hdfs            0 2021-03-08 17:40 /warehouse
```

使用admin账号创建的租户的keytab文件在hdfs的```/user/dpadmin/keytab```目录下

```hadoop fs -get /user/dpadmin/keytabs/htzn.htzn.keytab```将文件下载到本地当前目录。可以在后面加本地指定路径




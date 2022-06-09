# ceph文件存储系统

Ceph文件系统或CephFS是在Ceph的分布式对象存储RADOS之上构建的POSIX兼容文件系统。
文件元数据与文件数据存储在单独的pool中，并通过可调整大小的元数据服务器或MDS集群提供服务，该集群可扩展以支持更高吞吐量的元数据工作负载。文件系统的客户端可以直接访问RADOS以读取和写入文件数据块。

![img](https://preview.cloud.189.cn/image/imageAction?param=54AEF77B7EC3AFA45031A5815220BCBA17DCEAAACBFFEF9F01747A65558A4FD1E924A10F1403076E15568CA79B9A784457A9DF98FF3DE8BB23421BC961741DD8818575361AB8C68AAD3C32415370D119A18E87F4D230D09D0915AAC9655C24CA49449549A7763540D4E81541B3B55DD8)

前提是安装了mds

## 一、创建文件系统

一个Ceph文件系统至少需要两个Pool，一个用于数据，另一个用于元数据。
数据与元数据都是存储到OSD里。

#### 创建pool

```
[root@test-master-1 ~]# ceph osd pool create cephfs_data 128 128
pool 'cephfs_data' created
[root@test-master-1 ~]# ceph osd pool create cephfs_metadata 32 32
pool 'cephfs_metadata' created
```

> 通常，元数据池最多具有几GB的数据。因此，通常建议使用较少的PG。实际上，大型群集通常使用64或128。

我这里只是测试就32了。

#### 创建文件系统

```
ceph fs new <fs_name> <metadata> <data>
```

```
[root@test-master-1 ~]# ceph fs new mycephfs cephfs_metadata cephfs_data
new fs with metadata pool 6 and data pool 5
```

#### 查看信息

```
[root@test-master-1 ~]# ceph mds stat
mycephfs:1 {0=test-master-1=up:active} 2 up:standby
```

主备模式， test-master-1为主。

状态信息：

```
[root@test-master-1 ~]# ceph fs status mycephfs
mycephfs - 0 clients
========
+------+--------+---------------+---------------+-------+-------+
| Rank | State  |      MDS      |    Activity   |  dns  |  inos |
+------+--------+---------------+---------------+-------+-------+
|  0   | active | test-master-1 | Reqs:    0 /s |   10  |   13  |
+------+--------+---------------+---------------+-------+-------+
+-----------------+----------+-------+-------+
|       Pool      |   type   |  used | avail |
+-----------------+----------+-------+-------+
| cephfs_metadata | metadata | 1536k | 54.9G |
|   cephfs_data   |   data   |    0  | 54.9G |
+-----------------+----------+-------+-------+
+---------------+
|  Standby MDS  |
+---------------+
| test-master-2 |
| test-master-3 |
+---------------+
MDS version: ceph version 14.2.22 (ca74598065096e6fcbd8433c8779a2be0c889351) nautilus (stable)
```

查询cephfs列表：

```
[root@test-master-1 ~]# ceph fs ls
name: mycephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

客户端授权：

```
[root@test-master-1 ~]# ceph auth get-or-create client.cephfs mon 'allow r' mds 'allow rw' osd 'allow rw pool=cephfs_data, allow rw pool=cephfs_metadata'
[client.cephfs]
	key = AQDvy1JhYNLDIRAAa+qMWOBavV3JtPj0QVzFGg==
```

查看授权信息：

```
[root@test-master-1 ~]# ceph auth get client.cephfs
[client.cephfs]
	key = AQDvy1JhYNLDIRAAa+qMWOBavV3JtPj0QVzFGg==
	caps mds = "allow rw"
	caps mon = "allow r"
	caps osd = "allow rw pool=cephfs_data, allow rw pool=cephfs_metadata"
exported keyring for client.cephfs
```

## 二、挂载文件系统

挂载有两种方式:
内核挂载，就可客户端使用mount挂载。
FUSE挂载，使用ceph客户端工具挂载。

#### 如何选择

https://docs.ceph.com/docs/master/cephfs/mount-prerequisites/

> FUSE客户端是最易于访问且最容易升级到存储集群使用的Ceph版本的，而内核客户端将始终提供更好的性能。

#### 内核挂载

https://docs.ceph.com/docs/master/cephfs/mount-using-kernel-driver/

> 作为一个粗略的指导，从Ceph 10.x（Jewel）开始，您应该至少使用4.x内核。如果使用旧的内核，则应使用FUSE客户端而不是内核客户端。
> 如果您使用的Linux发行版包含CephFS支持，则此建议不适用，因为在这种情况下，发行商将负责将修补程序反向移植到其稳定的内核：请与供应商联系。

我这里挂载没问题： centos 7.6

##### 挂载命令:

```
mount -t ceph {device-string}:{path-to-mounted} {mount-point} -o {key-value-args} {other-args}
```

`device-string`： `monitor`节点地址
`path-to-mounted`: 要挂载的目录，一般都是挂载根`/`。
`mount-point`: 挂载点
`key-value-args`: 指定CephX凭据， 如果禁用了CephX，可以省略。

比如：

```
mount -t ceph test-master-1:6789,test-master-2:6789,test-master-3:6789:/ /ceph/mycephfs -o name=cephfs,secret=AQDvy1JhYNLDIRAAa+qMWOBavV3JtPj0QVzFGg==
```

`name`: 指定的是用户名的id，不是完整的type.id用户名。

```
[root@test-master-1 /]# df -h
Filesystem                                               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root                                   58G   14G   44G  24% /
devtmpfs                                                 3.9G     0  3.9G   0% /dev
tmpfs                                                    3.9G     0  3.9G   0% /dev/shm
tmpfs                                                    3.9G  392M  3.5G  10% /run
tmpfs                                                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1                                                797M  145M  652M  19% /boot
tmpfs                                                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-0
tmpfs                                                    3.9G   52K  3.9G   1% /var/lib/ceph/osd/ceph-1
tmpfs                                                    783M     0  783M   0% /run/user/0
10.6.105.223:6789,10.6.105.224:6789,10.6.105.225:6789:/   55G     0   55G   0% /ceph/mycephfs
```

为了防止shell历史记录里有秘钥，也可以从文件里读取秘钥。`secretfile`指定文件。 文件权限最好是`600`。

```
mount -t ceph :/ /mnt/mycephfs -o name=foo,secretfile=/etc/ceph/foo.secret
```

如果集群中有多个文件系统，非默认的需要添加参数`fs`参数。
我这个版本的ceph，多文件系统还是实验功能，这个`fs`参数不能用。新版本可能支持了。

```
mount -t ceph :/ /mnt/mycephfs2 -o name=fs,fs=mycephfs2
```

##### 永久生效

写到fstab里永久生效

```
[{ipaddress}:{port}]:/ {mount}/{mountpoint} ceph [name=username,secret=secretkey|secretfile=/path/to/secretfile],[{mount.options}]
```

比如：

```
172.100.102.91:6789,172.100.102.92:6789,172.100.102.93:6789:/  /mnt/mycephfs  ceph  name=cephfs,secret=AQDAV3NeMMdCLBAAr/YtWYB9kI9ZKFlewUEaKA==,_netdev,noatime 0 0
```

```
test-master-1:6789,test-master-2:6789,test-master-3:6789:/ /ceph/mycephfs ceph name=cephfs,secret=AQDvy1JhYNLDIRAAa+qMWOBavV3JtPj0QVzFGg==,_netdev,noatime 0 0
```

```shell
[root@test-master-2 /]# cd ceph/mycephfs/
[root@test-master-2 mycephfs]# mkdir jumpserver
[root@test-master-2 mycephfs]# ll
drwxr-xr-x 1 root root 0 Sep 29 15:20 jumpserver
#后面将使用/ceph/mycephfs/jumpserver作为后面测试用的目录
#并且需设置权限chmod 775 -R jumpserver，否则后面会报错
```

#### FUSE挂载（未亲自验证）

因为`ceph-fuse`是安装在用户空间的， 所以性能可能相对较低， 但是更易于管理。

##### 安装`ceph-fuse`

使用之前安装ceph的yum源。

#### FUSE挂载

因为`ceph-fuse`是安装在用户空间的， 所以性能可能相对较低， 但是更易于管理。

##### 安装`ceph-fuse`

使用之前安装ceph的yum源。

```
[root@test-master-1 /]# cat /etc/yum.repos.d/ceph.repo 
[ceph]
name=Ceph packages for $basearch
baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc


[ceph-noarch]
name=Ceph noarch packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

```



##### 将集群的ceph.conf拷贝到客户端并且把客户端使用的key放到文件里。

```
[root@test-master-1 /]# ls etc/ceph/
ceph.client.admin.keyring  ceph.conf  rbdmap  tmpCTTuzi
[root@test-master-1 /]# cat /etc/ceph/ceph.client.admin.keyring 
[client.admin]
	key = AQAgZ0Bh3tI5KxAAXEFU2l+J+EEudkfMidlTug==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"
```

`ceph.client.cephfs.keyring`文件是手动创建的，内容就是授权给客户端使用的key。

把key文件权限给一下。为了安全。

```
chmod 600 ceph.client.cephfs.keyring
```

##### 挂载

挂载命令：

```
ceph-fuse {mountpoint} {options}
```

如：

```
[root@client ceph]# ceph-fuse --keyring  /etc/ceph/ceph.client.cephfs.keyring  --name client.cephfs -m 172.100.102.91:6789,172.100.102.92:6789,172.100.102.93:6789 /mnt/mycephfs/
ceph-fuse[1437]: starting ceph client
2020-03-19 19:56:06.417 7f79f1ae0e00 -1 init, newargv = 0x557ebbb0c870 newargc=7
ceph-fuse[1437]: starting fuse
```

```
[root@client ceph]# df -Th
文件系统       类型            容量  已用  可用 已用% 挂载点
/dev/sda2      xfs             146G  1.8G  145G    2% /
devtmpfs       devtmpfs        990M     0  990M    0% /dev
tmpfs          tmpfs          1000M     0 1000M    0% /dev/shm
tmpfs          tmpfs          1000M  8.8M  991M    1% /run
tmpfs          tmpfs          1000M     0 1000M    0% /sys/fs/cgroup
tmpfs          tmpfs           200M     0  200M    0% /run/user/0
ceph-fuse      fuse.ceph-fuse   28G   28M   28G    1% /mnt/mycephfs
```

##### 永久生效

也是添加到fstab。

```
none    /mnt/mycephfs  fuse.ceph ceph.id={user-ID}[,ceph.conf={path/to/conf.conf}],_netdev,defaults  0 0
```

如：

```
none /mnt/mycephfs fuse.ceph ceph.id=cephfs,ceph.conf=/etc/ceph/ceph.conf,_netdev,defaults 0 0
```

```
[root@client ceph]# mount -a
ceph-fuse[1561]: starting ceph client
2020-03-19 20:43:09.051 7f1ac5334e00 -1 init, newargv = 0x55712f0a1850 newargc=9
ceph-fuse[1561]: starting fuse
```

想要挂载子目录,添加一个`ceph.client_mountpoint`参数就可以。

```
none /mnt/mycephfs fuse.ceph ceph.id=cephfs,ceph.client_mountpoint=/etc,ceph.conf=/etc/ceph/ceph.conf,_netdev,defaults 0 0
```

## 三、mds主从与多主模式

https://docs.ceph.com/docs/master/cephfs/multimds/

建议使用Linux内核客户端> = 4.14。

> 默认情况下，每个CephFS文件系统都为一个活动的MDS守护程序配置。为了扩展大型系统的元数据性能，您可以启用多个活动的MDS守护程序，这些守护程序将彼此共享元数据工作负载。

#### 什么时候应该使用多主

> 当默认运行的单个MDS上的元数据性能出现瓶颈时，应配置多个活动的MDS守护程序。

> 添加更多守护程序可能不会提高所有工作负载的性能。通常，除非单个应用程序并行执行大量元数据操作，否则运行在单个客户端上的单个应用程序将不会受益于MDS守护程序数量的增加。

> 通常，受益于许多活动MDS守护程序的工作负载是具有许多客户端的工作负载，这些工作负载可能在多个单独的目录上工作。

#### 需要有备用的mds

> 即使有多个活动的MDS守护程序，如果运行活动的守护程序的任何服务器发生故障，高可用性系统仍需要备用守护程序来接管。

> 因此，活动mds的最大数量最多只能比系统中MDS服务器的总数少一，也就是说最少要有一个是处于备用状态。

> 为了在多个服务器发生故障时保持可用，请增加系统中的备用守护程序的数量，与您希望承受的服务器故障的数量相匹配。

#### 设置

每个CephFS文件系统都有一个max_mds设置，可以用来修改活动的mds数量。

```
ceph fs set <fs_name> max_mds 2
```

先看一下mds当前的状态, mycephfs文件系统中是1个up并且活动状态，2个up并且备用状态：

```
[root@cephnode1 ~]# ceph mds statmycephfs:1 {0=cephnode2=up:active} 2 up:standby
```

```
[root@cephnode1 ~]# ceph fs set mycephfs max_mds 2
```

修改以后，mycephfs文件系统，2活动(cephnode2与cephnode3)，1备用

```
[root@cephnode1 ~]# ceph mds statmycephfs:2 {0=cephnode2=up:active,1=cephnode3=up:active} 1 up:standby
```

在重新设置为1个主以后，其他的主会慢慢退出活动状态，需要几秒到几分钟时间, 这时处于stopping状态。

```
[root@cephnode1 ~]# ceph mds statmycephfs:2 {0=cephnode2=up:active,1=cephnode1=up:stopping} 1 up:standby
```

## 四、kubernetes使用cephfs

现在cephfs已经部署完成，现在需要考虑k8s对接cephfs使用的问题了。k8s使用cephfs进行数据持久化时，主要有三种方式:

- 使用k8s支持的cephfs类型volume直接挂载。
- 使用k8s支持的PV&PVC方式进行数据卷的挂载。
- 使用社区提供的一个cephfs provisioner来支持以storageClass的方式动态的分配PV，来进行挂载。

以部署jumpserver为例：

将client.cephfs的keyring转成base64编码：

```
[root@test-master-1 ymal]# ceph auth get-key client.cephfs | base64
QVFEdnkxSmhZTkxESVJBQWErcU1XT0JhdlYzSnRQajBRVnpGR2c9PQ==
```

#### 创建secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-client-cephfs
  namespace: jumpserver
type: kubernetes.io/rbd
data:
  key: QVFEdnkxSmhZTkxESVJBQWErcU1XT0JhdlYzSnRQajBRVnpGR2c9PQ==
```

#### 创建PV和PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv
  labels:
    name: ceph-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  cephfs:
    monitors: 
    - test-master-1:6789
    - test-master-2:6789
    - test-master-3:6789
    path: /jumpserver
    user: cephfs
    secretRef:
      name: ceph-secret-client-cephfs
    readOnly: false
  persistentVolumeReclaimPolicy: Retain

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jumpserver-pvc
  namespace: jumpserver
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      name: ceph-pv
```

#### 创建MySQL

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: mysql
  namespace: jumpeserver
  labels:
    name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mysql
    spec:
      restartPolicy: Always
      securityContext: {}
      containers:
        - resources:
            limits:
              memory: 6Gi
            requests:
              cpu: '2'
              memory: 6Gi
          terminationMessagePath: /dev/termination-log
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: admin
          ports:
            - containerPort: 3306
              protocol: TCP
          imagePullPolicy: Always
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
          image: registry.ht-ocp.com/jumperserver/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: jumpserver-pvc
```

注意：

在cephfs的目录下需要手动创建/jumpserver/data目录，并且chmod 775权限，否则pod创建过程中会报错。

进入MySQL容器创建数据库及账号：

```sql
[root@test-master-1 ymal]# kubectl exec -it mysql-7d79f458cd-rgddv -n jumpserver -- /bin/bash
bash-4.2$ mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create user'admin'@'%'identified by'admin';
Query OK, 0 rows affected (0.01 sec)

mysql> CREATE DATABASE IF NOT EXISTS jumperserver DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected (0.00 sec)

mysql> grant create,alter,drop,select,insert,update,delete on jumperserver.* to admin@'%';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on *.* to 'admin'@'%' identified by 'admin' with grant option;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
bash-4.2$ exit
exit
```

创建MySQL的service：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mysql
  namespace: jumpserver
spec:
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 31060
  selector:
    app: mysql
  type: NodePort
```

#### 创建redis

创建configmap

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-config
  namespace: jumpserver
  labels:
    app: redis
data:
  redis.conf: |-
    dir /usr/local/etc/redis/data
    port 6379
    bind 0.0.0.0
    appendonly yes
    protected-mode no
    requirepass 123456
    pidfile /usr/local/etc/redis/redis-6379.pid
```

创建deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: jumpserver
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: registry.ht-ocp.com/jumperserver/redis
          command:
            - "sh"
            - "-c"
            - "redis-server /usr/local/etc/redis/redis.conf"
          ports:
            - containerPort: 6379
          resources:
            limits:
              cpu: 1000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          livenessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 300
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          volumeMounts:
            - name: redis-data
              mountPath: /usr/local/etc/redis/data
            - name: config
              mountPath: /usr/local/etc/redis/redis.conf
              subPath: jumperserver/redis/redis.conf
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: jumpserver-pvc
        - name: config
          configMap:
            name: redis-config
```

创建service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: redis
  namespace: jumpserver
spec:
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
      nodePort: 32379
  selector:
    app: redis
  type: NodePort
```

#### 部署jumpserver

部署deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jumpserver
  namespace: jumpserver
spec:
  selector:
    matchLabels:
      app: jumpserver
  replicas: 3
  template:
    metadata:
      labels:
        app: jumpserver
    spec:
      containers:
        - name: jumpserver
          image: registry.ht-ocp.com/jumperserver/jumperserver
          imagePullPolicy: IfNotPresent
          env:
          - name: SECRET_KEY
            value: "DUpluRqAbF4bd8ZdB2ftxjKdc6OldDMoSNDfYet9n94bKcFGQU"
          - name: BOOTSTRAP_TOKEN
            value: "WwmkZbjo3Wp2LvJz"
          - name: DB_ENGINE
            value: "mysql"
          - name: DB_HOST
            value: "mysql"
          - name: DB_PORT
            value: "3306"
          - name: DB_USER
            value: "admin"
          - name: "DB_PASSWORD"
            value: "admin"
          - name: DB_NAME
            value: "jumperserver"
          - name: REDIS_HOST
            value: "redis"
          - name: REDIS_PASSWORD
            value: ""
          - name: REDIS_PORT
            value: "6379"
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
            - containerPort: 22
              name: ssh
              protocol: TCP
          volumeMounts:
            - mountPath: /opt/jumpserver/data/media
              name: jumpserver-data
      volumes:
        - name: jumpserver-data
          persistentVolumeClaim:
            claimName: jumpserver-pvc
```

```
1.将相应的环境变量的值替换成自己的
2.SECRET_KEY和BOOTSTRAP_TOKEN的值可以通过jumpserver官网给的脚步生成
cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50  获取SECRET_KEY
cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16  获取BOOTSTRAP_TOKEN
3.数据库和redis的密码不要使用特殊符号，使用特殊符号在初始化的时候配置文件回不正常，导致初始化失败
```

部署service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jumpserver
  namespace: jumpserver
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: ssh
      protocol: TCP
      port: 2222
      targetPort: 2222
  selector:
    app: jumpserver
  type: NodePort
  sessionAffinity: None
```


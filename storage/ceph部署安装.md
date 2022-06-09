# ceph集群部署安装

# 1、安装环境准备

使用ceph-deploy工具安装。
https://docs.ceph.com/docs/master/start/

硬件建议： https://docs.ceph.com/docs/master/start/hardware-recommendations/
系统建议： https://docs.ceph.com/docs/master/start/os-recommendations/

| 节点          | IP            | 配置             |
| :------------ | :------------ | :--------------- |
| test-master-1 | xx.xx.105.223 | 4c8G，2块30G裸盘 |
| test-master-2 | xx.xx.105.224 | 4c8G，2块30G裸盘 |
| test-master-3 | xx.xx.105.225 | 4c8G，2块30G裸盘 |

这里采用的是k8s集群的三个master节点做测试。

磁盘情况，两块可用盘,sdb,sdc：

```
[root@test-master-1 /]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0   60G  0 disk 
├─sda1            8:1    0  800M  0 part /boot
└─sda2            8:2    0 59.2G  0 part 
  ├─centos-root 253:0    0 57.2G  0 lvm  /
  └─centos-swap 253:1    0    2G  0 lvm  
sdb               8:16   0   30G  0 disk 
sdc               8:32   0   30G  0 disk 
sr0              11:0    1 1024M  0 rom  
```

准备：所有节点：

## 1.1、关闭防火墙与selinux

```
systemctl disable firewalld && systemctl stop firewalld
```

selinux:
修改`/etc/selinux/config`， 设置 `SELINUX=disabled`

执行： `setenforce 0` # 都设置完以后最好重启一下，也就不需要这条了。

## 1.2、时间同步与时区

centos7使用chronyd作为时间服务器，能保持系统时间与时间服务器（NTP）同步。

```
systemctl restart chronyd.service && systemctl enable chronyd.service
```

可以修改/etc/chrony.conf添加内网使用的ntp服务器。

时区：

```
timedatectl set-timezone Asia/Shanghai
```

## 1.3、关闭NetworkManager

linux上有两套管理网络的工具：network, NetworkManager。
因为NetworkManager会自动的做一些网络相关配置，比如在网络中断以后清理路由。一般桌面环境才会使用这个工具。所以为了防止莫名其妙的网络问题，一般都会把NetworkManager关闭。

```
systemctl disable NetworkManager && systemctl stop NetworkManager
```

## 1.4、主机名解析

确保可以ping通短主机名，根据需要解决主机名解析问题。这里直接加`/etc/hosts`。
比如我这里：

```
[root@test-master-1 baremetal]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

xx.xx.105.223 test-master-1
xx.xx.105.224 test-master-2
xx.xx.105.225 test-master-3
xx.xx.105.226 test-node-1
xx.xx.105.227 test-node-2
xx.xx.105.228 test-node-3
```

## 1.5、优化配置以及内核参数

### 1.5.1、文件描述符

```
cat >> /etc/security/limits.conf << EOF
* soft nofile 65535
* hard nofile 65535
EOF
```

有一些默认配置，`/etc/security/limits.d/20-nproc.conf` 这个文件看情况修改或删除。

### 1.5.2、内核

设置系统全局进程数量，以及尽量不使用swap。

```
cat >> /etc/sysctl.conf << EOF
kernel.pid_max = 4194303
vm.swappiness = 0
EOF
sysctl -p
```

磁盘性能
提高数据预读的大小:

```
echo "8192" > /sys/block/sda/queue/read_ahead_kb
cat >> /etc/rc.d/rc.local << EOF
echo "8192" > /sys/block/sda/queue/read_ahead_kb
EOF
```

```
/etc/rc.d/rc.local
` 默认情况下没有执行权限，需要添加执行权限。`
chmod +x /etc/rc.d/rc.local
```

修改磁盘I/O调度策略：
ssd盘改成`noop`， sata/sas改成`deadline`。
比如我这里是sata，并且有两块硬盘可以用于ceph。

```
echo "deadline" > /sys/block/sdb/queue/scheduler
echo "deadline" > /sys/block/sdc/queue/scheduler
cat >> /etc/rc.d/rc.local << EOF
echo "deadline" > /sys/block/sdb/queue/scheduler
echo "deadline" > /sys/block/sdc/queue/scheduler
EOF
```

```
[root@test-master-1 baremetal]# cat /sys/block/sdb/queue/scheduler
noop [deadline] cfq 
[root@test-master-1 baremetal]# cat /sys/block/sdc/queue/scheduler
noop [deadline] cfq 
```

## 1.6、免密登录

执行`ceph-deploy`命令的节点需要免密ssh登录其他节点。
默认可能没有ssh登录用的key，需要执行`ssh-keygen`生成。

```
ssh-keygen
ssh-copy-id root@xx.xx.105.224
ssh-copy-id root@xx.xx.105.225
```

## 1.7、添加yum源

添加epel源
需要从epel下载依赖。我这里使用阿里的epel源。

```
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

添加`ceph-deploy`的yum源。使用国内的源，阿里，163都有。

```
vim /etc/yum.repos.d/ceph.repo
```

```shell
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

ceph-noarch里是一些管理工具以及`ceph-deploy`，主要的ceph软件在ceph里。

# 2、安装

## 2.1、安装ceph-deploy

```
yum install -y ceph-deploy
```

如果执行`ceph-deploy`命令报错：

```
[root@test-master-1 /]# ceph-deploy
Traceback (most recent call last):
  File "/usr/bin/ceph-deploy", line 18, in <module>
    from ceph_deploy.cli import main
  File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
    import pkg_resources
ImportError: No module named pkg_resources
```

需要安装`python-setuptools`:

```
yum install python-setuptools -y
```

执行`ceph-deploy`命令的过程中，会生成集群的配置文件与秘钥。
所以需要创建一个目录，确保是在目录里执行命令。

**从头开始**

在安装过程中如果想重新开始，执行：

```
ceph-deploy purge {ceph-node} [{ceph-node}]ceph-deploy purgedata {ceph-node} [{ceph-node}]ceph-deploy forgetkeysrm ceph.*
```

删除软件，删除配置数据，删除key， 最后的rm是删除执行命令的目录里的东西。

## 2.2、创建一个集群

需要安装`ceph monitor`组件的节点。
`ceph-deploy new <initial-monitor-node>`
这一步是确定`ceph monitor`组件的节点，我这里先设置两个节点，完了以后再扩展。 `monitor`组件本身需要实现高可用，所以也是N/2+1的规则，最好是奇数。

```
ceph-deploy new test-master-1 test-master-2
```

看一下执行过程，其实就是确定目标节点ip，确认monitor节点，生成monitor key.
目录中生成了这几个文件：

```
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
```

`ceph.conf`: 生成的配置文件
`ceph.mon.keyring`: 生成的monitor秘钥。
如果服务器是多网卡，并且划分了集群内部网络，外部访问网络。
可以修改`ceph.conf`, 参考：
https://docs.ceph.com/docs/master/rados/configuration/network-config-ref/

## 2.3、安装ceph软件包

所有ceph节点。

```
yum install ceph -y
```

也可以使用`ceph-deploy`工具安装，只是它会下载官方的源，导致安装的很慢。
安装完成以后，发现自动启动了一个进程：
`/usr/bin/python2.7 /usr/bin/ceph-crash`
暂时不确定这个进程的作用

安装epel-release及yum相关组件

```
yum -y install epel-release yum-plugin-priorities yum-utils
```

安装Ceph及相关组件

```
yum install -y ceph-deploy ceph ceph-radosgw snappy leveldb gdisk python-argparse gperftools-libs
```

查看ceph版本

```
[root@test-master-1 ~]# ceph -v
ceph version 14.2.22 (ca74598065096e6fcbd8433c8779a2be0c889351) nautilus (stable)
```

## 2.3、部署`monitor`组件以及收集各种key

```
ceph-deploy mon create-initial
```

执行成功以后，ps查看进程会发现上面设置的节点已经运行了`monitor`组件：

```
/usr/bin/ceph-mon -f --cluster ceph --id cephnode2 --setuser ceph --setgroup ceph
```

目录里会多出来很多key：

```
[root@test-master-1 ~]# ll
total 76
-rw-------. 1 root root  1655 Aug 24 09:32 anaconda-ks.cfg
-rw-------  1 root root   113 Sep 14 17:11 ceph.bootstrap-mds.keyring
-rw-------  1 root root   113 Sep 14 17:11 ceph.bootstrap-mgr.keyring
-rw-------  1 root root   113 Sep 14 17:11 ceph.bootstrap-osd.keyring
-rw-------  1 root root   113 Sep 14 17:11 ceph.bootstrap-rgw.keyring
-rw-------  1 root root   151 Sep 14 17:11 ceph.client.admin.keyring
-rw-r--r--  1 root root   231 Sep 14 16:54 ceph.conf
-rw-r--r--  1 root root 43805 Sep 14 17:11 ceph-deploy-ceph.log
-rw-------  1 root root    73 Sep 14 16:54 ceph.mon.keyring
```

查看刚才命令的执行过程，也会发现`ceph.conf`配置文件已经复制到了`/etc/ceph`下面。

## 2.4、发送ceph配置文件与管理秘钥到各节点

只是为了执行一些ceph cli命令时不需要再指定`monitor`节点地址与admin key文件。比如：ceph、rbd等命令就是需要使用`/etc/ceph/ceph.client.admin.keyring`来连接集群。

```
ceph-deploy admin test-master-1 test-master-2 test-master-3
```

```shell
[root@test-master-1 ~]# ceph-deploy admin test-master-1 test-master-2 test-master-3
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy admin test-master-1 test-master-2 test-master-3
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7ff62c6ea2d8>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  client                        : ['test-master-1', 'test-master-2', 'test-master-3']
[ceph_deploy.cli][INFO  ]  func                          : <function admin at 0x7ff62cf80230>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to test-master-1
[test-master-1][DEBUG ] connected to host: test-master-1 
[test-master-1][DEBUG ] detect platform information from remote host
[test-master-1][DEBUG ] detect machine type
[test-master-1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to test-master-2
[test-master-2][DEBUG ] connected to host: test-master-2 
[test-master-2][DEBUG ] detect platform information from remote host
[test-master-2][DEBUG ] detect machine type
[test-master-2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to test-master-3
[test-master-3][DEBUG ] connected to host: test-master-3 
[test-master-3][DEBUG ] detect platform information from remote host
[test-master-3][DEBUG ] detect machine type
[test-master-3][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
```

## 2.5、安装`ceph-manager`

只有 `luminous+, i.e >= 12.x`的版本需要安装, 只有大于等于12.x的版本需要。
`manager daemon`与`monitor daemon`一起为外部监控和管理系统提供额外的监控和接口。

> 从12.x版本开始，ceph的正常运行需要`mgr`，在11.x的版本还是可选组件。
> 如果没有运行`mgr`，会有一些运行状态警告，并且`ceph status`命令输出的某些信息会缺失 或者是旧的信息。
> 看来一些查看状态的命令也是从这个组件获取的信息。就跟k8s的`metrics server`一样。
> 默认情况下，`mgr`不需要其他配置就可以正常运行.
> 将`mgr`与`monitor`放到同一节点不是强制的，但几乎总是合理的。

```
ceph-deploy mgr create test-master-1 test-master-2 test-master-3
```

执行完毕多了一个进程：

```
/usr/bin/ceph-mgr -f --cluster ceph --id cephnode1 --setuser ceph --setgroup ceph
```

## 2.6、部署`rgw`

可选
提供对象存储，如果不使用对象存储，也就不用安装了。
需要安装`ceph-radosgw`软件，使用`ceph-deploy`应该也可以安装，`ceph-deploy install`里面有对应的参数。

在需要提供`rgw`的节点执行。

```
yum install -y ceph-radosgw
```

使用`ceph-deploy`添加到集群, 我这里只添加一个节点了。

```
ceph-deploy rgw create test-master-1
```

同样的执行完以后，也会多出来一个对应组件的进程。

```
/usr/bin/radosgw -f --cluster ceph --name client.rgw.cephnode1 --setuser ceph --setgroup ce
```

## 2.7、部署`MDS`

可选
提供元数据服务，文件系统存储使用。如果不使用，也就不用安装。

```
ceph-deploy mds create test-master-1 test-master-2 test-master-3
```

执行完毕多了`mds`组件的进程：

```
/usr/bin/ceph-mds -f --cluster ceph --id test-master-1 --setuser ceph --setgroup ceph
```

## 2.8、添加`osd`

使用未格式化的裸盘。硬盘随时可以加进集群。

```
ceph-deploy osd create --data {device} {ceph-node}
```

```
ceph-deploy osd create --data /dev/sdb test-master-1
ceph-deploy osd create --data /dev/sdc test-master-1
ceph-deploy osd create --data /dev/sdb test-master-2
ceph-deploy osd create --data /dev/sdc test-master-2
ceph-deploy osd create --data /dev/sdb test-master-3
ceph-deploy osd create --data /dev/sdc test-master-4
```

执行完成后：

```
[root@test-master-2 ~]# lsblk
NAME                                                                                                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                                                                                                     8:0    0   60G  0 disk 
├─sda1                                                                                                  8:1    0  800M  0 part /boot
└─sda2                                                                                                  8:2    0 59.2G  0 part 
  ├─centos-root                                                                                       253:0    0 57.2G  0 lvm  /
  └─centos-swap                                                                                       253:1    0    2G  0 lvm  
sdb                                                                                                     8:16   0   30G  0 disk 
└─ceph--e440ee50--0a77--4c5b--b824--87541bdc34f9-osd--block--144f6eb0--3b0e--4e60--a89a--8642dac63307 253:2    0   30G  0 lvm  
sdc                                                                                                     8:32   0   30G  0 disk 
└─ceph--dc5e2c25--ffd4--4903--8e95--575b39dd78f3-osd--block--7de8cd8b--b417--4deb--9542--6d87f69830fa 253:3    0   30G  0 lvm  
sr0                                                                                                    11:0    1 1024M  0 rom  
```

```
/usr/bin/ceph-osd -f --cluster ceph --id 2 --setuser ceph --setgroup ceph
/usr/bin/ceph-osd -f --cluster ceph --id 3 --setuser ceph --setgroup ceph
```

进程，每个osd进程对应一块磁盘，这个节点两块盘，也就两个osd进程。注意里面的id， 是全局唯一的，所有节点唯一的。

## 2.9、查看集群状态

```
[root@test-master-1 ~]# ceph health
HEALTH_WARN mons are allowing insecure global_id reclaim
[root@test-master-1 ~]# ceph status
  cluster:
    id:     4a7b358c-37ea-4f32-a7db-0e626d02c312
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
 
  services:
    mon: 2 daemons, quorum test-master-1,test-master-2 (age 21m)
    mgr: test-master-1(active, since 11m), standbys: test-master-2, test-master-3
    mds:  3 up:standby
    osd: 6 osds: 6 up (since 4m), 6 in (since 4m)
    rgw: 1 daemon active (test-master-1)
 
  task status:
 
  data:
    pools:   4 pools, 128 pgs
    objects: 187 objects, 1.2 KiB
    usage:   6.0 GiB used, 174 GiB / 180 GiB avail
    pgs:     128 active+clean
```

状态正常的话应该显示HEALTH_OK

如何解决：

禁用不安全模式

```
ceph config set mon auth_allow_insecure_global_id_reclaim false
```

```shell
[root@test-master-1 ~]# ceph config set mon auth_allow_insecure_global_id_reclaim false
[root@test-master-1 ~]# ceph -s
  cluster:
    id:     4a7b358c-37ea-4f32-a7db-0e626d02c312
    health: HEALTH_OK
 
  services:
    mon: 2 daemons, quorum test-master-1,test-master-2 (age 23m)
    mgr: test-master-1(active, since 14m), standbys: test-master-2, test-master-3
    mds:  3 up:standby
    osd: 6 osds: 6 up (since 7m), 6 in (since 7m)
    rgw: 1 daemon active (test-master-1)
 
  task status:
 
  data:
    pools:   4 pools, 128 pgs
    objects: 187 objects, 1.2 KiB
    usage:   6.0 GiB used, 174 GiB / 180 GiB avail
    pgs:     128 active+clean
```

`monitor` 组件两个节点，`cephnode1`,`cephnode2`
`mgr`: 3个节点，`cephnode1`活动，其他两个备用。
`mds`: 3 up:standby , 3个up并且备用节点，因为还没有创建cephfs文件系统，所以没有节点是活动状态。

## 2.10、如何重启

操作集群:
https://docs.ceph.com/docs/master/rados/operations/operating/

启动：`systemctl start ceph.target`
关闭: `systemctl stop ceph\*.service ceph\*.target`

按类型启动:

```
systemctl start ceph-osd.target
systemctl start ceph-mon.target
systemctl start ceph-mds.target
```

按类型关闭:

```
systemctl stop ceph-mon\*.service ceph-mon.target
systemctl stop ceph-osd\*.service ceph-osd.target
systemctl stop ceph-mds\*.service ceph-mds.target
```

## 2.11、安装dashboard

```
yum install -y ceph-mgr-dashboard     #安装dashboard     
ceph mgr module enable dashboard     #开启dashboard功能
ceph dashboard create-self-signed-cert  #创建证书
ceph config set mgr mgr/dashboard/ssl false   #禁用ssl
openssl req -new -nodes -x509 \
  -subj "/O=IT/CN=ceph-mgr-dashboard" -days 3650 \
  -keyout dashboard.key -out dashboard.crt -extensions v3_ca   #生成证书文件
vim password.txt #新建一个文件，通过这个文件设置密码。其实这块并没有理解，还存在问题
eph dashboard -h   #查看命令帮助
ceph dashboard ac-user-set-password admin -i password.txt  #给用户admin设置密码
ceph mgr services   #查看服务列表，打开dashboard后可以用刚刚设置的账号密码登陆
```

创建账号还有问题

dashboard相关的报错：

```shell
[root@test-ceph-1 ~]# ceph -s
  cluster:
    id:     7f485c64-943e-49fc-9638-bb633f2f7c81
    health: HEALTH_ERR
            Module 'dashboard' has failed: IOError("Port 8443 not free on '10.6.23.181'",)
 
  services:
    mon: 3 daemons, quorum test-ceph-1,test-ceph-2,test-ceph-3 (age 5h)
    mgr: test-ceph-2(active, since 2h), standbys: test-ceph-1, test-ceph-3
    mds: mycephfs:1 {0=test-ceph-1=up:active} 2 up:standby
    osd: 6 osds: 6 up (since 5h), 6 in (since 21h)
    rgw: 3 daemons active (test-ceph-1, test-ceph-2, test-ceph-3)
 
  task status:
 
  data:
    pools:   6 pools, 288 pgs
    objects: 318 objects, 154 MiB
    usage:   6.4 GiB used, 1.8 TiB / 1.8 TiB avail
    pgs:     288 active+clean
 
  io:
    client:   6.0 KiB/s wr, 0 op/s rd, 0 op/s wr
```

```bash
[root@test-ceph-1 ~]# ceph config dump
WHO   MASK LEVEL    OPTION                                VALUE       RO 
  mon      advanced auth_allow_insecure_global_id_reclaim false          
  mgr      advanced mgr/dashboard/server_addr             10.6.23.181 *  
  mgr      advanced mgr/dashboard/server_port             8443        *  
  mgr      advanced mgr/dashboard/ssl                     false       *  
[root@test-ceph-1 ~]# ceph mgr services
{
    "dashboard": "http://test-ceph-1:8443/",
    "prometheus": "http://test-ceph-2:9283/"
}
#注意ceph -s显示当前活跃的mgr是test-ceph-2
#尝试把dashboard修改到test-ceph-2节点上
[root@test-ceph-1 ~]# ceph config set mgr mgr/dashboard/server_addr test-ceph-2
[root@test-ceph-1 ~]# ceph mgr module disable dashboard
[root@test-ceph-1 ~]# ceph mgr module enable dashboard --force
[root@test-ceph-1 ~]# ceph config dump
WHO   MASK LEVEL    OPTION                                VALUE       RO 
  mon      advanced auth_allow_insecure_global_id_reclaim false          
  mgr      advanced mgr/dashboard/server_addr             test-ceph-2 *  
  mgr      advanced mgr/dashboard/server_port             8443        *  
  mgr      advanced mgr/dashboard/ssl                     false       *  
[root@test-ceph-1 ~]# ceph mgr services
{
    "dashboard": "http://test-ceph-2:8443/",
    "prometheus": "http://test-ceph-2:9283/"
}
#查看现在的状态，集群恢复正常
[root@test-ceph-1 ~]# ceph -s
  cluster:
    id:     7f485c64-943e-49fc-9638-bb633f2f7c81
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum test-ceph-1,test-ceph-2,test-ceph-3 (age 5h)
    mgr: test-ceph-2(active, since 21s), standbys: test-ceph-3, test-ceph-1
    mds: mycephfs:1 {0=test-ceph-1=up:active} 2 up:standby
    osd: 6 osds: 6 up (since 5h), 6 in (since 21h)
    rgw: 3 daemons active (test-ceph-1, test-ceph-2, test-ceph-3)
 
  task status:
 
  data:
    pools:   6 pools, 288 pgs
    objects: 318 objects, 155 MiB
    usage:   6.4 GiB used, 1.8 TiB / 1.8 TiB avail
    pgs:     288 active+clean
 
  io:
    client:   8.7 KiB/s wr, 0 op/s rd, 0 op/s wr

```



# 3、扩展集群

## 3.1、添加monitor

```
ceph-deploy mon add {ceph-nodes}
```

添加节点3

```
ceph-deploy mon add  test-master-3
```

发现抱错：

```shell
[test-master-3][WARNIN] Created symlink from /etc/systemd/system/ceph-mon.target.wants/ceph-mon@test-master-3.service to /usr/lib/systemd/system/ceph-mon@.service.
[test-master-3][INFO  ] Running command: systemctl start ceph-mon@test-master-3
[test-master-3][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.test-master-3.asok mon_status
[test-master-3][ERROR ] admin_socket: exception getting command descriptions: [Errno 2] No such file or directory
[test-master-3][WARNIN] test-master-3 is not defined in `mon initial members`
[test-master-3][WARNIN] monitor test-master-3 does not exist in monmap
[test-master-3][WARNIN] neither `public_addr` nor `public_network` keys are defined for monitors
[test-master-3][WARNIN] monitors may not be able to form quorum
[test-master-3][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.test-master-3.asok mon_status
[test-master-3][ERROR ] admin_socket: exception getting command descriptions: [Errno 2] No such file or directory
[test-master-3][WARNIN] monitor: mon.test-master-3, might not be running yet
```

通过日志   tail -f /var/log/messages   可以看出：

提示需要设置`public_addr or public_network`。
尝试过直接修改对应节点上的`/etc/ceph/ceph.conf`， 但是发现`ceph-deploy`命令还会对比目录里的`ceph.conf`文件，会报错。所以按下面的步骤来。

编辑执行`ceph-deploy`命令的目录里的`ceph.conf`文件.（这个例子中是在/root目录下）
添加`public network = xx.xx.105.0/24`，我这里的网卡网段。
执行：

```
ceph-deploy --overwrite-conf mon add test-master-3
```

就可以了。
创建集群的时候可能都是在`ceph-deploy new`之后就配置好配置文件。

添加新的节点以后，Ceph将开始同步监视器并形成仲裁。可以通过执行以下操作检查仲裁状态。

```
ceph quorum_status --format json-pretty
```

## 3.2、添加`manager`

跟部署的流程一样

```
ceph-deploy mgr create test-master-3
```

## 3.3、添加`rgw`

跟部署的流程一样。

```
yum install -y ceph-radosgw
ceph-deploy --overwrite-conf rgw create test-master-2 test-master-3
```

这里加--overwrite-conf因为添加`monitor`的时候改了目录里的配置文件。

# 4、一些查看集群状态的命令合集

## 4.1、集群总状态

```
ceph status
ceph -s
```

## 4.2、`monitor`统计

```
ceph mon stat
```

## 4.3、`monitor`基本信息

```
ceph mon dump
```

## 4.4、`monitor`状态

```
ceph mon_status
```

## 4.5、`mgr`信息

```
ceph mgr dump
```

## 4.6、`pg`统计

```
ceph pg stat
```

## 4.7、`pg`详细信息

```
ceph pg ls
```


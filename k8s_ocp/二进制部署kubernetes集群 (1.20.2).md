# 二进制部署kubernetes集群

通过kubeadm可以快速部署一套kubernetes集群，但是如果需要精细调整各组件服务的参数、安全配置及高可用模式等，管理员就可以使用二进制文件进行部署。

# 1、环境配置

## 1.1、机器规划及软件版本

| IP地址          | 机器名    | 操作系统  | 机器角色             | 安装软件                                                     |
| --------------- | --------- | --------- | -------------------- | ------------------------------------------------------------ |
| xx.xx...105.223 | xx-xxxx-3 | centos7.6 | master               | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| xx.xx...105.224 | xx-xxxx-4 | centos7.6 | master               | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| xx.xx...105.225 | xx-xxxx-5 | centos7.6 | master               | kube-apiserver、kube-controller-manager、kube-scheduler、etcd |
| xx.xx...105.226 | xx-xxxx-6 | centos7.6 | worker               | kubelet、kube-proxy                                          |
| xx.xx...105.227 | xx-xxxx-7 | centos7.6 | worker               | kubelet、kube-proxy                                          |
| xx.xx...105.228 | xx-xxxx-8 | centos7.6 | worker               | kubelet、kube-proxy                                          |
| xx.xx...105.231 |           | centos7.6 | 负载均衡VIP          | haproxy+keepalived                                           |
| 10.222.0.0/16   |           |           |                      |                                                              |
| 10.222.0.1      |           |           | service 网络的首个IP |                                                              |
| 10.222.0.2      |           |           | clusterDNS           |                                                              |
| 10.223.0.0/16   |           |           | clusterCIDR          |                                                              |



| 软件                                                         | 版本    |
| ------------------------------------------------------------ | ------- |
| kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy | v1.20.2 |
| etcd                                                         | v3.4.13 |
| calico                                                       | v3.14   |
| coredns                                                      | 1.7.0   |
| keepalived                                                   | 2.0.20  |
| haproxy                                                      | 18      |

## 1.2、配置阿里镜像

```shell
#先备份本地镜像源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
#下载阿里镜像源
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
或者
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
#更新缓存
yum makecache
```

## 1.3、关闭防火墙

```
systemctl stop firewalld
systemctl disable firewalld
```

## 1.4、关闭selinux

```
setenforce 0
sed -i '7s/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

## 1.5、关闭交换分区

```
swapoff -a
永久关闭，修改/etc/fstab,注释掉swap一行
```

## 1.6、时间同步

```
yum install -y chrony
systemctl start chronyd
systemctl enable chronyd
chronyc sources
```

## 1.7、 将桥接的IPV4流量传递到iptables的链

```
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## 1.8、加载IPVS模块

```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
lsmod | grep ip_vs
lsmod | grep nf_conntrack_ipv4
yum install -y ipvsadm
```

## 1.9、配置hosts文件

```
[root@xx-xxxx-8 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

xx.xx105.223 xx-xxxx-3
xx.xx105.224 xx-xxxx-4
xx.xx105.225 xx-xxxx-5
xx.xx105.226 xx-xxxx-6
xx.xx105.227 xx-xxxx-7
xx.xx105.228 xx-xxxx-8
```

## 1.10、配置工作目录

每台机器都需要配置证书文件、组件的配置文件、组件的服务启动文件，现专门选择 master1 来统一生成这些文件，然后再分发到其他机器。

```
mkdir -p /data/work
```

## 1.11、配置免密登陆

让master-1可以免密登陆到其他机器

```
ssh-keygen      //生成私钥
ssh-copy-id root@xx.xx....xx.xx...x      //向主机分发私钥
```

# 2、搭建etcd集群

## 2.1、创建etcd证书

### 2.1.1、安装cfssl证书生成工具

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

### 2.1.2、配置CA请求文件

```
vim /data/work/ca-csr.json 
```

```json
{
  "CN": "kubernetes",
  "key": {
      "algo": "rsa",
      "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "SuZhou",
      "O": "k8s",
      "OU": "system"
    }
  ],
  "ca": {
          "expiry": "87600h"
  }
}

```

<!--注：-->
<!--CN：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；-->
<!--O：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)-->

### 2.1.3、创建CA证书

```
[root@xx-xxxx-3 work]# cfssl gencert -initca ca-csr.json  | cfssljson -bare ca
2021/08/12 22:09:59 [INFO] generating a new CA key and certificate from CSR
2021/08/12 22:09:59 [INFO] generate received request
2021/08/12 22:09:59 [INFO] received CSR
2021/08/12 22:09:59 [INFO] generating key: rsa-2048
2021/08/12 22:09:59 [INFO] encoded CSR
2021/08/12 22:09:59 [INFO] signed certificate with serial number 267081203352079894854435011787957823136276238805
```

### 2.1.4、配置CA证书策略

```
vim /data/work/ca-config.json
```

```json
{
  "signing": {
      "default": {
          "expiry": "87600h"
        },
      "profiles": {
          "kubernetes": {
              "usages": [
                  "signing",
                  "key encipherment",
                  "server auth",
                  "client auth"
              ],
              "expiry": "87600h"
          }
      }
  }
}
```

### 2.1.5、配置etcd请求csr文件

```
vim /data/work/etcd-csr.json
```

```json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "xx.xx..105.223",
    "xx.xx..105.224",
    "xx.xx..105.225"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "CN",
    "ST": "JiangSu",
    "L": "SuZhou",
    "O": "k8s",
    "OU": "system"
  }]
}
```

### 2.1.6、生成证书

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
```

```
[root@xx-xxxx-3 work]# ll etcd*.pem
-rw------- 1 root root 1679 Aug 12 22:16 etcd-key.pem
-rw-r--r-- 1 root root 1432 Aug 12 22:16 etcd.pem
```

## 2.2、创建etcd集群

### 2.2.1、下载etcd二进制文件

```
https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
```

将安装包解压后放到/data/work目录下，将etcd和etcdctl文件拷贝至/usr/local/bin目录下（三个master节点）

```
tar -vxf etcd-v3.4.13-linux-amd64.tar.gz
cp -p etcd etcdctl /usr/local/bin
yum install rsync -y
rsync -vaz etcd-v3.4.13-linux-amd64/etcd* xx-xxxx-4:/usr/local/bin/
rsync -vaz etcd-v3.4.13-linux-amd64/etcd* xx-xxxx-5:/usr/local/bin/
```

### 2.2.2、创建配置文件

```
vim /data/work/etcd.conf
```

```shell
#[Member]
ETCD_NAME="etcd1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://xx.xx.105.223:2380"
ETCD_LISTEN_CLIENT_URLS="https://xx.xx.105.223:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://xx.xx.105.223:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://xx.xx.105.223:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://xx.xx.105.223:2380,etcd2=https://xx.xx.105.224:2380,etcd3=https://xx.xx.105.225:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

<!--注：-->
<!--ETCD_NAME：节点名称，集群中唯一-->
<!--ETCD_DATA_DIR：数据目录-->
<!--ETCD_LISTEN_PEER_URLS：集群通信监听地址-->
<!--ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址-->
<!--ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址-->
<!--ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址-->
<!--ETCD_INITIAL_CLUSTER：集群节点地址-->
<!--ETCD_INITIAL_CLUSTER_TOKEN：集群Token-->
<!--ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群-->

### 2.2.3、创建启动服务文件

方式一、有配置文件的启动

```
vim /data/work/etcd.service
```

```shell
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/etc/etcd/etcd.conf
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

方式二、无配置文件启动：

```shell
[root@master1 work]# vim etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --name=etcd1 \
  --data-dir=/var/lib/etcd/default.etcd \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --listen-peer-urls=https://172.10.1.11:2380 \
  --listen-client-urls=https://172.10.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.10.1.11:2379 \
  --initial-advertise-peer-urls=https://172.10.1.11:2380 \
  --initial-cluster=etcd1=https://172.10.1.11:2380,etcd2=https://172.10.1.12:2380,etcd3=https://172.10.1.13:2380 \
  --initial-cluster-token=etcd-cluster \
  --initial-cluster-state=new
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

注：本文采用第一种方式

### 2.2.4、同步相关文件到各个节点：

```
cp ca*.pem /etc/etcd/ssl/
cp etcd*.pem /etc/etcd/ssl/
cp etcd.conf /etc/etcd/
cp etcd.service /usr/lib/systemd/system/
for i in xx-xxxx-4 xx-xxxx-5;do rsync -vaz etcd.conf $i:/etc/etcd/;done
for i in xx-xxxx-4 xx-xxxx-5;do rsync -vaz etcd*.pem ca*.pem $i:/etc/etcd/ssl/;done
for i in xx-xxxx-4 xx-xxxx-5;do rsync -vaz etcd.service $i:/usr/lib/systemd/system/;done
```

<!--注：master2和master3分别修改配置文件中etcd名字和ip，并创建目录 /var/lib/etcd/default.etcd-->

创建工作目录：

```
mkdir -p /var/lib/etcd/default.etcd
```

### 2.2.5、启动etcd集群

```
mkdir -p /var/lib/etcd/default.etcd
systemctl daemon-reload
systemctl enable etcd.service
systemctl start etcd.service
systemctl status etcd
```

### 2.2.6、查看集群状态

```
ETCDCTL_API=3 /usr/local/bin/etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --endpoints=https://xx.xx105.223:2379,https://xx.xx105.224:2379,https://xx.xx105.225:2379 endpoint health
```

```
+---------------------------+--------+-------------+-------+
|         ENDPOINT          | HEALTH |    TOOK     | ERROR |
+---------------------------+--------+-------------+-------+
| https://xx.xx105.224:2379 |   true | 13.489066ms |       |
| https://xx.xx105.223:2379 |   true | 13.601289ms |       |
| https://xx.xx105.225:2379 |   true | 17.159935ms |       |
+---------------------------+--------+-------------+-------+
```

集群启动正常。

# 3、kubernetes组件部署

## 3.1、下载二进制文件

二进制文件下载完成后进行解压，然后将需要在master节点上部署服务的可执行的二进制文件复制到/usr/bin下，并分发到其他两个master节点的相同目录下。将需要在node节点上部署的服务的可执行二进制文件分发到相应的节点上的目录下。

```
wget https://dl.k8s.io/v1.20.1/kubernetes-server-linux-amd64.tar.gz
tar -vxf kubernetes-server-linux-amd64.tar
cd kubernetes/server/bin/
cp -p kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin
rsync -vaz kube-apiserver kube-controller-manager kube-scheduler kubectl xx-xxxx-4:/usr/local/bin/
rsync -vaz kube-apiserver kube-controller-manager kube-scheduler kubectl xx-xxxx-5:/usr/local/bin/
for i in xx-xxxx-6 xx-xxxx-7 xx-xxxx-8;do rsync -vaz kubelet kube-proxy $i:/usr/local/bin/;done
```

## 3.2、创建工作目录

```shell
mkdir -p /etc/kubernetes/          # kubernetes组件配置文件存放目录
mkdir -p /etc/kubernetes/ssl     # kubernetes组件证书文件存放目录
mkdir /var/log/kubernetes        # kubernetes组件日志文件存放目录
```

## 3.3、部署api-server

### 3.3.1、创建csr请求文件

```
vim kube-apiserver-csr.json
```

```json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "xx.xx105.223",
    "xx.xx105.224",
    "xx.xx105.225",
    "xx.xx105.226",
    "xx.xx105.227",
    "xx.xx105.228",
    "xx.xx105.231",
    "10.222.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "SuZhou",
      "O": "k8s",
      "OU": "system"
    }
  ]
}
```

<!--注：-->
<!--如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表。-->
<!--由于该证书后续被 kubernetes master 集群使用，需要将master节点的IP都填上，同时还需要填写 service 网络的首个IP。(一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.222.0.1)-->

### 3.3.2、生成证书和token文件

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver

[root@master1 work]# cat > token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

### 3.3.3、创建配置文件

```
vim kube-apiserver.conf
```

```shell
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --bind-address=xx.xx105.223 \
  --secure-port=6443 \
  --advertise-address=xx.xx105.223 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all=true \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.222.0.0/16 \
  --token-auth-file=/etc/kubernetes/token.csv \
  --service-node-port-range=30000-50000 \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \  # 1.20以上版本必须有此参数# \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \ # 1.20以上版本必须有此参数# \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --etcd-servers=https://xx.xx105.223:2379,https://xx.xx105.224:2379,https://xx.xx105.225:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=4"

```

注：
–logtostderr：启用日志
–v：日志等级
–log-dir：日志目录
–etcd-servers：etcd集群地址
–bind-address：监听地址
–secure-port：https安全端口
–advertise-address：集群通告地址
–allow-privileged：启用授权
–service-cluster-ip-range：Service虚拟IP地址段
–enable-admission-plugins：准入控制模块
–authorization-mode：认证授权，启用RBAC授权和节点自管理
–enable-bootstrap-token-auth：启用TLS bootstrap机制
–token-auth-file：bootstrap token文件
–service-node-port-range：Service nodeport类型默认分配端口范围
–kubelet-client-xxx：apiserver访问kubelet客户端证书
–tls-xxx-file：apiserver https证书
–etcd-xxxfile：连接Etcd集群证书
–audit-log-xxx：审计日志

### 3.3.4、创建服务启动文件

```
vim kube-apiserver.service
```

```shell
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 3.3.5、同步相关文件到各个节点

```shell
cp ca*.pem /etc/kubernetes/ssl/
cp kube-apiserver*.pem /etc/kubernetes/ssl/
cp token.csv /etc/kubernetes/
cp kube-apiserver.conf /etc/kubernetes/	
cp kube-apiserver.service /usr/lib/systemd/system/
rsync -vaz token.csv xx-xxxx-4:/etc/kubernetes/
rsync -vaz token.csv xx-xxxx-5:/etc/kubernetes/
rsync -vaz kube-apiserver*.pem xx-xxxx-4:/etc/kubernetes/ssl/     
# 主要rsync同步文件，只能创建最后一级目录，如果ssl目录不存在会自动创建，但是上一级目录kubernetes必须存在
rsync -vaz kube-apiserver*.pem xx-xxxx-5:/etc/kubernetes/ssl/
rsync -vaz ca*.pem xx-xxxx-4:/etc/kubernetes/ssl/
rsync -vaz ca*.pem xx-xxxx-5:/etc/kubernetes/ssl/
rsync -vaz kube-apiserver.conf xx-xxxx-4:/etc/kubernetes/
rsync -vaz kube-apiserver.conf xx-xxxx-5:/etc/kubernetes/
rsync -vaz kube-apiserver.service xx-xxxx-4:/usr/lib/systemd/system/
rsync -vaz kube-apiserver.service xx-xxxx-5:/usr/lib/systemd/system/
```

<!--注：master2和master3配置文件的IP地址修改为实际的本机IP-->

### 3.3.6、启动服务

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
测试
curl --insecure https://xx.xx105.223:6443/
有返回说明启动正常
```

## 3.4、部署kubectl

### 3.4.1、创建csr请求文件

```
vim /data/work/admin-csr.json
```

```json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "SuZhou",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
```

说明：
后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
O指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
注：
这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group；
“O”: “system:masters”, 必须是system:masters，否则后面kubectl create clusterrolebinding报错。

### 3.4.2、生成证书

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

cp admin*.pem /etc/kubernetes/ssl/
```

### 2.3.4.3、创建kubeconfig配置文件

kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书

```shell
#设置集群参数
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://xx.xx105.231:5000 --kubeconfig=kube.config
#设置客户端认证参数
kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
#设置上下文参数
kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
#设置默认上下文
kubectl config use-context kubernetes --kubeconfig=kube.config
mkdir ~/.kube
cp kube.config ~/.kube/config
#授权kubernetes证书访问kubelet api权限
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
```

```
[root@xx-xxxx-3 work]# kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://xx.xx105.231:5000 --kubeconfig=kube.config
Cluster "kubernetes" set.
[root@xx-xxxx-3 work]# kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
User "admin" set.
[root@xx-xxxx-3 work]# kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
Context "kubernetes" modified.
[root@xx-xxxx-3 work]# kubectl config use-context kubernetes --kubeconfig=kube.config
Switched to context "kubernetes".
[root@xx-xxxx-3 work]# cp kube.config ~/.kube/config
cp: overwrite ‘/root/.kube/config’? y
[root@xx-xxxx-3 work]# kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
clusterrolebinding.rbac.authorization.k8s.io/kube-apiserver:kubelet-apis created
```

### 3.4.4、查看集群组件状态

```shell
[root@xx-xxxx-3 work]# kubectl cluster-info
Kubernetes control plane is running at https://xx.xx105.231:5000

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@xx-xxxx-3 work]# kubectl get all --all-namespaces
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.222.0.1   <none>        443/TCP   5h18m
[root@xx-xxxx-3 work]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"}                                                                             
etcd-2               Healthy     {"health":"true"}                                                                             
etcd-1               Healthy     {"health":"true"} 
```

### 3.4.5、同步kubectl配置文件到其他节点

```
rsync -vaz /root/.kube/config xx-xxxx-4:/root/.kube/
rsync -vaz /root/.kube/config xx-xxxx-5:/root/.kube/
```

### 3.4.6、配置kubectl子命令补全

```
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
kubectl completion bash > ~/.kube/completion.bash.inc
source '/root/.kube/completion.bash.inc'  
source $HOME/.bash_profile
```

## 3.5、部署kube-controller-manager

### 3.5.1、创建csr请求文件

```
vim kube-controller-manager-csr.json
```

```json
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "xx.xx105.223",
      "xx.xx105.224",
      "xx.xx105.225"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "JiangSu",
        "L": "SuZhou",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
```

<!--注：-->
<!--hosts 列表包含所有 kube-controller-manager 节点 IP；-->
<!--CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限-->

### 3.5.2、生成证书

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

[root@xx-xxxx-3 work]# ls kube-controller-manager*.pem
kube-controller-manager-key.pem  kube-controller-manager.pem
```

### 3.5.3、创建kube-controller-manager的kubeconfig

```shell
#设置集群参数
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://xx.xx105.231:5000 --kubeconfig=kube-controller-manager.kubeconfig
#设置客户端认证参数
kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
#设置上下文参数
kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
#设置默认上下文
kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```

### 3.5.4、创建配置文件

```shell
# vim kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--secure-port=10257 \
  --bind-address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.222.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.223.0.0/16 \
  --experimental-cluster-signing-duration=87600h \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2"
```

### 3.5.5、创建启动文件

```shell
# vim kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.5.6、同步相关文件到各个节点

```
cp kube-controller-manager*.pem /etc/kubernetes/ssl/
cp kube-controller-manager.kubeconfig /etc/kubernetes/
cp kube-controller-manager.conf /etc/kubernetes/
cp kube-controller-manager.service /usr/lib/systemd/system/
rsync -vaz kube-controller-manager*.pem xx-xxxx-4:/etc/kubernetes/ssl/
rsync -vaz kube-controller-manager*.pem xx-xxxx-5:/etc/kubernetes/ssl/
rsync -vaz kube-controller-manager.kubeconfig kube-controller-manager.conf xx-xxxx-4:/etc/kubernetes/
rsync -vaz kube-controller-manager.kubeconfig kube-controller-manager.conf xx-xxxx-5:/etc/kubernetes/
rsync -vaz kube-controller-manager.service xx-xxxx-4:/usr/lib/systemd/system/
rsync -vaz kube-controller-manager.service xx-xxxx-5:/usr/lib/systemd/system/
```

### 3.5.7、启动服务

```
systemctl daemon-reload 
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

### 3.5.8、启动遇到的问题

```
[root@xx-xxxx-4 ssl]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                  ERROR
controller-manager   Unhealthy   HTTP probe failed with statuscode: 400   
scheduler            Healthy     ok                                       
etcd-0               Healthy     {"health":"true"}                        
etcd-2               Healthy     {"health":"true"}                        
etcd-1               Healthy     {"health":"true"}  
```

如果遇到以上情况，状态是unhealthy时，服务启动文件中将

  --port=0 \
  --secure-port=10252 \
  --bind-address=127.0.0.1 \

这三行删除。

!--这三行配置的功能是:--

- <!----port=0：关闭监听 http /metrics 的请求，同时 --address 参数无效，--bind-address 参数有效-->
- <!----secure-port=10252、--bind-address=0.0.0.0: 在所有网络接口监听 10252 端口的 https /metrics 请求-->

## 3.6、部署kube-scheduler

### 3.6.1、创建csr请求文件

```json
# vim kube-scheduler-csr.json
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "xx.xx105.223",
      "xx.xx105.224",
      "xx.xx105.225"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "JiangSu",
        "L": "SuZhou",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
```

<!--注：-->
<!--hosts 列表包含所有 kube-scheduler 节点 IP；-->
<!--CN 为 system:kube-scheduler、O 为 system:kube-scheduler，kubernetes 内置的 ClusterRoleBindings system:kube-scheduler 将赋予 kube-scheduler 工作所需的权限。-->

### 3.6.2、生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

# ls kube-scheduler*.pem
```

### 3.6.3、创建kube-scheduler的kubeconfig

```shell
#设置集群参数
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://xx.xx105.231:5000 --kubeconfig=kube-scheduler.kubeconfig
#设置客户端认证参数
kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
#设置上下文参数
kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
#设置默认上下文
kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

### 3.6.4、创建配置文件

```shell
# vim kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--address=127.0.0.1 \
--kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
--leader-elect=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=2"
```

### 3.6.5、创建服务启动文件

```shell
# vim kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.6.6、同步相关文件到各个节点

```shell
cp kube-scheduler*.pem /etc/kubernetes/ssl/
cp kube-scheduler.kubeconfig /etc/kubernetes/
cp kube-scheduler.conf /etc/kubernetes/
cp kube-scheduler.service /usr/lib/systemd/system/
rsync -vaz kube-scheduler*.pem xx-xxxx-4:/etc/kubernetes/ssl/
rsync -vaz kube-scheduler*.pem xx-xxxx-5:/etc/kubernetes/ssl/
rsync -vaz kube-scheduler.kubeconfig kube-scheduler.conf xx-xxxx-4:/etc/kubernetes/
rsync -vaz kube-scheduler.kubeconfig kube-scheduler.conf xx-xxxx-5:/etc/kubernetes/
rsync -vaz kube-scheduler.service xx-xxxx-4:/usr/lib/systemd/system/
rsync -vaz kube-scheduler.service xx-xxxx-5:/usr/lib/systemd/system/
```

### 3.6.7、启动服务

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

## 3.7、docker安装（在三个worker节点上安装）

### 3.7.1、安装docker

```
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum install -y docker-ce
systemctl enable docker
systemctl start docker
docker --version
```

### 3.7.2、修改docker源和驱动

```json
# vim /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://kfwkfulq.mirror.aliyuncs.com",
        "https://2lqq34jg.mirror.aliyuncs.com",
        "https://pee6w651.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
```

```
systemctl restart docker
docker info | grep "Cgroup Driver"
```

### 3.7.3、下载依赖镜像

```
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0 k8s.gcr.io/coredns:1.7.0
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0
```

## 3.8、部署kubelet

以下操作在master1（xx-xxxx-3）上操作

### 3.8.1、创建kubelet-bootstrap.kubeconfig

```shell
BOOTSTRAP_TOKEN=$(awk -F "," '{print $1}' /etc/kubernetes/token.csv)
#设置集群参数
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://xx.xx105.231:5000 --kubeconfig=kubelet-bootstrap.kubeconfig
#设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap --token=${BOOTSTRAP_TOKEN} --kubeconfig=kubelet-bootstrap.kubeconfig
#设置上下文参数
kubectl config set-context default --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=kubelet-bootstrap.kubeconfig
#设置默认上下文
kubectl config use-context default --kubeconfig=kubelet-bootstrap.kubeconfig
#创建角色绑定
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

### 3.8.2、创建配置文件

```json
#  vim kubelet.json
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "xx.xx105.226",
  "port": 10250,
  "readOnlyPort": 10255,
  "cgroupDriver": "systemd",                     # 如果docker的驱动为systemd，处修改为systemd。此处设置很重要，否则后面node节点无法加入到集群
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletClientCertificate": true,
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "cluster.local.",
  "clusterDNS": ["10.222.0.2"]
}
```

### 3.8.3、创建启动文件

```shell
# vim kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --config=/etc/kubernetes/kubelet.json \
  --network-plugin=cni \
  --pod-infra-container-image=k8s.gcr.io/pause:3.2 \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

<!--注：-->
<!----hostname-override：显示名称，集群中唯一-->
<!----network-plugin：启用CNI-->
<!----kubeconfig：空路径，会自动生成，后面用于连接apiserver-->
<!----bootstrap-kubeconfig：首次启动向apiserver申请证书-->
<!----config：配置参数文件-->
<!----cert-dir：kubelet证书生成目录-->
<!----pod-infra-container-image：管理Pod网络容器的镜像-->

### 3.8.4、同步相关文件到各个节点

```
cp kubelet-bootstrap.kubeconfig /etc/kubernetes/
cp kubelet.json /etc/kubernetes/
cp kubelet.service /usr/lib/systemd/system/
以上步骤，如果master节点不安装kubelet，则不用执行
for i in xx-xxxx-6 xx-xxxx-7 xx-xxxx-8;do rsync -vaz kubelet-bootstrap.kubeconfig kubelet.json $i:/etc/kubernetes/;done
for i in xx-xxxx-6 xx-xxxx-7 xx-xxxx-8;do rsync -vaz ca.pem $i:/etc/kubernetes/ssl/;done
for i in xx-xxxx-6 xx-xxxx-7 xx-xxxx-8;do rsync -vaz kubelet.service $i:/usr/lib/systemd/system/;done
```

注：kubelete.json配置文件address改为各个节点的ip地址

### 3.8.5、启动服务

在各个worker节点上操作：

```
mkdir /var/lib/kubelet
mkdir /var/log/kubernetes
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

确认kubelet服务启动成功后，接着到master上Approve一下bootstrap请求。执行如下命令可以看到三个worker节点分别发送了三个 CSR 请求：

```
kubectl get csr
```

```shell
[root@xx-xxxx-3 work]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-2tLJdK41cLatOLe9G4Jdvnm1mlcZxcb53o-Q1zFEZLI   73m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-eQQyA6BkWQfRF9A7NlVMkZi2HQ0RWBjzRVCx_mj8lms   73m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-yyzyZfDlTlt88IjjZayZ2zgfuq4qA10kiqpzOGbOo4c   72m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

```

```shell
[root@xx-xxxx-3 work]# kubectl certificate approve node-csr-2tLJdK41cLatOLe9G4Jdvnm1mlcZxcb53o-Q1zFEZLI
certificatesigningrequest.certificates.k8s.io/node-csr-2tLJdK41cLatOLe9G4Jdvnm1mlcZxcb53o-Q1zFEZLI approved
[root@xx-xxxx-3 work]# kubectl certificate approve node-csr-eQQyA6BkWQfRF9A7NlVMkZi2HQ0RWBjzRVCx_mj8lms
certificatesigningrequest.certificates.k8s.io/node-csr-eQQyA6BkWQfRF9A7NlVMkZi2HQ0RWBjzRVCx_mj8lms approved
[root@xx-xxxx-3 work]# kubectl certificate approve node-csr-yyzyZfDlTlt88IjjZayZ2zgfuq4qA10kiqpzOGbOo4c
certificatesigningrequest.certificates.k8s.io/node-csr-yyzyZfDlTlt88IjjZayZ2zgfuq4qA10kiqpzOGbOo4c approved

[root@xx-xxxx-3 work]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-2tLJdK41cLatOLe9G4Jdvnm1mlcZxcb53o-Q1zFEZLI   76m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-eQQyA6BkWQfRF9A7NlVMkZi2HQ0RWBjzRVCx_mj8lms   76m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-yyzyZfDlTlt88IjjZayZ2zgfuq4qA10kiqpzOGbOo4c   76m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

[root@xx-xxxx-5 kubernetes]# kubectl get node
NAME        STATUS     ROLES    AGE     VERSION
xx-xxxx-6   NotReady   <none>   4m47s   v1.20.1
xx-xxxx-7   NotReady   <none>   3m9s    v1.20.1
xx-xxxx-8   NotReady   <none>   5m38s   v1.20.1
```

## 3.9、部署kube-proxy

### 3.9.1、创建csr请求文件

```json
# vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "SuZhou",
      "O": "k8s",
      "OU": "system"
    }
  ]
}
```

### 3.9.2、生成证书

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
ls kube-proxy*.pem
```

### 3.9.3、创建kubeconfig文件

```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://xx.xx105.231:5000 --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --client-key=kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

### 3.9.4、创建kube-proxy配置文件

```shell
# vim kube-proxy.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: xx.xx105.226
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
clusterCIDR: 10.223.0.0/16                  # 此处网段必须与网络组件网段保持一致，否则部署网络组件时会报错
healthzBindAddress: xx.xx105.226:10256
kind: KubeProxyConfiguration
metricsBindAddress: xx.xx105.226:10249
mode: "ipvs"
```

### 3.9.5、创建服务启动文件

```shell
# vim kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 3.9.6、同步文件到各个节点

```shell
cp kube-proxy*.pem /etc/kubernetes/ssl/
cp kube-proxy.kubeconfig kube-proxy.yaml /etc/kubernetes/
cp kube-proxy.service /usr/lib/systemd/system/
#master节点不安装kube-proxy，则以上步骤不用执行
for i in xx-xxxx-6 xx-xxxx-7 xx-xxxx-8;do rsync -vaz kube-proxy.kubeconfig kube-proxy.yaml $i:/etc/kubernetes/;done
for i in xx-xxxx-6 xx-xxxx-7 xx-xxxx-8;do rsync -vaz kube-proxy.service $i:/usr/lib/systemd/system/;done
```

注：配置文件kube-proxy.yaml中address修改为各节点的实际IP

### 3.9.7、启动服务

```
mkdir -p /var/lib/kube-proxy
systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
systemctl status kube-proxy
```

# 4、配置网络组件

```shell
wget https://docs.projectcalico.org/v3.14/manifests/calico.yaml
添加一下两处信息：
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP_AUTODETECTION_METHOD    #需要添加的字段，指定本地网卡
              value: "interface=ens192"        #需要添加的字段，指定本地网卡
            - name: IP
              value: "autodetect"
 ###############################################################################################
             - name: CALICO_IPV4POOL_CIDR
               value: "10.223.0.0/16"           #填写CIDR的地址，和上面kube-controller-manager和kube-proxy.yaml里设置的一样，calico启动失败。如果该字段不填写，calico-kube-controller的POD会获得192.168...的默认随机地址，pod能正常运行。          
                            
kubectl apply -f calico.yaml 
```

此时再来查看各个节点，均为Ready状态

```
 kubectl get pods -A
 kubectl get nodes
```

```
[root@xx-xxxx-3 work]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS              RESTARTS   AGE
kube-system   calico-kube-controllers-6dfcd885bf-p7nq6   0/1     ContainerCreating   0          6m3s
kube-system   calico-node-tp26p                          0/1     Running             0          6m3s
kube-system   calico-node-x6hrr                          0/1     Running             1          6m3s
kube-system   calico-node-zjhqp                          0/1     Running             0          6m3s
[root@xx-xxxx-3 work]# kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
xx-xxxx-6   Ready    <none>   45m   v1.20.1
xx-xxxx-7   Ready    <none>   44m   v1.20.1
xx-xxxx-8   Ready    <none>   46m   v1.20.1
```

# 5、部署coreDNS

下载coredns yaml文件：[ https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed](https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed)
修改yaml文件:
kubernetes cluster.local in-addr.arpa ip6.arpa
forward . /etc/resolv.conf
clusterIP为：10.222.0.2（kubelet配置文件中的clusterDNS）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local  in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.222.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```

```
kubectl apply -f coredns.yaml
```

# 6、连接本地私有仓库harbor

## 6.1、准备工作

在harbor上新建一个账号k8s（密码：xxxxxx）

修改本地docker配置文件：（所有安装docker的节点）

```shell
# vim /etc/docker/daemon.json
```

```json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "insecure-registries": ["https://registry.xxxxxx.com"],   #新增这一行，本地私有仓库的地址
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://kfwkfulq.mirror.aliyuncs.com",
        "https://2lqq34jg.mirror.aliyuncs.com",
        "https://pee6w651.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
```

docker登陆测试：

```shell
[root@test-node-1 cpuset]# docker login https://registry.xxxxxx.com
Username: k8s
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

## 6.2、开始配置

### 6.2.1、创建secret

```
kubectl create secret docker-registry secret-name --namespace=default \
--docker-server=http://192.168.242.132 --docker-username=username \
--docker-password=password --docker-email=xxx@xxx.xx...x
```

 secret-name： secret的名称
 namespace： 命名空间
 docker-server： Harbor仓库地址
 docker-username： Harbor仓库登录账号
 docker-password： Harbor仓库登录密码
 docker-email： 邮件地址

```
kubectl create secret docker-registry registry-harbor \
--namespace=default \
--docker-server=https://registry.xxxxxx.com \
--docker-username=k8s \
--docker-password=xxxxxxx \
--docker-email=xxxxxx@xxxxx.com
```

```
[root@test-master-1 work]# kubectl create secret docker-registry registry-harbor \
> --namespace=default \
> --docker-server=http://xx.xx...xx.xx... --docker-username=k8s \
> --docker-password=xxxxxxxx --docker-email=k8s@123.com
secret/registry-harbor created
```

# 7、部署ingress

## 7.1、相关文件下载

官方版本发布页面：https://github.com/kubernetes/ingress-nginx/releases

下载tar.gz包。

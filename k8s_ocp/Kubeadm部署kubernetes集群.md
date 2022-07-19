# Kubeadm部署kubernetes集群

## 1、集群规划

节点名称 | IP地址
--- | ---
test-ht-k8s-master-1 | 10.6.23.111
test-ht-k8s-master-2 | 10.6.23.112
test-ht-k8s-master-3 | 10.6.23.113
test-ht-k8s-worker-1 | 10.6.23.114
test-ht-k8s-worker-2 | 10.6.23.115
test-ht-k8s-worker-3 | 10.6.23.116
test-ht-k8s-master-lb | 10.6.23.121

## 2、服务器环境配置

### 2.1、配置阿里云镜像地址

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

### 2.2 、关闭seLinux、防火墙

```shell
systemctl stop firewalld
systemctl disable firewalld
```

```shell
setenforce 0

sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

如果有高要求的主机防护策略，不能不关闭防火墙，则需要放行需要的端口

#### 控制面

协议 | 方向 | 端口范围 | 目的 | 使用者
--- | --- | --- | --- | ---
TCP | 入站 | 6443 | kubernetes API server | 所有
TCP | 入站 | 2379-2380 | etcd server client API | kube-apiserver，etcd
TCP | 入站 | 10250 | kubelet API | 自身，控制面
TCP | 入站 | 10259 | kube-scheduler | 自身
TCP | 入站 | 10257 | kube-controller-manager | 自身
TCP | 入站 | 8443 | haproxy | haproxy
尽管etcd的端口也列举在控制面部分，但是你也可以在外部自己托管etcd集群或者自定义端口。

#### 工作节点

协议 | 方向 | 端口范围 | 目的 | 使用者
--- | --- | --- | --- | ---
TCP | 入站 | 10250 | kubelet API | 自身，控制面
TCP | 入站 | 30000-32767 | NodePort Servicest | 所有

所有默认端口都可以重新配置。当使用自定义的端口时，你需要打开这些端口来代替这里提到的默认端口。

一个常见的例子是 API 服务器的端口有时会配置为443。或者你也可以使用默认端口，把 API 服务器放到一个监听443 端口的负载均衡器后面，并且路由所有请求到 API 服务器的默认端口。

### 2.3、关闭Linux的swap分区

```shell
swapoff -a    #重启后失效
```

修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m确认swap已经关闭

```shell
[root@xx-xx-1 ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           7821         172        7379           8         269        7389
Swap:             0           0           0
```

swappiness参数调整，修改/etc/sysctl.d/k8s.conf添加下面一行：

```shell
vm.swappiness = 0
```

执行`sysctl -p /etc/sysctl.d/k8s.conf`使修改生效。

```shell
[root@xx-xx-1 ~]# sysctl -p /etc/sysctl.d/k8s.conf
vm.swappiness = 0
```

<!--为什么关闭swap？-->

<!--如果一个node性能不足时，那么应该即时的迁移到性能足够的节点，不能让swap来拖慢这个进度，这是没有意义的。-->

### 2.4、 加载IPVS模块

```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/sh
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

```

开启执行权限并执行ipvs.modules&&确认ipvs模块加载成功:
```chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4```

### 2.5、 将桥接的IPV4流量传递到iptables的链

确保 br_netfilter 模块被加载。这一操作可以通过运行 lsmod | grep br_netfilter 来完成。若要显式加载该模块，可执行 sudo modprobe br_netfilter。

为了让你的 Linux 节点上的 iptables 能够正确地查看桥接流量，你需要确保在你的 sysctl 配置中将 net.bridge.bridge-nf-call-iptables 设置为 1。例如：

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

```shell
# sysctl --system      #使配置生效
```

可以使用 `sudo cat /sys/class/dmi/id/product_uuid` 命令对 product_uuid 校验
一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。 Kubernetes 使用这些值来唯一确定集群中的节点。 如果这些值在每个节点上不唯一，可能会导致安装 失败。

### 2.6、设置时间同步

```shell
yum install ntpdate  ntp -y
```

### 2.7、安装一些其他必要组件、升级系统组件

```shell
yum -y install bash-completion ipvsadm lrzsz net-tools telnet ipset sysstat conntrack
 
# 解决系统最小化安装后，命令不能自动补全的问题
$ source /usr/share/bash-completion/bash_completion
 
$ yum update -y
```

此步骤不知道是不是必须的。

### 2.8、配置hosts文件

在每个节点上将所有节点的信息都写入hosts文件

### 2.9、安装docker

```shell
# 安装依赖包
yum install -y yum-utils
# 设置镜像的仓库
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo #默认国外

sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo #推荐使用阿里云

#docker 离线安装包地址：https://download.docker.com/linux/centos/7/x86_64/stable/Packages/

# 更新yum软件包索引
yum makecache fast
# 安装docker相关 docker-ce社区版 ee企业版
sudo yum install docker-ce docker-ce-cli containerd.io
```

```shell
[root@xx-xx-1 /]# docker version
Client: Docker Engine - Community
 Version:           20.10.8
 API version:       1.41
 Go version:        go1.16.6
 Git commit:        3967b7d
 Built:             Fri Jul 30 19:55:49 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
[root@xx-xx-1 /]# systemctl start docker
[root@xx-xx-1 /]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

```shell
yum install *.rpm --downloadonly --downloaddir=./
#安装所需的其他依赖包
rpm -ivh *.rpm
```

检查docker使用的cgroup driver，这里显示为cgroupfs。
在 Linux 上，控制组（CGroup）用于限制分配给进程的资源。

当某个 Linux 系统发行版使用 systemd 作为其初始化系统时，初始化进程会生成并使用一个 root 控制组（cgroup），并充当 cgroup 管理器。 Systemd 与 cgroup 集成紧密，并将为每个 systemd 单元分配一个 cgroup。 你也可以配置容器运行时和 kubelet 使用 cgroupfs。 连同 systemd 一起使用 cgroupfs 意味着将有两个不同的 cgroup 管理器。

单个 cgroup 管理器将简化分配资源的视图，并且默认情况下将对可用资源和使用 中的资源具有更一致的视图。 当有两个管理器共存于一个系统中时，最终将对这些资源产生两种视图。 在此领域人们已经报告过一些案例，某些节点配置让 kubelet 和 docker 使用 cgroupfs，而节点上运行的其余进程则使用 systemd; 这类节点在资源压力下 会变得不稳定。

因此需要更改设置，令容器运行时和 kubelet 使用 systemd 作为 cgroup 驱动，以此使系统更为稳定。 对于 Docker, 设置 `native.cgroupdriver=systemd` 选项。

`注意：更改已加入集群的节点的 cgroup 驱动是一项敏感的操作。 如果 kubelet 已经使用某 cgroup 驱动的语义创建了 pod，更改运行时以使用 别的 cgroup 驱动，当为现有 Pods 重新创建 PodSandbox 时会产生错误。 重启 kubelet 也可能无法解决此类问题。 如果你有切实可行的自动化方案，使用其他已更新配置的节点来替换该节点， 或者使用自动化方案来重新安装。`

因此，我们需要修改docker的本地cgroupdriver为systemd。

```shell
[root@xx-xx-1 /]# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Build with BuildKit (Docker Inc., v0.6.1-docker)
  scan: Docker Scan (Docker Inc., v0.8.0)

Server:
 Containers: 1
  Running: 1
  Paused: 0
  Stopped: 0
 Images: 2
 Server Version: 20.10.8
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs       ######################
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 io.containerd.runtime.v1.linux runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: e25210fe30a0a703442421b0f60afac609f950a3
 runc version: v1.0.1-0-g4144b63
 init version: de40ad0
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 3.10.0-957.el7.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 7.638GiB
 Name: xx-xx-1
 ID: XTAF:74C5:UZYK:EJWJ:R6HG:3G5D:AZQY:YK4K:AH67:4EFZ:UBNX:SYVO
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
```

以下操作：在三个节点上为docker配置本地cgroupdriver，以及阿里云的镜像仓库，并reload配置、重启docker。

```shell
[root@xx-xx-1 docker]# cat daemon.json 
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
```

```shell
systemctl daemon-reload
systemctl restart docker.service
```

## 3、创建master节点的高可用

### 3.1、所有master节点安装haproxy和keepalived服务

`yum -y install haproxy keepalived`

### 3.2、修改master-1上的keepalived.conf文件

```shell
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL   #此处注意router_id为负载均衡标识，在局域网内应该是唯一的。
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"         # 检测脚本路径
    interval 2
    weight 2
}
vrrp_instance VI_1 {         # 虚拟路由的标识符
    state MASTER             # 状态只有MASTER和BACKUP两种，并且要大写，MASTER为工作状态，BACKUP是备用状态
    interface eth0           # 本机网卡名
    virtual_router_id 61     # 路由ID实例，此值必须是唯一的，如果在其他环境中存在相同的实例ID值，则会发生冲突。
    priority 100             # 权重100，此节点的优先级，主节点的优先级需要比其他节点高
    advert_int 1             # 通告的间隔时间，默认1秒
    nopreempt                # 设置为不抢占 注：这个配置只能设置在backup主机上，而且这个主机优先级要比另外一台高
    authentication {         # 设置认证
        auth_type PASS       # 认证方式，类型主要有PASS、AH 两种
        auth_pass 1111       # 认证密码
    }
    virtual_ipaddress {
        10.6.23.121      # 设置虚拟VIP，可以有多个地址，每个地址占一行，不需要子网掩码，同时这个ip 必须与我们在lvs 客户端设定的vip 相一致！
    }
    track_script {
        check_haproxy       # 模块
    }
}
```

### 3.3、 修改master-2上的keepalived.conf文件

```shell
vi  /etc/keepalived/keepalived.conf 
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"         # 检测脚本路径
    interval 2
    weight 2 
}
vrrp_instance VI_1 {
    state BACKUP            # BACKUP
    interface eth0        # 本机网卡名  
    virtual_router_id 61     # 路由ID实例，此值必须是唯一的，如果在其他环境中存在相同的实例ID值，则会发生冲突。
    priority 90             # 权重90
    advert_int 5
    nopreempt                # 设置为不抢占 注：这个配置只能设置在backup主机上，而且这个主机优先级要比另外一台高
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.6.23.121     # 虚拟IP
    }
    track_script {
        check_haproxy       # 模块
    }
}
```

### 3.4、 修改master-3上的keepalived.conf文件

```shell
vi  /etc/keepalived/keepalived.conf 
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}
vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"         # 检测脚本路径
    interval 2
    weight 2 
}
vrrp_instance VI_1 {
    state BACKUP            # BACKUP
    interface eth0        # 本机网卡名  
    virtual_router_id 61     # 路由ID实例，此值必须是唯一的，如果在其他环境中存在相同的实例ID值，则会发生冲突。
    priority 80             # 权重80
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.6.23.121     # 虚拟IP
    }
    track_script {
        check_haproxy       # 模块
    }
}
```

### 3.5 配置三台master节点的haproxy

三台master节点的配置文件一致：

可以先将原初始配置文件备份，再编辑新的文件
`vim /etc/haproxy/haproxy.cfg`

```shell
global
    log /dev/log    local0
    log /dev/log    local1 notice
    pidfile     /var/run/haproxy.pid
    chroot /var/lib/haproxy
    stats socket /var/run/haproxy-admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    nbproc 1
defaults
    log     global
    timeout connect 5000
    timeout client  10m
    timeout server  10m
listen  admin_stats
    bind 0.0.0.0:10080
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:123456
    stats hide-version
    stats admin if TRUE
listen kube-master
    bind 0.0.0.0:8443
    mode tcp
    option tcplog
    balance source
    server test-ht-k8s-master-1 10.6.23.111:6443 check inter 2000 fall 2 rise 2 weight 1
    server test-ht-k8s-master-2 10.6.23.112:6443 check inter 2000 fall 2 rise 2 weight 1
    server test-ht-k8s-master-3 10.6.23.113:6443 check inter 2000 fall 2 rise 2 weight 1
```

### 3.6、编写健康监测脚本

```shell
vim /etc/keepalived/check_haproxy.sh
#!/bin/sh
# HAPROXY down
A=`ps -C haproxy --no-header | wc -l`
if [ $A -eq 0 ]
then
systmectl start haproxy
if [ ps -C haproxy --no-header | wc -l -eq 0 ]
then
killall -9 haproxy
echo "HAPROXY down" | mail -s "haproxy"
sleep 3600
fi 

fi
```

给脚本赋予执行权限`chmod u+x /etc/keepalived/check_haproxy.sh`

启动keepalived和haproxy服务并加入开机启动

`systemctl start keepalived && systemctl enable keepalived`
`systemctl start haproxy && systemctl enable haproxy`

### 3.7、查看IP，可以看到虚拟IP已经生效

```shell
[root@test-ht-k8s-master-1 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 50:6b:8d:e1:cc:4f brd ff:ff:ff:ff:ff:ff
    inet 10.6.23.111/24 brd 10.6.23.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 10.6.23.121/32 scope global eth0       #########################虚拟IP#######################
       valid_lft forever preferred_lft forever
    inet6 fe80::526b:8dff:fee1:cc4f/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:de:f2:81:c1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

## 4、安装Kubeadm、kubectl、kubelet

添加kebernetes的yum源，采用阿里云镜像源安装

```shell
[root@xx-xx-1 yum.repos.d]# cat /etc/yum.repos.d/kebernetes.repo 
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

ps: 由于官网未开放同步方式, 可能会有索引gpg检查失败的情况, 这时请用 yum install -y --nogpgcheck kubelet kubeadm kubectl 安装

或者`gpgcheck=0 repo_gpgcheck=0`

查看可以安装的kubeadmin版本号：

```shell
setenforce 0
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

安装kubelet、kubeadm、kubectl

```shell
yum install -y kubelet-1.21.6 kubeadm-1.21.6 kubectl-1.21.6 --disableexcludes=kubernetes
#--disableexcludes=kubernetes  禁止除了这个之外的别的仓库
```

查看版本：

```shell
[root@xx-xx-1 /]# kubelet --version
Kubernetes v1.22.0
[root@xx-xx-1 /]# kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.0", GitCommit:"c2b5237ccd9c0f1d600d3072634ca66cefdf272f", GitTreeState:"clean", BuildDate:"2021-08-04T18:03:20Z", GoVersion:"go1.16.6", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
[root@xx-xx-1 /]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.0", GitCommit:"c2b5237ccd9c0f1d600d3072634ca66cefdf272f", GitTreeState:"clean", BuildDate:"2021-08-04T18:02:08Z", GoVersion:"go1.16.6", Compiler:"gc", Platform:"linux/amd64"}
```

kubelet加入开机启动之后不用启动，启动也会失败，初始化集群之后集群会自动启动kubelet服务。

```shell
systemctl enable kubelet && systemctl daemon-reload
```

## 5、创建集群

### 5.1、获取kubeadm init的默认配置

`kubeadm config print init-defaults > init.defaults.yaml`

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.6.23.111    #本机IP
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: test-ht-k8s-master-1       #本主机名
  taints: 
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "10.6.23.121:8443"       #虚拟IP和haproxy端口
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers      # 镜像仓库源要根据自己实际情况修改
kind: ClusterConfiguration
kubernetesVersion: 1.21.6   #k8s版本
networking:
  dnsDomain: cluster.local
  podSubnet: "10.221.0.0/16"            #pod使用的网络
  serviceSubnet: 10.222.0.0/16          #service使用的网络
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true

```

对生成的文件进行编辑可以生成合适的配置。比如仓库地址、IP范围、版本等。

在新旧版本之间进行配置转换：`kubeadm config migrate`

列出所需的镜像列表：`kubeadm config images list`

拉取镜像到本地`kubeadm config images pull`

### 5.2、初始化集群

```shell
[root@test-ht-k8s-master-1 ~]# kubeadm init --config kubeadm-config.yaml 
[init] Using Kubernetes version: v1.21.6
[preflight] Running pre-flight checks
[WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local test-ht-k8s-master-1] and IPs [10.48.0.1 10.6.23.111 10.6.23.121]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost test-ht-k8s-master-1] and IPs [10.6.23.111 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost test-ht-k8s-master-1] and IPs [10.6.23.111 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 15.013427 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node test-ht-k8s-master-1 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node test-ht-k8s-master-1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

kubeadm join 10.6.23.121:16443 --token abcdef.0123456789abcdef \
	                            --discovery-token-ca-cert-hash sha256:ef4dde83bc50707731e72fd46b6d73add7b2ce71bdd576add621b22c1567c407 \
                              --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.6.23.121:16443 --token abcdef.0123456789abcdef \
                            --discovery-token-ca-cert-hash sha256:ef4dde83bc50707731e72fd46b6d73add7b2ce71bdd576add621b22c1567c407 

```

上面的执行日志的最后两行，就是用于node节点加入到集群的命令，有效期为24小时，超过24小时如果需要加入新的节点，需要使用下面的命令在master重新生成。

`kubeadm token create --print-join-command`

### 5.3、证书分配

由于kubeadm默认使用CA证书，所以需要为kubectl配置证书才能访问master
在其他两个master节点创建以下目录：
`mkdir -p /etc/kubernetes/pki/etcd`
将主master节点的证书分别复制到其他master节点

```shell
scp /etc/kubernetes/pki/ca.* root@10.6.23.112:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* root@10.6.23.112:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* root@10.6.23.112:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* root@10.6.23.112:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf root@10.6.23.112:/etc/kubernetes/

scp /etc/kubernetes/pki/ca.* root@10.6.23.113:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* root@10.6.23.113:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* root@10.6.23.113:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* root@10.6.23.113:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf root@10.6.23.113:/etc/kubernetes/
```

把 master 主节点的 admin.conf 复制到其他 node 节点

```shell
scp /etc/kubernetes/admin.conf root@10.6.23.114:/etc/kubernetes/
scp /etc/kubernetes/admin.conf root@10.6.23.115:/etc/kubernetes/
scp /etc/kubernetes/admin.conf root@10.6.23.116:/etc/kubernetes/
```

### 5.4、将其他节点加入集群

master节点加入集群：

```shell
kubeadm join 10.6.23.121:16443 --token abcdef.0123456789abcdef \
	                             --discovery-token-ca-cert-hash sha256:ef4dde83bc50707731e72fd46b6d73add7b2ce71bdd576add621b22c1567c407 \
	                             --control-plane
```

node节点加入集群：

```shell
kubeadm join 10.6.23.121:16443 --token abcdef.0123456789abcdef \
                            --discovery-token-ca-cert-hash sha256:ef4dde83bc50707731e72fd46b6d73add7b2ce71bdd576add621b22c1567c407
```

### 5.5、所有master节点执行一下命令，node节点随意(执行后可以使用kubectl命令)

如果是root用户则也可以通过设置环境变量KUBECONFIG完成kubuctl的设置：
我们用的是root用户，所以采用此方法：

```shell
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source .bash_profile
```
<!--注：如果不操作的话后面执行kubectl命令的时候会报错：-->

```shell
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

非root用户执行以下命令：

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

<!--将master节点的/etc/kubernetes/admin.conf文件拷贝到node节点相同的位置，再同样执行上面的授权操作，否则node节点也不能正常执行kubectl命令。-->

然后就可以使用kubectl命令对集群进行操作了。例如：

```shell
[root@test-ht-k8s-master-1 ~]# kubectl get nodes
NAME                   STATUS     ROLES                  AGE    VERSION
test-ht-k8s-master-1   NotReady   control-plane,master   3h2m   v1.21.6
test-ht-k8s-master-2   NotReady   control-plane,master   129m   v1.21.6
test-ht-k8s-master-3   NotReady   control-plane,master   127m   v1.21.6
test-ht-k8s-worker-1   NotReady   <none>                 114m   v1.21.6
test-ht-k8s-worker-2   NotReady   <none>                 110m   v1.21.6
test-ht-k8s-worker-3   NotReady   <none>                 110m   v1.21.6
```

此时各个点的状态还是NotReady，那是因为CNI网络组件还没有部署。


## 6、 按装CNI网络插件

下载 `https://docs.projectcalico.org/manifests/calico.yaml` 文件

`wget https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate`


执行

```shell
[root@xx-xx-1 ~]# kubectl apply -f /root/calico.yaml
configmap/calico-config unchanged
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org configured
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org configured
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrole.rbac.authorization.k8s.io/calico-node unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-node unchanged
daemonset.apps/calico-node configured
serviceaccount/calico-node unchanged
deployment.apps/calico-kube-controllers unchanged
serviceaccount/calico-kube-controllers unchanged
poddisruptionbudget.policy/calico-kube-controllers configured
[root@xx-xx-1 ~]# kubectl get node
NAME        STATUS   ROLES                  AGE   VERSION
xx-xx-1   Ready    control-plane,master   98m   v1.21.3
xx-xx-2   Ready    <none>                 27m   v1.21.3
```

我们在看节点信息的时候就可以看到已经是ready状态了

```shell
[root@xx-xx-1 kubernetes]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS             RESTARTS   AGE
kube-system   calico-kube-controllers-58497c65d5-qkwdl   1/1     Running            0          27m
kube-system   calico-node-9qjtl                          1/1     Running            0          27m
kube-system   calico-node-wrp77                          1/1     Running            0          27m
kube-system   coredns-59d64cd4d4-5blgg                   0/1     ImagePullBackOff   0          116m
kube-system   coredns-59d64cd4d4-q79zd                   0/1     ImagePullBackOff   0          116m
kube-system   etcd-xx-xx-1                             1/1     Running            1          116m
kube-system   kube-apiserver-xx-xx-1                   1/1     Running            1          116m
kube-system   kube-controller-manager-xx-xx-1          1/1     Running            1          116m
kube-system   kube-proxy-bvmdr                           1/1     Running            1          116m
kube-system   kube-proxy-kfmkl                           1/1     Running            0          45m
kube-system   kube-scheduler-xx-xx-1                   1/1     Running            1          116m
```

查看pod状态的时候发现有两个镜像还是不能正常下载

```shell
[root@xx-xx-1 kubernetes]# docker pull registry.aliyuncs.com/google_containers/coredns:v1.8.0
Error response from daemon: manifest for registry.aliyuncs.com/google_containers/coredns:v1.8.0 not found: manifest unknown: manifest unknown
```

把v去掉就可以了，然后在node节点也同样重新下载和重命名

```shell
[root@xx-xx-1 kubernetes]# docker pull registry.aliyuncs.com/google_containers/coredns:1.8.0
1.8.0: Pulling from google_containers/coredns
c6568d217a00: Pull complete 
5984b6d55edf: Pull complete 
Digest: sha256:cc8fb77bc2a0541949d1d9320a641b82fd392b0d3d8145469ca4709ae769980e
Status: Downloaded newer image for registry.aliyuncs.com/google_containers/coredns:1.8.0
registry.aliyuncs.com/google_containers/coredns:1.8.0
```

```shell
docker tag registry.aliyuncs.com/google_containers/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns:v1.8.0
```

```shell
docker rmi registry.aliyuncs.com/google_containers/coredns:1.8.0
```

```shell
[root@xx-xx-1 kubernetes]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-58497c65d5-qkwdl   1/1     Running   0          33m
kube-system   calico-node-9qjtl                          1/1     Running   0          33m
kube-system   calico-node-wrp77                          1/1     Running   0          33m
kube-system   coredns-59d64cd4d4-5blgg                   1/1     Running   0          121m
kube-system   coredns-59d64cd4d4-q79zd                   1/1     Running   0          121m
kube-system   etcd-xx-xx-1                             1/1     Running   1          121m
kube-system   kube-apiserver-xx-xx-1                   1/1     Running   1          121m
kube-system   kube-controller-manager-xx-xx-1          1/1     Running   1          121m
kube-system   kube-proxy-bvmdr                           1/1     Running   1          121m
kube-system   kube-proxy-kfmkl                           1/1     Running   0          51m
kube-system   kube-scheduler-xx-xx-1                   1/1     Running   1          121m
```

这个时候就可以看到全部正常了

到此，我们使用kubeadmin创建k8s集群的工作就结束了。

## 7、清理

如果在安装部署过程中出现问题，需要清理之前安装的组件则需要按照以下方法进行

### 删除节点

`kubectl delete node <node name>`

使用适当的凭证与控制平面节点通信，运行：

`kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets`

### 重置kubeadmn安装状态

`kubeadm reset`

重置过程不会重置或清除 iptables 规则或 IPVS 表。如果你希望重置 iptables，则必须手动进行：

`iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X`
`sysctl net.bridge.bridge-nf-call-iptables=1`

如果要重置 IPVS 表，则必须运行以下命令：

`ipvsadm -C`

### 删除网络残留

`rm -rf /etc/cni/net.d/`


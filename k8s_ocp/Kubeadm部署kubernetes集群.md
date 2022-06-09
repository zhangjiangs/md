

# Kubeadm部署kubernetes集群

## 1、配置镜像源

配置阿里云镜像：

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

## 2、环境配置

### 2.1 、关闭seLinux、防火墙

```
systemctl stop firewalld
systemctl disable firewalld
```

```shell
setenforce 0

sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

### 2.2 、关闭Linux的swap分区：

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

```
vm.swappiness = 0
```

执行`sysctl -p /etc/sysctl.d/k8s.conf`使修改生效。

```shell
[root@xx-xx-1 ~]# sysctl -p /etc/sysctl.d/k8s.conf
vm.swappiness = 0
```

<!--为什么关闭swap？-->

<!--如果一个node性能不足时，那么应该即时的迁移到性能足够的节点，不能让swap来拖慢这个进度，这是没有意义的。-->

### 2.3、 将桥接的IPV4流量传递到iptables的链

```
[root@xx-xx-1 /]# cat etc/sysctl.d/k8s.conf 
vm.swappiness=0

net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

```shell
# sysctl --system      #使配置生效
```

### 2.4、 加载IPVS模块

```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
```

依次执行以上命令

### 2.5、设置时间同步

```
yum install ntpdate -y
```

### 2.6、安装一些其他必要组件、升级系统组件

```
yum -y install bash-completion ipvsadm lrzsz net-tools telnet
 
# 解决系统最小化安装后，命令不能自动补全的问题
$ source /usr/share/bash-completion/bash_completion
 
$ yum update -y
```

此步骤不知道是不是必须的。

## 3、安装docker

```shell
# 安装依赖包
yum install -y yum-utils
# 设置镜像的仓库
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo #默认国外

sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo #推荐使用阿里云
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

```
yum install *.rpm --downloadonly --downloaddir=./
#安装所需的其他依赖包
rpm -ivh *.rpm
```

检查docker使用的cgroup driver，这里显示为cgroupfs。

但是由于后面k8s集群在初始化的时候，会检测到docker使用了cgroupfs作为cgroup的驱动，而k8s推荐使用systemd作为cgroup的驱动。

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

```
systemctl daemon-reload
systemctl restart docker.service
```

## 4、安装Kubeadm

采用阿里云镜像源安装

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

安装kubelet、kubeadm、kubectl

```shell
yum install -y kubelet-1.21.3 kubeadm-1.21.3 kubectl-1.21.3 --disableexcludes=kubernetes
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

启动kubelet服务并设置为开机启动：

```shell
systemctl start kubelet
systemctl enable kubelet	
```

获取kubeadm init的默认配置：

```shell
kubeadm config print init-defaults > init.defaults.yaml
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
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
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: 1.22.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

获取kubeadm join的默认参数配置内容：

```shell
kubeadm config print join-defaults > join.defaults.yaml
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  bootstrapToken:
    apiServerEndpoint: kube-apiserver:6443
    token: abcdef.0123456789abcdef
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: abcdef.0123456789abcdef
kind: JoinConfiguration
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  imagePullPolicy: IfNotPresent
  name: xx-xx-1
  taints: null
```

对生成的文件进行编辑可以生成合适的配置。比如仓库地址、IP范围、版本等。

在新旧版本之间进行配置转换：

```
kubeadm config migrate
```

列出所需的镜像列表：

```
kubeadm config images list
```

拉取镜像到本地

```
kubeadm config images pull
```

## 5、创建集群

```shell
[root@xx-xx-1 ~]# kubeadm init    \
--apiserver-advertise-address=xx.xx105.221    \
--image-repository registry.aliyuncs.com/google_containers   \
--kubernetes-version v1.21.3   \
--service-cidr=10.220.0.0/16    \
--pod-network-cidr=10.221.0.0/16  \
--ignore-preflight-errors=all

[init] Using Kubernetes version: v1.21.3
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "xx-xx-1" could not be reached
	[WARNING Hostname]: hostname "xx-xx-1": lookup xx-xx-1 on xx.xx205.1:53: read udp xx.xx105.221:59862->xx.xx205.1:53: i/o timeout
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
	[WARNING ImagePull]: failed to pull image registry.aliyuncs.com/google_containers/coredns:v1.8.0: output: Error response from daemon: manifest for registry.aliyuncs.com/google_containers/coredns:v1.8.0 not found: manifest unknown: manifest unknown
, error: exit status 1
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [xx-xx-1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.220.0.1 xx.xx105.221]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [xx-xx-1 localhost] and IPs [xx.xx105.221 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [xx-xx-1 localhost] and IPs [xx.xx105.221 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
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
[apiclient] All control plane components are healthy after 16.002548 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node xx-xx-1 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node xx-xx-1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: s64i8y.tv16uysn83xr85us
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join xx.xx105.221:6443 --token s64i8y.tv16uysn83xr85us \
	--discovery-token-ca-cert-hash sha256:09177019b944ba1a14aeb716b6e03811c2be4030111f82174c8081a9c8b52d38 
```

上面的执行日志的最后两行，就是用于node节点加入到集群的命令，有效期为24小时，超过24小时如果需要加入新的节点，需要使用下面的命令在master重新生成。

```
kubeadm token create --print-join-command
```

由于kubeadm默认使用CA证书，所以需要为kubectl配置证书才能访问master

将admin.conf配置文件复制到HOME目录的.kube子目录下，命令如下：

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

如果是root用户则也可以通过设置环境变量KUBECONFIG完成kubuctl的设置：

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

<!--注：如果不操作的话后面执行kubectl命令的时候会报错：-->

```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

<!--将master节点的/etc/kubernetes/admin.conf文件拷贝到node节点相同的位置，再同样执行上面的授权操作，否则node节点也不能正常执行kubectl命令。-->

然后就可以使用kubectl命令对集群进行操作了。例如：

```
[root@xx-xx-1 ~]# kubectl -n kube-system get configmap
NAME                                 DATA   AGE
coredns                              1      31m
extension-apiserver-authentication   6      31m
kube-proxy                           2      31m
kube-root-ca.crt                     1      31m
kubeadm-config                       2      31m
kubelet-config-1.21                  1      31m
```

```
[root@xx-xx-1 ~]# kubectl get node
NAME        STATUS     ROLES                  AGE   VERSION
xx-xx-1   NotReady   control-plane,master   32m   v1.21.3
```

通过kubectl命令来查询我们创建的主节点了，此时主节点的状态还是NotReady，那是因为calico网络组件还没有部署。

将node节点加入集群：

首先按照上面的步骤安装好环境，从节点无需安装kubectl

```
# 从节点
 yum install -y kubelet-1.21.3 kubeadm-1.21.3 
 systemctl enable kubelet.service
```

安装好kubelet-1.21.3 kubeadm-1.21.3 kubectl-1.21.3后执行：

```
kubeadm join xx.xx105.221:6443 --token s64i8y.tv16uysn83xr85us --discovery-token-ca-cert-hash sha256:09177019b944ba1a14aeb716b6e03811c2be4030111f82174c8081a9c8b52d38
```

成功后再去主节点看，就能看到新加入的节点了

```
[root@xx-xx-1 ~]# kubectl get node
NAME        STATUS     ROLES                  AGE   VERSION
xx-xx-1   NotReady   control-plane,master   71m   v1.21.3
xx-xx-2   NotReady   <none>                 50s   v1.21.3
```

按装CNI网络插件：

下载https://docs.projectcalico.org/manifests/calico.yaml文件

然后将其中的

```
apiVersion: policy/v1beta1
 
修改为：
 
apiVersion: policy/v1
```

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

```
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

```
[root@xx-xx-1 kubernetes]# docker pull registry.aliyuncs.com/google_containers/coredns:v1.8.0
Error response from daemon: manifest for registry.aliyuncs.com/google_containers/coredns:v1.8.0 not found: manifest unknown: manifest unknown
```

把	v去掉就可以了，然后在node节点也同样重新下载和重命名

```
[root@xx-xx-1 kubernetes]# docker pull registry.aliyuncs.com/google_containers/coredns:1.8.0
1.8.0: Pulling from google_containers/coredns
c6568d217a00: Pull complete 
5984b6d55edf: Pull complete 
Digest: sha256:cc8fb77bc2a0541949d1d9320a641b82fd392b0d3d8145469ca4709ae769980e
Status: Downloaded newer image for registry.aliyuncs.com/google_containers/coredns:1.8.0
registry.aliyuncs.com/google_containers/coredns:1.8.0
```

```
docker tag registry.aliyuncs.com/google_containers/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns:v1.8.0
```

```
docker rmi registry.aliyuncs.com/google_containers/coredns:1.8.0
```

```
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


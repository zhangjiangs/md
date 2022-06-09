# 1、Kubernetes入门

## 1.1、Kubernetes是什么

Kubernetes是Google开源的一个容器编排引擎，它支持自动化部署、大规模可伸缩、应用容器化管理。

在Kubernetes中，我们可以创建多个容器，每个容器里面运行一个应用实例，然后通过内置的负载均衡策略，实现对这一组应用实例的管理、发现、访问，而这些细节都不需要运维人员去进行复杂的手工配置和处理。

云计算时代的操作系统。

## 1.2、Kubernetes基本概念

### 资源对象的概念：

Kubernetes中的基本概念和术语大多都是围绕资源对象（Resource Object）来说的，资源对象总体上可以分为2类：

（1）、某种资源的对象，例如节点（Node）、Pod、服务（service）、存储卷（Volume）

（2）、与资源对象相关的事物与操作，例如标签（Label）、注解（Annotation）、命名空间（Namespace）、部署（Deployment）、HPA、PVC等。

### 资源类

### 1.2.1、Master

Kubernetes里的Master指的是集群控制节点，在每个Kubernetes集群里都需要有一个Master来负责整个集群的管理和控制，基本上Kubernetes的所有控制命令都发给它，它负责具体的执行过程，我们后面执行的所有命令基本都是在Master上运行的。

在Master上运行着以下关键进程。

◎ Kubernetes API Server（kube-apiserver）：提供了HTTP Rest接口的关键服务进程，是Kubernetes里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程。

◎ Kubernetes Controller Manager（kube-controller-manager）：Kubernetes里所有资源对象的自动化控制中心，可以将其理解为资源对象的“大总管”。

◎ Kubernetes Scheduler（kube-scheduler）：负责资源调度（Pod调度）的进程，相当于公交公司的“调度室”。

另外，在Master上通常还需要部署etcd服务，因为Kubernetes里的所有资源对象的数据都被保存在etcd中。

### 1.2.2、Node

除了Master，Kubernetes集群中的其他机器被称为Node，在较早的版本中也被称为Minion。与Master一样，Node可以是一台物理主机，也可以是一台虚拟机。Node是Kubernetes集群中的工作负载节点，每个Node都会被Master分配一些工作负载（Docker容器），当某个Node宕机时，其上的工作负载会被Master自动转移到其他节点上。

在每个Node上都运行着以下关键进程。

◎ kubelet：负责Pod对应的容器的创建、启停等任务，同时与Master密切协作，实现集群管理的基本功能。

◎ kube-proxy：实现Kubernetes Service的通信与负载均衡机制的重要组件。

◎ Docker Engine（docker）：Docker引擎，负责本机的容器创建和管理工作。

Node可以在运行期间动态增加到Kubernetes集群中，前提是在这个节点上已经正确安装、配置和启动了上述关键进程，在默认情况下kubelet会向Master注册自己，这也是Kubernetes推荐的Node管理方式。一旦Node被纳入集群管理范围，kubelet进程就会定时向Master汇报自身的情报，例如操作系统、Docker版本、机器的CPU和内存情况，以及当前有哪些Pod在运行等，这样Master就可以获知每个Node的资源使用情况，并实现高效均衡的资源调度策略。而某个Node在超过指定时间不上报信息时，会被Master判定为“失联”，Node的状态被标记为不可用（Not Ready），随后Master会触发“工作负载大转移”的自动流程。

相关命令：

```shell
[root@hostname ~]# kubectl get nodes
#查看集群情况
[root@hostname ~]# kubectl describe node infra02.dev.xx-xxx.com
#查看具体节点信息
```

◎ Node的基本信息：名称、标签、创建时间等。

◎ Node当前的运行状态：Node启动后会做一系列的自检工作，比如磁盘空间是否不足（DiskPressure）、内存是否不足（MemoryPressure）、网络是否正常（NetworkUnavailable）、PID资源是否充足（PIDPressure）。在一切正常时设置Node为Ready状态（Ready=True），该状态表示Node处于健康状态，Master将可以在其上调度新的任务了（如启动Pod）。

◎ Node的主机地址与主机名。

◎ Node上的资源数量：描述Node可用的系统资源，包括CPU、内存数量、最大可调度Pod数量等。

◎ Node可分配的资源量：描述Node当前可用于分配的资源量。

◎ 主机系统信息：包括主机ID、系统UUID、Linux kernel版本号、操作系统类型与版本、Docker版本号、kubelet与kube-proxy的版本号等。

◎ 当前运行的Pod列表概要信息。

◎ 已分配的资源使用概要信息，例如资源申请的最低、最大允许使用量占系统总量的百分比。

◎ Node相关的Event信息。

如果某个node存在问题，比如存在安全隐患、硬件资源不足或者计划淘汰，我们就可以给该node打上一个特殊的标签——污点（Taint），这样可以避免新的容器被调度到该node上，而如果某些Pod可以容忍（Toleration）某种污点的存在，则可以继续将其调度到该Node上，Taint和Toleration这两个术语属于Kubernetes调度相关的重要概念。

### 1.2.3、Namespace命名空间

在大多数情况下Namespace用于实现多租户的资源隔离，可以创建多个Namespace，它们之间相互独立存在，属于不同命名空间的资源对象从逻辑上相互隔离。从而在管理中在逻辑上形成不同的分组方便管理。

在Kubernetes集群安装完成且正常运行之后，Master会自动创建两个命名空间：

a、一个是默认的default，用户创建的资源对象如果没有指定命名空间，则被默认存放到default下

b、另一个是系统及的kube-system，系统相关的资源对象比如网络组件、DNS组件、监控类组件等都被安装在此Namespace下。

## 应用类

### 1.2.4 、Service和Pod

应用类相关的资源对象主要是围绕Service和Pod这两个核心对象展开。

一般来说Service指的是无状态服务，通常由多个程序副本提供服务，在特殊情况下也会是有状态的单实例服务，比如MySQL这种数据存储类的服务。

与我们常规理解的服务不同，在kubernetes里的service具有一个全局唯一的虚拟ClusterIP地址，而且在Service整个生命周期中这个ClusterIP都不会改变，客户端可以通过这个虚拟IP+端口直接访问该服务，再通过部署集群中的DNS服务，就可以实现Service Name到ClusterIP的DNS映射功能，我们只要使用影身的域名就能完成到目标服务的访问请求，这就解决了一个问题：服务发现。同时解决负载均衡和故障自动恢复的高级特性。

通过分析、识别并建模系统中的所有服务为微服务——Kubernetes Service，我们的系统最终由多个提供不同业务能力而又彼此独立的微服务单元组成的，服务之间通过TCP/IP进行通信，从而形成了强大而又灵活的弹性网格，拥有强大的分布式能力、弹性扩展能力、容错能力，程序架构也变得简单和直观许多，如图所示。

![img](https://preview.cloud.189.cn/image/imageAction?param=5B155D42C871116330554E2656B8E3D6A52A45960A229C50D22A3190DECE91A806E60F797032C20D725216246760CED72EDE72FA2C151327BDF3CACE27F90D05A6AF63A769187CF707723E68B4EBD58CBF235A71A6FEF894C19FCE38859D1EC8AD69B3AB59447113F2E377B7B4100271)

Service服务也是Kubernetes里的核心资源对象之一，Kubernetes里的每个Service其实就是我们经常提起的微服务架构中的一个微服务，之前讲解Pod、RC等资源对象其实都是为讲解KubernetesService做铺垫的。下图显示了Pod、RC与Service的逻辑关系。

![img](https://preview.cloud.189.cn/image/imageAction?param=E2BF8E2BADF6523C621AD9CB3D8B77DE5CDCEA0C4BB29EE6CF0AB7468560BF9450CA0033A21E39FDD52D629C935A57CF40AF6C57B44B129BDB158F9C0216C3178C7C7C3BBFABDB8AED001EFC1CC7A5933C0FDB68242DB9A1A3DA646066C623D18372B017BD2D7081016995345151066C)

Kubernetes的Service定义了一个服务的访问入口地址，前端的应用（Pod）通过这个入口地址访问其背后的一组由Pod副本组成的集群实例，Service与其后端Pod副本集群之间则是通过Label Selector来实现无缝对接的。RC的作用实际上是保证Service的服务能力和服务质量始终符合预期标准。

**接下来说说与Service密切相关的核心资源对象——Pod**

Pod是Kubernetes最重要的基本概念，每个Pod都有一个特殊的被称为“根容器”的Pause容器。Pause容器对应的镜像属于Kubernetes平台的一部分，除了Pause容器，每个Pod还包含一个或多个紧密相关的用户业务容器。

为什么Kubernetes会设计出一个全新的Pod的概念并且Pod有这样特殊的组成结构？

原因之一：为多进程之间的协作提供一个抽象模型，使用Pod作为基本的调度、复制等管理工作的最小单位，让多个应用进程能一起有效的调度和伸缩。

原因之二：Pod里的多个业务容器共享Pause容器的IP，共享Pause容器挂接的Volume，这样既简化了密切关联的业务容器之间的通信问题，也很好地解决了它们之间的文件共享问题。

Kubernetes为每个Pod都分配了唯一的IP地址，称之为Pod IP，一个Pod里的多个容器共享Pod IP地址。Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟二层网络技术来实现，例如Flannel、Open vSwitch等。

Pod其实有两种类型：普通的Pod及静态Pod（Static Pod）。后者比较特殊，它并没被存放在Kubernetes的etcd存储里，而是被存放在某个具体的Node上的一个具体文件中，并且只在此Node上启动、运行。而普通的Pod一旦被创建，就会被放入etcd中存储，随后会被Kubernetes Master调度到某个具体的Node上并进行绑定（Binding），随后该Pod被对应的Node上的kubelet进程实例化成一组相关的Docker容器并启动。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问题并且重新启动这个Pod（重启Pod里的所有容器），如果Pod所在的Node宕机，就会将这个Node上的所有Pod重新调度到其他节点上。Pod、容器与Node的关系如图所示：

![img](https://preview.cloud.189.cn/image/imageAction?param=F58EB732EB52799B84184B47AD1D1493CCC6CEEAEB13FA65E39A289A563D8B423466CB10150E53D89E993E5B8A2D78EC8114E04AA3B17FDF12303791141181342C5F7B29A4C2A14921302EBF01FDF893F7BB3583C72351F60D498BEF55B48D397042691541BB7269F7587A5AF04D6FA7)

每个Pod都可以对其能使用的服务器上的计算资源设置限额，当前可以设置限额的计算资源有CPU与Memory两种，其中CPU的资源单位为CPU（Core）的数量，是一个绝对值而非相对值。

对于绝大多数容器来说，一个CPU的资源配额相当大，所以在Kubernetes里通常以千分之一的CPU配额为最小单位，用m来表示。通常一个容器的CPU配额被定义为100～300m，即占用0.1～0.3个CPU。由于CPU配额是一个绝对值，所以无论在拥有一个Core的机器上，还是在拥有48个Core的机器上，100m这个配额所代表的CPU的使用量都是一样的。与CPU配额类似，Memory配额也是一个绝对值，它的单位是内存字节数。在Kubernetes里，一个计算资源进行配额限定时需要设定以下两个参数。

◎ Requests：该资源的最小申请量，系统必须满足要求。

◎ Limits：该资源最大允许使用的量，不能被突破，当容器试图使用超过这个量的资源时，可能会被Kubernetes“杀掉”并重启。

通常，我们会把Requests设置为一个较小的数值，符合容器平时的工作负载情况下的资源需求，而把Limit设置为峰值负载情况下资源占用的最大量。

```shell
kubectl describe pod $podname   #查看pod信息
kubectl get pod $podname -o yaml    #查看pod的yaml文件内容
```

**Pod的IP加上容器的端口**（containerPort）组成了另一个重要的概念——Endpoint，代表此Pod里的一个服务进程的对外通信地址，一个Pod也存在具有多个Endpoint的情况，比如我们吧Tomcat定义为一个Pod的时候，可以对外暴露管理端口和服务端口这两个Endpoint。

Docker Volume的在概念在kubernetes中对应Pod Volume，它被挂载在Pod上，然后被各个容器挂载到自己的文件系统中。Volume简单来说就是被挂载到Pod里的文件目录。

### 1.2.5、Label与标签选择器

Label（标签）是Kubernetes系统中另外一个核心概念。一个Label是一个key=value的键值对，其中key与value由用户自己指定。Label可以被附加到各种资源对象上，例如Node、Pod、Service、RC等，一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上。Label通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。通过定义多个不同的Label来实现多维度的资源分组管理功能，以便灵活、方便的进行资源分配、调度、配置、部署等管理工作。

Label相当于我们熟悉的“标签”。给某个资源对象定义一个Label，就相当于给它打了一个标签，随后可以通过**Label Selector（标签选择器）**查询和筛选拥有某些Label的资源对象，Kubernetes通过这种方式实现了类似SQL的简单又通用的对象查询机制。

可以通过多个Label Selector表达式的组合实现复杂的条件选择，多个表达式之间用“，”进行分隔即可，几个条件之间是“AND”的关系，即同时满足多个条件，比如下面的例子：

```
name=redis-slave,env!=production
name not in (php-frontend,php-php),env!=production
```

![img](https://preview.cloud.189.cn/image/imageAction?param=462989A9B9538C7EF67ADA3283AD47709F6E81641913AC5E9332E9296091D310EB6628326321228A9BDF2D5448D457F81990F9C666145C954425A8B0A415AB4B0EF5DB27B5963A54F6665AD6A82BB546071E2F3CC5840AFC48864D546E7608C6BAF2E24CAAB1ECD3FBCF30860EDED456)

matchLabels用于定义一组Label，与直接写在Selector中的作用相同；matchExpressions用于定义一组基于集合的筛选条件，可用的条件运算符包括In 、 NotIn 、 Exists和DoesNotExist。

如果同时设置了matchLabels和matchExpressions，则两组条件为AND关系，即需要同时满足所有条件才能完成Selector的筛选。

Label Selector在Kubernetes中的重要使用场景如下。

◎ kube-controller进程通过在资源对象RC上定义的Label Selector来筛选要监控的Pod副本数量，使Pod副本数量始终符合预期设定的全自动控制流程。

◎ kube-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立每个Service到对应Pod的请求转发路由表，从而实现Service的智能负载均衡机制。

◎ 通过对某些Node定义特定的Label，并且在Pod定义文件中使用NodeSelector这种标签调度策略，kube-scheduler进程可以实现Pod定向调度的特性。

例子：假设为Pod定义了3个Label：release、env和role，不同的Pod定义了不同的Label值，如图所示，如果设置“role=frontend”的Label Selector，则会选取到Node 1和Node 2上的Pod。

![img](https://preview.cloud.189.cn/image/imageAction?param=A5D2DB29059E708FB0D3B6CB653CDD19AA272243FC127B55B4EA07BAAC801C21FF5472CB483D07B515B5EC42F10855B82F7770D27E3C02269AFAEA62AB1F21B16681E6FDCF9CE4F4FD1256E382C9D9B99910D336D9171747DDFBC637C7AAEAD9EF7056BC9A9D0EE6D8E265EF500C8078)

如果设置“release=beta”的Label Selector，则会选取到Node 2和Node 3上的Pod，如图所示:

![img](https://preview.cloud.189.cn/image/imageAction?param=945336990E684BFAD1E0A78A779B08373B42AD50722D9FD6B779DBC704B371AFE8351FC15F30E624602481B15E6D7803CB0B955669589FF88A50AD81772759D30C20CFEE68F63B43668D5D478DEBBC333930920FF94CF7184B83B2B0ADEF4DE84EABB9DE61C19E26E74F6500ABE5014A)

### 1.2.6、Replication Controller

RC是Kubernetes系统中的核心概念之一，简单来说，它其实定义了一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合某个预期值，所以RC的定义包括如下几个部分。

◎ Pod期待的副本数量。

◎ 用于筛选目标Pod的Label Selector。

◎ 当Pod的副本数量小于预期数量时，用于创建新Pod的Pod模板（template）。

在我们定义了一个RC并将其提交到Kubernetes集群中后，Master上的Controller Manager组件就得到通知，定期巡检系统中当前存活的目标Pod，并确保目标Pod实例的数量刚好等于此RC的期望值，如果有过多的Pod副本在运行，系统就会停掉一些Pod，否则系统会再自动创建一些Pod。可以说，通过RC，Kubernetes实现了用户应用集群的高可用性，并且大大减少了系统管理员在传统IT环境中需要完成的许多手工运维工作（如主机监控脚本、应用监控脚本、故障恢复脚本等）。

我们可以根据replicas的值来定义Pod的运行数量，此外，在运行时，我们可以通过修改RC的副本数量，来实现Pod的动态缩放（Scaling），这可以通过执行kubectl scale命令来一键完成：

```
kubectl scale rc redis-salve --replicas=3
```

需要注意的是，删除RC并不会影响通过该RC已创建好的Pod。为了删除所有Pod，可以设置replicas的值为0，然后更新该RC。另外，kubectl提供了stop和delete命令来一次性删除RC和RC控制的全部Pod。

应用升级时，通常会使用一个新的容器镜像版本替代旧版本。我们希望系统平滑升级，比如在当前系统中有10个对应的旧版本的Pod，则最佳的系统升级方式是旧版本的Pod每停止一个，就同时创建一个新版本的Pod，在整个升级过程中此消彼长，而运行中的Pod数量始终是10个，几分钟以后，当所有的Pod都已经是新版本时，系统升级完成。通过RC机制，Kubernetes很容易就实现了这种高级实用的特性，被称为“滚动升级”（Rolling Update）。

Replication Controller由于与Kubernetes代码中的模块Replication Controller同名，同时“Replication Controller”无法准确表达它的本意，所以在Kubernetes 1.2中，升级为另外一个新概念——Replica Set，官方解释其为“下一代的RC”。Replica Set与RC当前的唯一区别是，Replica Sets支持基于集合的Label selector（Set-based selector），而RC只支持基于等式的LabelSelector（equality-based selector），这使得Replica Set的功能更强。

kubectl命令行工具适用于RC的绝大部分命令同样适用于Replica Set。此外，我们当前很少单独使用Replica Set，它主要被Deployment这个更高层的资源对象所使用，从而形成一整套Pod创建、删除、更新的编排机制。我们在使用Deployment时，无须关心它是如何创建和维护Replica Set的，这一切都是自动发生的。

### 1.2.7、Deployment

Deployment所作的工作是通过模板（Template）自动创建指定数量的Pod。

看看Deployment的一些内容：

```shell
apiVersion: apps/v1
kind: Deployment
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: edpa-admin
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: edpa-admin
    spec:

```

几个重要的属性：

a、replicas：Pod的副本数量

b、selector：目标Pod的标签选择器

c、template：用于自动创建新Pod副本的模板

Deployment另外一个很重要的特性：自动控制。当Pod所在节点出现宕机的时候，Kubernetes会发现故障并自动创建一个新的Pod将其调度到其他合适的Node上，会按照Deployment的声明控制Pod的副本数量。

```shell
kubectl get deployments 
#查看deployment信息
kubectl describe deployment
#查看具体的deployment内容
```

Deployment资源对象其实还与ReplicaSet资源对象密切相关，Kubernetes内部会根据Deployment对象自动创建相关联的ReplicaSet对象。而且Pod的命名也是以Deployment对应的ReplicaSet对象的名称为前缀，这样就清晰的表明了一个ReplicaSet对象创建了哪些Pod，对于Pod滚动升级这种复杂的操作过程和容易排错。

### 1.2.8、Service的ClusterIP地址

ClusterIP是一种虚拟IP地址：

a、clusterIP地址仅仅作用于kubernetes service这个对象，并由kubernetes管理和分配（来源于clusterIP地址池），与Node和Master所在的物理网络完全无关；

b、因为没有一个“实体网络对象”来相应，所以clusterIP地址无法被ping通，它只能和service port组成一个具体的服务访问端点，单独的clusterIP不具备TCP/IP的通信基础

c、clusterIP属于kubernetes集群这个封闭空间，集群外的节点要访问这个通信端口需要做一些额外的工作。

除了正常的service以外，还有一种特殊的service：headless service。只要在service的定义中设置了clusterIP=None，就定义了一个headless service，他没有clusterIP，如果解析headless service的DNS域名，则返回的是该服务对应的全部pod的endpoint列表，这意味着客户端是直接与后端的pod建立TCP/IP连接的，没有通过clusterIP进行转发，因此通信性能更高，等同于“原生网络通信”。

### 1.2.9、service的外网访问问题

先搞明白kubernetes的三种IP：

a、Node IP：Node的IP地址

b、Pod IP：Pod的IP地址

c、service IP：service的IP地址

当使用NodePort的时候，每一个service都要占用Node的一个端口，而端口又是很有限的物理资源，那能不能让多个service共用一个对外端口呢？这就是后来又增加的Ingress资源对象所要解决的问题。在一定程度上我们可以把Ingress的实现机制理解为基于Nginx的支持虚拟主机的HTTP代理。Ingress其实只能将多个HTTP（HTTPS）的service“聚合”，通过虚拟域名或者URL Path的特征进行路由转发功能。

### 1.2.10、有状态应用集群

有状态集群应用一般有如下特性：

a、每个节点都有固定的身份ID，通过这个ID集群中的成员可以互相发现并通信；

b、集群的规模是比较固定的，集群规模不能随意变动；

c、集群中的每个节点都是有状态的，通常会持久化数据到永久存储中，每个节点在重启后都需要使用原有的持久化数据；

d、集群中成员节点的启停顺序通常是固定的；

e、如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损。

如果使用Deployment控制Pod副本数量来实现以上有状态的集群，我们发现很多特性都无法满足，比如Pod的名称就是随机产生的，所以这时候引入了statefulset。

StatefulSet从本质上来说，可以看作Deployment/RC的一个特殊变种，它有如下特性。

◎ StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设StatefulSet的名称为kafka，那么第1个Pod叫kafka-0，第2个叫kafka-1，以此类推。

◎ StatefulSet控制的Pod副本的启停顺序是受控的，操作第n个Pod时，前n-1个Pod已经是运行且准备好的状态。

◎ StatefulSet里的Pod采用稳定的持久化存储卷，通过PV或PVC来实现，删除Pod时默认不会删除与StatefulSet相关的存储卷（为了保证数据的安全）。

StatefulSet除了要与PV卷捆绑使用以存储Pod的状态数据，还要与Headless Service配合使用，即在每个StatefulSet定义中都要声明它属于哪个Headless Service。Headless Service与普通Service的关键区别在于，它没有Cluster IP，如果解析Headless Service的DNS域名，则返回的是该Service对应的全部Pod的Endpoint列表。StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod实例都创建了一个DNS域名，这个域名的格式为：

```
$(podname).$(headless service name)
```

比如一个3节点的Kafka的StatefulSet集群对应的Headless Service的名称为kafka，StatefulSet的名称为kafka，则StatefulSet里的3个Pod的DNS名称分别为kafka-0.kafka、kafka-1.kafka、kafka-3.kafka，这些DNS名称可以直接在集群的配置文件中固定下来。

但是随着技术的发展，发现statefulset也不能满足需求了（建模能力有限），所以就有了后来的kubernetes operator框架和众多的operator实现了。但是kubernetes operator框架是面向开发者的，他们可以借助operator框架提供的API开发一个类似statefulset的控制器，在这个控制器里开发者通过编码方式实现对目标集群的自定义操控，包括集群的部署、故障发现和调优等操作，从而实现更好的自动部署和智能运维。从发展的趋势来看，未来主流的有状态集群基本都会以operator方式部署到k8s集群中。

### 1.2.11、批处理应用

除了有状态应用和无状态应用之外，还有批处理应用。它的特点是一个或者多个进程处理一组数据（图像，文件，视频等），在这组数据都处理完成后批处理任务自动结束。为了支持这类应用，kubernetes引入了新的资源对象——Job。

Job控制器提供了两个控制并发的参数：completions和parallelism

completions标识需要运行任务的总数

parallelism表示并发运行的个数，例如设置为1的时候，则会依次执行任务。

Job所控制的Pod副本是短暂运行的，可以将其视为一组容器，其中的每个容器都仅运行一次。当Job控制的所有Pod都运行结束时，对应的Job也就结束了。Job生成的Pod副本是不能自动重启的，对应Pod副本的restartPolicy都被设置成never。后来kubernetes增加了cronjob，可以周期性的执行某个任务。

### 1.2.12、应用的配置问题

现在总结一下三种应用建模的资源对象：

a、无状态服务的建模：deployment

b、有状态集群的建模：statefulset

c、批处理应用的建模：Job

在应用建模的过程中如何解决应用需要在不同的环境中修改配置的问题呢？这时候就设计configmap和secret两个对象。

configmap的具体做法：

a、用户将配置文件的内容保存到configmap中，文件名可以作为key，value就是整个文件的内容，多个配置文件都可以放入到一个configmap中；

b、在建模应用的时候，在Pod里将configmap定义为特殊的volume进行挂载，在pod被调度到某个Node上时，configmap里的配置文件会被自动还原到本地的目录下，然后映射到Pod里的指定目录下，这样用户的程序就可以无感知的读取配置了；

c、在configmap的内容别修改以后，k8s会自动重新获取其内容，并在目标节点上更新对应的文件。

接下来说说secret：

它也用于解决应用配置的问题，不过解决的是对敏感信息的配置问题，比如数据库密码，应用的数字证书、token、SSH密钥及其他需要保密的敏感配置。对于这类敏感信息，我们可以创建一个secret对象，然后被Pod引用。

### 1.2.13、应用的运维问题

为了实现分布式应用系统能够根据当前的负载的变化自动触发水平扩容或者缩容，K8s引入了Horizontal Pod Autoscaler对象。HPA与之前的RC、Deployment一样，也属于一种Kubernetes资源对象。通过追踪分析指定RC控制的所有目标Pod的负载变化情况，来确定是否需要有针对性地调整目标Pod的副本数量，这是HPA的实现原理。当前，HPA有以下两种方式作为Pod负载的度量指标。

◎ CPUUtilizationPercentage。

◎ 应用程序自定义的度量指标，比如服务在每秒内的相应请求数（TPS或QPS）。

CPUUtilizationPercentage是一个算术平均值，即目标Pod所有副本自身的CPU利用率的平均值。一个Pod自身的CPU利用率是该Pod当前CPU的使用量除以它的Pod Request的值。如果某一时刻CPUUtilizationPercentage的值超过80%，则意味着当前Pod副本数量很可能不足以支撑接下来更多的请求，需要进行动态扩容，而在请求高峰时段过去后，Pod的CPU利用率又会降下来，此时对应的Pod副本数应该自动减少到一个合理的水平。如果目标Pod没有定义Pod Request的值，则无法使用CPUUtilizationPercentage实现Pod横向自动扩容。

除了可以通过直接定义YAML文件并且调用kubectrl create的命令来创建一个HPA资源对象的方式，还可以通过下面的简单命令行直接创建等价的HPA对象：

```
kubectl autoscale deployment php-apache --cpu-percent=90 --min=1 --max=10
```

接下来就是VPA（vertical pod autoscaler），即垂直pod自动扩容，他根据容器资源使用率自动推测并设置pod合理的CPU和内存的需求指标，从而更加精确的调度Pod，实现整体上节省集群资源的目标。VPA属于比较新的特性，也不能与HPA共同操控同一组Pod，目前只需要关注其发展状况即可。

## 存储类

存储类的资源对象主要包括：volume、persistent volume、PVC和storgeclass

volume是Pod中能够被多个容器访问的共享目录，他被定义在Pod上，被一个Pod里的多个容器挂载到具体的文件目录下，它的生命周期和Pod相同，但是与容器的生命周期不相关，当容器终止或者重启的时候，volume中的数据也不会丢失。k8s支持多种类型的volume，例如ceph、glusterFS等等。

kubernetes提供了非常丰富的volume类型供容器使用，例如临时目录、宿主机目录、共享存储等。

### 1.2.14、emptyDir

一个emptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且无需指定宿主机上对应的目录文件，当Pod从Node上移除的时候empytDir中的数据也会被永久删除。主要用途如下：

a、临时空间、无需永久保留的目录

b、长时间任务执行过程中使用的临时目录

c、一个容器需要从另一个容器中获取数据的目录（多容器共享目录）

在默认情况下emptyDir使用的是节点的存储介质，还可以设定empytDir.medium属性，设置为memory就可以使用更快的基于内存的后端存储了，但是这样消耗的内存会计算到容器的资源消耗。

### 1.2.15、hostPath

hostPath为在Pod上挂载宿主机上的文件或者目录，通常用于：

a、在容器应用程序生成的日志文件需要永久保存时可以使用宿主机的文件系统进行存储

b、需要访问宿主机上的docker引擎内部的数据结构的容器应用时，可以通过定义hostPath为宿主机/var/lib/docker目录，使容器内部的应用可以直接访问docker 的文件系统。

使用这种volume的时候的注意点：

a、在不通的Node上具有相同配置的Pod，可能会因为宿主机上的目录和文件不同而导致对volume上目录和文件的读取结果不一致

b、如果使用了资源配置管理，K8s无法将hostPath在宿主机上使用的资源纳入管理。

除此之外还有公有云volume、iSCSI、nfs、congfigmap、secret等volume。

### 1.2.16、动态存储管理

所谓动态存储管理，其实就是实现存储的自动化管理，相关的三个核心资源对象：PV、PVC、storageclass

PV表示有系统动态创建的一个存储卷，可以被理解成k8s集群中某个网络存储对应的一块存储，它和volume类似，但是PV并不是被定义在Pod上，而是独立于Pod之外定义的。主要支持FC、iSCSI、nfs、cephfs等等

那么系统怎么知道从那个存储系统中创建什么规格的PV存储呢，这时候就需要用到storageclass和PVC：

storageclass用来描述和定义某种存储系统的特征：provisioner（第三方插件）、parameters（必要参数）、reclaimpolicy（回收策略，删除或保留），需要注意的是storageclass的名称会在PVC中出现。

PVC（PV Claim）表示应用希望申请的PV的规格，重要的属性有accessmodes（存储访问模式）、storageclassName（用那种storageclass来实现动态创建）、resources（存储的具体规格）

有了storageclass和PVC为基础的动态PV管理机制，我们就很容易管理和使用volume了，只要在Pod中引用PVC即可达到目的。

## 安全类

只有通过认证的用户才能通过kubernetes的API Server进行查询、创建及维护等工作。

K8s有两类用户：

普通用户：平时使用kubectl进行运维的工作人员（人）

关键用户：Pod应用需要通过API server查询、创建及管理其他资源对象（Pod）

为此就出现了Service Account（SA）这个特殊的资源对象，代表Pod应用的账号，为Pod提供必要的身份认证。

k8s进一步实现和完善了基于角色的访问控制权限系统——RBAC（Role-Based Access Control）

SA不是全局的，每个namespace都会自动创建一个默认的default的SA，创建Pod的时候会自动与其绑定作为“公民身份证”。

SA是通过secret来保存对应的用户（应用）身份凭证的，这些凭证信息有CA根证书数据（ca.crt）和签名后的token信息。API Server通过接收到的token信息就能确定SA的身份。

当Pod里的容器被创建时，k8s会把对应的secret对象中的身份信息（ca.crt、token等）持久化保存到容器里固定位置的本地文件中，当容器里的用户进程通过k8s提供的客户端API去访问API Server时，这些API会自动读取这些身份信息文件，并将其附加到HTTPS请求中传递给API Server以完成身份认证逻辑。身份认证通过后就要设计访问授权的问题了。

### 1.2.17、role

包括role（局限于某个namespace的角色）和clusterrole（作用于整个集群范围）两种类型的角色。

下面这个例子表示在命名空间default中定义一个role对象，用于授予对Pod资源的读访问权限，绑定到该role的用户则具有对Pod资源的get、watch和list权限：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]   #空字符串“”表明使用core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

创建好role后可以通过rolebinding与clusterrolebinding

举例：在default命名空间中将pod-reader角色授予用户caden，结合对应的role定义，表明这一授权将允许用户caden从命名空间default中读取pod

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: Caden
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

在RB中使用subject（目标主体）来表示要授权的对象，我们可以授权三种目标账号：group、user、SA

### 1.2.18、NetworkPolicy

一种特殊的资源对象，网络策略。它是网络安全相关的资源对象，用于解决用户应用之间的网络隔离和授权的问题。

当多家公司或者厂商共用一个K8s集群的时候，特别是在公有云环境下，不同厂商的应用需要进行隔离以增加安全性，这个时候就要用到networkPolicy了。



# 2、存储原理及应用

## 2.1  持久卷（Persistent Volume）详解

PV（持久卷）是对存储资源的抽象，将存储定义为一种容器应用可以使用的资源。

PVC则是用户对存储资源的一个申请。

StorageClass用于标记存储资源的特性，根据PVC的需求动态供给合适的PV资源。

## 


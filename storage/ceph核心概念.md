# ceph核心概念

# 1、ceph简介

Ceph 是分布式，高可用，高性能，高拓展性的存储系统，个人认为这是目前最适合云计算的存储系统,，它是软件定义存储（SDS），它目前提供对象存储, 块设备存储, 文件系统存储三种存储应用

Ceph是一个真正的SDS解决方案，它可以从软件层面正确提供所有的企业级存储特性。低成本、可靠性、可扩展性是Ceph的主要特点

高扩展性：使用普通x86服务器，支持TB到PB级的扩展。
高可靠性：多数据副本，没有单点故障，自动管理，自动修复。
高性能：数据分布均衡，并行化度高。对于对象存储和块设备存储,不需要元数据服务器。
Ceph是一种软件定义存储，可以运行在几乎所有主流的Linux发行版（比如CentOS和Ubuntu）和其它类UNIX操作系统（典型如FreeBSD）。

Ceph的分布式基因使其可以轻易管理成百上千个节点、PB级及以上存储容量的大规模集群，同时基于计算的扁平寻址设计使得Ceph客户端可以直接和服务端的任意节点通信，从而避免因为存在访问热点而导致性能瓶颈。

Ceph是一个统一存储系统，即支持传统的块、文件存储协议，例如SAN和NAS；也支持对象存储协议，例如S3和Swift。



### Ceph核心组件

集群运行需要 `Monitor`,`OSD`,`Mgr`组件。
其中`Monitor`是集群的中心，数据库, 需要第一个启动，其他组件启动会注册信息到`Monitor`里。`OSD`负责存储。 `Mgr`提供集群监控以及管理接口。
另外如果需要对象存储加上`RGW`,需要文件系统存储加上`MDS`。

#### OSD

> OSD是负责物理存储的进程，一般配置成和磁盘一一对应，一块磁盘启动一个OSD进程。主要功能是存储数据、复制数据、平衡数据、恢复数据，以及与其它OSD间进行心跳检查，负责响应客户端请求返回具体数据的进程等；

#### Monitor（ mon）

> 一个Ceph集群需要多个Monitor组成小集群，它们通过Paxos同步数据，用来保存OSD的元数据。负责坚实整个Ceph集群运行的Map视图（如OSD Map、Monitor Map、PG Map和CRUSH Map），维护集群的健康状态，维护展示集群状态的各种图表，管理集群客户端认证与授权信息。
> 管理关键的群集状态，例如群集成员身份和身份验证信息。

#### MDS

> MDS全称Ceph Metadata Server，是CephFS服务依赖的元数据服务。负责保存文件系统的元数据，管理目录结构。对象存储和块设备存储不需要元数据服务；
> 如果不使用文件系统存储，可以不安装。

#### Mgr

> ceph 官方开发了 ceph-mgr(ceph-manager)，主要目标实现 ceph 集群的管理，为外界提供统一的入口。例如cephmetrics、zabbix、calamari、promethus

#### RGW

> RGW全称RADOS gateway，是Ceph对外提供的对象存储服务，接口与Amazon S3和OpenStack Swift 兼容。

#### Admin

> Ceph常用管理接口通常都是命令行工具，如rados、ceph、rbd等命令，另外Ceph还有可以有一个专用的管理节点，在此节点上面部署专用的管理工具来实现近乎集群的一些管理工作，如集群部署，集群组件管理等。

![img](https://preview.cloud.189.cn/image/imageAction?param=CB9681556E78A1A18B298622A3DD28DE598918EC93DF118A6E8F27A0A78B013B47238C589A21A82B85E02907492D263D683ABB21B5DD75FE5A51524FE5DB85FBAB9D365263025FDC5D595741C932B030F261A681ED7107096276544C6B41E9E54EF761E9E691DD679147D083F6A2F134)



![](https://preview.cloud.189.cn/image/imageAction?param=2B8B6171B6EE5757CB090A4B1F4D46F4A3B07EB64EB6B6C5B6B62C3E3EFB3BED0F2560BE44CC110832EE1E2A1BDCDC26839B53ABBECC90418054E0B582928D005A17A3D9F829661361C5585BACE19314E2D9EDA2D70FBEFA38B54062334C34E6B2DA685C04B8C35A3A8A95049C5B5E21)



客户端获取数据的流程：

![img](https://preview.cloud.189.cn/image/imageAction?param=C0249D2334AFAA8222AFC502CB40FAF4E48D23543BD125056B42AC5F5CB6B94B186DE30D240F67FEBBE1CA3F2345DF81779ACE1A2BA1C0319B98887A4B9133DDB340095618F1EB51FF19F350C82D62FCCF9596779AE0ECF3480AC9958196A875F812C629E51B7B5959A1ACD54942945A)



Ceph是当前非常流行的开源分布式存储系统,具有高扩展性、高性能、高可靠性等优点。可以同时提供

块存储(rbd),

对象存储(rgw),

文件系统存储(cephfs)。

块存储: 就是如磁盘一样挂载。
对象存储: 通过api接口访问。
文件系统存储: 可以通过mount挂载。

### 核心概念

http://docs.ceph.org.cn/architecture/

#### RADOS

> 全称Reliable Autonomic Distributed Object Store，即可靠的、自动化的、分布式对象存储系统。RADOS是Ceph集群的精华，用户实现数据分配、故障转移等集群操作。

#### Librados

> Rados提供库，因为RADOS是协议很难直接访问，因此上层的RBD、RGW和CephFS都是通过librados访问的，目前提供PHP、Ruby、Java、Python、C和C++支持。

#### Crush

http://docs.ceph.org.cn/architecture/#crush

> Crush算法是Ceph的两大创新之一，通过Crush算法的寻址操作，Ceph得以摒弃了传统的集中式存储元数据寻址方案。而Crush算法在一致性哈希基础上很好的考虑了容灾域的隔离，使得Ceph能够实现各类负载的副本放置规则，例如跨机房、机架感知等。同时，Crush算法有相当强大的扩展性，理论上可以支持数千个存储节点，这为Ceph在大规模云环境中的应用提供了先天的便利。

> 存储集群的客户端和各个 Ceph OSD 守护进程使用 CRUSH 算法高效地计算数据位置，而不是依赖于一个中心化的查询表。

#### Pool

> Pool是存储对象的逻辑分区，它规定了数据冗余的类型和对应的副本分布策略，支持两种类型：副本（replicated）和 纠删码（ Erasure Code）；

可以配置最大最小副本数，配额，pg数量，crush规则等。

我理解的是：
数据存储会通过pool的副本配置，crush规则等决定数据如何存储。
数据读取也会通过pool的相关配置，决定是否可以读取。如，在有多少副本坏了的情况下就不允许读取了。
因为读取都是通过pool， 就实现了数据逻辑分区。

#### PG

> PG（ placement group）是一个放置策略组，它是对象的集合，该集合里的所有对象都具有相同的放置策略，简单点说就是相同PG内的对象都会放到相同的硬盘上，PG是 ceph的逻辑概念，服务端数据均衡和恢复的最小粒度就是PG，一个PG包含多个OSD。引入PG这一层其实是为了更好的分配数据和定位数据；

#### Object

> 简单来说块存储读写快，不利于共享，文件存储读写慢，利于共享。能否弄一个读写快，利于共享的出来呢。于是就有了对象存储。最底层的存储单元，包含元数据和原始数据。

一个文件会被切割成多个Object分别存储。


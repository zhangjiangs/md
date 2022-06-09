​                          

# OCP4环境使用Q&A

 

# 1. 概述

​    本文档记录对亨通OCP4.6.18环境的使用常见问题与处理。

# 2. 常见问题与处理

## 2.2.连接不上外部服务

### 2.2.1.     问题

集群内应用连接不上集群外服服务，比如kafka服务，地址为10.6.23.41:9092

### 2.2.2.     解决

经测试，ping目标服务器地址可以ping通，但是telnet ip port却报 no route to host。这中现象很可能是网络防火墙禁用了相应的端口，应该排查目标设备或途径的网络设备防火墙的配置，允许相应的端口通信即可。

## 2.3.NFS挂载

### 2.3.1.     问题

如果普通应用直接挂载nfs的话，则会遇到启动失败的问题，查看console页面响应的event，则有显示权限不足，无法挂载nfs的警告事件。

### 2.3.2.     解决

ocp默认使用default  service account运行pod，其对应的SecurityContextConstraints为restricted，不允许进行hostpath和nfs挂载。需创建新的service account并赋予 hostmount-anyuid scc 即可，需挂载nfs的应用使用此service account即可。

操作步骤如下：

集群创建好后，创建如下clusterrole （此操作只需要执行一次即可）

```yaml
cat <<EOF | oc apply -f -

apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRole

metadata:

 name: system:openshift:scc:hostmount-anyuid

rules:

- apiGroups:

 - security.openshift.io

 resourceNames:

 - hostmount-anyuid

 resources:

 - securitycontextconstraints

 verbs:

 - use

EOF
```

 

为特定项目组，例如 test， 新建服务账户 (对应console页面的 user management/service accounts)

```
oc -n test create sa nfsmount
```

为该账户赋予挂载nfs的权限 (对应console页面的 user management/role bindings)

```
oc -n test adm policy add-role-to-user system:openshift:scc:hostmount-anyuid -z nfsmount
```

部署应用时使用此服务账户，添加 serviceAccountName 行

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

 name: nfsmount

spec:

 template:

  spec:

   serviceAccountName: nfsmount
```

备注：如何删除账户和权限？

```
oc -n test adm policy remove-role-from-user system:openshift:scc:hostmount-anyuid -z nfsmount

oc -n test delete sa nfsmount
```



## 2.4.对接外部elasticsearch

### 2.4.1.     问题

外部es集群使用用户名密码保护，是否可以使用fluentd对接到kafka

### 2.4.2.     解决

Ocp的logging 转发是支持直接对接es的，但对es的连接方式有要求，一种是http直连，另一种是https加密通信，目前暂不支持用户名密码保护的es直接对接。建议前端再放一个kafka避免直接对接的问题，同时可以避免日志高峰造成的对es集群的冲击。

## 2.5.边缘节点

### 2.5.1.     问题

物联网应用需要添加边缘节点到集群中，ocp集群是否支持？

### 2.5.2.     解决

OCP有 remote worker nodes 的方式支持边缘节点的加入。但如何使用需要进一步沟通与讨论。Todo.

## 2.6.应用自定义监控

### 2.6.1.     问题

应用开发需要单独的prometheus和grafana部署

### 2.6.2.     解决

在console界面中的Operatorhub中提供社区版本的prometheus和grafana operator，用户可以在特定namespace中进行安装。用户同样也可以使用yaml安装原生的prometheus和grafana。

## 2.7.登录保持时长

### 2.7.1.     问题

OC login的会话保持时常有多久，登陆后多长时间会失效？

### 2.7.2.     解决

使用token的方式登陆的，其资源存储在oauthaccesstoken资源对象中，其token默认的有效时长是24小时。

## 2.8.通过gitlab拉取源代码Build 时提示 SSL 错误

 

### 2.8.1.     解决

 

或者全局增加信任证书对应域名的CA 

 

## 2.9.NFS实现statefull应用配置

 

 

## 2.10. HAproxy配置使得容器中应用获取客户端源IP

客户端访问容器服务的流量路径如下：

客户端 -> 负载均衡（haproxy） -> Router（haproxy） -> 容器

在负载均衡器中，默认配置的是tcp代理，此时客户端的访问源ip会被替换为负载均衡器的ip，由负载均衡器代理访问后端的容器服务（俗称反向代理）。如果需要把客户端ip传递给容器，就需要把4层（tcp）负载均衡改为7层（http）负载均衡，同时在http header中添加通用的header 头（X-Forwarded-For），写入访问客户端的源ip，容器中的应用可以解析header 头，来获取客户端的真实ip。

前端负载均衡haprxoy的配置修改如下：

 

注意，因为此修改需要把tcp通信的内容解析为http协议，并对通信内容进行修改，所以此修改只对未加密的80端口适用。通过tls加密的443端口，因负载均衡器无法对通信内容进行解密，因此无法还原为http原始内容并添加源ip信息。

## 2.11. 如何获取永久token，并能执行OC命令

### 2.11.1.   问题

IOT平台需要一个长期token

### 2.11.2.   解决

建一个登陆的帐号，和项目，并在这个项目下建一个SA，给SA赋予对应的cluster-reader权限，然后在页面上去找这个SA的token。

在ht-iot-insights下新建一个名为compute的service accounts

执行命令：

进入ht-iot-insights项目：

oc project ht-iot-insights

赋权：

oc adm policy add-cluster-role-to-user cluster-reader -z compute

## 2.12. 如何给应用开启特权，使用host port和host path

### 2.12.1.   问题

给应用直接映射node端口和使用node path

### 2.12.2.   解决

赋权：

oc adm policy add-scc-to-user anyuid system:serviceaccount:hostnetwork-right:compute

//可以以root运行

oc adm policy add-scc-to-user privileged system:serviceaccount:hostnetwork-right:compute

//具有最高权限(创建的POD可以使用host network)

oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:hostnetwork-right:compute

//可以读取这个Openshift cluster的信息

再次部署就可以了

## 2.13. 给用户访问别的project的权限

oc adm policy add-role-to-user edit 监造平台 -n yun-mom

//给用户添加对其他project的编辑权限

oc adm policy add-role-to-user view 监造平台 -n yun-mom

//给用户添加对其他project的查看权限

注意：用户名需要使用oc get users查看出来的

 

oc adm policy add-role-to-group edit test-supervision -n yun-mom

//给用户组添加对其他project的编辑权限

## 2.14. 应用timeout时间设定

### 2.14.1.   问题

数据平台的dataapi-gateway（api网关服务）设置timeout时间不起效果，设置时间为3m，但是api每次1min就报超时错误

haproxy.router.openshift.io/timeout: 3m

### 2.14.2.   解决

以 baremetal 安装方式的集群来看，它的流量是这样的

 

​                 —————————— router pod1

​                 ｜

client -- Internet DNS-- LB -----｜

​                 ｜————————— router pod2

 

访问流程

\1. 客户端访问比如 http://xxxx.com 通过 DNS 解析，解析到 LB 上

\2. LB 吧用户的请求转到 router pod 上

 

当前设置的 timeout 时间是在 route 这个定义上，那么如果 LB 这一层没有对 timeout 做过配置，您只配置 ocp 的 route 我理解可能是不起作用的。

 

所以为了撇清关系，我们把访问路径做一下变更

 

client ----> DNS ---> router pod 1

 

让他满足这样流量走向，也就是不过 LB，看是否可以正常工作

 

如果这个地方测试成功了，说明 router 的timeout 是有效的，需要调整 LB

请您在 dns server 上面，创建一个新的 A 记录，然后让这个 A 记录解析指向 router pod IP 就可以了，因为这样可以撇清到底问题发生在哪个层面

 

具体如何做：

 

1. 如何查询 router pod IP，

```
# oc get po -n openshift-ingress -o wide

NAME               READY  STATUS  RESTARTS  AGE  IP       NODE             NOMINATED NODE  READINESS GATES

router-default-6898f4fb9d-gmkfr  1/1   Running  0     47h  192.168.1.207  worker-1.ocp.nielasaran.com  <none>      <none>

router-default-6898f4fb9d-qh6mx  1/1   Running  0     47h  192.168.1.206  worker-0.ocp.nielasaran.com  <none>      <none>
```

 

先查看一下 router pod 目前已有的 IP 是多少，正常来说 router pod ip 就是 worker 节点的 ip，我们需要确认他在哪个 worker 上运行

 

2. 请把这个解析地址专门写一个 A 记录，指向 router pod 所在的 node ip

 

比如，事先给 a.apps.ocp.nielasaran.com 上设置过 timeout，那么，请在 DNS server 中增加这样的一条记录

 

a.apps.ocp.nielasaran.com.        IN   A    192.168.1.207

 

设置完 dns 之后，要确认一下解析

 

nslookup a.apps.ocp.nielasaran.com

 

看是否可以正常解析到 router pod 的地址，如果是正常的，再测试

按照以上步骤测试发现3m的设置生效，说明问题处在LB的设置

于是修改haproxy的配置文件（LB服务器）

将/etc/haproxy/haproxy.cfg

timeout server 的值修改为6m，然后将前面添加的DNS解析删除。最终解决问题

haproxy的相关配置解释可参考：

```
https://www.papertrail.com/solution/tips/haproxy-logging-how-to-tune-timeouts-for-performance/
```

## 2.15.  集群APIServer证书更新

### 2.15.1.   APIServer证书更新过程

```
OCP集群APIServer证书使用时间约一年的80%（大约292天后），OpenShift 集群的 APIServer 证书 kube-apiserver-to-kubelet-signer 会自动进行轮换更新
这个更新是为了防止您的证书过期，然后重新生成下一年新的证书，因为 openshift 4 开始采用 machine config 进行 os 级别的文件更新，所以到达 292 天时 MCO 也会触发系统重启来应用新的证书。如果 292 天不是您的停机维护窗口时间，那么请参考红帽的 KCS，暂时停止证书轮转，达到不重启的效果
https://access.redhat.com/solutions/5484051
但是请注意如果暂停证书轮转，一定要在集群运行 365 天之前放行，如下所示 AGE 为集群运行的天数 63d 是 63 天
 
# oc get nodes
NAME                        STATUS     ROLES    AGE   VERSION
master01.ocp4.example.com   Ready      master   63d   v1.21.1+9807387
master02.ocp4.example.com   Ready      master   63d   v1.21.1+9807387
master03.ocp4.example.com   Ready      master   63d   v1.21.1+9807387
worker01.ocp4.example.com   Ready      worker   63d   v1.21.1+9807387
worker02.ocp4.example.com   Ready      worker   63d   v1.21.1+9807387
worker03.ocp4.example.com   Ready      worker   63d   v1.21.1+9807387
 
请保证一定要在 365 天之前，申请停机维护窗口，放行让 OCP 集群重启更新证书，如果超过 365 天仍没有放行重启，那么就会发生集群节点 not ready 的证书过期问题，届时会导致集群无法正常工作。
如果 292 天正好是您的运维窗口时间，那么您等待它自动重启就好，它是每次重启一个 node，重启之前会尝试驱离这个 node 上的 pod，所以不会对您的业务造成影响，但是也建议您关注一下整个重启过程，比如突然某个 pod 驱离失败，那么可能整个重启流程会卡住，到时候需要您帮忙手动删除这个无法驱离的 pod
```

### 2.15.2.   更新操作会在当天几点开始操作？

您可以通过有 kubeadmin 权限的用户查询

```
 oc get secret -A -o json | jq -r '.items[] | select(.metadata.annotations."auth.openshift.io/certificate-not-after"!=null) | select(.metadata.name|test("-[0-9]+$")|not) | "\(.metadata.namespace) \(.metadata.name) \(.metadata.annotations."auth.openshift.io/certificate-not-after")"' | column -t
```

 以我的测试环境为例 

```
# oc get secret -A -o json | jq -r '.items[] | select(.metadata.annotations."auth.openshift.io/certificate-not-after"!=null) | select(.metadata.name|test("-[0-9]+$")|not) | "\(.metadata.namespace) \(.metadata.name) \(.metadata.annotations."auth.openshift.io/certificate-not-after")"' | column -t

openshift-config-managed          kube-controller-manager-client-cert-key  2022-01-09T10:59:38Z

openshift-config-managed          kube-scheduler-client-cert-key      2022-01-09T10:59:37Z

openshift-kube-apiserver-operator      aggregator-client-signer         2022-01-12T05:08:24Z

openshift-kube-apiserver-operator      kube-apiserver-to-kubelet-signer     2022-12-10T10:37:11Z
```

 这个环境是 2021年 12 月10日 安装的，他的过期时间是 2022年12月10日，具体的过期时间是 2022-12-10T10:37:11Z，这个是系统的 UTC 时间，转化成北京时间是2022-12-10 18:37 分，为 365 天，基于这个时间，往前数 73 天，也就是 292 天的 18:37 分开始创建 mc，并开始重启操作

##  2.16 OCP4：CVE-2021-44228 影响 Elasticsearch（Red Hat OpenShift Logging）

### 解决方法

如果尚无法升级，则有一种解决方法。

> **重要：** 如果您遵循可选步骤 8，请确保将 Openshift 日志记录和/或 Elasticsearch 运算符恢复 `managementState` 为 `managed` 状态以升级到包含修补程序的版本。
>
> **注意：** 此变通办法以 ElasticSearch for Openshift Logging 堆栈为例，但该变通办法可以以相同的方式应用于使用官方 Red Hat ElasticSearch Operator 部署的 Elasticsearch。

1. 更改为部署了日志记录堆栈的项目（默认情况下） `openshift-logging` 项目）：

   ```
   $ oc project openshift-logging
   ```

2. 查找 `replicasets` 已部署的 Elasticsearch，以便稍后传递到 `oc set env` 命令：

   ```
   $ oc get rs -l component=elasticsearch
   NAME                                      DESIRED   CURRENT   READY   AGE
   elasticsearch-cdm-ba9c6evk-1-796f6cfdbc   1         1         1       16d
   elasticsearch-cdm-ba9c6evk-2-7959d4d857   1         1         1       16d
   elasticsearch-cdm-ba9c6evk-3-5f9c5d668c   1         1         1       17d
   ```

3. 在 `ES_JAVA_OPTS` Elasticsearch 容器中将系统属性的环境变量 `log4j2.formatMsgNoLookups` 设置为 `true`:

   ```
   $ oc set env -c elasticsearch replicaset/<elasticsearch_replicaset_name> ES_JAVA_OPTS="-Dlog4j2.formatMsgNoLookups=true"
   ```

   3.1. 您可以通过以下方式确认这一点：

   ```
   $ oc set env -c elasticsearch rs -l component=elasticsearch --list | grep ES_JAVA_OPTS
   
   ES_JAVA_OPTS=-Dlog4j2.formatMsgNoLookups=true
   ES_JAVA_OPTS=-Dlog4j2.formatMsgNoLookups=true
   ES_JAVA_OPTS=-Dlog4j2.formatMsgNoLookups=true
   ```

4. 缩减 Elasticsearch `ReplicaSet` 因此，之后会生成新的 ES pod：

   ```
   $ oc scale replicaset/<elasticsearch_replicaset_name> --replicas=0
   ```

5. 检查新的 ES pod 在缩减 `ReplicaSet` 到 `0` （无需扩展，因为这将自动完成）：

   ```
   $ oc get pods -l component=elasticsearch
   
   NAME                                            READY   STATUS    RESTARTS   AGE
   elasticsearch-cdm-ba9c6evk-1-796f6cfdbc-4dqc6   2/2     Running   0          27m
   elasticsearch-cdm-ba9c6evk-2-7959d4d857-z5km9   2/2     Running   0          2d9h
   elasticsearch-cdm-ba9c6evk-3-5f9c5d668c-cr8lj   2/2     Running   0          2d9h
   
   $ oc  set env -c elasticsearch pods -l component=elasticsearch --list | grep ES_JAVA_OPTS
   
   ES_JAVA_OPTS=-Dlog4j2.formatMsgNoLookups=true
   ES_JAVA_OPTS=-Dlog4j2.formatMsgNoLookups=true
   ES_JAVA_OPTS=-Dlog4j2.formatMsgNoLookups=true
   ```

6. `oc exec` 进入新生成的 ES pod 以检查正确传递的 Java 命令行参数，包括 `-Dlog4j2.formatMsgNoLookups=true`:

   ```
   $ oc exec -c elasticsearch <elasticsearch_pod> -- grep -a log4j2.formatMsgNoLookups /proc/1/cmdline
   ```

   `-Dlog4j2.formatMsgNoLookups=true` 应该在上面的输出中可见。

7. 对每个 Elasticsearch 重复步骤 3 到 6 `ReplicaSet` 一次缓慢地一个，以允许 ES pod 在保持 ES 集群仲裁的同时重生：

8. （可选）要避免 OpenShift 日志记录运算符和 Elasticsearch 运算符在修复程序仍在进行时重写更改，可以将其设置为 `unmanaged`:

   ```
   $ oc edit clusterlogging/instance
   [...]
   spec:
   [...]
     managementState: Unmanaged
   [...]
   $ oc edit elasticsearch/elasticsearch
   [...]
   spec:
   [...]
     managementState: Unmanaged
   [...]
   ```

   并通过检查来确认：

   ```
   $ oc get clusterlogging
   NAME       MANAGEMENT STATE
   instance   Unmanaged
   
   $ oc get elasticsearch
   NAME            MANAGEMENT STATE   HEALTH   NODES   DATA NODES   SHARD ALLOCATION   INDEX MANAGEMENT
   elasticsearch   Unmanaged          green    3       3            all    
   ```

- （可选）完成上述更改后，可以在部署级别执行该过程，而不是 `ReplicaSets` 如步骤 2 到 5 中所述：

  ```
  $ oc patch deployment/<elasticsearch_deployment_name> --type=merge -p '{"spec":{"paused": false}}'
  
  $ oc set env deployment/<elasticsearch_deployment_name> -c elasticsearch ES_JAVA_OPTS="-Dlog4j2.formatMsgNoLookups=true"
  ```

  这将创建新的 `ReplicaSets` 自动和 pod 定义了环境。之后，可以执行步骤5到7。 如果执行步骤 2 到 5，则不需要执行此方法。

### 根本原因

[CVE-2021-44228](https://access.redhat.com/security/cve/cve-2021-44228) [布吉拉](https://bugzilla.redhat.com/show_bug.cgi?id=2030932)

### 诊断步骤

检查 OpenShift 日志记录运算符是否尚未还原 `ES_JAVA_OPTS` 进行更改后添加环境变量：

```
$ oc describe rs/<elasticsearch_replicaset_name> | grep ES_JAVA_OPTS
```

预期输出：

```
      ES_JAVA_OPTS:             -Dlog4j2.formatMsgNoLookups=true
```
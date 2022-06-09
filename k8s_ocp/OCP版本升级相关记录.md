

#                     Openshift离线升级文档

离线环境，随着使用可能需要更新 operator hub 中的内容，具体操作方法如下

需求1. 升级原有离线 operator hub 中内容的版本

查看 原 operator hub 中的 catalog 信息

```
# oc get po 

NAME                                                              READY   STATUS                  RESTARTS   AGE
marketplace-operator-7fc454d4bc-tpgtj                             1/1     Running                 0          17h
origin-mhh7c                                                      1/1     Running                 0          17h
```

获取 catalog 中的 image 信息

```
# oc describe pod origin-mhh7c |grep -i image
    Image:          harbor.ocp4.example.com/olm/redhat-operator-index:v4.6
    Image ID:       harbor.ocp4.example.com/olm/redhat-operator-index@sha256:2ae14e2c3507efd0cfa4319cb6d0b668a0116d538a799b6d1118885429f90ba2
```

获取原 catalog 中包含的 operator 信息，这一步需要开 2 个终端

在 Terminal-1 创建用于运行 index container

```
# podman run -p50051:50051 -it harbor.ocp4.example.com/olm/redhat-operator-index:v4.6
```

在 Terminal-2 通过 grpcurl 命令获取 redhat operator index 中包含的 operator 列表

```
# mkdir /tmp/olm 
# grpcurl -plaintext localhost:50051 api.Registry/ListPackages > /tmp/olm/old-operator-packages.out
```

假设看到的离线环境 operator 包含这些 operator

```shell
# cat /tmp/olm/old-operator-packages.out
{
  "name": "cluster-logging"
}
{
  "name": "elasticsearch-operator"
}
{
  "name": "local-storage-operator"
}
{
  "name": "mtc-operator"
}
{
  "name": "nfd"
}
{
  "name": "ocs-operator"
}
```

升级方法如下

1. 首先把所有已经安装的 Subscription 从 Automatic 改为 manual，防止新的 index 进来之后自动更新了 operator，所有已安装的，一定要改

![img](C:\Users\zhangjs\AppData\Local\YNote\data\zjs_55@163.com\00395bc6ab34482b844a9e4497f652ef\截图.png)

2. 重新创建新的 index image

```shell
# mkdir new-operator

# cd new-operator/

[root@bastion new-operator]# opm index prune -f registry.redhat.io/redhat/redhat-operator-index:v4.6 \
 -p elasticsearch-operator,cluster-logging,local-storage-operator,mtc-operator,nfd,ocs-operator \
 -t harbor.ocp4.example.com/olm/redhat-operator-index:v4.6
```

新的 image 和之前的 image 有相同的 tag，推送到 registry

```
# podman push harbor.ocp4.example.com/olm/redhat-operator-index:v4.6
```

3. 生成新的 mapping 文件和 image source 文件

```shell
# oc adm catalog mirror harbor.ocp4.example.com/olm/redhat-operator-index:v4.6 harbor.ocp4.example.com/olm/redhat-operator-index:v4.6 -a /root/pull-secret --filter-by-os='linux/amd64' --manifests-only

# ls -l
total 0
drwxr-xr-x 2 root root 88 Aug 14 08:12 manifests-redhat-operator-index-1628946743

# cd manifests-redhat-operator-index-1628946743/

# ls -l
total 64
-rwxr-xr-x 1 root root   253 Aug 14 08:12 catalogSource.yaml
-rwxr-xr-x 1 root root 10749 Aug 14 08:12 imageContentSourcePolicy.yaml
-rw-rjiang--r-- 1 root root 48722 Aug 14 08:12 mapping.txtfdfd
```

4. 用 skopeo 命令拷贝 image 到离线仓库

```shell
cat mapping.txt
以这个内容为例
registry.redhat.io/openshift4/ose-elasticsearch-operator@sha256:a8d4313aa52019135487478483b1e86e1c2bab115cbbaff17345bc5058bb0a79=harbor.ocp4.example.com/olm/redhat-operator-index-openshift4-ose-elasticsearch-operator:8a8e75c1

获得 mapping 信息之后，用这个命令做
skopeo copy -a docker://registry.redhat.io/openshift4/ose-elasticsearch-operator@sha256:a8d4313aa52019135487478483b1e86e1c2bab115cbbaff17345bc5058bb0a79 docker://harbor.ocp4.example.com/olm/redhat-operator-index-openshift4-ose-elasticsearch-operator

注意不要带 image 后面的 :8a8e75c1

用这个方法拷贝所有 image 
```

也可以使用这个脚本一次全部拷贝到离线仓库：

```
while read source local ; do skopeo copy --authfile /root/pull-secret.json --all docker://$source docker://$local ; done <<< `awk -F '=|:' '{print $1":"$2,$3}' mapping.txt` 
```

5.找到上一次做 离线 operator hub 时候的 imageContentSourcePolicy.yaml

```shell
把新的 imageContentSourcePolicy.yaml 的内容，放到原 image content 之后

  - mirrors:
    - harbor.ocp4.example.com/olm/ocs4/mcg-core-rhel8 <===== 之前第一次做的
    source: registry.redhat.io/ocs4/mcg-core-rhel8
  - mirrors:
    - harbor.ocp4.example.com/olm2/redhat-operator-index-openshift4-ose-cluster-logging-operator-bundle
    source: registry.redhat.io/openshift4/ose-cluster-logging-operator-bundle
  - mirrors:
    - harbor.ocp4.example.com/olm2/redhat-operator-index-openshift4-ose-logging-fluentd <===== 升级版本新增的
    source: registry.redhat.io/openshift4/ose-logging-fluentd
```

6 replace 这个资源

```shell
# oc apply -f imageContentSourcePolicy.yaml

4.6 版本会触发 mco 重启

# oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-0997bd3d12c3d9577e60be7fddcb412e   False     True       False      3              0                   0                     0                      9h
worker   rendered-worker-d8efeb1bc448c1750e78461864dfebeb   False     True       False      3              0                   0                     0                      9h
```

7. approve installPlan，升级完毕

升级前

![img](https://preview.cloud.189.cn/image/imageAction?param=F96AA9E4A020D33A5AE41AB4A612B62FD6D1C74B550E292AB53FD71E85BBE5CDAEAE530E7C10142486E74CBE8645662E3E0580491FD62FC72E287EC53E6B4F79E7416F43B73E143D8749910047AAECB3422BF4BFD3F1929C9A8024B4077D802A8456DBD9C731DD0440DC467650DAF99F)

approve install plan

![img](https://preview.cloud.189.cn/image/imageAction?param=E49FDDC929564C7AC0ECFFB45B29DE2DED064CEC5AA76DD68B730519E64DA1813E2EC63960ABB38D81A656D2E5BE66FD76B5B850639592BB49181AD316A97E6FFE025E35A4DFA1BC610F88EE2D4DE5DD1AE29546F1B83FF1BEC452A667675847E9326FB6782F132F8D43B4596423D908)

点 approve

![img](https://preview.cloud.189.cn/image/imageAction?param=219F0CCD371DC407E33BF9CFD981C5E01E88B0FBD355F66869658395B5E066B25CBF15ED98E1264615A2837D3A757950B2CB7638AAB9C5A3B11D799390C1EA97762FE0DD19CA9110D6FAA5EDAEEE370DD5E312687DBB5C6C1146892B34CE70AE5414E3F7A216D1FA5CA98535251DBBE7)

查看升级后的结果

![img](https://preview.cloud.189.cn/image/imageAction?param=ED0FFE4BDE8265FF29C47994D05B94AEF9D480E0D0C1EA5CF3FE7179B1D459AD3D76C4D7516055745915E61DA4A3C1E0816D3A73D5DA36C268EE4D32E9B679A25CC9F5FF452ABD3434AC806217E056AAE703B62C2055AC27A539DF953F56A92B4CB3E1A40C7CF495333DBE0A809B29AA)





其他一些记录信息：

将已安装的operator更新模式改成手动

```shell
#ssh登陆节点
ssh core@master01
```

```shell
#查看当前集群配置的仓库信息

#更改集群的pull-secret文件，用来设置集群的镜像仓库地址，可以达到在线升级或者离线升级的效果
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=<pull_secret_location> 

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/root/pull-secret

#设置环境变量，用以从Redhat官方仓库将需要的版本镜像拉取到本地harbor仓库
##先缓存当前版本 4.6.51
export OCP_RELEASE=4.6.51   
##这个地方写您的 harbor 仓库的 https 访问地址即可，如果用的不是 443 端口，需要加端口号
LOCAL_REGISTRY='<local_registry_host_name>:<local_registry_host_port>'
LOCAL_REGISTRY=harbor.xx-xxx.com
##需要您在 harbor 仓库当中创建一个 公开的项目，叫做 ocp4651，然后再定义这个变量
LOCAL_REPOSITORY='ocp4651/openshift4' 
##固定值
PRODUCT_REPO='openshift-release-dev'
##指定身份信息文件位置
LOCAL_SECRET_JSON='/root/pull-secret.json'
##固定值
RELEASE_NAME="ocp-release"
##镜像位数信息
ARCHITECTURE=<server_architecture>
ARCHITECTURE=x86_64
##执行该命令开始下载镜像
oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --apply-release-image-signature
##镜像缓存成功后将最后的信息中的以下部分复制，并以此生成一个image4824.txt的文件：
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - harbor.xx-xxx.com/ocp4824/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - harbor.xx-xxx.com/ocp4824/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

##设置需要更新的目标版本仓库，比如要升级至4.8.24版本：
###该操作在4.7版本之前是需要所有节点重启的，4.8版本以后不需要重启，该设置完成后即可在web console上update
oc create -f image4824.txt
```


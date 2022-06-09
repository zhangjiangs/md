#                  OpenShift中备份etcd数据库

1. 为 control plane 节点启动一个 debug 会话：

   ```terminal
   $ oc debug node/<node_name>
   ```

2. 将您的根目录改为主机：

   ```terminal
   sh-4.2# chroot /host
   ```

3. 如果启用了集群范围的代理，请确定已导出了 `NO_PROXY`、`HTTP_PROXY`和 `HTTPS_PROXY` 环境变量。

4. 运行 `cluster-backup.sh` 脚本，输入保存备份的位置。

   提示

   `cluster-backup.sh` 脚本作为 etcd Cluster Operator 的一个组件被维护，它是 `etcdctl snapshot save` 命令的包装程序（wrapper）。

   ```terminal
   sh-4.4# /usr/local/bin/cluster-backup.sh /home/core/assets/backup
   ```

   **脚本输出示例**

   ```terminal
   found latest kube-apiserver: /etc/kubernetes/static-pod-resources/kube-apiserver-pod-6
   found latest kube-controller-manager: /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-7
   found latest kube-scheduler: /etc/kubernetes/static-pod-resources/kube-scheduler-pod-6
   found latest etcd: /etc/kubernetes/static-pod-resources/etcd-pod-3
   ede95fe6b88b87ba86a03c15e669fb4aa5bf0991c180d3c6895ce72eaade54a1
   etcdctl version: 3.4.14
   API version: 3.4
   {"level":"info","ts":1624647639.0188997,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/home/core/assets/backup/snapshot_2021-06-25_190035.db.part"}
   {"level":"info","ts":"2021-06-25T19:00:39.030Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
   {"level":"info","ts":1624647639.0301006,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://10.0.0.5:2379"}
   {"level":"info","ts":"2021-06-25T19:00:40.215Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
   {"level":"info","ts":1624647640.6032252,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://10.0.0.5:2379","size":"114 MB","took":1.584090459}
   {"level":"info","ts":1624647640.6047094,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/home/core/assets/backup/snapshot_2021-06-25_190035.db"}
   Snapshot saved at /home/core/assets/backup/snapshot_2021-06-25_190035.db
   {"hash":3866667823,"revision":31407,"totalKey":12828,"totalSize":114446336}
   snapshot db and kube resources are successfully saved to /home/core/assets/backup
   ```

   

   在这个示例中，在 control plane 主机上的 `/home/core/assets/backup/` 目录中创建了两个文件：

   - `snapshot_<datetimestamp>.db`：这个文件是 etcd 快照。`cluster-backup.sh` 脚本确认其有效。

   - `static_kuberesources_<datetimestamp>.tar.gz`：此文件包含静态 pod 的资源。如果启用了 etcd 加密，它也包含 etcd 快照的加密密钥。

     注意

     如果启用了 etcd 加密，建议出于安全考虑，将第二个文件与 etcd 快照分开保存。但是，需要这个文件才能从 etcd 快照中进行恢复。

     请记住，etcd 仅对值进行加密，而不对键进行加密。这意味着资源类型、命名空间和对象名称是不加密的。

   


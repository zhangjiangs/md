# Environment Overview

## Environment Overview

You will be interacting with an OpenShift 4 cluster that is running on Amazon Web Services. During the lab you will also install OpenShift Container Storage, based on Rook/Ceph.

The basics of the OpenShift 4 installation have been completed in advance. The OpenShift cluster is essentially set to all defaults and looks like the following:

- 3 master nodes
- 3 worker nodes
- 1 bastion host

[Homeroom](https://github.com/openshift-labs/workshop-dashboard) is a solution that provides this integrated lab guide, terminal, and web console pane. It is actually running inside the cluster that you will be interacting with.

## Conventions

You will see various code and command blocks throughout these exercises. Some of the command blocks can be executed directly. Others will require modification of the command before execution. If you see a command block with a red border (see below), the command will copy to clipboard for slight required modification.

The icon beside the command blocks should tell you if the commands will be executed or copied.

- This command block will be copied to your clipboard for modification.

```none
some command to modify
```

To paste the copied command try the following

- Cmd + V *tested to work in Chrome on macOS*
- Ctrl + Shift + V *tested to work in Chrome and Firefox on Windows 10*
- Right click + paste in the terminal window *tested to work on Edge on Windows 10*

- This will execute in the console

```none
echo Hello World\!
```

Most command blocks support auto highlighting or executing with a click. If you hover over the command block above and left-click, it should automatically highlight all the text to make for easier copying. Look at the symbol next to the block to see if it will copy or execute.

### Cluster Admin Authentication

The login you provided to access this lab guide actually has nothing to do with the terminal or web console you will interact with. We use a feature of Kubernetes called `ServiceAccounts` which are non-human user accounts. The terminal and web console tabs are interacting with the OpenShift API using one of these `ServiceAccounts`, and that account has been given the *cluster-admin* `ClusterRole`. This allows the terminal and web console to perform administrative / privileged actions against the APIs.

Privileges in OpenShift are controlled through a set of roles, policies, and bindings which you will learn more about in one of the exercises in this workshop.

As a quick example, you can execute the following to learn more about what a `Role` is:

```bash
oc explain Role
```

Inspect how `ClusterRole` differs:

```bash
oc explain ClusterRole
```

You can execute the following to learn more about `RoleBinding`:

```bash
oc explain RoleBinding
```

Inspect how `ClusterRoleBinding` differs:

```bash
oc explain ClusterRoleBinding
```

You can always use `oc explain [RESOURCE]` to get more explanation about what various objects are.

Let’s look at PolicyRules defined in the `ClusterRole` *cluster-admin*:

```bash
oc get clusterrole cluster-admin -o yaml
```

Notice how under rules, an account with the *cluster-admin* role has wildcard `*` access to all `resources` and `verbs` of an apiGroup and all `verbs` in `nonResourceURLs`.

`verbs` are actions that you perform against resources. Things like `delete` and `get` are `verbs` in OpenShift.

To learn more about certain verbs, run `oc [verb] --help`

Let’s learn more about the `verb` *whoami*:

```bash
oc whoami --help
```

We will now run `oc whoami` to see what account you will be using today:

```bash
oc whoami
```

Let’s inspect *dashboard-cluster-admin* `ClusterRoleBinding` that gave our `ServiceAccount` *cluster-admin* `ClusterRole`:

```bash
oc get clusterrolebinding dashboard-cluster-admin -o yaml
```

Notice that our `ServiceAccount` is a subject in this `ClusterRoleBinding` with a role referenced being the *cluster-admin* `ClusterRole`

As a cluster-admin throughout the exercises, you will be able to do anything with the cluster as you have noted earlier, so follow instructions carefully.

# Installation and Verification

## Installation and Verification

The scope of the new installer-provisioned infrastructure (IPI) OpenShift 4 installation is purposefully narrow. It is designed for simplicity and ensured success. Many of the items and configurations that were previously handled by the installer are now expected to be "Day 2" operations, performed just after the installation of the control plane and basic workers completes. The installer provides a guided experience for provisioning the cluster on a particular platform.

This IPI installation has already been performed for you, and the cluster is in its basic, default state.

### Logging in

To inspect the cluster installation, you can simply SSH to the bastion host where it was installed on like so:

```bash
ssh -l cloud-user helper-odflab-55.odflab-55.container-contest.top -o ServerAliveInterval=120
```

Here is your ssh password:

```bash
mqpuboxtskdwxdlu
```

Once you’ve SSH-ed into the bastion host, become the `cloud-user`:

```bash
sudo su - cloud-user
```

You’ll notice that there is a 4-digit alphanumeric string (eg: f4a3) in the hostname. This string is your `GUID`. Since you will often use `GUID`, it makes sense to export it as an environment variable:

```bash
export GUID=`hostname | cut -d. -f2 |cut -d- -f3 `
```

### Master Components

![openshift master 4 responsibilities](https://preview.cloud.189.cn/image/imageAction?param=0CF3B21302089871D3AFB5DE2CFA00DF49FAA34CE549DC23FD77E0FA81AF6059C1EA2BD522377AE45131943D55C0A7064E9917A691A3C96157A0C7D16286DBA00B1567BBF09C35436B52F303E32C726465BB300875FA93D934BFA9B27A804EDCB5EE02342BCB905E841AD9A552B997E5)

Figure 1. OpenShift Master’s 4 main responsibilities.

#### API/Authentication

The Kubernetes API server validates and configures the resources that make up a Kubernetes cluster.

Common things that interact with the Kubernetes API server are:

- OpenShift Web Console
- OpenShift `oc` command line tool
- OpenShift Node
- Kubernetes Controllers

All interactions with the API server are secured using TLS. In addition, all API calls must be authenticated (the user is who they say they are) and authorized (the user has rights to make the requested API calls).

#### Data Store

The OpenShift Data Store (etcd) stores the persistent master state while other components watch etcd for changes to bring themselves into the desired state. etcd can be optionally configured for high availability, typically deployed with 2n+1 peer services.

etcd stores the cluster’s state. It is not used to store user application data.

#### Scheduler

The pod scheduler is responsible for determining placement of new pods onto nodes within the cluster.

The scheduler is very flexible and can take the physical topology of the cluster into account (racks, datacenters, etc).

#### Health / Scaling

Each pod can register both liveness and readiness probes.

Liveness probes tell the system if the pod is healthy or not. If the pod is not healthy, it can be restarted automatically.

Readiness probes tell the system when the pod is ready to take traffic. This, for example, can be used by the cluster to know when to put a pod into the load balancer.

For more information on the OpenShift Master’s areas of responsibility, please refer to the [infrastructure components section](https://docs.openshift.com/container-platform/4.6/architecture/control-plane.html) of the product documentation.

### Examining the installation artifacts

OpenShift 4 installs with two effective superusers:

- `kubeadmin` (technically an alias for `kube:admin`)
- `system:admin`

Why two? Because `system:admin` is a user that uses a certificate to login and has no password. Therefore this superuser cannot log-in to the web console (which requires a password).

If you want additional users to be able to authenticate to and use the cluster, you need to configure your desired authentication mechanisms using CustomResources and Operators as previously discussed. LDAP-based authentication will be configured as one of the lab exercises.

### Verifying the Installation

Let’s do some basic tests with your installation. As an administrator, most of your interaction with OpenShift will be from the command line. The `oc` program is a command line interface that talks to the OpenShift API.

#### Login to OpenShift

When the installation completed, the installer left some artifacts that contain the various URLs and passwords required to access the environment. The installation program was run under the `cloud-user` account.

```bash
ls -al
```

您将看到以下内容：

```
总用量 1296
drwx------. 5 cloud-user cloud-user     236 5月  28 22:48 .
drwxr-xr-x. 3 root       root            24 5月  29 2021 ..
drwxr-x---. 2 cloud-user cloud-user      50 5月  28 21:15 auth
-rw-------. 1 cloud-user cloud-user      21 5月  28 21:55 .bash_history
-rw-r--r--. 1 cloud-user cloud-user      18 8月  21 2019 .bash_logout
-rw-r--r--. 1 cloud-user cloud-user     193 8月  21 2019 .bash_profile
-rw-r--r--. 1 cloud-user cloud-user     282 5月  28 21:15 .bashrc
drwxr-x---. 3 cloud-user cloud-user      19 5月  28 21:26 .kube
-rw-r-----. 1 root       root           112 5月  28 21:15 metadata.json
-rw-r--r--. 1 root       root         61391 5月  28 21:15 .openshift_install.log
-rw-r-----. 1 root       root       1239035 5月  28 21:15 .openshift_install_state.json
drwx------. 2 cloud-user cloud-user      29 5月  29 2021 .ssh
-rw-rw-r--. 1 cloud-user cloud-user     256 5月  28 21:51 workshop-settings.sh
```

The OpenShift 4 IPI installation embeds Terraform in order to create some of the cloud provider resources. You can see some of its outputs here. The important file right now is the `.openshift_install.log`. Its last few lines contain the relevant output to figure out how to access your environment:

```bash
tail -n5 ~/.openshift_install.log
```

- You will see something like the following

```
time="2021-05-28T21:15:42-04:00" level=debug msg="  Reusing previously-fetched Install Config"
time="2021-05-28T21:15:42-04:00" level=debug msg="  Fetching Bootstrap Ignition Config..."
time="2021-05-28T21:15:42-04:00" level=debug msg="  Reusing previously-fetched Bootstrap Ignition Config"
time="2021-05-28T21:15:42-04:00" level=debug msg="Generating Metadata..."
time="2021-05-28T21:15:42-04:00" level=info msg="Ignition-Configs created in: /home/cloud-user and /home/cloud-user/auth"
sudo /usr/local/bin/openshift-install wait-for install-complete --log-level debug
```

- You will see something like the following

```
DEBUG OpenShift Installer 4.7.10
DEBUG Built from commit 3d157f47000c2a9963527ad1dc8c69b77053a4a6
DEBUG Loading Install Config...
DEBUG   Loading SSH Key...
DEBUG   Loading Base Domain...
DEBUG     Loading Platform...
DEBUG   Loading Cluster Name...
DEBUG     Loading Base Domain...
DEBUG     Loading Platform...
DEBUG   Loading Networking...
DEBUG     Loading Platform...
DEBUG   Loading Pull Secret...
DEBUG   Loading Platform...
DEBUG Using Install Config loaded from state file
INFO Waiting up to 40m0s for the cluster at https://api.rhocslab-120.container-contest.top:6443 to initialize...
DEBUG Cluster is initialized
INFO Waiting up to 10m0s for the openshift-console route to be created...
DEBUG Route found in openshift-console namespace: console
DEBUG OpenShift console route is admitted
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/cloud-user/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.rhocslab-120.container-contest.top
INFO Login to the console with user: "kubeadmin", and password: "nhQ7W-XZTkb-sabSx-FhBqJ"
INFO Time elapsed: 0s
```

The installation was run as a different system user, and the artifacts folder is read-only mounted into your `lab-user` folder. While the installer has fortunately given you a convenient `export` command to run, you don’t have write permissions to the path that it shows. The `oc` command will try to write to the `KUBECONFIG` file, which it can’t, so you’ll get errors later if you try it.

Our installation process has actually already copied the config you need to `~/.kube/config`, so you are already logged in. Try the following:

```bash
oc whoami
```

The `oc` tool should already be in your path and be executable.

#### Examine the Cluster Version

First, you can check the current version of your OpenShift cluster by executing the following:

```bash
oc get clusterversion
```

And you will see some output like:

```
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.7.10    True        False         67m     Cluster version is 4.7.10
```

For more details, you can execute the following command:

```bash
oc describe clusterversion
```

Which will give you additional details, such as available updates:

```
Name:         version
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  config.openshift.io/v1
Kind:         ClusterVersion
Metadata:
  Creation Timestamp:  2021-05-29T01:19:32Z
  Generation:          1
  Managed Fields:
    API Version:  config.openshift.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:channel:
        f:clusterID:
    Manager:      cluster-bootstrap
    Operation:    Update
    Time:         2021-05-29T01:19:32Z
    API Version:  config.openshift.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:availableUpdates:
        f:conditions:
        f:desired:
          .:
          f:image:
          f:url:
          f:version:
        f:history:
        f:observedGeneration:
        f:versionHash:
    Manager:         cluster-version-operator
    Operation:       Update
    Time:            2021-05-29T01:19:32Z
  Resource Version:  52175
  Self Link:         /apis/config.openshift.io/v1/clusterversions/version
  UID:               feed1054-fa32-40bc-a21f-1bb4d7789a76
Spec:
  Channel:     stable-4.7
  Cluster ID:  8d06a718-a5dc-4749-a503-03df84a3716e
Status:
  Available Updates:  <nil>
  Conditions:
    Last Transition Time:  2021-05-29T01:47:37Z
    Message:               Done applying 4.7.10
    Status:                True
    Type:                  Available
    Last Transition Time:  2021-05-29T01:59:52Z
    Status:                False
    Type:                  Failing
    Last Transition Time:  2021-05-29T01:47:37Z
    Message:               Cluster version is 4.7.10
    Status:                False
    Type:                  Progressing
    Last Transition Time:  2021-05-29T01:19:32Z
    Message:               Unable to retrieve available updates: Get "https://api.openshift.com/api/upgrades_info/v1/graph?arch=amd64&channel=stable-4.7&id=8d06a718-a5dc-4749-a503-03df84a3716e&version=4.7.10": dial tcp 100.25.9.144:443: connect: connection timed out
    Reason:                RemoteFailed
    Status:                False
    Type:                  RetrievedUpdates
  Desired:
    Image:    quay.io/openshift-release-dev/ocp-release@sha256:24f0bcf67474e06ceb1091fc63bddd6010e1d13f5fe5604962a4579ee98b8e22
    URL:      https://access.redhat.com/errata/RHSA-2021:1483
    Version:  4.7.10
  History:
    Completion Time:    2021-05-29T01:47:37Z
    Image:              quay.io/openshift-release-dev/ocp-release@sha256:24f0bcf67474e06ceb1091fc63bddd6010e1d13f5fe5604962a4579ee98b8e22
    Started Time:       2021-05-29T01:19:32Z
    State:              Completed
    Verified:           false
    Version:            4.7.10
  Observed Generation:  1
  Version Hash:         S3hfdRoYCxY=
Events:                 <none>
```

#### Look at the Nodes

Execute the following command to see a list of the **Nodes** that OpenShift knows about:

```bash
oc get nodes
```

The output should look something like the following:

```
NAME               STATUS   ROLES    AGE   VERSION
master-ocs-0-120   Ready    master   95m   v1.20.0+e3fdce4
master-ocs-1-120   Ready    master   95m   v1.20.0+e3fdce4
master-ocs-2-120   Ready    master   95m   v1.20.0+e3fdce4
worker-ocs-0-120   Ready    worker   85m   v1.20.0+e3fdce4
worker-ocs-1-120   Ready    worker   86m   v1.20.0+e3fdce4
worker-ocs-2-120   Ready    worker   86m   v1.20.0+e3fdce4
```

You have 3 masters and 3 workers. The OpenShift **Master** is also a **Node** because it needs to participate in the software defined network (SDN). If you need additional nodes for additional purposes, you can create them very easily when using IPI and leveraging the cloud provider operators. You will create nodes to run OpenShift infrastructure components (registry, router, etc.) in a subsequent exercise.

Exit out of the `cloud-user` user shell.

```
exit
```

#### Check the Web Console

OpenShift provides a web console for users, developers, application operators, and administrators to interact with the environment. Many of the cluster administration functions, including upgrading the cluster itself, can be performed simply by using the web console.

The web console actually runs as an application inside the OpenShift environment and is exposed via the OpenShift Router. You will learn more about the router in a subsequent exercise. For now, you can simply control+click the link:

[https://console-openshift-console.apps.odflab-55.container-contest.top:6443](https://console-openshift-console.apps.odflab-55.container-contest.top:6443/)

#### You will now exit the ssh session

```
exit
```

If you accidentally hit exit more than once and connection to the console closed, refresh the webpage to reconnect.

You might receive a self-signed certificate error in your browser when you first visit the web console. When OpenShift is installed, by default, a CA and SSL certificates are generated for all inter-component communication within OpenShift, including the web console. Some lab instances were installed with Let’s Encrypt certificates, so not all will get this warning.

# Application Management Basics

## Application Management Basics

In this module, you will deploy a sample application using the `oc` tool and learn about some of the core concepts, fundamental objects, and basics of application management on OpenShift Container Platform.

### Core OpenShift Concepts

As a future administrator of OpenShift, it is important to understand several core building blocks as it relates to applications. Understanding these building blocks will help you better see the big picture of application management on the platform.

### Projects

A **Project** is a "bucket" of sorts. It’s a meta construct where all of a user’s resources live. From an administrative perspective, each **Project** can be thought of like a tenant. **Projects** may have multiple users who can access them, and users may be able to access multiple **Projects**. Technically speaking, a user doesn’t own the resources, the **Project** does. Deleting the user doesn’t affect any of the created resources.

For this exercise, first create a **Project** to hold some resources:

```bash
oc new-project app-management
```

### Deploy a Sample Application

The `new-app` command provides a very simple way to tell OpenShift to run things. You simply provide it with one of a wide array of inputs, and it figures out what to do. Users will commonly use this command to get OpenShift to launch existing images, to create builds of source code and ultimately deploy them, to instantiate templates, and so on.

You will now launch a specific image that exists on Dockerhub

```bash
oc new-app quay.io/thoraxe/mapit
```

The output will look like:

```
--> Found container image 7ce7ade (3 years old) from quay.io for "quay.io/thoraxe/mapit"

    * An image stream tag will be created as "mapit:latest" that will track this image

--> Creating resources ...
    imagestream.image.openshift.io "mapit" created
    deployment.apps "mapit" created
    service "mapit" created
--> Success
    Application is not exposed. You can expose services to the outside world by executin
g one or more of the commands below:
     'oc expose service/mapit'
    Run 'oc status' to view your app.
```

You can see that OpenShift automatically created several resources as the output of this command. We will take some time to explore the resources that were created.

For more information on the capabilities of `new-app`, take a look at its help message by running `oc new-app -h`.

### Pods

![openshift pod](https://preview.cloud.189.cn/image/imageAction?param=C40AC7F4567E62F798A05F715112810FAF554A4C00D7EB8D9CA12FE8C26B4DFCCEE06E07E0BE65E4C6F8FCA75489AC6BF64268DB21A0F6E1FDDA12715CCF41DD8C90A827C65D90D2FD9637847720837563078D6152869D82C4D007AA067DE4B0FB290483093864E5158F71070648B004)

Figure 1. OpenShift Pods

Pods are one or more containers deployed together on a host. A pod is the smallest compute unit you can define, deploy and manage in OpenShift. Each pod is allocated its own internal IP address on the SDN and owns the entire port range. The containers within pods can share local storage space and networking resources.

Pods are treated as **static** objects by OpenShift, i.e., one cannot change the pod definition while running.

You can get a list of pods:

```bash
oc get pods
```

And you will see something like the following:

```
NAME                     READY   STATUS    RESTARTS   AGE
mapit-764c5bf8b8-l49z7   1/1     Running   0          2m53s
```

| NOTE | Pod names are dynamically generated as part of the deployment process, which you will learn about shortly. Your name will be slightly different. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The `describe` command will give you more information on the details of a pod. In the case of the pod name above:

```bash
oc describe pod -l deployment=mapit
```

| NOTE | The `-l deployment=mapit` selects the pod that is related to the `Deployment` which will be discussed later. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

And you will see output similar to the following:

```
Name:         mapit-764c5bf8b8-l49z7
Namespace:    app-management
Priority:     0
Node:         ip-10-0-128-29.us-west-2.compute.internal/10.0.128.29
Start Time:   Tue, 10 Nov 2020 21:01:09 +0000
Labels:       deployment=mapit
              pod-template-hash=764c5bf8b8
Annotations:  k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "",
                    "interface": "eth0",
                    "ips": [
                        "10.129.2.99"
                    ],
                    "default": true,
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "",
                    "interface": "eth0",
                    "ips": [
                        "10.129.2.99"
                    ],
                    "default": true,
                    "dns": {}
                }]
              openshift.io/generated-by: OpenShiftNewApp
              openshift.io/scc: restricted
Status:       Running
IP:           10.129.2.99
IPs:
  IP:           10.129.2.99
Controlled By:  ReplicaSet/mapit-764c5bf8b8
Containers:
  mapit:
    Container ID:   cri-o://fb708e659c19c6aaf8211bf7e3029f8adc8cf14959bcaefa5c7e6df17d37
feaf
    Image:          quay.io/thoraxe/mapit@sha256:8c7e0349b6a016e3436416f3c54debda4594ba0
9fd34b8a0dee0c4497102590d
    Image ID:       quay.io/thoraxe/mapit@sha256:8c7e0349b6a016e3436416f3c54debda4594ba0
9fd34b8a0dee0c4497102590d
    Ports:          9779/TCP, 8080/TCP, 8778/TCP
    Host Ports:     0/TCP, 0/TCP, 0/TCP
    State:          Running
      Started:      Tue, 10 Nov 2020 21:01:29 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-v7fpq (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-v7fpq:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-v7fpq
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason          Age    From               Message
  ----    ------          ----   ----               -------
  Normal  Scheduled       6m50s  default-scheduler  Successfully assigned app-management
/mapit-764c5bf8b8-l49z7 to ip-10-0-128-29.us-west-2.compute.internal
  Normal  AddedInterface  6m48s  multus             Add eth0 [10.129.2.99/23]
  Normal  Pulling         6m48s  kubelet            Pulling image "quay.io/thoraxe/mapit
@sha256:8c7e0349b6a016e3436416f3c54debda4594ba09fd34b8a0dee0c4497102590d"
  Normal  Pulled          6m31s  kubelet            Successfully pulled image "quay.io/t
horaxe/mapit@sha256:8c7e0349b6a016e3436416f3c54debda4594ba09fd34b8a0dee0c4497102590d" in
 16.762028989s
  Normal  Created         6m31s  kubelet            Created container mapit
  Normal  Started         6m31s  kubelet            Started container mapit
```

This is a more detailed description of the pod that is running. You can see what node the pod is running on, the internal IP address of the pod, various labels, and other information about what is going on.

### Services

![openshift service](https://preview.cloud.189.cn/image/imageAction?param=0508F4D9054FA3B42996FE4C0687306F17BF0FC4AE9A3FDC3AE913DAA8C025F047B1040138433C9FB824F8A7E477A49BAE3F4388A09781B84A5198F04CC3364A270826AC271B4402980D33D6A0C08EEE5381FE9F3C058F765AB7711D9D65E6B45597F499F351FFC94DD5983F7DAD1981)

Figure 2. OpenShift Service

**Services** provide a convenient abstraction layer inside OpenShift to find a group of like **Pods**. They also act as an internal proxy/load balancer between those **Pods** and anything else that needs to access them from inside the OpenShift environment. For example, if you needed more `mapit` instances to handle the load, you could spin up more **Pods**. OpenShift automatically maps them as endpoints to the **Service**, and the incoming requests would not notice anything different except that the **Service** was now doing a better job handling the requests.

When you asked OpenShift to run the image, the `new-app` command automatically created a **Service** for you. Remember that services are an internal construct. They are not available to the "outside world", or anything that is outside the OpenShift environment. That’s OK, as you will learn later.

The way that a **Service** maps to a set of **Pods** is via a system of **Labels** and **Selectors**. **Services** are assigned a fixed IP address and many ports and protocols can be mapped.

There is a lot more information about [Services](https://docs.openshift.com/container-platform/4.6/architecture/understanding-development.html#understanding-kubernetes-pods), including the YAML format to make one by hand, in the official documentation.

You can see the current list of services in a project with:

```bash
oc get services
```

You will see something like the following:

```
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
mapit   ClusterIP   172.30.167.160   <none>        8080/TCP,8778/TCP,9779/TCP   26
```

| NOTE | Service IP addresses are dynamically assigned on creation and are immutable. The IP of a service will never change, and the IP is reserved until the service is deleted. Your service IP will likely be different. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Just like with pods, you can `describe` services, too. In fact, you can `describe` most objects in OpenShift:

```bash
oc describe service mapit
```

You will see something like the following:

```
Name:              mapit
Namespace:         app-management
Labels:            app=mapit
                   app.kubernetes.io/component=mapit
                   app.kubernetes.io/instance=mapit
Annotations:       openshift.io/generated-by: OpenShiftNewApp
Selector:          deployment=mapit
Type:              ClusterIP
IP:                172.30.167.160
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.129.2.99:8080
Port:              8778-tcp  8778/TCP
TargetPort:        8778/TCP
Endpoints:         10.129.2.99:8778
Port:              9779-tcp  9779/TCP
TargetPort:        9779/TCP
Endpoints:         10.129.2.99:9779
Session Affinity:  None
Events:            <none>
```

Information about all objects (their definition, their state, and so forth) is stored in the etcd datastore. etcd stores data as key/value pairs, and all of this data can be represented as serializable data objects (JSON, YAML).

Take a look at the YAML output for the service:

```bash
oc get service mapit -o yaml
```

You will see something like the following:

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  creationTimestamp: "2020-11-10T21:01:09Z"
  labels:
    app: mapit
    app.kubernetes.io/component: mapit
    app.kubernetes.io/instance: mapit
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:openshift.io/generated-by: {}
        f:labels:
          .: {}
          f:app: {}
          f:app.kubernetes.io/component: {}
          f:app.kubernetes.io/instance: {}
      f:spec:
        f:ports:
          .: {}
          k:{"port":8080,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
          k:{"port":8778,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
          k:{"port":9779,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:selector:
          .: {}
          f:deployment: {}
        f:sessionAffinity: {}
        f:type: {}
    manager: oc
    operation: Update
    time: "2020-11-10T21:01:09Z"
  name: mapit
  namespace: app-management
  resourceVersion: "106194"
  selfLink: /api/v1/namespaces/app-management/services/mapit
  uid: 921c2e2c-a53e-4f83-8e76-9df962069314
spec:
  clusterIP: 172.30.167.160
  ports:
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: 8778-tcp
    port: 8778
    protocol: TCP
    targetPort: 8778
  - name: 9779-tcp
    port: 9779
    protocol: TCP
    targetPort: 9779
  selector:
    deployment: mapit
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Take note of the `selector` stanza. Remember it.

It is also of interest to view the YAML of the **Pod** to understand how OpenShift wires components together. Go back and find the name of your `mapit` **Pod**, and then execute the following:

```bash
oc get pod -l deployment=mapit -o jsonpath='{.items[*].metadata.labels}' | jq -r
```

| NOTE | The `-o jsonpath` selects a specific field. In this case we are asking for the `labels` section in the manifest. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The output should look something like this:

```
{
  "deployment": "mapit",
  "pod-template-hash": "764c5bf8b8"
}
```

- The **Service** has a `selector` stanza that refers to `deployment: mapit`.
- The **Pod** has multiple **Labels**:
  - `deployment: mapit`
  - `pod-template-hash: 764c5bf8b8`

**Labels** are just key/value pairs. Any **Pod** in this **Project** that has a **Label** that matches the **Selector** will be associated with the **Service**. If you look at the `describe` output again, you will see that there is one endpoint for the service: the existing `mapit` **Pod**.

The default behavior of `new-app` is to create just one instance of the item requested. We will see how to modify/adjust this in a moment, but there are a few more concepts to learn first.

### Background: Deployment Configurations and Replica Sets

While **Services** provide routing and load balancing for **Pods**, which may go in and out of existence, **ReplicaSets** (RS) are used to specify and then ensure the desired number of **Pods** (replicas) are in existence. For example, if you always want an application to be scaled to 3 **Pods** (instances), a **ReplicaSet** is needed. Without an RS, any **Pods** that are killed or somehow die/exit are not automatically restarted. **ReplicaSets** are how OpenShift "self heals".

A **Deployments** (deploy) defines how something in OpenShift should be deployed. From the [deployments documentation](https://docs.openshift.com/container-platform/4.6/applications/deployments/what-deployments-are.html#deployments-kube-deployments_what-deployments-are):

```
Deployments describe the desired state of a particular component of an
application as a Pod template. Deployments create ReplicaSets, which
orchestrate Pod lifecycles.
```

In almost all cases, you will end up using the **Pod**, **Service**, **ReplicaSet** and **Deployment** resources together. And, in almost all of those cases, OpenShift will create all of them for you.

There are some edge cases where you might want some **Pods** and an **RS** without a **Deployments** or a **Service**, and others, but these are advanced topics not covered in these exercises.

| NOTE | Earlier versions of OpenShift used something called a **DeploymentConfig**. While still a valid deployment mechanism, moving forward the **Deployment** will be what will be created with `oc new-app`. See the [official documentation](https://docs.openshift.com/container-platform/4.6/applications/deployments/what-deployments-are.html#deployments-comparing-deploymentconfigs_what-deployments-are) for more details. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Exploring Deployment-related Objects

Now that we know the background of what a **ReplicaSet** and **Deployment** are, we can explore how they work and are related. Take a look at the **Deployment** (deploy) that was created for you when you told OpenShift to stand up the `mapit` image:

```bash
oc get deploy
```

You will see something like the following:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mapit   1/1     1            1           76m
```

To get more details, we can look into the **ReplicaSet** (**RS**).

Take a look at the **ReplicaSet** (RS) that was created for you when you told OpenShift to stand up the `mapit` image:

```bash
oc get rs
```

You will see something like the following:

```
NAME               DESIRED   CURRENT   READY   AGE
mapit-764c5bf8b8   1         1         1       77m
```

This lets us know that, right now, we expect one **Pod** to be deployed (`Desired`), and we have one **Pod** actually deployed (`Current`). By changing the desired number, we can tell OpenShift that we want more or less **Pods**.

### Scaling the Application

Let’s scale our mapit "application" up to 2 instances. We can do this with the `scale` command.

```bash
oc scale --replicas=2 deploy/mapit
```

To verify that we changed the number of replicas, issue the following command:

```bash
oc get rs
```

You will see something like the following:

```
NAME               DESIRED   CURRENT   READY   AGE
mapit-764c5bf8b8   2         2         2       79m
mapit-8695cb9c67   0         0         0       79m
```

| NOTE | The "older" version was kept. This is to we can "rollback" to a previous version of the application. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

You can see that we now have 2 replicas. Let’s verify the number of pods with the `oc get pods` command:

```bash
oc get pods
```

You will see something like the following:

```
NAME                     READY   STATUS    RESTARTS   AGE
mapit-764c5bf8b8-b4vpn   1/1     Running   0          112s
mapit-764c5bf8b8-l49z7   1/1     Running   0          81m
```

And lastly, let’s verify that the **Service** that we learned about in the previous lab accurately reflects two endpoints:

```bash
oc describe svc mapit
```

You will see something like the following:

```
Name:              mapit
Namespace:         app-management
Labels:            app=mapit
                   app.kubernetes.io/component=mapit
                   app.kubernetes.io/instance=mapit
Annotations:       openshift.io/generated-by: OpenShiftNewApp
Selector:          deployment=mapit
Type:              ClusterIP
IP:                172.30.167.160
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.128.2.19:8080,10.129.2.99:8080
Port:              8778-tcp  8778/TCP
TargetPort:        8778/TCP
Endpoints:         10.128.2.19:8778,10.129.2.99:8778
Port:              9779-tcp  9779/TCP
TargetPort:        9779/TCP
Endpoints:         10.128.2.19:9779,10.129.2.99:9779
Session Affinity:  None
Events:            <none>
```

Another way to look at a **Service**'s endpoints is with the following:

```bash
oc get endpoints mapit
```

And you will see something like the following:

```
NAME    ENDPOINTS                                                        AGE
mapit   10.128.2.19:8080,10.129.2.99:8080,10.128.2.19:9779 + 3 more...   81m
```

Your IP addresses will likely be different, as each pod receives a unique IP within the OpenShift environment. The endpoint list is a quick way to see how many pods are behind a service.

Overall, that’s how simple it is to scale an application (**Pods** in a **Service**). Application scaling can happen extremely quickly because OpenShift is just launching new instances of an existing image, especially if that image is already cached on the node.

One last thing to note is that there are actually several ports defined on this **Service**. Earlier we said that a pod gets a single IP and has control of the entire port space on that IP. While something running inside the **Pod** may listen on multiple ports (single container using multiple ports, individual containers using individual ports, a mix), a **Service** can actually proxy/map ports to different places.

For example, a **Service** could listen on port 80 (for legacy reasons) but the **Pod** could be listening on port 8080, 8888, or anything else.

In this `mapit` case, the image we ran has several `EXPOSE` statements in the `Dockerfile`, so OpenShift automatically created ports on the service and mapped them into the **Pods**.

### Application "Self Healing"

Because OpenShift’s **RSs** are constantly monitoring to see that the desired number of **Pods** are actually running, you might also expect that OpenShift will "fix" the situation if it is ever not right. You would be correct!

Now that we have two **Pods** running right now, let’s see what happens when we delete them. Frist, run the `oc get pods` command, and make note of the **Pod** names:

```bash
oc get pods
```

You will see something like the following:

```
NAME                     READY   STATUS    RESTARTS   AGE
mapit-764c5bf8b8-lxnvw   1/1     Running   0          2m28s
mapit-764c5bf8b8-rscss   1/1     Running   0          2m54s
```

Now, delete the pods that belog to the **Deployment** `mapit`:

```bash
oc delete pods -l deployment=mapit
```

Run the `oc get pods` command once again:

```bash
oc get pods
```

Did you notice anything? There are new containers already running!

The **Pods** has a different name. That’s because OpenShift almost immediately detected that the current state (0 **Pods**, because they were deleted) didn’t match the desired state (2 **Pods**), and it fixed it by scheduling the **Pods**.

### Background: Routes

![openshift route](https://preview.cloud.189.cn/image/imageAction?param=6B19D51D38A55E6390CE082D100C09CD6E97EA60B51EF4DC6451D21468007AB0E815698A9F4275BB41097224AD71C16A2C779C3176CD340CA0A5F591B58827392D85E4D6527EC992DDA2A6E68AAA96B2A406BB0D3C50049E0489D314863CA5FB44CCB49281CE6463718BC60857D4AE20)

Figure 3. OpenShift Route

While **Services** provide internal abstraction and load balancing within an OpenShift environment, sometimes clients (users, systems, devices, etc.) **outside** of OpenShift need to access an application. The way that external clients are able to access applications running in OpenShift is through the OpenShift routing layer. And the data object behind that is a **Route**.

The default OpenShift router (HAProxy) uses the HTTP header of the incoming request to determine where to proxy the connection. You can optionally define security, such as TLS, for the **Route**. If you want your **Services** (and by extension, your **Pods**) to be accessible to the outside world, then you need to create a **Route**.

Do you remember setting up the router? You probably don’t. That’s because the installation deployed an Operator for the router, and the operator created a router for you! The router lives in the `openshift-ingress` **Project**, and you can see information about it with the following command:

```bash
oc describe deployment router-default -n openshift-ingress
```

You will explore the Operator for the router more in a subsequent exercise.

### Creating a Route

Creating a **Route** is a pretty straight-forward process. You simply `expose` the **Service** via the command line. If you remember from earlier, your **Service** name is `mapit`. With the **Service** name, creating a **Route** is a simple one-command task:

```bash
oc expose service mapit
```

You will see:

```
route.route.openshift.io/mapit exposed
```

Verify the **Route** was created with the following command:

```bash
oc get route
```

You will see something like:

```
NAME    HOST/PORT                                             PATH   SERVICES   PORT       TERMINATION   WILDCARD
mapit   mapit-app-management.apps.odflab-55.container-contest.top:6443                   mapit      8080-tcp                 None
```

If you take a look at the `HOST/PORT` column, you’ll see a familiar looking FQDN. The default behavior of OpenShift is to expose services on a formulaic hostname:

```
{SERVICENAME}-{PROJECTNAME}.{ROUTINGSUBDOMAIN}
```

In the subsequent router Operator labs we’ll explore this and other configuration options.

While the router configuration specifies the domain(s) that the router should listen for, something still needs to get requests for those domains to the Router in the first place. There is a wildcard DNS entry that points `*.apps...` to the host where the router lives. OpenShift concatenates the **Service** name, **Project** name, and the routing subdomain to create this FQDN/URL.

You can visit this URL using your browser, or using `curl`, or any other tool. It should be accessible from anywhere on the internet.

The **Route** is associated with the **Service**, and the router automatically proxies connections directly to the **Pod**. The router itself runs as a **Pod**. It bridges the "real" internet to the SDN.

If you take a step back to examine everything you’ve done so far, in three commands you deployed an application, scaled it, and made it accessible to the outside world:

```
oc new-app quay.io/thoraxe/mapit
oc scale --replicas=2 deploy/mapit
oc expose service mapit
```

### Scale Down

Before we continue, go ahead and scale your application down to a single instance:

```bash
oc scale --replicas=1 deploy/mapit
```

### Application Probes

OpenShift provides rudimentary capabilities around checking the liveness and/or readiness of application instances. If the basic checks are insufficient, OpenShift also allows you to run a command inside the **Pod**/container in order to perform the check. That command could be a complicated script that uses any language already installed inside the container image.

There are two types of application probes that can be defined:

**Liveness Probe**

A liveness probe checks if the container in which it is configured is still running. If the liveness probe fails, the container is killed, which will be subjected to its restart policy.

**Readiness Probe**

A readiness probe determines if a container is ready to service requests. If the readiness probe fails, the endpoint’s controller ensures the container has its IP address removed from the endpoints of all services that should match it. A readiness probe can be used to signal to the endpoint’s controller that even though a container is running, it should not receive any traffic.

More information on probing applications is available in the [Application Health](https://docs.openshift.com/container-platform/4.6/applications/application-health.html) section of the documentation.

### Add Probes to the Application

The `oc set` command can be used to perform several different functions, one of which is creating and/or modifying probes. The `mapit` application exposes an endpoint which we can check to see if it is alive and ready to respond. You can test it using `curl`:

```bash
curl mapit-app-management.apps.odflab-55.container-contest.top:6443/health
```

You will get some JSON as a response:

```json
{"status":"UP","diskSpace":{"status":"UP","total":10724835328,"free":10257825792,"threshold":10485760}}
```

We can ask OpenShift to probe this endpoint for liveness with the following command:

```bash
oc set probe deploy/mapit --liveness --get-url=http://:8080/health --initial-delay-seconds=30
```

You can then see that this probe is defined in the `oc describe` output:

```bash
oc describe deploy mapit
```

You will see a section like:

```
...
  Containers:
   mapit:
    Image:        quay.io/thoraxe/mapit@sha256:8c7e0349b6a016e3436416f3c54debda
4594ba09fd34b8a0dee0c4497102590d
    Ports:        9779/TCP, 8080/TCP, 8778/TCP
    Host Ports:   0/TCP, 0/TCP, 0/TCP
    Liveness:     http-get http://:8080/health delay=30s timeout=1s period=10s
#success=1 #failure=3
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
...
```

Similarly, you can set a readiness probe in the same manner:

```bash
oc set probe deploy/mapit --readiness --get-url=http://:8080/health --initial-delay-seconds=30
```

### Examining Deployments and ReplicaSets

Each change to the **Deployment** is counted as a *configuration* change, which *triggers* a new *deployment*. The **Deployment** in in charge of which **ReplicaSet** to deploy. The *newest* is always deployed.

Execute the following:

```bash
oc get deployments
```

You should see something like:

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mapit   1/1     1            1           131m
```

You made two material configuration changes (plus a scale), after the initial deployment, thus you are now on the fourth revision of the **Deployment**.

Execute the following:

```bash
oc get replicasets
```

You should see something like:

```
NAME               DESIRED   CURRENT   READY   AGE
mapit-5f695ff4b8   1         1         1       4m19s
mapit-668f69cdd5   0         0         0       6m18s
mapit-764c5bf8b8   0         0         0       133m
mapit-8695cb9c67   0         0         0       133m
```

Each time a new deployment is triggered, the deployer pod creates a new **ReplicaSet** which then is responsible for ensuring that pods exist. Notice that the old RSs have a desired scale of zero, and the most recent RS has a desired scale of 1.

If you `oc describe` each of these RSs you will see how earlier versions have no probes, and the latest running RS has the new probes.

# Application Storage Basics

## Application Storage Basics

If a pod in OpenShift needs reliable storage, for example, to host a database, we would need to supply a **persistent** volume to the pod. This type of storage outlives the container, i.e. it persists when the pod dies. It typically comes from an external storage system.

Doing these exercises requires that you already have deployed the application featured in the Application Management Basics exercises. You should do those exercises now before continuing.

The `mapit` application currently doesn’t leverage any persistent storage. If the pod dies, so does all the content inside the container.

We will talk about this concept in more detail later. But let’s imagine for a moment, the `mapit` application needs persistent storage available under the `/app-storage` directory inside the container.

The directories that make up the container’s internal filesystem are a blend of the read-only layers of the container image and the top-most writable layer that is added as soon as a container instance is started from the image. The writable layer is disposed of once the container is deleted which can happen regularly in a dynamic container orchestration environment.

You should be in the `app-management` project from the previous lab. To make sure, run the following command:

```bash
oc project app-management
```

### Adding Persistent Volume Claims

Here’s how you would instruct OpenShift to create a `PersistentVolume` object, which represents external, persistent storage, and have it **mounted** inside the container’s filesystem:

```bash
oc set volume deploy/mapit --add --name=mapit-storage -t pvc --claim-mode=ReadWriteOnce --claim-size=1Gi --claim-name=mapit-storage --mount-path=/app-storage
```

The output looks like this:

```
deployment.apps/mapit volume updated
```

In the first step a **PersistentVolumeClaim** was created. This object represents a request for storage of a certain kind, with a certain capacity from the user to OpenShift.

Next the `Deployment` of `mapit` is updated to reference this storage and make it available under the `/app-storage` directory inside the pod.

You can see the new `Deployment` like this:

```bash
oc get deploy mapit
```

The output will show that a new deployment was created by inspecting the `AGE` column.

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mapit   1/1     1            1           14m
```

Likely, depending when you ran the command, you may or may not see that the new pod is still being spawned:

```bash
oc get pod
NAME             READY     STATUS              RESTARTS   AGE
mapit-3-ntd9w    1/1       Running             0          9m
mapit-4-d872b    0/1       ContainerCreating   0          5s
mapit-4-deploy   1/1       Running             0          10s
```

Take a look at the `Deployment` now:

```bash
oc describe deploy mapit
```

You will see there is now both `Mounts` and `Volumes` details about the new storage:

```
...
    Mounts:
      /app-storage from mapit-storage (rw)
  Volumes:
   mapit-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  mapit-storage
    ReadOnly:   false
...
```

But what is happening under the covers?

### Storage Classes

When OpenShift 4 was first installed, a dynamic storage provider for AWS EBS was configured. You can see this `StorageClass` with the following:

```bash
oc get storageclass
```

And you will see something like:

```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                   47m
gp2-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   47m
```

Any time a request for a volume is made (`PersistentVolumeClaim`) that doesn’t specify a `StorageClass`, the default will be used. In this case, the default is an EBS provisioner that will create an EBS GP2 volume of the requested size (in our example, 1Gi).

You’ll note that there is also a `gp2-csi`. This implements the [CSI interface](https://github.com/container-storage-interface/spec), which stands for "Container Storage Interface". This specification enables storage vendors to develop their plugins once, and have it work across various container orchestration systems.

### Persistent Volume (Claims)

The command you ran earlier referenced a `claim`. Storage in a Kubernetes environment uses a system of volume claims and volumes. A user makes a `PersistentVolumeClaim` and Kubernetes tries to find a `PersistentVolume` that matches. In the case where a volume does not already exist, if there is a dynamic provisioner that satisfies the claim, a `PersistentVolume` is created.

Execute the following:

```bash
oc get persistentvolume
```

You will see something like the following:

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pvc-4397c6be-9f53-490e-960d-c1b77de6000c   1Gi        RWO            Delete           Bound    app-management/mapit-storage   gp2                     12m
```

This is the volume that was created as a result of your earlier claim. Note that the volume is **bound** to the claim that exists in the `app-management` project.

Now, execute:

```bash
oc get persistentvolumeclaim -n app-management
```

You will see something like:

```
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mapit-storage   Bound    pvc-4397c6be-9f53-490e-960d-c1b77de6000c   1Gi        RWO            gp2            14m
```

### Testing Persistent Storage

Log on to the pod using the remote-shell capability of the `oc` client:

```bash
oc rsh $(oc get pods -l deployment=mapit -o name)
```

**Being in the container’s shell session**, list the content of the root directory from the perspective of the container’s namespace:

```bash
ls -ahl /
```

You will see a directory there called `/app-storage`:

```
total 20K
drwxr-xr-x.   1 root  root         81 Apr 12 19:11 .
drwxr-xr-x.   1 root  root         81 Apr 12 19:11 ..
-rw-r--r--.   1 root  root        16K Dec 14  2016 anaconda-post.log
drwxrwsr-x.   3 root  1000570000 4.0K Apr 12 19:10 app-storage (1)
lrwxrwxrwx.   1 root  root          7 Dec 14  2016 bin -> usr/bin
drwxrwxrwx.   1 jboss root         45 Aug  4  2017 deployments
drwxr-xr-x.   5 root  root        360 Apr 12 19:11 dev
drwxr-xr-x.   1 root  root         93 Jan 18  2017 etc
drwxr-xr-x.   2 root  root          6 Nov  5  2016 home
lrwxrwxrwx.   1 root  root          7 Dec 14  2016 lib -> usr/lib
lrwxrwxrwx.   1 root  root          9 Dec 14  2016 lib64 -> usr/lib64
drwx------.   2 root  root          6 Dec 14  2016 lost+found
drwxr-xr-x.   2 root  root          6 Nov  5  2016 media
drwxr-xr-x.   2 root  root          6 Nov  5  2016 mnt
drwxr-xr-x.   1 root  root         19 Jan 18  2017 opt
dr-xr-xr-x. 183 root  root          0 Apr 12 19:11 proc
dr-xr-x---.   2 root  root        114 Dec 14  2016 root
drwxr-xr-x.   1 root  root         21 Apr 12 19:11 run
lrwxrwxrwx.   1 root  root          8 Dec 14  2016 sbin -> usr/sbin
drwxr-xr-x.   2 root  root          6 Nov  5  2016 srv
dr-xr-xr-x.  13 root  root          0 Apr 10 14:34 sys
drwxrwxrwt.   1 root  root         92 Apr 12 19:11 tmp
drwxr-xr-x.   1 root  root         69 Dec 16  2016 usr
drwxr-xr-x.   1 root  root         41 Dec 14  2016 var
```

1. This is where the persistent storage appears inside the container

Amazon EBS volumes are read-write-once. In other words, because they are block storage, they may only be attached to one EC2 instance at a time, which means that only one container can use an EBS-based `PersistentVolume` at a time. In other words: read-write-once.

Execute the following inside the remote shell session:

```bash
echo "Hello World from OpenShift" > /app-storage/hello.txt
exit
```

Then, make sure your file is present:

```bash
oc rsh $(oc get pods -l deployment=mapit -o name) cat /app-storage/hello.txt
```

Now, to verify that persistent storage really works, delete your pod:

```bash
oc delete pods -l deployment=mapit && oc get pod
```

After some time, your new pod will be ready and running. Once it’s running, check the file:

```bash
oc rsh $(oc get pods -l deployment=mapit -o name) cat /app-storage/hello.txt
```

It’s still there. In fact, the new pod may not even be running on the same node as the old pod, which means that, under the covers, Kubernetes and OpenShift automatically attached the real, external storage to the right place at the right time.

If you needed read-write-many storage, file-based storage solutions can provide it. OpenShift Container Storage is a hyperconverged storage solution that can run inside OpenShift and provide file, block and even object storage by turning locally attached storage devices into storage pools and then creating volumes out of them.

# MachineSets, Machines, and Nodes

## MachineSets, Machines, and Nodes

Kubernetes `Nodes` are where containers are orchestrated and run in `Pods`. OpenShift 4 is fundamentally different than OpenShift 3 with respect to its focus on automated operations through the use of `Operators`. With respect to `Nodes`, there is a set of `Operators` and controllers that are focused on maintaining the state of the cluster size — including creating and destroying `Nodes`!

### MachineSets and Machines

As you saw in the application management exercises, there is a basic fundamental relationship between a `ReplicaSet`/`ReplicationController` and the `Pods` it creates. Similarly, there is a relationship between a `MachineSet` and a `Machine`.

The `MachineSet` defines a desired state for a set of `Machine` objects. When using IPI installations, then, there is an `Operator` whose job it is to make sure that there is actually an underlying instance for each `Machine` and, finally, that every `Machine` becomes a `Node`.

Execute the following:

```bash
oc get machineset -n openshift-machine-api
```

You will see something like:

```
NAME                                   DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-f4a3-lpxbs-worker-us-east-2a   1         1         1       1           47h
cluster-f4a3-lpxbs-worker-us-east-2b   1         1         1       1           47h
cluster-f4a3-lpxbs-worker-us-east-2c   1         1         1       1           47h
```

When OpenShift was installed, the installer interrogated the cloud provider to learn about the available AZs (since this is on AWS). It then ultimately created a `MachineSet` for each AZ and then scaled those sets, in order, until it reached the desired number of `Machines`. Since the default installation has 3 workers, the first 3 AZs got one worker each. The rest got zero.

```bash
oc get machine -n openshift-machine-api
```

You will see something like:

```
NAME                                         INSTANCE              STATE     TYPE         REGION      ZONE         AGE
cluster-f4a3-lpxbs-master-0                  i-04280885cafad3130   running   m4.xlarge    us-east-2   us-east-2a   47h
cluster-f4a3-lpxbs-master-1                  i-0def910edcae51d11   running   m4.xlarge    us-east-2   us-east-2b   47h
cluster-f4a3-lpxbs-master-2                  i-0beb5e40214d706fc   running   m4.xlarge    us-east-2   us-east-2c   47h
cluster-f4a3-lpxbs-worker-us-east-2a-b94pr   i-0a922c0fe765caa3c   running   m5.2xlarge   us-east-2   us-east-2a   47h
cluster-f4a3-lpxbs-worker-us-east-2b-m8gbx   i-0fb8d960b8a3a3343   running   m5.2xlarge   us-east-2   us-east-2b   47h
cluster-f4a3-lpxbs-worker-us-east-2c-5tmg7   i-0151c72cd85f85038   running   m5.2xlarge   us-east-2   us-east-2c   47h
```

Each `Machine` has a corresponding `INSTANCE`. Do those IDs look familiar? They are AWS EC2 instance IDs. You also see `Machines` for the OpenShift masters. They are not part of a `MachineSet` because they are somewhat stateful and their management is handled by different operators and through a different process.

There is currently no protection for the master `Machines`. Do not accidentally or intentionally delete them, as this will potentially break your cluster. It is repairable, but it is not fun.

Lastly, execute:

```bash
oc get nodes
```

You will see something like:

```
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-133-191.us-east-2.compute.internal   Ready    worker   44m   v1.16.2
ip-10-0-136-83.us-east-2.compute.internal    Ready    master   51m   v1.16.2
ip-10-0-152-132.us-east-2.compute.internal   Ready    worker   44m   v1.16.2
ip-10-0-157-139.us-east-2.compute.internal   Ready    master   51m   v1.16.2
ip-10-0-167-9.us-east-2.compute.internal     Ready    worker   44m   v1.16.2
ip-10-0-169-121.us-east-2.compute.internal   Ready    master   51m   v1.16.2
```

Each `Machine` ends up corresponding to a `Node`. With IPI, there is a bootstrap process where the machine operator will create an EC2 instance and then Ignition inside the CoreOS operating system will receive initial instructions from the operator. This results in the EC2 instance being configured as an OpenShift node and joining the cluster.

If you spend some time using `oc describe` and the various `Machine` and `Node` objects, you will figure out which ones correlate with which.

### Cluster Scaling

Because of the magic of `Operators` and the way in which OpenShift uses them to manage `Machines` and `Nodes`, scaling your cluster in OpenShift 4 is extremely trivial.

Look at the list of `MachineSets` again:

```bash
oc get machineset -n openshift-machine-api
```

Within that list, find the `MachineSet` that ends in `-2c`. It should have a scale of one. Now, let’s scale it to have two instances:

```bash
oc scale machineset cluster-5fa6-hx2ml-worker-us-east-2c -n openshift-machine-api --replicas=2
```

Take special note because your `MachineSet` name is likely different from the one in the lab guide. You should see a note that the `MachineSet` was successfully scaled. Now, look at the list of `Machines`:

```bash
oc get machines -n openshift-machine-api
```

You probably already have a new entry for a `Machine` with a `STATE` of `Pending`. After a few moments, it will have a corresponding EC2 instance ID and will look something like:

```
cluster-f4a3-lpxbs-worker-us-east-2c-h7gdt   i-0b9208ec47f0e206b   running   m5.2xlarge     us-east-2   us-east-2c   47s
```

At this point, in the background, the bootstrap process is happening automatically. After several minutes (up to five or so), take a look at the output of:

```bash
oc get nodes
```

You should see your fresh and happy new node as the one with a very young age:

```
ip-10-0-166-103.us-east-2.compute.internal   Ready    worker   1m   v1.16.2
```

It can take several minutes for a `Machine` to be prepared and added as a `Node`.

Scale the `MachineSet` from two back down to one before continuing.

```bash
oc scale machineset cluster-5fa6-hx2ml-worker-us-east-2c -n openshift-machine-api --replicas=1
```

# Infrastructure Nodes and Operators

## OpenShift Infrastructure Nodes

The OpenShift subscription model allows customers to run various core infrastructure components at no additional charge. In other words, a node that is only running core OpenShift infrastructure components is not counted in terms of the total number of subscriptions required to cover the environment.

OpenShift components that fall into the infrastructure categorization include:

- kubernetes and OpenShift control plane services ("masters")
- router
- container image registry
- cluster metrics collection ("monitoring")
- cluster aggregated logging
- service brokers

Any node running a container/pod/component not described above is considered a worker and must be covered by a subscription.

### More MachineSet Details

In the `MachineSets` exercises you explored using `MachineSets` and scaling the cluster by changing their replica count. In the case of an infrastructure node, we want to create additional `Machines` that have specific Kubernetes labels. Then, we can configure the various infrastructure components to run specifically on nodes with those labels.

Currently the operators that are used to control infrastructure components do not all support the use of taints and tolerations. This means that infrastructure workload will go onto the infrastructure nodes, but other workload is not specifically prevented from running on the infrastructure nodes. In other words, user workload may commingle with infrastructure workload until full taint/toleration support is implemented in all operators.

The use of taints/tolerations is not covered in any of these exercises.

To accomplish this, you will create additional `MachineSets`.

In order to understand how `MachineSets` work, run the following.

This will allow you to follow along with some of the following discussion.

```bash
oc get machineset -n openshift-machine-api -o yaml cluster-5fa6-hx2ml-worker-us-east-2c
```

#### Metadata

The `metadata` on the `MachineSet` itself includes information like the name of the `MachineSet` and various labels.

```YAML
metadata:
  creationTimestamp: 2019-01-25T16:00:34Z
  generation: 1
  labels:
    machine.openshift.io/cluster-api-cluster: 190125-3
    machine.openshift.io/cluster-api-machine-role: worker
    machine.openshift.io/cluster-api-machine-type: worker
  name: 190125-3-worker-us-east-1b
  namespace: openshift-machine-api
  resourceVersion: "9027"
  selfLink: /apis/cluster.k8s.io/v1alpha1/namespaces/openshift-machine-api/machinesets/190125-3-worker-us-east-1b
  uid: 591b4d06-20ba-11e9-a880-068acb199400
```

You might see some `annotations` on your `MachineSet` if you dumped one that had a `MachineAutoScaler` defined.

#### Selector

The `MachineSet` defines how to create `Machines`, and the `Selector` tells the operator which machines are associated with the set:

```YAML
spec:
  replicas: 2
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: 190125-3
      machine.openshift.io/cluster-api-machineset: 190125-3-worker-us-east-1b
```

In this case, the cluster name is `190125-3` and there is an additional label for the whole set.

### Template Metadata

The `template` is the part of the `MachineSet` that templates out the `Machine`. The `template` itself can have metadata associated, and we need to make sure that things match here when we make changes:

```YAML
  template:
    metadata:
      creationTimestamp: null
      labels:
        machine.openshift.io/cluster-api-cluster: 190125-3
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: 190125-3-worker-us-east-1b
```

#### Template Spec

The `template` needs to specify how the `Machine`/`Node` should be created. You will notice that the `spec` and, more specifically, the `providerSpec` contains all of the important AWS data to help get the `Machine` created correctly and bootstrapped.

In our case, we want to ensure that the resulting node inherits one or more specific labels. As you’ve seen in the examples above, labels go in `metadata` sections:

```YAML
  spec:
      metadata:
        creationTimestamp: null
      providerSpec:
        value:
          ami:
            id: ami-08871aee06d13e584
...
```

By default the `MachineSets` that the installer creates do not apply any additional labels to the node.

### Defining a Custom MachineSet

Now that you’ve analyzed an existing `MachineSet` it’s time to go over the rules for creating one, at least for a simple change like we’re making:

1. Don’t change anything in the `providerSpec`
2. Don’t change any instances of `machine.openshift.io/cluster-api-cluster: <clusterid>`
3. Give your `MachineSet` a unique `name`
4. Make sure any instances of `machine.openshift.io/cluster-api-machineset` match the `name`
5. Add labels you want on the nodes to `.spec.template.spec.metadata.labels`
6. Even though you’re changing `MachineSet` `name` references, be sure not to change the `subnet`.

This sounds complicated, but we have a little program and some steps that will do the hard work for you:

```bash
bash /opt/app-root/src/support/machineset-generator.sh 1 infra 0 | oc create -f -
export MACHINESET=$(oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=infra -o jsonpath='{.items[0].metadata.name}')
oc patch machineset $MACHINESET -n openshift-machine-api --type='json' -p='[{"op": "add", "path": "/spec/template/spec/metadata/labels", "value":{"node-role.kubernetes.io/worker":"", "node-role.kubernetes.io/infra":""} }]'
oc scale machineset $MACHINESET -n openshift-machine-api --replicas=3
```

Then go ahead and run:

```bash
oc get machineset -n openshift-machine-api
```

You should see the new infra set listed with a name similar to the following:

```
...
cluster-city-56f8-mc4pf-infra-us-east-2a    1         1                             13s
...
```

We don’t yet have any ready or available machines in the set because the instances are still coming up and bootstrapping. You can check `oc get machine -n openshift-machine-api` to see when the instance finally starts running. Then, you can use `oc get node` to see when the actual node is joined and ready.

It can take several minutes for a `Machine` to be prepared and added as a `Node`.

```bash
oc get nodes
NAME                                         STATUS   ROLES          AGE     VERSION
ip-10-0-133-134.us-east-2.compute.internal   Ready    infra,worker   8m     v1.16.2
ip-10-0-133-191.us-east-2.compute.internal   Ready    worker         61m    v1.16.2
ip-10-0-136-83.us-east-2.compute.internal    Ready    master         67m    v1.16.2
ip-10-0-138-24.us-east-2.compute.internal    Ready    infra,worker   8m1s   v1.16.2
ip-10-0-139-81.us-east-2.compute.internal    Ready    infra,worker   8m3s   v1.16.2
ip-10-0-152-132.us-east-2.compute.internal   Ready    worker         61m    v1.16.2
ip-10-0-157-139.us-east-2.compute.internal   Ready    master         67m    v1.16.2
ip-10-0-167-9.us-east-2.compute.internal     Ready    worker         61m    v1.16.2
ip-10-0-169-121.us-east-2.compute.internal   Ready    master         67m    v1.16.2
```

If you’re having trouble figuring out which node is the new one, take a look at the `AGE` column. It will be the youngest! Also, in the `ROLES` column you will notice that the new node has both a `worker` and an `infra` role.

### Check the Labels

In our case, the youngest node was named `ip-10-0-128-138.us-east-1.compute.internal`, so we can ask what its labels are:

```bash
oc get node ip-10-0-139-81.us-east-2.compute.internal --show-labels
```

And, in the `LABELS` column we see:

```
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m5.2xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-2,failure-domain.beta.kubernetes.io/zone=us-east-2a,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-10-0-140-3,kubernetes.io/os=linux,node-role.kubernetes.io/infra=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos
```

It’s hard to see, but our `node-role.kubernetes.io/infra` label is there.

### Add More Machinesets (or scale, or both)

In a realistic production deployment, you would want at least 3 `MachineSets` to hold infrastructure components. Both the logging aggregation solution and the service mesh will deploy ElasticSearch, and ElasticSearch really needs 3 instances spread across 3 discrete nodes. Why 3 `MachineSets`? Well, in theory, having multiple `MachineSets` in different AZs ensures that you don’t go completely dark if AWS loses an AZ.

The `MachineSet` you created with the scriptlet already created 3 replicas for you, so you don’t have to do anything for now. Don’t create any additional ones yourself, either — the AWS limits on the account you are using are purposefully small.

### Extra Credit

In the `openshift-machine-api` project are several `Pods`. One of them has a name like `machine-api-controllers-56bdc6874f-86jnb`. If you use `oc logs` on the various containers in that `Pod`, you will see the various operator bits that actually make the nodes come into existence.

## Quick Operator Background

Operators are just `Pods`. But they are special `Pods`. They are software that understands how to deploy and manage applications in a Kubernetes environment. The power of Operators relies on a recent Kubernetes feature called `CustomResourceDefinitions` (`CRD`). A `CRD` is exactly what it sounds like. They are a way to define a custom resource which is essentially extending the Kubernetes API with new objects.

If you wanted to be able to create/read/update/delete `Foo` objects in Kubernetes, you would create a `CRD` that defines what a `Foo` resource is and how it works. You can then create `CustomResources` (`CRs`) — instances of your `CRD`.

With Operators, the general pattern is that an Operator looks at `CRs` for its configuration, and then it *operates* on the Kubernetes environment to do whatever the configuration specifies. Now you will take a look at how some of the infrastructure operators in OpenShift do their thing.

## Moving Infrastructure Components

Now that you have some special nodes, it’s time to move various infrastructure components onto them.

### Router

The OpenShift router is managed by an `Operator` called `openshift-ingress-operator`. Its `Pod` lives in the `openshift-ingress-operator` project:

```bash
oc get pod -n openshift-ingress-operator
```

The actual default router instance lives in the `openshift-ingress` project. Take a look at the `Pods`.

```bash
oc get pods -n openshift-ingress -o wide
```

And you will see something like:

```
NAME                              READY   STATUS    RESTARTS   AGE   IP           NODE                                        NOMINATED NODE
router-default-7bc4c9c5cd-clwqt   1/1     Running   0          9h    10.128.2.7   ip-10-0-144-70.us-east-2.compute.internal   <none>
router-default-7bc4c9c5cd-fq7m2   1/1     Running   0          9h    10.131.0.7   ip-10-0-138-38.us-east-2.compute.internal   <none>
```

Review a `Node` on which a router is running:

```bash
oc get node ip-10-0-144-70.us-east-2.compute.internal
```

You will see that it has the role of `worker`.

```
NAME                                        STATUS   ROLES    AGE   VERSION
ip-10-0-144-70.us-east-2.compute.internal   Ready    worker   9h    v1.12.4+509916ce1
```

The default configuration of the router operator is to pick nodes with the role of `worker`. But, now that we have created dedicated infrastructure nodes, we want to tell the operator to put the router instances on nodes with the role of `infra`.

The OpenShift router operator uses a custom resource definition (`CRD`) called `ingresses.config.openshift.io` to define the default routing subdomain for the cluster:

```bash
oc get ingresses.config.openshift.io cluster -o yaml
```

The `cluster` object is observed by the router operator as well as the master. Yours likely looks something like:

```YAML
apiVersion: config.openshift.io/v1
kind: Ingress
metadata:
  creationTimestamp: 2019-04-08T14:37:49Z
  generation: 1
  name: cluster
  resourceVersion: "396"
  selfLink: /apis/config.openshift.io/v1/ingresses/cluster
  uid: e1ec463c-5a0b-11e9-93e8-028b0fb1636c
spec:
  domain: apps.odflab-55.container-contest.top:6443
status: {}
```

Individual router deployments are managed via the `ingresscontrollers.operator.openshift.io` CRD. There is a default one created in the `openshift-ingress-operator` namespace:

```bash
oc get ingresscontrollers.operator.openshift.io default -n openshift-ingress-operator -o yaml
```

Yours looks something like:

```YAML
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  creationTimestamp: 2019-04-08T14:46:15Z
  finalizers:
  - ingress.openshift.io/ingress-controller
  generation: 2
  name: default
  namespace: openshift-ingress-operator
  resourceVersion: "2056085"
  selfLink: /apis/operator.openshift.io/v1/namespaces/openshift-ingress-operator/ingresscontrollers/default
  uid: 0fac160d-5a0d-11e9-a3bb-02d64e703494
spec: {}
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2019-04-08T14:47:14Z
    status: "True"
    type: Available
  domain: apps.cluster-f4a3.f4a3.openshiftworkshop.com
  endpointPublishingStrategy:
    type: LoadBalancerService
  selector: ingress.operator.openshift.io/ingress-controller-deployment=default
```

To specify a `nodeSelector` that tells the router pods to hit the infrastructure nodes, we can apply the following configuration:

```bash
oc apply -f /opt/app-root/src/support/ingresscontroller.yaml
```

Run:

```bash
oc get pod -n openshift-ingress -o wide
```

Your session may timeout during the router move. Please refresh the page to get your session back. You will not lose your terminal session but may have to navigate back to this page manually.

If you’re quick enough, you might catch either `Terminating` or `ContainerCreating` pods. The `Terminating` pod was running on one of the worker nodes. The `Running` pods eventually are on one of our nodes with the `infra` role.

## Registry

The registry uses a similar `CRD` mechanism to configure how the operator deploys the actual registry pods. That CRD is `configs.imageregistry.operator.openshift.io`. You will edit the `cluster` CR object in order to add the `nodeSelector`. First, take a look at it:

```bash
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
```

You will see something like:

```YAML
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  creationTimestamp: "2019-08-06T13:57:22Z"
  finalizers:
  - imageregistry.operator.openshift.io/finalizer
  generation: 2
  name: cluster
  resourceVersion: "13218"
  selfLink: /apis/imageregistry.operator.openshift.io/v1/configs/cluster
  uid: 1cb6272a-b852-11e9-9a54-02fdf1f6ca7a
spec:
  defaultRoute: false
  httpSecret: fff8bb0952d32e0aa56adf0ac6f6cf5267e0627f7b42e35c508050b5be426f8fd5e5108bea314f4291eeacc0b95a2ea9f842b54d7eb61522238f2a2dc471f131
  logging: 2
  managementState: Managed
  proxy:
    http: ""
    https: ""
    noProxy: ""
  readOnly: false
  replicas: 1
  requests:
    read:
      maxInQueue: 0
      maxRunning: 0
      maxWaitInQueue: 0s
    write:
      maxInQueue: 0
      maxRunning: 0
      maxWaitInQueue: 0s
  storage:
    s3:
      bucket: image-registry-us-east-2-0a598598fc1649d8b96ed91a902b982c-1cbd
      encrypt: true
      keyID: ""
      region: us-east-2
      regionEndpoint: ""
status:
...
```

If you run the following command:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster -p '{"spec":{"nodeSelector":{"node-role.kubernetes.io/infra": ""}}}' --type=merge
```

It will modify the `.spec` of the registry CR in order to add the desired `nodeSelector`.

At this time the image registry is not using a separate project for its operator. Both the operator and the operand are housed in the `openshift-image-registry` project.

After you run the patch command you should see the registry pod being moved to the infra node. The registry is in the `openshift-image-registry` project. If you execute the following quickly enough:

```bash
oc get pod -n openshift-image-registry
```

You might see the old registry pod terminating and the new one starting. Since the registry is being backed by an S3 bucket, it doesn’t matter what node the new registry pod instance lands on. It’s talking to an object store via an API, so any existing images stored there will remain accessible.

Also note that the default replica count is 1. In a real-world environment you might wish to scale that up for better availability, network throughput, or other reasons.

If you look at the node on which the registry landed (see the section on the router), you’ll note that it is now running on an infra worker.

Lastly, notice that the `CRD` for the image registry’s configuration is not namespaced — it is cluster scoped. There is only one internal/integrated registry per OpenShift cluster.

## Monitoring

The Cluster Monitoring operator is responsible for deploying and managing the state of the Prometheus+Grafana+AlertManager cluster monitoring stack. It is installed by default during the initial cluster installation. Its operator uses a `ConfigMap` in the `openshift-monitoring` project to set various tunables and settings for the behavior of the monitoring stack.

The following `ConfigMap` definition will configure the monitoring solution to be redeployed onto infrastructure nodes.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
```

There is no `ConfigMap` created as part of the installation. Without one, the operator will assume default settings. Verify the `ConfigMap` is not defined in your cluster:

```bash
oc get configmap cluster-monitoring-config -n openshift-monitoring
```

You should see:

```
Error from server (NotFound): configmaps "cluster-monitoring-config" not found
```

The operator will, in turn, create several `ConfigMap` objects for the various monitoring stack components, and you can see them, too:

```bash
oc get configmap -n openshift-monitoring
```

You can create the new monitoring config with the following command:

```bash
oc create -f /opt/app-root/src/support/cluster-monitoring-configmap.yaml
```

Watch the monitoring pods move from `worker` to `infra` `Nodes` with:

```bash
watch 'oc get pod -n openshift-monitoring'
```

or:

```bash
oc get pod -w -n openshift-monitoring
```

## Logging

OpenShift’s log aggregation solution is not installed by default. There is a dedicated lab exercise that goes through the configuration and deployment of logging.

# Deploying and Managing OpenShift Container Storage

## Lab Overview

This module is for both system administrators and application developers interested in learning how to deploy and manage OpenShift Container Storage (OCS). In this module you will be using OpenShift Container Platform (OCP) 4.x and the OCS operator to deploy Ceph and the Multi-Cloud-Gateway (MCG) as a persistent storage solution for OCP workloads.

### In this lab you will learn how to

- Configure and deploy containerized Ceph and MCG
- Validate deployment of containerized Ceph and MCG
- Deploy the Rook toolbox to run Ceph and RADOS commands
- Create an application using Read-Write-Once (RWO) PVC that is based on Ceph RBD
- Create an application using Read-Write-Many (RWX) PVC that is based on CephFS
- Use OCS for Prometheus and AlertManager storage
- Use the MCG to create a bucket and use in an application
- Add more storage to the Ceph cluster
- Review OCS metrics and alerts
- Use must-gather to collect support information

![Showing OCS4 pods](https://preview.cloud.189.cn/image/imageAction?param=2FCC1C4F12EC878B6813F9D8F37800C5C79588C0E04A8643BC7ABA33A330148D91C18B59B7E90A52BBB015ECB91FB401FFE9AFF2CD1AEEACEE479E362C2D6C40175D1196D0D3B68448CFA97AB1148E002AE854EEE667E2FB613D255BE07D38B322727DBCA035FC75F2F107A801B5EFEF)

Figure 1. OpenShift Container Storage components

| NOTE | If you want more information about how Ceph works please review [Introduction to Ceph](https://dashboard-lab-ocp-cns.apps.odflab-55.container-contest.top/workshop/ocs4#_introduction_to_ceph) section before starting the exercises in this module. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

## Deploy your storage backend using the OCS operator

### Scale OCP cluster and add new worker nodes

In this section, you will first validate the OCP environment has 2 or 3 worker nodes before increasing the cluster size by additional 3 worker nodes for OCS resources. The `NAME` of your OCP nodes will be different than shown below.

```
oc get nodes -l node-role.kubernetes.io/worker -l '!node-role.kubernetes.io/infra','!node-role.kubernetes.io/master'
```

Example output:

```
NAME                                        STATUS   ROLES    AGE    VERSION
ip-10-0-153-37.us-east-2.compute.internal   Ready    worker   4d4h   v1.19.0+9f84db3
ip-10-0-170-25.us-east-2.compute.internal   Ready    worker   4d4h   v1.19.0+9f84db3
```

Now you are going to add 3 more OCP compute nodes to cluster using **machinesets**.

```
oc get machinesets -n openshift-machine-api | grep -v infra
```

This will show you the existing **machinesets** used to create the 2 or 3 worker nodes in the cluster already. There is a **machineset** for each of 3 AWS Availability Zones (AZ). Your **machinesets** `NAME` will be different than below. In the case of only 2 workers one of the **machinesets** will not have any machines (i.e., DESIRED=0) created.

Example output:

```
NAME                                        DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-ocs4-8613-bc282-worker-us-east-2a   1         1         1       1           4d4h
cluster-ocs4-8613-bc282-worker-us-east-2b   1         1         1       1           4d4h
cluster-ocs4-8613-bc282-worker-us-east-2c   0         0                             4d4h
```

Similar to the infrastructure nodes lab, create new **MachineSets** that will in turn create storage-specific nodes for your OCP cluster in the AWS AZs:

```
bash /opt/app-root/src/support/machineset-generator.sh 3 workerocs 0 | oc create -f -
oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=workerocs -o name | xargs oc patch -n openshift-machine-api --type='json' -p '[{"op": "add", "path": "/spec/template/spec/metadata/labels", "value":{"node-role.kubernetes.io/worker":"", "role":"storage-node", "cluster.ocs.openshift.io/openshift-storage":""} }]'
oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=workerocs -o name | xargs oc scale -n openshift-machine-api --replicas=1
```

Check that you have new **machines** created.

```
oc get machines -n openshift-machine-api | egrep 'NAME|workerocs'
```

They will be in `Provisioning` for sometime and eventually in a `Running` PHASE. The `NAME` of your machines will be different than shown below.

Example output:

```
NAME                                                 PHASE     TYPE          REGION      ZONE
         AGE
cluster-ocs4-8613-bc282-workerocs-us-east-2a-g6cfz   Running   m5.4xlarge    us-east-2   us-east-2a   3m48s
cluster-ocs4-8613-bc282-workerocs-us-east-2b-2zdgx   Running   m5.4xlarge    us-east-2   us-east-2b   3m48s
cluster-ocs4-8613-bc282-workerocs-us-east-2c-gg7br   Running   m5.4xlarge    us-east-2   us-east-2c   3m48s
```

You can see that the workerocs **machines** are also using the AWS EC2 instance type `m5.4xlarge`. The `m5.4xlarge` instance type has 16 cpus and 64 GB memory.

Now you want to see if our new **machines** are added to the OCP cluster.

```
watch "oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'"
```

This step could take more than 5 minutes. The result of this command needs to look like below before you proceed. All new workerocs **machinesets** should have an integer, in this case `1`, filled out for all rows and under columns `READY` and `AVAILABLE`. The `NAME` of your **machinesets** will be different than shown below.

Example output:

```
NAME                                           DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-ocs4-8613-bc282-workerocs-us-east-2a   1         1         1       1           16m
cluster-ocs4-8613-bc282-workerocs-us-east-2b   1         1         1       1           16m
cluster-ocs4-8613-bc282-workerocs-us-east-2c   1         1         1       1           16m
```

You can exit by pressing Ctrl+C.

Now check to see that you have 3 new OCP worker nodes. The `NAME` of your OCP nodes will be different than shown below.

```
oc get nodes -l node-role.kubernetes.io/worker -l '!node-role.kubernetes.io/infra','!node-role.kubernetes.io/master'
```

Example output:

```
NAME                                         STATUS   ROLES    AGE    VERSION
ip-10-0-147-230.us-east-2.compute.internal   Ready    worker   14m    v1.19.0+9f84db3
ip-10-0-153-37.us-east-2.compute.internal    Ready    worker   4d4h   v1.19.0+9f84db3
ip-10-0-170-25.us-east-2.compute.internal    Ready    worker   4d4h   v1.19.0+9f84db3
ip-10-0-175-8.us-east-2.compute.internal     Ready    worker   14m    v1.19.0+9f84db3
ip-10-0-209-53.us-east-2.compute.internal    Ready    worker   14m    v1.19.0+9f84db3
```

Let’s check to make sure the new OCP nodes have the OCS label. This label was added in the `workerocs` **machinesets** so every **machine** created using these **machinesets** will have this label.

```
oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
```

Example output:

```
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-147-230.us-east-2.compute.internal   Ready    worker   15m   v1.19.0+9f84db3
ip-10-0-175-8.us-east-2.compute.internal     Ready    worker   15m   v1.19.0+9f84db3
ip-10-0-209-53.us-east-2.compute.internal    Ready    worker   15m   v1.19.0+9f84db3
```

### Installing the OCS operator

In this section you will be using three of the worker OCP 4 nodes to deploy OCS 4 using the OCS Operator in OperatorHub. The following will be installed:

- An OCS **OperatorGroup**
- An OCS **Subscription**
- All other OCS resources (Operators, Ceph Pods, NooBaa Pods, StorageClasses)

Start with creating the `openshift-storage` namespace.

```
oc create namespace openshift-storage
```

You must add the monitoring label to this namespace. This is required to get prometheus metrics and alerts for the OCP storage dashboards. To label the `openshift-storage` namespace use the following command:

```
oc label namespace openshift-storage "openshift.io/cluster-monitoring=true"
```

| NOTE | The creation of the `openshift-storage` namespace, and the monitoring label added to this namespace, can also be done during the OCS operator installation using the **Openshift Web Console**. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Now switch over to your **Openshift Web Console**:

[https://console-openshift-console.apps.odflab-55.container-contest.top:6443](https://console-openshift-console.apps.odflab-55.container-contest.top:6443/)

Remember that the login is `kubeadmin` and the password is:

```
VyYA4-KWyRT-9bFCf-2Frvz
```

Once you are logged in, navigate to the **Operators** → **OperatorHub** menu.

![OCP OperatorHub](https://preview.cloud.189.cn/image/imageAction?param=43D23219991B7FECAEF98297219AEA777644E782C4C9068478F7A9C142F98AC830C20D9C427ECC294BF45D4E8AF5D79DD4ED7484F43824CB59B95ED2424ACBDBA9A3C7F9CC2FAF5D3240FA25636D49459DCE2595D5ABC3D0E8AA3ACBD6385196674B1230576DCC2679281778C2565AC3)

Figure 2. OCP OperatorHub

Now type `openshift container storage` in the **Filter by \*keyword…\*** box.

![OCP OperatorHub Filter](https://preview.cloud.189.cn/image/imageAction?param=AB74EF837E16963BA7FC1F55C8201A01114052724126E48423E953431A8D9F898DAA07A4BE35491260E678ED46815D8E0F75B566F9622427C2F2888AC6C11B0D3DF1ED3545AB871B97CD360C9638A6C3BFBCE8137BDA30BE11AEDF90A47876160496799F256363496BE101D7FE143EBC)

Figure 3. OCP OperatorHub filter on OpenShift Container Storage Operator

Select `OpenShift Container Storage Operator` and then select **Install**.

![OCP OperatorHub Install](https://preview.cloud.189.cn/image/imageAction?param=A4611EC05F939EEB8607D326519A116F150F0F665857BDC957C57E249156A21CFD13753D402BB233E5C3C229AC643B264BBA07DF5CEE2E55AB399893F8B99C38FCDC364D4ED61BE17E14264922478223292A8B8ACB67953B2C7CCBCC074B9085ACF543FAC627F6C1D0602E63BE153642)

Figure 4. OCP OperatorHub Install OpenShift Container Storage

On the next screen make sure the settings are as shown in this figure.

![OCP OperatorHub Subscribe](https://preview.cloud.189.cn/image/imageAction?param=127E5C99ABD88EFCCAF3B195180D30C3BF508F970EDF6DE457449336E8680BA71350DB8FD8E946255AE5AFA3EDF735D7EA50460479D9A286596D5CA3D7100C09137034EF9BF406DC2C0901ACCD146DDF293889BDC7658D80DCBFC61021CB2D953161762AAEB5E04F4A48B41C0C18BB28)

Figure 5. OCP Subscribe to OpenShift Container Storage

Click `Install`.

Now you can go back to your terminal window to check the progress of the installation.

```
watch oc -n openshift-storage get csv
```

Example output:

```
NAME                  DISPLAY                       VERSION   REPLACES   PHASE
ocs-operator.v4.6.0   OpenShift Container Storage   4.6.0                Succeeded
```

You can exit by pressing Ctrl+C.

The resource `csv` is a shortened word for `clusterserviceversions.operators.coreos.com`.

| CAUTION | Please wait until the operator `PHASE` changes to `Succeeded`This will mark that the installation of your operator was successful. Reaching this state can take several minutes. |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

You will now also see new operator pods in `openshift-storage` namespace:

```
oc -n openshift-storage get pods
```

Example output:

```
NAME                                   READY   STATUS    RESTARTS   AGE
noobaa-operator-698746cd47-sp6w9       1/1     Running   0          108s
ocs-metrics-exporter-78bc44687-pg4hk   1/1     Running   0          107s
ocs-operator-6d99bc6787-d7m9d          1/1     Running   0          108s
rook-ceph-operator-59f7fb95d6-sdjd8    1/1     Running   0          108s
```

Now switch back to your **Openshift Web Console** for the remainder of the installation for OCS 4.

Select `View Operator` in figure below to get to the OCS configuration screen.

![View Operator in openshift-storage namespacee](https://preview.cloud.189.cn/image/imageAction?param=A24FCC88CEE4ECC744F8A0C7F2128C9CE59EEFC0DC1CEE7E217688F1A7A1D7CE44505FE6C449B12B6AFE1492A7014C8EE41C7F70E78E137F87F4ED0F3185E768412E781DE360E6A00D7384D938DC7A4DD345E21DAF8083370D4C1EC5798FA616E2505365932CD16847D59D0955D70E70)

Figure 6. View Operator in openshift-storage namespace

![OCS configuration screen](https://preview.cloud.189.cn/image/imageAction?param=A2E81F4C82590DD9B5F0A0233DD2AD0227FE2513D4A5791DFD1A00FC6FDABE4DE51C6D2D047C9FCE66F55273CD6047CA632E4177E3D513F996D78FBCBB8457EF2A06545D7CB9E96C3E049B479861B12790516577BF65CD4C5CBF1E862ADEA6AD3CE02428C9FC99E869743220C7682843)

Figure 7. OCS configuration screen

On the top of the OCS configuration screen, scroll over to the right and click on `Storage Cluster` and then click on `Create Storage Cluster` to the far right. If you do not see `Create Storage Cluster` refresh your browser window.

![Create Storage Cluster](https://preview.cloud.189.cn/image/imageAction?param=00A801A49A1AF927712B471B0ACECE085AD3CB3E72A554995AD0464A425AF0FA0833E9C7319E0537DFB73E57FF055E3D5FEA575F0DD538EC3D511CBF3575124DB32947D47CE5B96F3A0EAF6BE8C9E5296B76704B4E7CD10AA8F886E3F06F47F9344B72305806F30FDEA9C69F88E36F73)

Figure 8. Create Storage Cluster

The `Create Storage Cluster` screen will display.

![Create Storage Cluster default settings](https://preview.cloud.189.cn/image/imageAction?param=9F384211FED7FC5BB08CF504753D943AA167475F5EB68326ADB64A3FC3005D0DFB51E2C210FD63759662ED0188AEEE035C51132A386BA2794070A901A7612AD389696B53658AEF3C06E814FDCDC5A351E764BD979EAAF032D064E93C19C84551F6A5B56A391DD7CA757E4C170A80FD78)

Figure 9. Create Storage Cluster default settings

Leave the default selection of `Internal`, `gp2`, `2 TiB` and Encryption `Disabled`.

![Create a new storage cluster](https://preview.cloud.189.cn/image/imageAction?param=EEFD649B274C9C089212F6929E198F1C3E336AAB061835EDD699D78D5B2759AEC221E0E8804B766960884392890CA1F53A5730ECAB3D143E8ADF5073E6134678144DD7D693333BD490809774A54DAAC4F31FF359B1A4C6109F5F47F0DA39493B217667CEF101F20A52E10063EC070764)

Figure 10. Create a new storage cluster

There should be 3 worker nodes already selected that had the OCS label applied in the last section. Execute command below and make sure they are all selected.

```
oc get nodes --show-labels | grep ocs | cut -d ' ' -f1
```

Then click on the button `Create` below the dialog box with the 3 workers selected with a `checkmark`.

You can watch the deployment using the **Openshift Web Console** by going back to the `Openshift Container Storage Operator` screen and selecting `All instances`.

Please wait until all **Pods** are marked as `Running` in the CLI or until you see all instances shown below as `Ready` Status in the Web Console as shown in the following diagram:

![OCS instance overview after cluster install is finished](https://preview.cloud.189.cn/image/imageAction?param=D53D484319A1FD03EB14CC5112E3056497740B71AC85895EDD2E9EDA9C96AD2A6F344C7149371DC8923D80F64AF57A49297C142023A7243C33E754E32E204382DEE712609AE7E23E106176F5063243061C1981FA9F61FBED2E7ACE3DB71F591D0E5326B22EA47AE43F7D20C53D6DB034)

Figure 11. OCS instance overview after cluster install is finished

```
oc -n openshift-storage get pods
```

Output when the cluster installation is finished

```
NAME                                                              READY   STATUS      RESTART
S   AGE
csi-cephfsplugin-875xd                                            3/3     Running     0
    23m
csi-cephfsplugin-bncsj                                            3/3     Running     0
    23m
csi-cephfsplugin-hjv77                                            3/3     Running     0
    23m
csi-cephfsplugin-lch4m                                            3/3     Running     0
    23m
csi-cephfsplugin-provisioner-6cfdc4bfbb-cklxs                     6/6     Running     0
    23m
csi-cephfsplugin-provisioner-6cfdc4bfbb-krkq5                     6/6     Running     0
    23m
csi-cephfsplugin-wtp4v                                            3/3     Running     0
    23m
csi-rbdplugin-7clqf                                               3/3     Running     0
    23m
csi-rbdplugin-8nllt                                               3/3     Running     0
    23m
csi-rbdplugin-d267h                                               3/3     Running     0
    23m
csi-rbdplugin-provisioner-b46dd5c7-vd58q                          6/6     Running     0
    23m
csi-rbdplugin-provisioner-b46dd5c7-z8mx6                          6/6     Running     0
    23m
csi-rbdplugin-tdj8f                                               3/3     Running     0
    23m
csi-rbdplugin-wp65b                                               3/3     Running     0
    23m
noobaa-core-0                                                     1/1     Running     0
    19m
noobaa-db-0                                                       1/1     Running     0
    19m
noobaa-endpoint-86cc5df669-ffqj2                                  1/1     Running     0
    16m
noobaa-operator-698746cd47-sp6w9                                  1/1     Running     0
    17h
ocs-metrics-exporter-78bc44687-pg4hk                              1/1     Running     0
    17h
ocs-operator-6d99bc6787-d7m9d                                     1/1     Running     0
    17h
rook-ceph-crashcollector-ip-10-0-147-230-7cbf854757-chlgs         1/1     Running     0
    20m
rook-ceph-crashcollector-ip-10-0-175-8-5779d5d5df-p6hkl           1/1     Running     0
    21m
rook-ceph-crashcollector-ip-10-0-209-53-7ccc4cc785-wjxzd          1/1     Running     0
    21m
rook-ceph-drain-canary-128c383c26627b938ab0fd7f47f58d33-665pbsg   1/1     Running     0
    19m
rook-ceph-drain-canary-84c954eec459013180f78efd0a35792c-7b6qdnj   1/1     Running     0
    19m
rook-ceph-drain-canary-ip-10-0-175-8.us-east-2.compute.intrh526   1/1     Running     0
    19m
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-756df8b4kp9kr   1/1     Running     0
    18m
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-64585764bbg6b   1/1     Running     0
    18m
rook-ceph-mgr-a-5c74bb4b85-5x26g                                  1/1     Running     0
    20m
rook-ceph-mon-a-746b5457c-hlh7n                                   1/1     Running     0
    21m
rook-ceph-mon-b-754b99cfd-xs9g4                                   1/1     Running     0
    21m
rook-ceph-mon-c-7474d96f55-qhhb6                                  1/1     Running     0
    20m
rook-ceph-operator-59f7fb95d6-sdjd8                               1/1     Running     0
    17h
rook-ceph-osd-0-7d45696497-jwgb7                                  1/1     Running     0
    19m
rook-ceph-osd-1-6f49b665c7-gxq75                                  1/1     Running     0
    19m
rook-ceph-osd-2-76ffc64cd-9zg65                                   1/1     Running     0
    19m
rook-ceph-osd-prepare-ocs-deviceset-gp2-0-data-0-9977n-49ngd      0/1     Completed   0
    20m
rook-ceph-osd-prepare-ocs-deviceset-gp2-1-data-0-nnmpv-z8vq6      0/1     Completed   0
    20m
rook-ceph-osd-prepare-ocs-deviceset-gp2-2-data-0-mtbtj-xrj2n      0/1     Completed   0
    20m
```

The great thing about operators and OpenShift is that the operator has the intelligence about the deployed components built-in. And, because of the relationship between the `CustomResource` and the operator, you can check the status by looking at the `CustomResource` itself. When you went therough the UI dialogs, ultimately in the back-end an instance of a `StorageCluster` was created:

```
oc get storagecluster -n openshift-storage
```

You can check the status of the storage cluster with the following:

```
oc get storagecluster -n openshift-storage ocs-storagecluster -o jsonpath='{.status.phase}{"\n"}'
```

If it says `Ready`, you can continue.

### Getting to know the Storage Dashboards

You can now also check the status of your storage cluster with the OCS specific **Dashboards** that are included in your **Openshift Web Console**. You can reach this by clicking on `Overview` on your left navigation bar, then selecting `Persistent Storage` on the top navigation bar of the content page.

![Location of OCS Dashboards](https://preview.cloud.189.cn/image/imageAction?param=686789CA518A22E705EBC73CEA8AB1D72A2D7B7DF6B8B0FF4ABE847496E07A5B3D0937206EACFF60BA8848AB4E705D69DC647A136A8E34B3220C1A88DCCAEA933C01EE30221A69961EA8F3924432C7F3E3A76589118AD40598E33CD50BB8737AB78ABCE3AAF36FE0BAFD014763B12F34)

Figure 12. Location of OCS Dashboards

| NOTE | If you just finished your OCS 4 deployment it could take 5-10 minutes for your **Dashboards** to fully populate. Different versions of OCP 4 may have minor differences in **Dashboard** sections and naming of **Dashboards**. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

![Storage Dashboard after successful storage installation](https://preview.cloud.189.cn/image/imageAction?param=7E23E4A1ED7B09B65BED79AB21E716943F89C858D5794993FE045F7AAF58FEF67B5983AD764DDFCA53451982434505828C2A0DE17A6C64E98062930504C4921FA0B712D904D288F950F5209B417530BE104FB2294326D81DEADA64C6E3F967E0426F6DEA70E0E26C41F1C5AB33256A53)

Figure 13. Storage Dashboard after successful storage installation

| **1** | Health      | Quick overview of the general health of the storage cluster  |
| ----- | ----------- | ------------------------------------------------------------ |
| **2** | Details     | Overview of the deployed storage cluster version and backend provider |
| **3** | Inventory   | List of all the resources that are used and offered by the storage system |
| **4** | Events      | Live overview of all the changes that are being done affecting the storage cluster |
| **5** | Utilization | Overview of the storage cluster usage and performance        |

OCS ships with a **Dashboard** for the Object Store service as well. From the **Overview** click on the `Object Service` on the top navigation bar of the content page.

![OCS Multi-Cloud-Gateway Dashboard after successful installation](https://preview.cloud.189.cn/image/imageAction?param=8DA0BA779A740DA462107387E8C4B55DF3F7844D2846B36B56D983FF9A59906B16259C7C308E0C56AEA5EAC88437442BE1FB6A9241D62A5E86AEA1856865D55738356CA4B74CCAB0C95D4A39033622E05DE82FA86B885F94EB28128467918D32C21A60F75CCE14F129D1E3C504E19EDB)

Figure 14. OCS Multi-Cloud-Gateway Dashboard after successful installation

| **1** | Health             | Quick overview of the general health of the Multi-Cloud-Gateway |
| ----- | ------------------ | ------------------------------------------------------------ |
| **2** | Details            | Overview of the deployed MCG version and backend provider including a link to the MCG Console |
| **3** | Buckets            | List of all the ObjectBucket with are offered and ObjectBucketClaims which are connected to them |
| **4** | Resource Providers | Shows the list of configured Resource Providers that are available as backing storage in the MCG |
| **5** | Counters           | Shows the current numbers of reads and writes issued against each provider |
| **6** | Events             | Live overview of all the changes that are being done affecting the MCG |

Once this is all healthy, you will be able to use the three new **StorageClasses** created during the OCS 4 Install:

- ocs-storagecluster-ceph-rbd
- ocs-storagecluster-cephfs
- openshift-storage.noobaa.io

You can see these three **StorageClasses** from the Openshift Web Console by expanding the `Storage` menu in the left navigation bar and selecting `Storage Classes`. You can also run the command below:

```
oc -n openshift-storage get sc
```

Please make sure the three storage classes are available in your cluster before proceeding.

| NOTE | The NooBaa pod used the `ocs-storagecluster-ceph-rbd` storage class for creating a PVC for mounting to the `db` container. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Using the Rook-Ceph toolbox to check on the Ceph backing storage

Since the Rook-Ceph **toolbox** is not shipped with OCS, we need to deploy it manually.

You can patch the `OCSInitialization ocsinit` using the following command line:

```
oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
```

After the `rook-ceph-tools` **Pod** is `Running` you can access the **toolbox** like this:

```
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD
```

Once inside the **toolbox**, try out the following Ceph commands:

```
ceph status
ceph osd status
ceph osd tree
ceph df
rados df
ceph versions
```

Example output:

```
sh-4.2# ceph status
  cluster:
    id:     e3398039-f8c6-4937-ba9d-655f5c01e0ae
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 6h)
    mgr: a(active, since 6h)
    mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:standby-replay
    osd: 3 osds: 3 up (since 6h), 3 in (since 6h)

  task status:
    scrub status:
        mds.ocs-storagecluster-cephfilesystem-a: idle
        mds.ocs-storagecluster-cephfilesystem-b: idle

  data:
    pools:   3 pools, 96 pgs
    objects: 120 objects, 245 MiB
    usage:   3.5 GiB used, 6.0 TiB / 6 TiB avail
    pgs:     96 active+clean

  io:
    client:   853 B/s rd, 16 KiB/s wr, 1 op/s rd, 1 op/s wr
```

You can exit the toolbox by either pressing Ctrl+D or by executing exit.

```
exit
```

## Create a new OCP application deployment using Ceph RBD volume

In this section the `ocs-storagecluster-ceph-rbd` **StorageClass** will be used by an OCP application + database **Deployment** to create RWO (ReadWriteOnce) persistent storage. The persistent storage will be a Ceph RBD (RADOS Block Device) volume in the Ceph pool `ocs-storagecluster-cephblockpool`.

To do so we have created a template file, based on the OpenShift rails-pgsql-persistent template, that includes an extra parameter STORAGE_CLASS that enables the end user to specify the **StorageClass** the PVC should use. Feel free to download `https://github.com/red-hat-storage/ocs-training/blob/master/training/modules/ocs4/attachments/configurable-rails-app.yaml` to check on the format of this template. Search for `STORAGE_CLASS` in the downloaded content.

Make sure that you completed all previous sections so that you are ready to start the Rails + PostgreSQL **Deployment**.

Start by creating a new project:

```
oc new-project my-database-app
```

Then use the `rails-pgsql-persistent` template to create the new application.

```
oc new-app -f /opt/app-root/src/support/ocslab_rails-app.yaml -p STORAGE_CLASS=ocs-storagecluster-ceph-rbd -p VOLUME_CAPACITY=5Gi
```

After the deployment is started you can monitor with these commands.

```
oc status
```

Check the PVC is created.

```
oc get pvc -n my-database-app
```

This step could take 5 or more minutes. Wait until there are 2 **Pods** in `Running` STATUS and 4 **Pods** in `Completed` STATUS as shown below.

```
watch oc get pods -n my-database-app
```

Example output:

```
NAME                                READY   STATUS      RESTARTS   AGE
postgresql-1-deploy                 0/1     Completed   0          5m48s
postgresql-1-lf7qt                  1/1     Running     0          5m40s
rails-pgsql-persistent-1-build      0/1     Completed   0          5m49s
rails-pgsql-persistent-1-deploy     0/1     Completed   0          3m36s
rails-pgsql-persistent-1-hook-pre   0/1     Completed   0          3m28s
rails-pgsql-persistent-1-pjh6q      1/1     Running     0          3m14s
```

You can exit by pressing Ctrl+C.

Once the deployment is complete you can now test the application and the persistent storage on Ceph.

```
oc get route rails-pgsql-persistent -n my-database-app -o jsonpath --template="http://{.spec.host}/articles{'\n'}"
```

This will return a route similar to this one:

Example output:

```
http://rails-pgsql-persistent-my-database-app.apps.cluster-ocs4-8613.ocs4-8613.sandbox944.opentlc.com/articles
```

Copy your route (different than above) to a browser window to create articles.

Enter the `username` and `password` below to create articles and comments. The articles and comments are saved in a PostgreSQL database which stores its table spaces on the Ceph RBD volume provisioned using the `ocs-storagecluster-ceph-rbd` **StorageClass** during the application deployment.

```
username: openshift
password: secret
```

Lets now take another look at the Ceph `ocs-storagecluster-cephblockpool` created by the `ocs-storagecluster-ceph-rbd` **StorageClass**. Log into the **toolbox** pod again.

```
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD
```

Run the same Ceph commands as before the application deployment and compare to results in prior section. Notice the number of objects in `ocs-storagecluster-cephblockpool` has increased. The third command lists RBD volumes and we should now have two RBDs.

```
ceph df
rados df
rbd -p ocs-storagecluster-cephblockpool ls | grep vol
```

You can exit the toolbox by either pressing Ctrl+D or by executing exit.

```
exit
```

### Matching PVs to RBDs

A handy way to match OCP persistent volumes (**PVs**)to Ceph RBDs is to execute:

```
oc get pv -o 'custom-columns=NAME:.spec.claimRef.name,PVNAME:.metadata.name,STORAGECLASS:.spec.storageClassName,VOLUMEHANDLE:.spec.csi.volumeHandle'
```

Example output:

```
NAME                      PVNAME                                     STORAGECLASS                  VOLUMEHANDLE
ocs-deviceset-0-0-d2ppm   pvc-2c08bd9c-332d-11ea-a32f-061f7a67362c   gp2                           <none>
ocs-deviceset-1-0-9tmc6   pvc-2c0a0ed5-332d-11ea-a32f-061f7a67362c   gp2                           <none>
ocs-deviceset-2-0-qtbfv   pvc-2c0babb3-332d-11ea-a32f-061f7a67362c   gp2                           <none>
db-noobaa-core-0          pvc-4610a3ce-332d-11ea-a32f-061f7a67362c   ocs-storagecluster-ceph-rbd   0001-0011-openshift-storage-0000000000000001-4a74e248-332d-11ea-9a7c-0a580a820205
postgresql                pvc-874f93cb-3330-11ea-90b1-0a10d22e734a   ocs-storagecluster-ceph-rbd   0001-0011-openshift-storage-0000000000000001-8765a21d-3330-11ea-9a7c-0a580a820205
rook-ceph-mon-a           pvc-d462ecb0-332c-11ea-a32f-061f7a67362c   gp2                           <none>
rook-ceph-mon-b           pvc-d79d0db4-332c-11ea-a32f-061f7a67362c   gp2                           <none>
rook-ceph-mon-c           pvc-da9cc0e3-332c-11ea-a32f-061f7a67362c   gp2                           <none>
```

The second half of the `VOLUMEHANDLE` column mostly matches what your RBD is named inside of Ceph. All you have to do is append `csi-vol-` to the front like this:

Get the full RBD name and the associated information for your postgreSQL **PV**

```
CSIVOL=$(oc get pv $(oc get pv | grep my-database-app | awk '{ print $1 }') -o jsonpath='{.spec.csi.volumeHandle}' | cut -d '-' -f 6- | awk '{print "csi-vol-"$1}')
echo $CSIVOL
```

Examplet output:

```
csi-vol-8765a21d-3330-11ea-9a7c-0a580a820205
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD rbd -p ocs-storagecluster-cephblockpool info $CSIVOL
```

Example output:

```
rbd image 'csi-vol-8765a21d-3330-11ea-9a7c-0a580a820205':
        size 5 GiB in 1280 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 17e811c7f287
        block_name_prefix: rbd_data.17e811c7f287
        format: 2
        features: layering
        op_features:
        flags:
        create_timestamp: Thu Jan  9 22:36:51 2020
        access_timestamp: Thu Jan  9 22:36:51 2020
        modify_timestamp: Thu Jan  9 22:36:51 2020
```

### Expand RBD based PVCs

OpenShift 4.5 and later versions let you expand an existing PVC based on the `ocs-storagecluster-ceph-rbd` **StorageClass**. This section walks you through the steps to perform a PVC expansion.

We will first artificially fill up the PVC used by the application you have just created.

```
oc rsh -n my-database-app $(oc get pods -n my-database-app|grep postgresql | grep -v deploy | awk {'print $1}')
df
```

Example output:

```
Filesystem                           1K-blocks     Used Available Use% Mounted on
overlay                              125277164 12004092 113273072  10% /
tmpfs                                    65536        0     65536   0% /dev
tmpfs                                 32571336        0  32571336   0% /sys/fs/cgroup
shm                                      65536        8     65528   1% /dev/shm
tmpfs                                 32571336    10444  32560892   1% /etc/passwd
/dev/mapper/coreos-luks-root-nocrypt 125277164 12004092 113273072  10% /etc/hosts
/dev/rbd1                              5095040    66968   5011688   2% /var/lib/pgsql/data
tmpfs                                 32571336       28  32571308   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                 32571336        0  32571336   0% /proc/acpi
tmpfs                                 32571336        0  32571336   0% /proc/scsi
tmpfs                                 32571336        0  32571336   0% /sys/firmware
```

As observed in the output above the device named `/dev/rbd1` is mounted as `/var/lib/pgsql/data`. This is the directory we will artificially fill up.

```
dd if=/dev/zero of=/var/lib/pgsql/data/fill.up bs=1M count=3850
```

Example output:

```
3850+0 records in
3850+0 records out
4037017600 bytes (4.0 GB) copied, 13.6446 s, 296 MB/s
```

Let’s verify the volume mounted has increased.

```
df
```

Example output:

```
Filesystem                           1K-blocks     Used Available Use% Mounted on
overlay                              125277164 12028616 113248548  10% /
tmpfs                                    65536        0     65536   0% /dev
tmpfs                                 32571336        0  32571336   0% /sys/fs/cgroup
shm                                      65536        8     65528   1% /dev/shm
tmpfs                                 32571336    10444  32560892   1% /etc/passwd
/dev/mapper/coreos-luks-root-nocrypt 125277164 12028616 113248548  10% /etc/hosts
/dev/rbd1                              5095040  4009372   1069284  79% /var/lib/pgsql/data
tmpfs                                 32571336       28  32571308   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                 32571336        0  32571336   0% /proc/acpi
tmpfs                                 32571336        0  32571336   0% /proc/scsi
tmpfs                                 32571336        0  32571336   0% /sys/firmware
```

As observed in the output above, the filesystem usage for `/var/lib/pgsql/data` has increased up to 79%. By default OCP will generate a PVC alert when a PVC crosses the 75% full threshold.

Now exit the pod.

```
exit
```

Let’s verify an alert has appeared in the OCP event log.

![PVC nearfull alert](https://preview.cloud.189.cn/image/imageAction?param=B68B9F1C09DBB4377BDF30F4D09A4F7C057180306845C725CDD76ECACA0810C33821FA5A0E6394C19CB2EADC22E8A5BE3139DC81BB9E283AE74409DE8B81F7A4C760B949FB88814A8982C67CD9B084DF3DD1BFD5E6124147D1112A1808127D7008ABB0CF6B703F92923BD5441F6DE208)

Figure 15. OpenShift Container Platform Events

#### Expand applying a modified PVC YAML file

To expand a **PVC** we simply need to change the actual amount of storage that is requested. This can easily be performed by exporting the **PVC** specifications into a YAML file with the following command:

```
oc get pvc postgresql -n my-database-app -o yaml > pvc.yaml
```

In the file `pvc.yaml` that was created, search for the following section using your favorite editor.

Example output:

```yaml
[truncated]
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
  volumeName: pvc-4d6838df-b4cd-4bb1-9969-1af93c1dc5e6
status: {}
```

Edit `storage: 5Gi` and replace it with `storage: 10Gi`. The resulting section in your file should look like the output below.

Example output:

```yaml
[truncated]
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ocs-storagecluster-ceph-rbd
  volumeMode: Filesystem
  volumeName: pvc-4d6838df-b4cd-4bb1-9969-1af93c1dc5e6
status: {}
```

Now you can apply your updated PVC specifications using the following command:

```
oc apply -f pvc.yaml -n my-database-app
```

Example output:

```
Warning: oc apply should be used on resource created by either oc create
--save-config or oc apply persistentvolumeclaim/postgresql configured
```

You can visualize the progress of the expansion of the PVC using the following command:

```
oc describe pvc postgresql -n my-database-app
```

Example output:

```
[truncated]
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      10Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    postgresql-1-p62vw
Events:
  Type     Reason                      Age   From                                                                                                                Message
  ----     ------                      ----  ----                                                                                                                -------
  Normal   ExternalProvisioning        120m  persistentvolume-controller                                                                                         waiting for a volume to be created, either by external provisioner "openshift-storage.rbd.csi.ceph.com" or manually created by system administrator
  Normal   Provisioning                120m  openshift-storage.rbd.csi.ceph.com_csi-rbdplugin-provisioner-66f66699c8-gcm7t_3ce4b8bc-0894-4824-b23e-ed9bd46e7b41  External provisioner is provisioning volume for claim "my-database-app/postgresql"
  Normal   ProvisioningSucceeded       120m  openshift-storage.rbd.csi.ceph.com_csi-rbdplugin-provisioner-66f66699c8-gcm7t_3ce4b8bc-0894-4824-b23e-ed9bd46e7b41  Successfully provisioned volume pvc-4d6838df-b4cd-4bb1-9969-1af93c1dc5e6
  Warning  ExternalExpanding           65s   volume_expand                                                                                                       Ignoring the PVC: didn't find a plugin capable of expanding the volume; waiting for an external controller to process this PVC.
  Normal   Resizing                    65s   external-resizer openshift-storage.rbd.csi.ceph.com                                                                 External resizer is resizing volume pvc-4d6838df-b4cd-4bb1-9969-1af93c1dc5e6
  Normal   FileSystemResizeRequired    65s   external-resizer openshift-storage.rbd.csi.ceph.com                                                                 Require file system resize of volume on node
  Normal   FileSystemResizeSuccessful  23s   kubelet, ip-10-0-199-224.us-east-2.compute.internal                                                                 MountVolume.NodeExpandVolume succeeded for volume "pvc-4d6838df-b4cd-4bb1-9969-1af93c1dc5e6"
```

| NOTE | The expansion process commonly takes over 30 seconds to complete and is based on the workload of your pod. This is due to the fact that the expansion requires the resizing of the underlying RBD image (pretty fast) while also requiring the resize of the filesystem that sits on top of the block device. To perform the latter the filesystem must be quiesced to be safely expanded. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

| CAUTION | Reducing the size of a **PVC** is NOT supported. |
| ------- | ------------------------------------------------ |
|         |                                                  |

Another way to check on the expansion of the **PVC** is to simply display the **PVC** information using the following command:

```
oc get pvc -n my-database-app
```

Example output:

```
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
postgresql   Bound    pvc-4d6838df-b4cd-4bb1-9969-1af93c1dc5e6   10Gi       RWO            ocs-storagecluster-ceph-rbd   121m
```

| NOTE | The `CAPACITY` column will reflect the new requested size when the expansion process is complete. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Another method to check on the expansion of the **PVC** is to go through two specific fields of the PVC object via the CLI.

The current allocated size for the **PVC** can be checked this way:

```
echo $(oc get pvc postgresql -n my-database-app -o jsonpath='{.status.capacity.storage}')
```

Example output:

```
10Gi
```

The requested size for the **PVC** can be checked this way:

```
echo $(oc get pvc postgresql -n my-database-app -o jsonpath='{.spec.resources.requests.storage}')
```

Example output:

```
10Gi
```

| NOTE | When both results report the same value, the expansion was successful. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

#### Expand via the User Interface

The last method available to expand a **PVC** is to do so through the **OpenShift Web Console**. Proceed as follow:

First step is to select the project to which the **PVC** belongs to.

![Select project](https://preview.cloud.189.cn/image/imageAction?param=B9947E739C6E506030F4F4CE1CA09B07C0CDEB6509B9257B0236CAD74FD2AE6EAC258784F2ACF9FA966FF1DE0F1EEBADB746DF443EBF97F67983FBD204031A6889AF8A36F81AEAF4AD785A2D348EBC3B58EDE27CE240A2CFA87B37569E49C6C68C14E8F156BC1A4202C37B119DCF60E4)

Figure 16. Select the appropriate project

Choose `Expand PVC` from the contextual menu.

![Choose expand from the contextual menu](https://preview.cloud.189.cn/image/imageAction?param=A1C28E37EDF63673163AF46DB1A1BF83AB8646F44B915AEC975E9A47A6EE5B2099BAD578DB369BE180BB3CD00D574DC82BDAF3A59CEC6466DF46157A76D94318AEA25F201C39D2E3DE8FAF5A540E713BAAD7F772C374AD2A38FF8068A94687F3C61237017E448D4D95F718B8CF3769DD)

Figure 17. Choose Expand from menu

In the dialog box that appears enter the new capacity for the **PVC**.

| CAUTION | You can NOT reduce the size of a **PVC**. |
| ------- | ----------------------------------------- |
|         |                                           |

![Enter new size](https://preview.cloud.189.cn/image/imageAction?param=CA3B84CCBAE515F4FC9865E881DE71BE79F2A036E1F4EF1F34D02E52997B01450D1422518ED48971B79E29E09BFA1BA542C7A5A9D69C87A0A0C4F843E87302701D1077C7636D7995CA38CA64D0AA84FF86110DDA80AC9F0739FAC6CBD3A2DA1C7C79CCCCE15ADFC8A8631D489129E213)

Figure 18. Enter the new size for the **PVC**

You now simply have to wait for the expansion to complete and for the new size to be reflected in the console (15 GiB).

![Wait for expansion](https://preview.cloud.189.cn/image/imageAction?param=4ED69CCFC62B08E08CCAB5DEA64B3D245F10D29226BE21743EE489DCE2D381A80D5231BEF35B2023E7AF0F3925EB7756C38BD71DCC0026FDCFB780E62C8D1847C8F9B2C51A42E1020E224623348D69E0E2D4A3AEB503C5FCB47DB0D62A9473B371DFE53AD56074CB01A5EA42BB626ADF)

Figure 19. Wait for the expansion to complete

## Create a new OCP application deployment using CephFS volume

In this section the `ocs-storagecluster-cephfs` **StorageClass** will be used to create a RWX (ReadWriteMany) **PVC** that can be used by multiple pods at the same time. The application we will use is called `File Uploader`.

Create a new project:

```
oc new-project my-shared-storage
```

Next deploy the example PHP application called `file-uploader`:

```
oc new-app openshift/php:7.2-ubi8~https://github.com/christianh814/openshift-php-upload-demo --name=file-uploader
```

Example Output:

```
--> Found image 4f2dcc0 (9 days old) in image stream "openshift/php" under tag "7.2-ubi8" for "openshift/php:7.2-
ubi8"

    Apache 2.4 with PHP 7.2
    -----------------------
    PHP 7.2 available as container is a base platform for building and running various PHP 7.2 applications and f
rameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynam
ically generated web pages. PHP also offers built-in database integration for several commercial and non-commerci
al database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common
use of PHP coding is probably as a replacement for CGI scripts.

    Tags: builder, php, php72, php-72

    * A source build using source code from https://github.com/christianh814/openshift-php-upload-demo will be cr
eated
      * The resulting image will be pushed to image stream tag "file-uploader:latest"
      * Use 'oc start-build' to trigger a new build

--> Creating resources ...
    imagestream.image.openshift.io "file-uploader" created
    buildconfig.build.openshift.io "file-uploader" created
    deployment.apps "file-uploader" created
    service "file-uploader" created
--> Success
    Build scheduled, use 'oc logs -f buildconfig/file-uploader' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the comm
ands below:
     'oc expose service/file-uploader'
    Run 'oc status' to view your app.
```

Watch the build log and wait for the application to be deployed:

```
oc logs -f bc/file-uploader -n my-shared-storage
```

Sample Output:

```
Cloning "https://github.com/christianh814/openshift-php-upload-demo" ...

[...]

Generating dockerfile with builder image image-registry.openshift-image-regis
try.svc:5000/openshift/php@sha256:d97466f33999951739a76bce922ab17088885db610c
0e05b593844b41d5494ea
STEP 1: FROM image-registry.openshift-image-registry.svc:5000/openshift/php@s
ha256:d97466f33999951739a76bce922ab17088885db610c0e05b593844b41d5494ea
STEP 2: LABEL "io.openshift.build.commit.author"="Christian Hernandez <christ
ian.hernandez@yahoo.com>"       "io.openshift.build.commit.date"="Sun Oct 1 1
7:15:09 2017 -0700"       "io.openshift.build.commit.id"="288eda3dff43b02f7f7
b6b6b6f93396ffdf34cb2"       "io.openshift.build.commit.ref"="master"       "
io.openshift.build.commit.message"="trying to modularize"       "io.openshift
.build.source-location"="https://github.com/christianh814/openshift-php-uploa
d-demo"       "io.openshift.build.image"="image-registry.openshift-image-regi
stry.svc:5000/openshift/php@sha256:d97466f33999951739a76bce922ab17088885db610
c0e05b593844b41d5494ea"
STEP 3: ENV OPENSHIFT_BUILD_NAME="file-uploader-1"     OPENSHIFT_BUILD_NAMESP
ACE="my-shared-storage"     OPENSHIFT_BUILD_SOURCE="https://github.com/christ
ianh814/openshift-php-upload-demo"     OPENSHIFT_BUILD_COMMIT="288eda3dff43b0
2f7f7b6b6b6f93396ffdf34cb2"
STEP 4: USER root
STEP 5: COPY upload/src /tmp/src
STEP 6: RUN chown -R 1001:0 /tmp/src
STEP 7: USER 1001
STEP 8: RUN /usr/libexec/s2i/assemble
---> Installing application source...
=> sourcing 20-copy-config.sh ...
---> 17:24:39     Processing additional arbitrary httpd configuration provide
d by s2i ...
=> sourcing 00-documentroot.conf ...
=> sourcing 50-mpm-tuning.conf ...
=> sourcing 40-ssl-certs.sh ...
STEP 9: CMD /usr/libexec/s2i/run
STEP 10: COMMIT temp.builder.openshift.io/my-shared-storage/file-uploader-1:3
b83e447
Getting image source signatures

[...]

Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/my-share
d-storage/file-uploader@sha256:929c0ce3dcc65a6f6e8bd44069862858db651358b88065
fb483d51f5d704e501
Push successful
```

The command prompt returns out of the tail mode once you see *Push successful*.

| NOTE | This use of the `new-app` command directly asked for application code to be built and did not involve a template. That is why it only created a **single Pod** deployment with a **Service** and no **Route**. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Let’s make our application production ready by exposing it via a `Route` and scale to 3 instances for high availability:

```
oc expose svc/file-uploader -n my-shared-storage
oc scale --replicas=3 deploy/file-uploader -n my-shared-storage
oc get pods -n my-shared-storage
```

You should have 3 `file-uploader` **Pods** in a few minutes. Repeat the command above until there are 3 `file-uploader` **Pods** in `Running` STATUS.

| CAUTION | Never attempt to store persistent data in a **Pod** that has no persistent volume associated with it. **Pods** and their containers are ephemeral by definition, and any stored data will be lost as soon as the **Pod** terminates for whatever reason. |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

We can fix this by providing shared persistent storage to this application.

You can create a **PersistentVolumeClaim** and attach it into an application with the `oc set volume` command. Execute the following

```
oc set volume deploy/file-uploader --add --name=my-shared-storage \
-t pvc --claim-mode=ReadWriteMany --claim-size=1Gi \
--claim-name=my-shared-storage --claim-class=ocs-storagecluster-cephfs \
--mount-path=/opt/app-root/src/uploaded \
-n my-shared-storage
```

This command will:

- create a **PersistentVolumeClaim**
- update the **Deployment** to include a `volume` definition
- update the **Deployment** to attach a `volumemount` into the specified `mount-path`
- cause a new deployment of the 3 application **Pods**

For more information on what `oc set volume` is capable of, look at its help output with `oc set volume -h`. Now, let’s look at the result of adding the volume:

```
oc get pvc -n my-shared-storage
```

Sample Output:

```
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
my-shared-storage   Bound    pvc-c34bb9db-43a7-4eca-bc94-0251d7128721   1Gi        RWX            ocs-storagecluster-cephfs   47s
```

Notice the `ACCESSMODE` being set to **RWX** (short for `ReadWriteMany`).

All 3 `file-uploader`**Pods** are using the same **RWX** volume. Without this `ACCESSMODE`, OpenShift will not attempt to attach multiple **Pods** to the same **PersistentVolume** reliably. If you attempt to scale up deployments that are using **RWO** or `ReadWriteOnce` storage, the **Pods** will actually all become co-located on the same node.

Now let’s use the file uploader web application using your browser to upload new files.

First, find the **Route** that has been created:

```
oc get route file-uploader -n my-shared-storage -o jsonpath --template="http://{.spec.host}{'\n'}"
```

This will return a route similar to this one:

Example Output:

```
http://file-uploader-my-shared-storage.apps.cluster-ocs4-abdf.ocs4-abdf.sandbox744.opentlc.com
```

Point your browser to the web application using your route above. **Your `route` will be different.**

The web app simply lists all uploaded files and offers the ability to upload new ones as well as download the existing data. Right now there is nothing.

Select an arbitrary file from your local machine and upload it to the app.

![uploader screen upload](https://preview.cloud.189.cn/image/imageAction?param=EACA0B0C3721FF0DCA0C61BBB9AE1D5855EE7F42FED3FB8CEF39CC528BD6E2B1790385BB6DB4C065CF4080A4F2822BA3CB42E21101051D50C4C40E3AC055A361F5A63E360F7B4689387259745B8B5FF365A1C059B8D642337C6ABD82005E38DDF9E03EC509F3344692CD9571DCD8753A)

Figure 20. A simple PHP-based file upload tool

Once done click ***List uploaded files\*** to see the list of all currently uploaded files.

### Expand CephFS based PVCs

OpenShift 4.5 and later versions let you expand an existing **PVC** based on the `ocs-storagecluster-cephfs` **StorageClass**. This chapter walks you through the steps to perform a PVC expansion through the CLI.

| NOTE | All the other methods described for expanding a Ceph RBD based **PVC** are also available. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The `my-sharged-storage` **PVC** size is currently `1Gi`. Let’s increase the size to `5Gi` using the **oc patch** command.

```
oc patch pvc my-shared-storage -n my-shared-storage --type json --patch  '[{ "op": "replace", "path": "/spec/resources/requests/storage", "value": "5Gi" }]'
```

Example output:

```
persistentvolumeclaim/my-shared-storage patched
```

Now let’s verify the RWX **PVC** has been expanded.

```
echo $(oc get pvc my-shared-storage -n my-shared-storage -o jsonpath='{.spec.resources.requests.storage}')
```

Example output:

```
5Gi
echo $(oc get pvc my-shared-storage -n my-shared-storage -o jsonpath='{.status.capacity.storage}')
```

Example output:

```
5Gi
```

Repeat both commands until output values are identical.

| NOTE | CephFS based RWX **PVC** resizing, as opposed to RBD based **PVCs**, is almost instantaneous. This is due to the fact that resizing such PVC does not involved resizing a filesystem but simply involves updating a quota for the mounted filesystem. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

| CAUTION | Reducing the size of a CephFS **PVC** is NOT supported. |
| ------- | ------------------------------------------------------- |
|         |                                                         |

## PVC Clone and Snapshot

Starting with version OCS version 4.6, the `Container Storage Interface` (CSI) features of being able to clone or snapshot a persistent volume are now supported. These new capabilities are very important for protecting persistent data and can be used with third party `Backup and Restore` vendors that have CSI integration.

In addition to third party backup and restore vendors, OCS snapshot for Ceph RBD and CephFS PVCs can be triggered using `OpenShift APIs for Data Protection` (OADP) which is an un-supported community operator in **OperatorHub** that can be very useful for testing backup and restore of persistent data including OpenShift metadata (definition files for pods, service, routes, deployments, etc.).

### PVC Clone

A CSI volume clone is a duplicate of an existing persistent volume at a particular point in time. Cloning creates an exact duplicate of the specified volume in OCS. After dynamic provisioning, you can use a volume clone just as you would use any standard volume.

#### Provisioning a CSI Volume clone

For this exercise we will use the already created **PVC** `postgresql` that was just expanded to 15 GiB. Make sure you have done section [Create a new OCP application deployment using Ceph RBD volume](https://dashboard-lab-ocp-cns.apps.odflab-55.container-contest.top/workshop/ocs4#_create_a_new_ocp_application_deployment_using_ceph_rbd_volume) before proceeding.

```
oc get pvc -n my-database-app | awk {'print $1}'
```

Example output:

```
NAME
postgresql
```

| CAUTION | Make sure you expanded the `postgresql` **PVC** to 15Gi before proceeding. If not expanded go back and complete this section [Expand RBD based PVCs](https://dashboard-lab-ocp-cns.apps.odflab-55.container-contest.top/workshop/ocs4#_expand_rbd_based_pvcs). |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

Before creating the PVC clone make sure to create and save at least one new article so there is new data in the `postgresql` **PVC**.

```
oc get route rails-pgsql-persistent -n my-database-app -o jsonpath --template="http://{.spec.host}/articles{'\n'}"
```

This will return a route similar to this one.

Example output:

```
http://rails-pgsql-persistent-my-database-app.apps.cluster-ocs4-8613.ocs4-8613.sandbox944.opentlc.com/articles
```

Copy your route (different than above) to a browser window to create articles.

Enter the `username` and `password` below to create a new article.

```
username: openshift
password: secret
```

To protect the data (articles) in this **PVC** we will now clone this PVC. The operation of creating a clone can be done using the **OpenShift Web Console** or by creating the resource via a YAML file.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-clone
  namespace: my-database-app
spec:
  storageClassName: ocs-storagecluster-ceph-rbd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 15Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: postgresql
```

Doing the same operation in the **OpenShift Web Console** would require navigating to `Storage` → `Persistent Volume Claim` and choosing `Clone PVC`.

![Persistent Volume Claim clone PVC using UI](https://preview.cloud.189.cn/image/imageAction?param=AED02EE52BC1CEA974C7E67EC318FC946CAA314FACE9820525425A0E8CC3280D87A747D0325036AB92F6E254766CF48D872FE02B03E77F6E79600ACC17C115FDDC774EF6955F1BE130A32EFB779F51C73941E96D254BF947030985724579779CE07DC99227F924185C1CD32B29F6BE75)

Figure 21. Persistent Volume Claim clone PVC using UI

Size of new clone **PVC** is greyed out. The new **PVC** will be the same size as the original.

![Persistent Volume Claim clone configuration](https://preview.cloud.189.cn/image/imageAction?param=FB98DF2E2D287A48BD199B558A3798058845DACBF5B53855FD422FF18EC281ED32321E699B18DDDAAD629F073BE3886ADE195EDC7589E4A9D772B30D98BE2806712EFDA8B99A7DF906FEE7C81BB3989D5C71C24848389C5C1020808DA1A61D98ACAB17E70844563E95F061A25A611506)

Figure 22. Persistent Volume Claim clone configuration

Now create a **PVC** clone for `postgresql`.

```
oc apply -f /opt/app-root/src/support/postgresql-clone.yaml
```

Example output:

```
persistentvolumeclaim/postgresql-clone created
```

Now check to see there is a new **PVC**.

```
oc get pvc -n my-database-app | grep clone
```

Example output:

```
postgresql-clone   Bound    pvc-f5e09c63-e8aa-48a0-99df-741280d35e42   15Gi       RWO            ocs-storagecluster-ceph-rbd   3m47s
```

You can also check the new clone **PVC** in the **OpenShift Web Console**.

![Persistent Volume Claim clone view in UI](https://preview.cloud.189.cn/image/imageAction?param=76E56FD9368B9446EC0C2CDE10B05E159816F9C8B741ECA273623D5945A2DA85B5716F42F4B681378C58E2C0AF59BABF0E41E472EC7278240F980C75E286F92CEECCB63D4FEDFFF54F8E0A813D96B45F02204BF352F29998EB52AB051D4B29ED4D17BB9CB276698C10B32C5EAE790D9B)

Figure 23. Persistent Volume Claim clone view in UI

#### Using a CSI Volume clone for application recovery

Now that you have a clone for `postgresql` **PVC** you are ready to test by corrupting the database.

The following command will print all `postgresql` tables before deleting the article tables in the database and after the tables are deleted.

```
oc rsh -n my-database-app $(oc get pods -n my-database-app|grep postgresql | grep -v deploy | awk {'print $1}') psql -c "\c root" -c "\d+" -c "drop table articles cascade;" -c "\d+"
```

Example output:

```
You are now connected to database "root" as user "postgres".
                               List of relations
 Schema |         Name         |   Type   |  Owner  |    Size    | Description
--------+----------------------+----------+---------+------------+-------------
 public | ar_internal_metadata | table    | userXNL | 16 kB      |
 public | articles             | table    | userXNL | 16 kB      |
 public | articles_id_seq      | sequence | userXNL | 8192 bytes |
 public | comments             | table    | userXNL | 8192 bytes |
 public | comments_id_seq      | sequence | userXNL | 8192 bytes |
 public | schema_migrations    | table    | userXNL | 16 kB      |
(6 rows)

NOTICE:  drop cascades to constraint fk_rails_3bf61a60d3 on table comments
DROP TABLE
                               List of relations
 Schema |         Name         |   Type   |  Owner  |    Size    | Description
--------+----------------------+----------+---------+------------+-------------
 public | ar_internal_metadata | table    | userXNL | 16 kB      |
 public | comments             | table    | userXNL | 8192 bytes |
 public | comments_id_seq      | sequence | userXNL | 8192 bytes |
 public | schema_migrations    | table    | userXNL | 16 kB      |
(4 rows)
```

Now go back to the browser tab where you created your article using this link:

```
oc get route rails-pgsql-persistent -n my-database-app -o jsonpath --template="http://{.spec.host}/articles{'\n'}"
```

If you refresh the browser you will see the application has failed.

![Application failed because database table removed](https://preview.cloud.189.cn/image/imageAction?param=26D54F80B23AC3B215CC0D5AD1F6256AE3BE597874E55AC5042818D418FE94065194268F18ED7401F6528C65E26DF155D814F2897696AE3DA145EBE29A7C17D7801E14DD9B4E48389924FBA89F9A844E637799023421C5AB0CF67E091102F52B6D2516DD747D395F47243E9A60A389E8)

Figure 24. Application failed because database table removed

Remember a **PVC** clone is an exact duplica of the original **PVC** at the time the clone was created. Therefore you can use you `postgresql` clone to recover the application.

First you need to scale the `rails-pgsql-persistent` deployment down to zero so the **Pod** will be deleted.

```
oc scale deploymentconfig rails-pgsql-persistent -n my-database-app --replicas=0
```

Example output:

```
deploymentconfig.apps.openshift.io/rails-pgsql-persistent scaled
```

Verify the **Pod** is gone.

```
oc get pods -n my-database-app | grep rails | egrep -v 'deploy|build|hook' | awk {'print $1}'
```

Wait until there is no result for this command. Repeat if necessary.

Now you need to patch the deployment for `postgesql` and modify to use the `postgresql-clone` **PVC**. This can be done using the `oc patch` command.

```
oc patch dc postgresql -n my-database-app --type json --patch  '[{ "op": "replace", "path": "/spec/template/spec/volumes/0/persistentVolumeClaim/claimName", "value": "postgresql-clone" }]'
```

Example output:

```
deploymentconfig.apps.openshift.io/postgresql patched
```

After modifying the deployment with the clone **PVC** the `rails-pgsql-persistent` deployment needs to be scaled back up.

```
oc scale deploymentconfig rails-pgsql-persistent -n my-database-app --replicas=1
```

Example output:

```
deploymentconfig.apps.openshift.io/rails-pgsql-persistent scaled
```

Now check to see that there is a new `postgresql` and `rails-pgsql-persistent` **Pod**.

```
oc get pods -n my-database-app | egrep 'rails|postgresql' | egrep -v 'deploy|build|hook'
```

Example output:

```
postgresql-4-hv5kb                  1/1     Running     0          5m58s
rails-pgsql-persistent-1-dhwhz      1/1     Running     0          5m10s
```

Go back to the browser tab where you created your article using this link:

```
oc get route rails-pgsql-persistent -n my-database-app -o jsonpath --template="http://{.spec.host}/articles{'\n'}"
```

If you refresh the browser you will see the application is back online and you have your articles. You can even add more articles now.

This process shows the pratical reasons to create a **PVC** clone if you are testing an application where data corruption is a possibility and you want a known good copy or `clone`.

Let’s next look at a similar feature, creating a **PVC** snapshot.

### PVC Snapshot

Creating the first snapshot of a PVC is the same as creating a clone from that PVC. However, after an initial PVC snapshot is created, subsequent snapshots only save the delta between the initial snapshot the current contents of the PVC. Snapshots are frequently used by backup utilities which schedule incremental backups on a periodic basis (e.g. hourly). Snapshots are more capacity efficient than creating full clones each time period (e.g. hourly), as only the deltas to the PVC are stored in each snapshot.

A snapshot can be used to provision a new volume by creating a **PVC** clone. The volume clone can be used for application recovery as demonstrated in the previous section.

#### VolumeSnapshotClass

To create a volume snapshot there first must be **VolumeSnapshotClass** resources that will be referenced in the **VolumeSnapshot** definition. The deployment of OCS (must be version 4.6 or greater) creates two **VolumeSnapshotClass** resources for creating snapshots.

```
oc get volumesnapshotclasses
```

Example output:

```
$ oc get volumesnapshotclasses
NAME                                        DRIVER                                  DELETIONPOLICY   AGE
ocs-storagecluster-cephfsplugin-snapclass   openshift-storage.cephfs.csi.ceph.com   Delete           4d23h
ocs-storagecluster-rbdplugin-snapclass      openshift-storage.rbd.csi.ceph.com      Delete           4d23h
```

You can see by the naming of the **VolumeSnapshotClass** that one is for creating CephFS volume snapshots and the other is for Ceph RBD.

#### Provisioning a CSI Volume snapshot

For this exercise we will use the already created **PVC** `my-shared-storage`. Make sure you have done section [Create a new OCP application deployment using CephFS volume](https://dashboard-lab-ocp-cns.apps.odflab-55.container-contest.top/workshop/ocs4#_create_a_new_ocp_application_deployment_using_cephfs_volume) before proceeding.

The operation of creating a snapshot can be done using the **OpenShift Web Console** or by creating the resource via a YAML file.

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: my-shared-storage-snapshot
  namespace: my-shared-storage
spec:
  volumeSnapshotClassName: ocs-storagecluster-cephfsplugin-snapclass
  source:
    persistentVolumeClaimName: my-shared-storage
```

Doing the same operation in the **OpenShift Web Console** would require navigating to `Storage` → `Persistent Volume Claim` and choosing `Create Snapshot`.

![Persistent Volume Claim snapshot using UI](https://preview.cloud.189.cn/image/imageAction?param=8CC2BC28AC16D40C6ED934F605B2C42DDB2CD57681923B92E93763CD6122D5532E2E6FB191B3456386CD07F81C78DF6FA9C05C3D246CA97E15C62E6152CD6BDFC898380AD47E89AE1EB736F6505DEFA861602B60FC7242BDAF7A04F7FD14022F502F6D9F3CF3663BC6FDF64C8066F1C7)

Figure 25. Persistent Volume Claim snapshot using UI

Now create a snapshot for CephFS volume `my-shared-storage`.

```
oc apply -f /opt/app-root/src/support/my-shared-storage-snapshot.yaml
```

Example output:

```
volumesnapshot.snapshot.storage.k8s.io/my-shared-storage-snapshot created
```

Now check to see there is a new **VolumeSnapshot**.

```
oc get volumesnapshot -n my-shared-storage
```

Example output:

```
NAME                         READYTOUSE   SOURCEPVC           SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS                               SNAPSHOTCONTENT                                   CREATIONTIME   AGE
my-shared-storage-snapshot   true         my-shared-storage                           5Gi           ocs-storagecluster-cephfsplugin-snapclass   snapcontent-2d4729bc-a127-4da6-930d-2a7d0125d3b7   24s            26s
```

#### Restoring Volume Snapshot to clone PVC

You can now restore the new **VolumeSnapshot** in the **OpenShift Web Console**. Navigate to `Storage` → `Volume Snapshots`. Select `Restore as new PVC`. Make sure to have the `my-shared-storage` project selected at the top left.

![Persistent Volume Claim snapshot restore in UI](https://preview.cloud.189.cn/image/imageAction?param=379C840B04C903D56576392B5531527D5B4EC082715F780A6AC55059FB2B30E017FEA29C889AEDB33C5563A5125B8E3D7491B0C919815AF5CC183C65734846B3F2094ADE7B59229B943C9C21186EED6BC1D8E9BC4F1BFFB4FB1BD4DD22A6860D7424D5A01AD0E2CE4C2A2E7A27B7B766)

Figure 26. Persistent Volume Claim snapshot restore in UI

Chose the correct **StorageClass** to create the new clone from snapshot **PVC** and select `Restore`. The size of the new **PVC** is greyed out and is same as the `parent` or original **PVC** `my-shared-storage`.

![Persistent Volume Claim snapshot restore configuration](https://preview.cloud.189.cn/image/imageAction?param=C0F3FEC5147FACBC6003F1DA6E79628532C17CB0B5E13A007E99B7578CCE1EB5E1C30A91F068E279A5854D57E04CB598FA7756A991A8D841B160821234F6A2DAB52D2020F79D4C418475DBAA4488AB69760CA852F8B127E63A39B602010C7E24D1F7C138EE8779A1D78A03AE9877CDC3)

Figure 27. Persistent Volume Claim snapshot restore configuration

Check to see if there is a new **PVC** restored from the **VolumeSnapshot**.

```
oc get pvc -n my-shared-storage | grep restore
```

Example output:

```
my-shared-storage-snapshot-restore   Bound    pvc-24999d30-09f1-4142-b150-a5486df7b3f1   5Gi        RWX            ocs-storagecluster-cephfs   108s
```

The output shows a new **PVC** that could be used to recover an application if there is corruption or lost data.

## Using OCS for Prometheus Metrics

OpenShift ships with a pre-configured and self-updating monitoring stack that is based on the Prometheus open source project and its wider eco-system. It provides monitoring of cluster components and ships with a set of alerts to immediately notify the cluster administrator about any occurring problems. For production environments, it is highly recommended to configure persistent storage using block storage technology. OCS 4 provide block storage using Ceph RBD volumes. Running cluster monitoring with persistent storage means that your metrics are stored to a persistent volume and can survive a pod being restarted or recreated. This section will detail how to migrate Prometheus and AlertManager storage to Ceph RBD volumes for persistence.

First, let’s discover what **Pods** and **PVCs** are installed in the `openshift-monitoring` namespace. In the prior module, OpenShift Infrastructure Nodes, the Prometheus and AlertManager resources were moved to the OCP infra nodes.

```
oc get pods,pvc -n openshift-monitoring
```

Example output:

```
NAME                                               READY   STATUS         RESTARTS   AGE
pod/alertmanager-main-0                            5/5     Running        0          6d21h
pod/alertmanager-main-1                            5/5     Running        0          6d21h
pod/alertmanager-main-2                            5/5     Running        0          6d21h
pod/cluster-monitoring-operator-595888fddd-mcgnl   2/2     Running        0          4h49m
pod/grafana-65454464fd-5spx2                       2/2     Running        0          26h
pod/kube-state-metrics-7cb89d65d4-p9hbd            3/3     Running        0          6d21h
pod/node-exporter-96zjb                            2/2     Running        0          6d21h
pod/node-exporter-9jjdk                            2/2     Running        0          2d17h
pod/node-exporter-dhnt4                            2/2     Running        0          6d21h
pod/node-exporter-kg2fb                            2/2     Running        0          2d17h
pod/node-exporter-l27n2                            2/2     Running        0          16h
pod/node-exporter-qq4g7                            2/2     Running        0          16h
pod/node-exporter-rfnxb                            2/2     Running        0          16h
pod/node-exporter-v8kpq                            2/2     Running        0          2d17h
pod/node-exporter-wvm8n                            2/2     Running        0          6d21h
pod/node-exporter-wwcr9                            2/2     Running        0          6d21h
pod/node-exporter-z8r98                            2/2     Running        0          6d21h
pod/openshift-state-metrics-57969c7f87-h8fm4       3/3     Running        0          6d21h
pod/prometheus-adapter-cb658c44-zmcww              1/1     Running        0          2d22h
pod/prometheus-adapter-cb658c44-zsn85              1/1     Running        0          2d22h
pod/prometheus-k8s-0                               6/6     Running        0          6d21h
pod/prometheus-k8s-1                               6/6     Running        0          6d21h
pod/prometheus-operator-8594bd77df-ftwvl           2/2     Running        0          26h
pod/telemeter-client-79d7ddbf84-ft97l              3/3     Running        0          42h
pod/thanos-querier-787547fbd6-qw9tr                5/5     Running        0          6d21h
pod/thanos-querier-787547fbd6-xdsmm                5/5     Running        0          6d21h
```

At this point there are no **PVC** resources because Prometheus and AlertManager are both using ephemeral (EmptyDir) storage. This is the way OpenShift is initially installed. The Prometheus stack consists of the Prometheus database and the alertmanager data. Persisting both is best-practice since data loss on either of these will cause you to lose your metrics and alerting data.

### Modifying your Prometheus environment

For Prometheus every supported configuration change is controlled through a central **ConfigMap**, which needs to exist before we can make changes. When you start off with a clean installation of Openshift, the ConfigMap to configure the Prometheus environment may not be present. To check if your ConfigMap is present, execute this:

```
oc -n openshift-monitoring get configmap cluster-monitoring-config
```

Output if the ConfigMap is not yet created:

```
Error from server (NotFound): configmaps "cluster-monitoring-config" not found
```

Output if the ConfigMap is created:

```
NAME                        DATA   AGE
cluster-monitoring-config   1      116m
```

If you are missing the **ConfigMap**, create it using this command:

```
oc apply -f /opt/app-root/src/support/ocslab_cluster-monitoring-noinfra.yaml
```

Example output:

```
configmap/cluster-monitoring-config created
```

If the **ConfigMap** already exists because of completing prior module `OpenShift Infrastructure Nodes`, you will apply changes to the existing **ConfigMap**.

```
oc apply -f /opt/app-root/src/support/ocslab_cluster-monitoring-withinfra.yaml
```

Example output:

```
configmap/cluster-monitoring-config updated
```

You can view the **ConfigMap** with the following command:

| NOTE | The size of the Ceph RBD volumes, `40Gi`, can be modified to be larger or smaller depending on requirements. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

```
oc -n openshift-monitoring get configmap cluster-monitoring-config -o yaml | more
```

ConfigMap sample output:

```yaml
[...]
      volumeClaimTemplate:
        metadata:
          name: prometheusdb
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd
          resources:
            requests:
              storage: 40Gi
[...]
      volumeClaimTemplate:
        metadata:
          name: alertmanager
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd
          resources:
            requests:
              storage: 40Gi
[...]
```

Once you create this new **ConfigMap** `cluster-monitoring-config`, the affected **Pods** will automatically be restarted and the new storage will be mounted in the Pods.

| NOTE | It is not possible to retain data that was written on the default EmptyDir-based or ephemeral installation. Thus you will start with an empty DB after changing the backend storage thereby starting over with metric collection and reporting. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

After a couple of minutes, the AlertManager and Prometheus **Pods** will have restarted and you will see new **PVCs** in the `openshift-monitoring` namespace that they are now providing persistent storage.

```
oc get pods,pvc -n openshift-monitoring
```

Example output:

```
NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
[...]
alertmanager-alertmanager-main-0   Bound    pvc-733be285-aaf9-4334-9662-44b63bb4efdf   40Gi       RWO            ocs-storagecluster-ceph-rbd   3m37s
alertmanager-alertmanager-main-1   Bound    pvc-e07ebe61-de5d-404c-9a25-bb3a677281c5   40Gi       RWO            ocs-storagecluster-ceph-rbd   3m37s
alertmanager-alertmanager-main-2   Bound    pvc-9de2edf2-9f5e-4f62-8aa7-ecfd01957748   40Gi       RWO            ocs-storagecluster-ceph-rbd   3m37s
prometheusdb-prometheus-k8s-0      Bound    pvc-5b845908-d929-4326-976e-0659901468e9   40Gi       RWO            ocs-storagecluster-ceph-rbd   3m31s
prometheusdb-prometheus-k8s-1      Bound    pvc-f2d22176-6348-451f-9ede-c00b303339af   40Gi       RWO            ocs-storagecluster-ceph-rbd   3m31s
```

You can validate that Prometheus and AlertManager are working correctly after moving to persistent storage [Monitoring the OCS environment](https://dashboard-lab-ocp-cns.apps.odflab-55.container-contest.top/workshop/ocs4#_monitoring_the_ocs_environment) in a later section of this lab guide.

## Using the Multi-Cloud-Gateway

In this section, you will deploy a new OCP application that uses `Object Bucket Claims` (OBCs) to create dynamic buckets via the `Multicloud Object Gateway` (MCG). You will also use the `MCG Console` to validate new objects in the `Object Bucket`.

| NOTE | The `MCG Console` is not fully integrated with the **Openshift Web Console** and resources created in the `MCG Console` are not synchronized back to the Openshift Cluster. For MCG features such as Namespace buckets, please use the MCG console to configure. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Checking on the MCG status

The MCG status can be checked with the NooBaa CLI. Make sure you are in the `openshift-storage` project when you execute this command.

```
noobaa status -n openshift-storage
```

Example output:

```
INFO[0000] CLI version: 5.6.0
INFO[0000] noobaa-image: noobaa/noobaa-core:5.6.0
INFO[0000] operator-image: noobaa/noobaa-operator:5.6.0
INFO[0000] Namespace: openshift-storage
INFO[0000]
INFO[0000] CRD Status:
INFO[0000] ✅ Exists: CustomResourceDefinition "noobaas.noobaa.io"
INFO[0000] ✅ Exists: CustomResourceDefinition "backingstores.noobaa.io"
INFO[0000] ✅ Exists: CustomResourceDefinition "bucketclasses.noobaa.io"
INFO[0000] ✅ Exists: CustomResourceDefinition "objectbucketclaims.objectbucket.io"
INFO[0000] ✅ Exists: CustomResourceDefinition "objectbuckets.objectbucket.io"
INFO[0000]
INFO[0000] Operator Status:
INFO[0000] ✅ Exists: Namespace "openshift-storage"
INFO[0000] ✅ Exists: ServiceAccount "noobaa"
INFO[0000] ✅ Exists: Role "ocs-operator.v4.6.0-noobaa-6649766bf4"
INFO[0000] ✅ Exists: RoleBinding "ocs-operator.v4.6.0-noobaa-6649766bf4"
INFO[0000] ✅ Exists: ClusterRole "ocs-operator.v4.6.0-65577bfbc"
INFO[0000] ✅ Exists: ClusterRoleBinding "ocs-operator.v4.6.0-65577bfbc"
INFO[0000] ✅ Exists: Deployment "noobaa-operator"
INFO[0000]
INFO[0000] System Status:
INFO[0000] ✅ Exists: NooBaa "noobaa"
INFO[0000] ✅ Exists: StatefulSet "noobaa-core"
INFO[0000] ✅ Exists: Service "noobaa-mgmt"
INFO[0000] ✅ Exists: Service "s3"
INFO[0000] ✅ Exists: StatefulSet "noobaa-db"
INFO[0000] ✅ Exists: Service "noobaa-db"
INFO[0000] ✅ Exists: Secret "noobaa-server"
INFO[0000] ✅ Exists: Secret "noobaa-operator"
INFO[0000] ✅ Exists: Secret "noobaa-endpoints"
INFO[0000] ✅ Exists: Secret "noobaa-admin"
INFO[0000] ✅ Exists: StorageClass "openshift-storage.noobaa.io"
INFO[0000] ✅ Exists: BucketClass "noobaa-default-bucket-class"
INFO[0000] ✅ Exists: Deployment "noobaa-endpoint"
INFO[0000] ✅ Exists: HorizontalPodAutoscaler "noobaa-endpoint"
INFO[0000] ✅ (Optional) Exists: BackingStore "noobaa-default-backing-store"
INFO[0000] ✅ (Optional) Exists: CredentialsRequest "noobaa-aws-cloud-creds"
INFO[0000] ⬛ (Optional) Not Found: CredentialsRequest "noobaa-azure-cloud-creds"
INFO[0000] ⬛ (Optional) Not Found: Secret "noobaa-azure-container-creds"
INFO[0000] ⬛ (Optional) Not Found: Secret "noobaa-gcp-bucket-creds"
INFO[0000] ⬛ (Optional) Not Found: CredentialsRequest "noobaa-gcp-cloud-creds"
INFO[0000] ✅ (Optional) Exists: PrometheusRule "noobaa-prometheus-rules"
INFO[0000] ✅ (Optional) Exists: ServiceMonitor "noobaa-service-monitor"
INFO[0000] ✅ (Optional) Exists: Route "noobaa-mgmt"
INFO[0000] ✅ (Optional) Exists: Route "s3"
INFO[0000] ✅ Exists: PersistentVolumeClaim "db-noobaa-db-0"
INFO[0000] ✅ System Phase is "Ready"
INFO[0000] ✅ Exists:  "noobaa-admin"

#------------------#
#- Mgmt Addresses -#
#------------------#

ExternalDNS : [https://noobaa-mgmt-openshift-storage.apps.cluster-ocs4-8613.ocs4-8613.sandbox944.opentlc.com https://af3f0dd25ab0f4c7ba70f101f112ef0c-11
5712529.us-east-2.elb.amazonaws.com:443]
ExternalIP  : []
NodePorts   : [https://10.0.209.53:31759]
InternalDNS : [https://noobaa-mgmt.openshift-storage.svc:443]
InternalIP  : [https://172.30.22.156:443]
PodPorts    : [https://10.131.2.11:8443]

#--------------------#
#- Mgmt Credentials -#
#--------------------#

email    : admin@noobaa.io
password : Mph5Mhg/r2lCWj99O4jWjw==

#----------------#
#- S3 Addresses -#
#----------------#

ExternalDNS : [https://s3-openshift-storage.apps.cluster-ocs4-8613.ocs4-8613.sandbox944.opentlc.com https://a2087e1ee6e754d70bb96dd8922435b3-1451584877.
us-east-2.elb.amazonaws.com:443]
ExternalIP  : []
NodePorts   : [https://10.0.147.230:32297]
InternalDNS : [https://s3.openshift-storage.svc:443]
InternalIP  : [https://172.30.54.94:443]
PodPorts    : [https://10.130.2.70:6443]

#------------------#
#- S3 Credentials -#
#------------------#

AWS_ACCESS_KEY_ID     : SBC4HsLagqAy7IrGK2A3
AWS_SECRET_ACCESS_KEY : anilMy0atqj/QlVXMwNwbGasUpRJTXDM7/Mmt/AN

#------------------#
#- Backing Stores -#
#------------------#

NAME                           TYPE     TARGET-BUCKET                                       PHASE   AGE
noobaa-default-backing-store   aws-s3   nb.1610563076824.ocs4-8613.sandbox944.opentlc.com   Ready   7h9m5s

#------------------#
#- Bucket Classes -#
#------------------#

NAME                          PLACEMENT                                                             PHASE   AGE
noobaa-default-bucket-class   {Tiers:[{Placement: BackingStores:[noobaa-default-backing-store]}]}   Ready   7h9m5s

#-----------------#
#- Bucket Claims -#
#-----------------#

No OBCs found.
```

The NooBaa status command will first check on the environment and will then print all the information about the environment. Besides the status of the MCG, the second most intersting information for us are the available S3 addresses that we can use to connect to our MCG buckets. We can chose between using the external DNS which incurs DNS traffic cost, or route internally inside of our Openshift cluster.

You can get a more basic overview of the MCG status using the `Object Service` **Dashboard**. To reach this, log into the **Openshift Web Console**, click on `Home` and select the `Overview` item. In the main view, select `Object Service` in the top navigation bar. This dashboard does not give you connection information for your S3 endpoint, but offers Graphs and runtime information about the usage of your S3 backend as well as a link to the `MCG Console`.

### Creating and Using Object Bucket Claims

MCG **ObjectBucketClaims** (OBCs) are used to dynamically create S3 compatible buckets that can be used by an OCP application. When an OBC is created MCG creates a new **ObjectBucket** (OB), **ConfigMap** (CM) and **Secret** that together contain all the information your application needs to connect to the new bucket from within your deployment.

To demonstrate this feature we will use the Photo-Album demo application.

Run the application startup script which will build and deploy the application to your cluster.

```
cd /opt/app-root/src/support/photo-album
./demo.sh
```

| NOTE | Please make sure you follow the continuation prompts by pressing enter. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Example output:

```
[ OK    ] Using apps.cluster-7c31.7c31.sandbox905.opentlc.com as our base domain

Object Bucket Demo

 * Cleanup existing environment

Press any key to continue...
[ OK    ] oc delete --ignore-not-found=1 -f app.yaml

[ OK    ] oc delete --ignore-not-found=1 bc photo-album -n demo
buildconfig.build.openshift.io "photo-album" deleted

 * Import dependencies and create build config
-./demo.sh
[ OK    ] Using apps.cluster-7c31.7c31.sandbox905.opentlc.com as our base domain

Object Bucket Demo

 * Cleanup existing environment

Press any key to continue...
[ OK    ] oc delete --ignore-not-found=1 -f app.yaml

[ OK    ] oc delete --ignore-not-found=1 bc photo-album -n demo
buildconfig.build.openshift.io "photo-album" deleted

 * Import dependencies and create build config

[...]
 OK    ] oc start-build photo-album --from-dir . -F -n demo
photo-album setup
/opt/app-root/src/demo-apps/photo-album
```

| IMPORTANT | **Deployment might take up to 5 minutes or more to complete.** |
| --------- | ------------------------------------------------------------ |
|           |                                                              |

Check the photo-album deployment is complete by running:

```
oc -n demo get pods
```

Example output:

```
NAME                   READY   STATUS      RESTARTS   AGE
photo-album-1-build    0/1     Completed   0          10m
photo-album-1-deploy   0/1     Completed   0          10m
photo-album-1-rtplt    1/1     Running     0          10m
```

Now that the photo-album application has been deployed you can view the **ObjectBucketClaim** it created. Run the following:

```
oc -n demo get obc
```

Example output:

```
NAME          STORAGE-CLASS                 PHASE   AGE
photo-album   openshift-storage.noobaa.io   Bound   23m
```

To view the **ObjectBucket** (OB) that was created by the **OBC** above run the following:

```
oc get ob
```

Example output:

```
NAME                   STORAGE-CLASS                 CLAIM-NAMESPACE   CLAIM-NAME    RECLAIM-POLICY   PHASE   AGE
obc-demo-photo-album   openshift-storage.noobaa.io   demo              photo-album   Delete           Bound   23m
```

| NOTE | **OBs**, similar to **PVs**, are cluster-scoped resources so therefore adding the namespace is not needed. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

You can also view the new bucket **ConfigMap** and **Secret** using the following commands.

The **ConfigMap** will contain important information such as the bucket name, service and port. All are used to configure the connection from within the deployment to the s3 endpoint.

To view the **ConfigMap** created by the OBC, run the following:

```
oc -n demo get cm photo-album -o yaml | more
```

Example output:

```yaml
apiVersion: v1
data:
  BUCKET_HOST: s3.openshift-storage.svc
  BUCKET_NAME: photo-album-2c0d8504-ae02-4632-af83-b8b458b9b923
  BUCKET_PORT: "443"
  BUCKET_REGION: ""
  BUCKET_SUBREGION: ""
kind: ConfigMap
[...]
```

The **Secret** will contain the credentials required for the application to connect and access the new object bucket. The credentials or keys are `base64` encoded in the **Secret**.

To view the **Secret** created for the OBC run the following:

```
oc -n demo get secret photo-album -o yaml | more
```

Example output:

```yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: MTAyc3pJNnBsM3dXV0hOUzUyTEk=
  AWS_SECRET_ACCESS_KEY: cWpyWWhuendDcjNaR1ZyVkZVN1p4c2hRK2xicy9XVW1ETk50QmJpWg==
kind: Secret
[...]
```

As you can see when the new **OBC** and **OB** are created, MCG creates an associated **Secret** and **ConfigMap** which contain all the information required for our photo-album application to use the new bucket.

To view exactly how the application uses the information in the new **Secret** and **ConfigMap** have a look at the file `photo-album/app.yaml` after you have deployed the app.

In order to view the details of the **ObjectBucketClaim** view the start of `photo-album/app.yaml`. In the **DeploymentConfig** specification section, find `env:` and you can see how the **ConfigMap** and **Secret** details are mapped to environment variables.

```
cat /opt/app-root/src/support/photo-album/app.yaml | more
```

Example output:

```yaml
---
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: "photo-album"
  namespace: demo
spec:
  generateBucketName: "photo-album"
  storageClassName: openshift-storage.noobaa.io
---
[...]
     spec:
        containers:
        - image: image-registry.openshift-image-registry.svc:5000/default/photo-album
          name: photo-album
          env:
            - name: ENDPOINT_URL
              value: 'https://s3-openshift-storage.apps.cluster-7c31.7c31.sandbox905.opentlc.com'
            - name: BUCKET_NAME
              valueFrom:
                configMapKeyRef:
                  name: photo-album
                  key: BUCKET_NAME
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: photo-album
                  key: AWS_ACCESS_KEY_ID
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: photo-album
                  key: AWS_SECRET_ACCESS_KEY
[...]
```

In order to create objects in your new bucket you must first find the route for the `photo-album` application.

```
oc get route photo-album -n demo -o jsonpath --template="http://{.spec.host}{'\n'}"
```

Example output:

```
http://photo-album.apps.cluster-7c31.7c31.sandbox905.opentlc.com
```

Copy and paste this route into a web browser tab.

![Select Photo and Upload](https://preview.cloud.189.cn/image/imageAction?param=261F91A7E797BF0A83617E323BC87FFE0EC0B76C84AF1673F1A982A0C335851C03AE2A6A29AB95E3C7B01EC75895BAEB5904A29B5B537F2DD9411C55E658BD75FCA535E5833F511AC3D47C4F947A88A3F614CF551E4BD06B4A2E76A640DB41F0059E496595B409452B9E00F5E010E674)

Figure 28. Select Photo and Upload

Select one or more photos of your choosing on your local machine. Then make sure to click the `Upload` button for each photo.

![View photos after uploading](https://preview.cloud.189.cn/image/imageAction?param=5D8FB9FEA3FEDC88D7FA9B463DADE209ED5B4D4C05F295B0F877FDF01A36B946703154665EFAEC372997DA54DD427A4F44547688B15D2DA1968134F28CE571DBCD849488F95CDE1E881318E224A2D5D81D26E31AE2AA4781A0821676E1C60334E8B17FE1B1C1431C12510775485912CD)

Figure 29. View photos after uploading

To view the photos in your object bucket navigate to the `MCG Console` by viewing the **Object Service** dashboard viewed previously. Select the `Mulitcloud Object Gateway` link under `System Name`.

![Launch MCG console from Object Service dashboard](https://preview.cloud.189.cn/image/imageAction?param=596DFECF99F70CFD769AC2E2C0A7AA3EA95CA48B98F4FE90E2AB9F85AB17152ACCA616F0E82389BDE902BAFE8F46F82ADE4DDF4BC17982A30AE8C71E8BEDA93B38FEF009D3558A9BF486B8875F9578D9FBE8B6D75FB8D7E1E204996009B5812D1F5F34FEEFEB7DEE0CB5025D891AFB65)

Figure 30. Launch MCG console from Object Service dashboard

Login to the `MCG Console` using `username` kubeadmin and your `password`.

```
VyYA4-KWyRT-9bFCf-2Frvz
```

You can navigate to the bucket details by selecting the `Buckets` on the far right side. Now select `Object Buckets`.

![Login to MCG Console and select Buckets](https://preview.cloud.189.cn/image/imageAction?param=C7652A7A18AF535F2E4E87A4B6140563BDDF4861C4B797F387021626BA8C52A885AA4BB0A46DDBE88AB0D20EF2B0129E440BA0981C258E99675885657F2DFA064F8EFA548A6846D25DB7657D9F5C6EE9940DF8E3079A136EB724502A7341D1288B34DA8384E1B1BEA79E401A37E47BCF)

Figure 31. Login to MCG Console and select Buckets

Select your bucket name under `Object Buckets` and then select the `Objects` tab to view the individual objects create when you uploaded your photos.

![Validate uploaded photos are in your object bucket](https://preview.cloud.189.cn/image/imageAction?param=9F7C691083165CC677D9F8E5CFE60118B25D8047B19C19A25E5E47718491CAE34005D1108D6B1ED0F7A89D2285A23DC80CE4BD23A0C5126CBE2E2173FBEB443C4F7266241972540F3AFF2564E94A0C48EC5686A54E14F886A0CA35B495AD522211FE5E6EEA7ADC5E92DB23E579F43538)

Figure 32. Validate uploaded photos are in your Object Bucket

## Adding storage to the Ceph Cluster

Adding storage to OCS adds capacity and performance to your already present cluster.

| NOTE | The reason for adding more OCP worker nodes for storage is because the existing nodes do not have adequate CPU and/or Memory available. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Add storage worker nodes

This section will explain how one can add more worker nodes to the present storage cluster. Afterwards follow the next sub-section on how to extend the OCS cluster to provision storage on these new nodes.

To add more nodes, we could either add more **machinesets** like we did before, or scale the already present OCS **machinesets**. For this training, we will spawn more workers by scaling the already present OCS worker instances up from 1 to 2 **machines**.

Check on our current workerocs **machinesets** and **machine** counts:

```
oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'
```

Example output:

```
NAME                                           DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster-ocs4-8613-bc282-workerocs-us-east-2a   1         1         1       1           2d
cluster-ocs4-8613-bc282-workerocs-us-east-2b   1         1         1       1           2d
cluster-ocs4-8613-bc282-workerocs-us-east-2c   1         1         1       1           2d
```

Let’s scale the workerocs machinesets up with this command:

```
oc get machinesets -n openshift-machine-api -o name | grep workerocs | xargs -n1 -t oc scale -n openshift-machine-api --replicas=2
```

Example output:

```
oc scale -n openshift-machine-api --replicas=2 machineset.machine.openshift.io/cluster-ocs4-8613-bc282-workerocs-us-east-2a
machineset.machine.openshift.io/cluster-ocs4-8613-bc282-workerocs-us-east-2a scaled
oc scale -n openshift-machine-api --replicas=2 machineset.machine.openshift.io/cluster-ocs4-8613-bc282-workerocs-us-east-2b
machineset.machine.openshift.io/cluster-ocs4-8613-bc282-workerocs-us-east-2b scaled
oc scale -n openshift-machine-api --replicas=2 machineset.machine.openshift.io/cluster-ocs4-8613-bc282-workerocs-us-east-2c
machineset.machine.openshift.io/cluster-ocs4-8613-bc282-workerocs-us-east-2c scaled
```

Wait until the new OCP workers are available. This could take 5 minutes or more so be patient. You will know the new OCP worker nodes are available when you have the number `2` in all columns.

```
watch "oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'"
```

You can exit by pressing Ctrl+C.

Once they are available, you can check to see if the new OCP worker nodes have the OCS label applied. The total of OCP nodes with the OCS label should now be six.

| NOTE | The OCS label `cluster.ocs.openshift.io/openshift-storage=` is already applied because it is configured in the workerocs **machinesets** that you used to create the new worker nodes. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

```
oc get nodes -l cluster.ocs.openshift.io/openshift-storage -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}'
```

Example output:

```
ip-10-0-147-230.us-east-2.compute.internal
ip-10-0-157-22.us-east-2.compute.internal
ip-10-0-175-8.us-east-2.compute.internal
ip-10-0-183-84.us-east-2.compute.internal
ip-10-0-209-53.us-east-2.compute.internal
ip-10-0-214-36.us-east-2.compute.internal
```

Now that you have the new instances created with the OCS label, the next step is to add more storage to the Ceph cluster. The OCS operator will prefer the new OCP nodes with the OCS label because they have no OCS **Pods** scheduled yet.

### Add storage capacity

In this section we will add storage capacity and performance to the configured OCS worker nodes and the Ceph cluster. If you have followed the previous section you should now have 6 OCS nodes.

To add storage, go to the **Openshift Web Console** and follow these steps to reach the OCS storage cluster overview:

- Click on `Operators` on the left navigation bar
- Select `Installed Operators` and select `openshift-storage` project
- Click on `Openshift Container Storage Operator`
- In the top navigation bar, scroll right to find the item `Storage Cluster` and click on it

![OCS4 OCP4 Storage Cluster overview reachit](https://preview.cloud.189.cn/image/imageAction?param=509E3A4CC643D2C7555081D3691D16FA824B6654DE590E440E8563A32BADF4BBB6A60D49D7EA0302F2A1419F6B75D7D4BF5095F647E0C4674AE5995B65C914CF2AAA16B8CC775A6DDF94BE880B8391FFB695FF6441BFC17516631AF02397F5FA61BA951DBD59C03DA7F7633160906CBD)

- The visible list should list only one item - click on the three dots on the far right to extend the options menu
- Select `Add Capacity` from the options menu

![Add Capacity dialog](https://preview.cloud.189.cn/image/imageAction?param=88D04ABDDCE53E5FF3B903877D49BC4652A8E36085423F248791FCE89A81BDBB4A6087A825E51259C7869176C3E70494095027AE8112B2F623E30BDD7E169DF09A5BA439E99248ED4FD480BAC506B6386AD31AF933E2545F1DE03BD3D7EE53663C2D8BF32927C0929B8C55ADDCDB5885)

Figure 33. Add Capacity dialog

The storage class should be set to `gp2`. The added provisioned capacity will be three times as much as you see in the `Raw Capacity` field, because OCS uses a replica count of 3.

| NOTE | **The size chosen for OCS Service Capacity during the initial deployment of OCS is greyed out and cannot be changed.** |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Once you are done with your setting, proceed by clicking on `Add`. You will see the Status of the Storage Cluster is `Ready`.

| CAUTION | It may take more than 5 minutes for new OSD pods to be in a `Running` state. |
| ------- | ------------------------------------------------------------ |
|         |                                                              |

Use this command to see the new OSD pods:

```
oc get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName -n openshift-storage | grep osd | grep -v prepare
```

Example output:

```
rook-ceph-osd-0-7d45696497-jwgb7            Running     ip-10-0-147-230.us-east-
2.compute.internal
rook-ceph-osd-1-6f49b665c7-gxq75            Running     ip-10-0-209-53.us-east-2
.compute.internal
rook-ceph-osd-2-76ffc64cd-9zg65             Running     ip-10-0-175-8.us-east-2.
compute.internal
rook-ceph-osd-3-97b5d9844-jpwgm             Running     ip-10-0-157-22.us-east-2
.compute.internal
rook-ceph-osd-4-9cb667b76-mftt9             Running     ip-10-0-214-36.us-east-2
.compute.internal
rook-ceph-osd-5-55b8d97855-2bp85            Running     ip-10-0-157-22.us-east-2
.compute.internal
```

This is everything that you need to do to extend the OCS storage.

### Verify new storage

Once you added the capacity and made sure that the OSD pods are present, you can also optionally check the additional storage capacity using the Ceph **toolbox** created earlier. Follow these steps:

```
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD
```

Check the status of the Ceph cluster:

```
ceph status
```

Example output:

```
sh-4.2# ceph status
  cluster:
    id:     e3398039-f8c6-4937-ba9d-655f5c01e0ae
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 25m)
    mgr: a(active, since 24m)
    mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 38s), 6 in (since 38s) (1)

  task status:
      scrub status:
          mds.ocs-storagecluster-cephfilesystem-a: idle
          mds.ocs-storagecluster-cephfilesystem-b: idle

  data:
    pools:   3 pools, 192 pgs
    objects: 92 objects, 81 MiB
    usage:   6.1 GiB used, 12 TiB / 12 TiB avail (2)
    pgs:     192 active+clean

  io:
    client:   1.2 KiB/s rd, 1.7 KiB/s wr, 2 op/s rd, 0 op/s wr
```

In the Ceph status output, we can already see that:

1. We now use 6 osds in total and they are `up` and `in` (meaning the daemons are running and being used to store data)
2. The available raw capacity has increased from 6 TiB to 12 TiB

Besides that, nothing has changed in the output.

Check the topology of your cluster:

```
ceph osd crush tree
```

Example output:

```
ID  CLASS WEIGHT   TYPE NAME
 -1       12.00000 root default
 -5       12.00000     region us-east-2
 -4        4.00000         zone us-east-2a
 -3        2.00000             host ocs-deviceset-gp2-0-data-0-9977n
  0   ssd  2.00000                 osd.0
-21        2.00000             host ocs-deviceset-gp2-2-data-1-nclgr (1)
  4   ssd  2.00000                 osd.4
-14        4.00000         zone us-east-2b
-13        2.00000             host ocs-deviceset-gp2-1-data-0-nnmpv
  2   ssd  2.00000                 osd.2
-19        2.00000             host ocs-deviceset-gp2-0-data-1-mg987 (1)
  3   ssd  2.00000                 osd.3
-10        4.00000         zone us-east-2c
 -9        2.00000             host ocs-deviceset-gp2-2-data-0-mtbtj
  1   ssd  2.00000                 osd.1
-17        2.00000             host ocs-deviceset-gp2-0-data-2-l8tmb (1)
  5   ssd  2.00000                 osd.5
```

1. We now have additional hosts, which are extending the storage in the respective zone.

Since our Ceph cluster’s CRUSH rules are set up to replicate data between the zones, this is an effective way to reduce the load on the 3 initial nodes.

Existing data on the original OSDs will be balanced out automatically, so that the old and the new OSDs share the load.

You can exit the **toolbox** by either pressing Ctrl+D or by executing exit.

```
exit
```

## Monitoring the OCS environment

This section covers the different tools available when it comes to monitoring OCS the environment. This section relies on using the **OpenShift Web Console**.

Individuals already familiar with OCP will feel comfortable with this section but for those who are not, it will be a good primer.

The monitoring tools are accessible through the main **OpenShift Web Console** left pane. Click the **Monitoring** menu item to expand and have access to the following 3 selections:

- Alerting
- Metrics
- Dashboards

### Alerting

Click on the **Alerting** item to open the Alert window as illustrated in the screen capture below.

![OCP Monitoring Menu](https://preview.cloud.189.cn/image/imageAction?param=DE2FA3F488F72586A075EC09AA793BEA7E3695698FEC5211A7DD86DB99584A4162ECF3C5338E689D9C6C53404E8A45C439C069DAA8D62009D18E6BE767CD0BA992177298BFC627EB3CAB5FF761DEA96F60E7139E3BA89A16A5D9B42397DDFC30CEF114826759B85EB2D76AFA3EF56E4F)

Figure 34. OCP Monitoring Menu

This will take you to the **Alerting** homepage as illustrated below.

![OCP Alerting Homepage](https://preview.cloud.189.cn/image/imageAction?param=CD18574FBE569B8723C1D02DBB472E492686C50CB2CD93B9EDB030F056E87F8D7B639A1050E93824D07DA428CC3D63161E3867F49AEA04E6905809AF86CA9F2F5DB64F758EDDB59E908F05555E2AA6B65AC55C93A4C9A501C9B24CB2008A53F0D7BE889A964A6BBD2FBF5AE58F46AEB9)

Figure 35. OCP Alerting Homepage

You can display the alerts in the main window using the filters at your disposal.

- 1 - Will let you select alerts by State, Severity and Sourc
- 2 - Will let you select if you want to search a specific character string using either the `Name` or the `Label`
- 3 - Will let you enter the character string you are searching for

The alert `State` can be.

- `Firing` - Alert has been confirmed
- `Silenced` - Alerts that have been silenced while they were in `Pending` or `Firing` state
- `Pending` - Alerts that have been triggered but not confirmed

| NOTE | An alert transitions from `Pending` to `Firing` state if the alert persists for more than the amount of time configured in the alert definition (e.g. 10 minutes for the `CephClusterWarningState` alert). |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The alert `Severity` can be.

- `Critical` - Alert is tagged as critical
- `Warning` - Alert is tagged as warning
- `Info` - Alert is tagged as informational
- `None` - The alert has no `Severity` assigned

The alert `Source` can be.

- `Platform` - Alert is generated by an OCP component
- `User` - Alert is generated by a user application

As illustrated below, alerts can be filtered precisely using multiple criteria.

![OCP Alert Status Filtering](https://preview.cloud.189.cn/image/imageAction?param=C557A43A4A1F1F56FA6D553848B991CB9FA32102E20E65839404FC77678E8E7CC8875A1911555429789ECAC1F23DB9B17049591A64173EB01648CECFAFBF7757C8BBBBF75064C415E8ED9AD5A82A6ADB521B2BC6CB16F5E56F822612580FE8045ABA6282D43C85C207F7D2DCB4F9698E)

Figure 36. OCP Alerting Status Filtering

| NOTE | You can clear all filters to view all the existing alerts. |
| ---- | ---------------------------------------------------------- |
|      |                                                            |

If you select `View Alerting Rule` you will get access to the details of the rule that triggered the alert. The details include the Prometheus query used by the alert to perform the detection of the condition.

![OCP Alert Contextual Menu](https://preview.cloud.189.cn/image/imageAction?param=E6A897F50744E359186560C5AF1280855EBD5194921DDAA8C619E02B798B95746F3D52F82C6562C4E2AAEE027580E78CE7DA441649AB95C7F7C776F1D406530C199CC6438B00E17F4C1DDB967CA2FB1DD8AFA4475F15EBA7E9A394B50A61632495C5780039D96F19BDE452DC21F9FB83)

Figure 37. OCP Alert Contextual Menu

![OCP Alert Detailed Display](https://preview.cloud.189.cn/image/imageAction?param=845BDDAE938C200186C5D964C977C8340BDD6FFEE0D97050088DDF52D68DB5B1E2DB101E57F178EB9678F92626EBDF0EC5E25D67452FA11DADF2A0FD2E1C9E999CADA5EEA6D95B4DD5DF8AD395895ABA0B62B490F177631A168DAED7BF46BECDAE0B4450A385A1D9EBE2E28C9272D849)

Figure 38. OCP Alert Detail Display

| NOTE | If desired, you can click the Prometheus query embedded in the alert. Doing so will take you to the **Metrics** page where you will be able to execute the query for the alert and if desired make changes to the rule. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Metrics

Click on the **Metrics** item as illustrated below in the `Monitoring` menu.

![OCP Metrics Menu](https://preview.cloud.189.cn/image/imageAction?param=AD20E8C2CDBC256D166C13CE7FED99A289FE45B1383A867EC92BD3E383FD4D26B6D8464560970480E912FB549495AFD9317BD87358CD45925FC69A98BE4A89F4D959FDBE3C48DE90E47DBE4B52F413EA29BFE18DEA25429605B499A9035DED6E15569A60746DE2F4DCC414245617161E)

Figure 39. OCP Metrics Menu

This will take you to the **Metrics** homepage as illustrated below.

![OCP Monitoring Metrics Homepage](https://preview.cloud.189.cn/image/imageAction?param=DF346FB327E630469834C5F20C2C6CEA172051AE0FE05C62148D2CFC181635EE7148F2AF472FCA6088BFC4FFD6FBDD07C260E676113A7352A38EBD40B6A03B92DD290CF71C7812B55DDD6BFED74D9C8A21F7EAC953E301B9BDEF9090995B81E608C8249F8EF8D9ADEF0C5C12EEE35C94)

Figure 40. OCP Monitoring Metrics Homepage

Use the query field to either enter the formula of your choice or to search for metrics by name. The metrics available will let you query both OCP related information or OCS related information. The queries can be simple or complex using the Prometheus query syntax and all its available functions.

Let’s start testing a simple query example and enter the following text `ceph_osd_op` in the query field. When you are done typing, simply hit `[Enter]` or select `Run Queries`.

![Ceph Simple Query](https://preview.cloud.189.cn/image/imageAction?param=70A699642B6BEAD6EC070E4CEA6BF8549AE0D9FFA5111EB2906ED591C73AFE44B1AF706CEB40B16A0D40A0EA53EA54A0C1D32089834F31DEBEF5BBCA48CAF24F86E7BAC61C50836968EC8ACF2968FC04D47757CB188F53668E1298E449E271351F41D5081FDA73B88EB95EEB251F1A7F)

Figure 41. Simple Ceph Query

The window should refresh with a graph similar to the one below.

![Ceph Simple Graph](https://preview.cloud.189.cn/image/imageAction?param=42D0B04AF949EBE976FC00E18480C76A970268D8CDAC96579E160F74CCA78E9833FB80D7A3D0EB5C582440621F12268D6E559C840152908DD718F19ECDEBD1BA7AFD14C9A065D7F501F09CB9617050B28A615D767D22C00E3D1E8DAE0BEA20CFA9F1ACBD326845D86EFC3B7EE6912198)

Figure 42. Simple Ceph Graph

Then let’s try a more relevant query example and enter the following text `rate(ceph_osd_op[5m])` or `irate(ceph_osd_op[5m])` in the query field. When you are done typing, simply hit `[Enter]`or select `Run Queries`.

![Ceph Complex Query](https://preview.cloud.189.cn/image/imageAction?param=D26BC4CBA5FB421BAC651F926DFE32543B7A27CD66F108458F12174150FE5B7369DFF24707159DF7D9C4F1869254DCBDBD8180A4852F7385ADF5846533601771B2CF344E7E7586072DCAA33FDFD7421DD0212F11368B211EAA4702933C9AA8CAEE3051EB44CF73BDC99C8AF970321932)

Figure 43. Complex Ceph Query

The window should refresh with a graph similar to the one below.

![Ceph Complex Graph](https://preview.cloud.189.cn/image/imageAction?param=EECAF7C2180C79C622D2F548400FBC2E8EDA477FEB10EF2FA21274A4A693F2B7A3A7A0A6E2C9BF04F659469AA3942D9FFE4D503859FF5DBA18F70D40CF59E0DA831DE01B02D30F9BF6A8559EA20B8A482E40B7C37C3BA13F7B3F7216D12AAFFDDD717F1BF976A25BA1130825DB3315F8)

Figure 44. Complex Ceph Graph

All OCP metrics are also available through the integrated **Metrics** window. Feel free to try with any of the OCP related metrics such as `irate(process_cpu_seconds_total[5m])` for example.

![OCP Complex Graph](https://preview.cloud.189.cn/image/imageAction?param=238F5220B0A055EA916CB040810B0206D9FCF942639B0F521890E19406ECAA00145C9C7C7946518C55B7A3BD3CEA4D16B22D18130D2E3785C3CF8014A97BEDEE2495676843BBE66C12B2E926FE3F980AF8F441D8F9F8A0B7010984992C44034EF47E3C03F22ADA36F1B143C110FD2D01)

Figure 45. Complex OCP Graph

Have a look at the difference between `sum(irate(process_cpu_seconds_total[5m]))` and the last query `irate(process_cpu_seconds_total[5m])`.

| NOTE | For more information on the Prometheus query language visit the [Prometheus Query Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/). |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

## Using must-gather

Must-gather is a tool for collecting data about your running Openshift cluster. It loads a predefined set of containers that execute multiple programs and write results on the local workstation’s filesystem. The local files can then be uploaded to a Red Hat case and used by a remote support engineer to debug a problem without needing direct access to your cluster. This utility and method for diagnostic collection is similar to sosreports for RHEL hosts.

The OCS team has released its own must-gather image for the must-gather tool that runs storage specific commands.

You can run this diagnostic tool like this for generic OpenShift debugging:

```
oc adm must-gather
```

Or like this for OCS specific results:

```
oc adm must-gather --image=registry.redhat.io/ocs4/ocs-must-gather-rhel8:v4.6
```

The output will then be saved in the current directory inside of a new folder called `must-gather.local.(random)`

More runtime options can be displayed with this command:

```
oc adm must-gather -h
```

Example output:

```
Launch a pod to gather debugging information

 This command will launch a pod in a temporary namespace on your cluster that gathers debugging information and then
downloads the gathered information.

 Experimental: This command is under active development and may change without notice.

Usage:
  oc adm must-gather [flags]

Examples:
  # gather information using the default plug-in image and command, writing into ./must-gather.local.<rand>
  oc adm must-gather

  # gather information with a specific local folder to copy to
  oc adm must-gather --dest-dir=/local/directory

  # gather information using multiple plug-in images
  oc adm must-gather --image=quay.io/kubevirt/must-gather --image=quay.io/openshift/origin-must-gather

  # gather information using a specific image stream plug-in
  oc adm must-gather --image-stream=openshift/must-gather:latest

  # gather information using a specific image, command, and pod-dir
  oc adm must-gather --image=my/image:tag --source-dir=/pod/directory -- myspecial-command.sh

Options:
      --dest-dir='': Set a specific directory on the local machine to write gathered data to.
      --image=[]: Specify a must-gather plugin image to run. If not specified, OpenShift's default must-gather image
will be used.
      --image-stream=[]: Specify an image stream (namespace/name:tag) containing a must-gather plugin image to run.
      --node-name='': Set a specific node to use - by default a random master will be used
      --source-dir='/must-gather/': Set the specific directory on the pod copy the gathered data from.

Use "oc adm options" for a list of global command-line options (applies to all commands).
```

## Appendix A: Introduction to Ceph

This section will go through Ceph fundamental knowledge for a better understanding of the underlying storage solution used by OCS 4.

| NOTE | The content in this Appendix is relevant to learning about the critical components of Ceph and how Ceph works. OCS 4 uses Ceph in a prescribed manner for providing storage to OpenShift applications. Using **Operators** and **CustomResourceDefinitions** (CRDs) for deploying and managing OCS 4 may restrict some of Ceph’s advanced features when compared to general use outside of OCP 4. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

**Timeline**

The Ceph project has a long history as you can see in the timeline below.

![Ceph Project Timeline](https://preview.cloud.189.cn/image/imageAction?param=24D8A6F8415329C965CDC8681DB7FDDDBD70D1B400CAF897A561D063E136ADD46F83B7286724310E3C2F73625B60F528323839FF8B2D501E96AD28599D5988EBD3379F9BC5E87A8DDE2034739566EFB297701750551DA5A92EF2FA0B38B80E4F018431E16AC4BAC13990617E1C771B50)

Figure 46. Ceph Project History

It is a battle-tested software defined storage (SDS) solution that has been available as a storage backend for OpenStack and Kubernetes for quite some time.

**Architecture**

The Ceph cluster provides a scalable storage solution while providing multiple access methods to enable the different types of clients present within the IT infrastructure to get access to the data.

![Ceph From Above](https://preview.cloud.189.cn/image/imageAction?param=13F9D48922C00CC873A73C5A08336E7E5DE18940513745B1690ABDC514F4E1D0384DAA7BE4CF4EA99B58D41915C648E36294922FAAF62221C4C11174FF8FAEC5DAC2EFF8D349B3E77BB7CB3B7B11B99F3242C5487201D21C32F23F706DE6D87E68B90CF4C39DAB0E654650B540488760)

Figure 47. Ceph Architecture

The entire Ceph architecture is resilient and does not present any single point of failure (SPOF).

**RADOS**

The heart of Ceph is an object store known as RADOS (Reliable Autonomic Distributed Object Store) bottom layer on the screen. This layer provides the Ceph software defined storage with the ability to store data (serve IO requests, to protect the data, to check the consistency and the integrity of the data through built-in mechanisms. The RADOS layer is composed of the following daemons:

1. MONs or Monitors
2. OSDs or Object Storage Devices
3. MGRs or Managers
4. MDSs or Meta Data Servers

***Monitors\***

The Monitors maintain the cluster map and state and provide distributed decision-making while configured in an odd number, 3 or 5 depending on the size and the topology of the cluster, to prevent split-brain situations. The Monitors are not in the data-path and do not serve IO requests to and from the clients.

***OSDs\***

One OSD is typically deployed for each local block devices and the native scalable nature of Ceph allows for thousands of OSDs to be part of the cluster. The OSDs are serving IO requests from the clients while guaranteeing the protection of the data (replication or erasure coding), the rebalancing of the data in case of an OSD or a node failure, the coherence of the data (scrubbing and deep-scrubbing of the existing data).

***MGRs\***

The Managers are tightly integrated with the Monitors and collect the statistics within the cluster. Additionally they provide an extensible framework for the cluster through a pluggable Python interface aimed at expanding the Ceph existing capabilities. The current list of modules developed around the Manager framework are:

- Balancer module
- Placement Group auto-scaler module
- Dashboard module
- RESTful module
- Prometheus module
- Zabbix module
- Rook module

***MDSs\***

The Meta Data Servers manage the metadata for the POSIX compliant shared filesystem such as the directory hierarchy and the file metadata (ownership, timestamps, mode, …). All the metadata is stored with RADOS and they do not server any data to the clients. MDSs are only deployed when a shared filesystem is configured in the Ceph cluster.

If we look at the Ceph cluster foundation layer, the full picture with the different types of daemons or containers looks like this.

![RADOS Overview](https://preview.cloud.189.cn/image/imageAction?param=553F173FBE3605F63E44356B8F525C526674130B1CE60990356F5BFF9DCCF9AFC854638F6ED4BA4C55B273F9E65FAF6DBF8E5088B6C821B2AF56130E2CBA9681CCD92070D37B02CC6FFDE7D45C0F4E8F95C87DE9DDFD0FA8117BFFEA25B3E8BDC6BC8D877BFAC883C1CDE8AB3BCA22EB)

Figure 48. RADOS as it stands

The circle represent the MONs, the 'M' represent the MGRs and the squares with the bars represent the OSDs. In the diagram above, the cluster operates with 3 Monitors, 2 Managers and 23 OSDs.

**Access Methods**

Ceph was designed to provides the IT environment with all the necessary access methods so that any application can use what is the best solution for its use-case.

![Ceph Access Modes](https://preview.cloud.189.cn/image/imageAction?param=B425FFBBF37FEAD248130D7B722A2C5F15AC4A25A82E62E6A2FE9CBD394D2B894AC87418EFB5CA5E83A23E2D0877FE2A6A68A9F03654181FC638E35885CDD7FC18B246EBE4340E1240D946FCBBBAADB27F09CAE8D00736C83CC57C510061B4A6EAAF1EF5E0812BFA3956445397340670)

Figure 49. Different Storage Types Supported

Ceph supports block storage through the RADOS Block Device (aka RBD) access method, file storage through the Ceph Filesystem (aka CephFS) access method and object storage through its native `librados` API or through the RADOS Gateway (aka RADOSGW or RGW) for compatibility with the S3 and Swift protocols.

**Librados**

Librados allows developers to code natively against the native Ceph cluster API for maximum efficiency combined with a small footprint.

![librados](https://preview.cloud.189.cn/image/imageAction?param=E39E2450D91B7C68AADB581B74445C4B92359DE96C1E1D5236A4944AF6C1D8261E6930D98A0637B2D698A36DA64082D006312CB5C38081062C0D9D6D7A13579F33BB5260CE0884BF862BF1E3833E13D3B632C2146825B0E8AC473CF2C6736EBEF42E0165E7A9B81AAFF21CD4A33E58DC)

Figure 50. Application Native Object API

The Ceph native API offers different wrappers such as C, C++, Python, Java, Ruby, Erlang, Go and Rust.

**RADOS Block Device (RBD)**

This access method is used in Red Hat Enterprise Linux or OpenShift version 3.x or 4.x. RBDs can be accessed either through a kernel module (RHEL, OCS4) or through the `librbd` API (RHOSP). In the OCP world, RBDs are designed to address the need for RWO PVCs.

***Kernel Module (kRBD)\***

The kernel RBD driver offers superior performance compared to the userspace `librbd` method. However, kRBD is currently limited and does not provide the same level of functionality. e.g., no RBD Mirroring support.

![Kernel based RADOS Block Device](https://preview.cloud.189.cn/image/imageAction?param=E99C6DDD2C3736EE6C255ED1223C501CB2C3874B0642BC495E1B07B76D6E0F8B7A063CD11A6480E5DBCF6510666595AEB27C46268CAE00962F5581C0DDA2DF9064194303ECE18FB755B722F6D113D2F52F458F6E12A3D32DC24F3242926242B994ED7395AF18029EE8352FE67DE42FE7)

Figure 51. kRBD Diagram

***Userspace RBD (librbd)\***

This access method is used in Red Hat OpenStack Environment or OpenShift through the RBD-NBD driver when available starting in the RHEL 8.1 kernel. This mode allows us to leverage all existing RBD features such as RBD Mirroring.

![Userspace RADOS Block Device](https://preview.cloud.189.cn/image/imageAction?param=B36F3B9C0062813738FAFC8244D9F25818ADF413EA58AE1381269D8C492267BDE93388E2FD5B471C14B27995BD47BE6E27B4D1E23E834876E1AC263F96B97ADC997B6A7ED39EFD25D976873891AC104BE331A1623EE6198FD7FFD4D2C679F07C6CBA64AF4F44D313732A9AAC3A2326F6)

Figure 52. librbd Diagram

***Shared Filesystem (CephFS)\***

This method allows clients to jointly access a shared POSIX compliant filesystem. The client initially contacts the Meta Data Server to obtain the location of the object(s) for a given inode and then communicates directly with an OSD to perform the final IO request.

![Kernel Based CephFS Client](https://preview.cloud.189.cn/image/imageAction?param=8DB1780CAEC1A3120A0B71C9A0FD07F4FB32DD79863B2385515D4F931034A63E8E37496B6F650A8E3A6FD4538E4ED890F37C277D54C68EFD807521FCFA661A52D0259C0D5AE6C03D7EC0D00ECE09D05A3B352BEC51DA8495E60DB99EFD86B1BDB57DB5E86BD2BC480D76FC52EE8BAA74)

Figure 53. File Access (Ceph Filesystem or CephFS)

CephFS is typically used for RWX claims but can also be used to support RWO claims.

***Object Storage, S3 and Swift (Ceph RADOS Gateway)\***

This access method offers support for the Amazon S3 and OpenStack Swift support on top of a Ceph cluster. The Openshift Container Storage Multi Cloud Gateway can leverage the RADOS Gateway to support Object Bucket Claims. From the Multi Cloud Gateway perspective the RADOS Gateway will be tagged as a compatible S3 endpoint.

![S3 and Swift Support](https://preview.cloud.189.cn/image/imageAction?param=3F0F9FB4B28E7F43B93988DBB071BC6F962BA4E3C5D3F0043CA717EE2C5CF7A25760CA4B41DE985211524CB5E9CF9FD217F7AB5F3576049A568CE43F4E8F1EB29E30B18921E65E292791D06479ECF76ED50816D7EDFB6D1138EBDAB76FC4CCB08531D1046135F5D377FB66C6B088E4A8)

Figure 54. Amazone S3 or OpenStack Swift (Ceph RADOS Gateway)

**CRUSH**

The Ceph cluster being a distributed architecture some solution had to be designed to provide an efficient way to distribute the data across the multiple OSDs in the cluster. The technique used is called CRUSH or Controlled Replication Under Scalable Hashing. With CRUSH, every object is assigned to one and only one hash bucket known as a Placement Group (PG).

![From Object to OSD](https://preview.cloud.189.cn/image/imageAction?param=3859B42431EF698BAA585C803C98DB05C6C3ECA090DD9B880226981F2756FA78FA52C7F6C85DCBF1EAB6EAC8779EC24F3E71553FE27A325E9B5CF3E0EFEF69C4D316BBBA39371D7637FA12CFE38AB9E0D70135BA61BA6E72C09255560BF1F13C0E6BE061770C19319BAAD6E0F116B89F)

CRUSH is the central point of configuration for the topology of the cluster. It offers a pseudo-random placement algorithm to distribute the objects across the PGs and uses rules to determine the mapping of the PGs to the OSDs. In essence, the PGs are an abstraction layer between the objects (application layer) and the OSDs (physical layer). In case of failure, the PGs will be remapped to different physical devices (OSDs) and eventually see their content resynchronized to match the protection rules selected by the storage administrator.

**Cluster Partitioning**

The Ceph OSDs will be in charge of the protection of the data as well as the constant checking of the integrity of the data stored in the entire cluster. The cluster will be separated into logical partitions, known as pools. Each pool has the following properties that can be adjusted:

- An ID (immutable)
- A name
- A number of PGs to distribute the objects across the OSDs
- A CRUSH rule to determine the mapping of the PGs for this pool
- A type of protection (Replication or Erasure Coding)
- Parameters associated with the type of protection
  - Number of copies for replicated pools
  - K and M chunks for Erasure Coding
- Various flags to influence the behavior of the cluster

**Pools and PGs**

![From Object to OSD](https://preview.cloud.189.cn/image/imageAction?param=25139337EEE98EF7716E899FA30A116E08CEA6D16297B86E6ADA5B0F323A11ADCD1E3F3CF9018C09FAC7EA981737933FEC7DAFCC16FEF020DC1BB6F77DCA6F2FE9DA457E08D35B2C75321F72312106F9EC4C83ED7CA0B7028BBCF4C3E4407423B1EDE9711990C20B97D2291F717B417B)

Figure 55. Pools and PGs

The diagram above shows the relationship end to end between the object at the access method level down to the OSDs at the physical layer.

| NOTE | A Ceph pool has no size and is able to consume the space available on any OSD where it’s PGs are created. A Placement Group or PG belongs to only one pool. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

**Data Protection**

Ceph supports two types of data protection presented in the diagram below.

![Replicated Pools vs Erasure Coded Pools](https://preview.cloud.189.cn/image/imageAction?param=9CD8D7B973303E235AE0E28126BF3BE2F0FBE4D2862731D9CE85B864678D99FF17ED02EB7D35545C783C35DC3F694DA2B5474CBD050C4EDB70D667E0A8B3ECE1BF33D0F6A1D2EB371521C02ED8072C4011095195F9C0739EA2365DFF69721108ADA4DA8DD7DAF6A4F6AE935858DA33F2)

Figure 56. Ceph Data Protection

Replicated pools provide better performance in almost all cases at the cost of a lower usable to raw storage ratio (1 usable byte is stored using 3 bytes of raw storage) while `Erasure Coding` provides a cost efficient way to store data with less performance. Red Hat supports the following `Erasure Coding` profiles with their corresponding usable to raw ratio:

- 4+2 (1:2 ratio)
- 8+3 (1:1.375 ratio)
- 8+4 (1:2 ratio)

Another advantage of `Erasure Coding` (EC) is its ability to offer extreme resilience and durability as we can configure the number of parities being used. EC can be used for the RADOS Gateway access method and for the RBD access method (performance impact).

**Data Distribution**

To leverage the Ceph architecture at its best, all access methods but librados, will access the data in the cluster through a collection of objects. Hence a 1GB block device will be a collection of objects, each supporting a set of device sectors. Therefore, a 1GB file is stored in a CephFS directory will be split into multiple objects. Also a 5GB S3 object stored through the RADOS Gateway via the Multi Cloud Gateway will be divided in multiple objects.

![RADOS Block Device Layout](https://preview.cloud.189.cn/image/imageAction?param=F8FB921A0B0E5DB0C264559CA8EEFDFC1740595CEC18E8F63087D3BB2360D4EEEE59221C1557BB004630CA80EA2C477DFBD39FAE8EC458DA959509B8EE1A161EC92B66D35E1FA1377DDAC9173829F8B142C36C876D451F6CA24D27053B54AA9924F11766869C1764949D47A1E3F521A4)

Figure 57. Data Distribution

| NOTE | By default, each access method uses an object size of 4MB. The above diagram details how a 32MB RBD (Block Device) supporting a RWO PVC will be scattered throughout the cluster. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

# OpenShift Log Aggregation

## OpenShift Log Aggregation

In this lab you will explore the logging aggregation capabilities of OpenShift.

An extremely important function of OpenShift is collecting and aggregating logs from the environments and the application pods it is running. OpenShift ships with an elastic log aggregation solution: **EFK**. (**E**lasticSearch, **F**luentd and **K**ibana)

The cluster logging components are based upon Elasticsearch, Fluentd, and Kibana (EFK). The collector, Fluentd, is deployed to each node in the OpenShift cluster. It collects all node and container logs and writes them to Elasticsearch (ES). Kibana is the centralized, web UI where users and administrators can create rich visualizations and dashboards with the aggregated data. Administrators can see and search through all logs. Application owners and developers can allow access to logs that belong to their projects. The EFK stack runs on top of OpenShift.

This lab requires that you have completed the infra-nodes lab. The logging stack will be installed on the `infra` nodes that were created in that lab.

More information may be found on the official OpenShift documentation site found here:

```
https://docs.openshift.com/container-platform/4.6/logging/cluster-logging.html
```

This exercise is done almost entirely using the OpenShift web console. All of the interactions with the web console are effectively creating or manipulating API objects in the background. It is possible to fully automate the process and/or do it using the CLI or other tools, but these methods are not covered in the exercise or documentation at this time.

### Deploying OpenShift Logging

OpenShift Container Platform cluster logging is designed to be used with the default configuration, which is tuned for small to medium sized OpenShift Container Platform clusters. The installation instructions that follow include a sample Cluster Logging Custom Resource (CR), which you can use to create a cluster logging instance and configure your cluster logging deployment.

If you want to use the default cluster logging install, you can use the sample CR directly.

If you want to customize your deployment, make changes to the sample CR as needed. The following describes the configurations you can make when installing your cluster logging instance or modify after installtion. See the Configuring sections for more information on working with each component, including modifications you can make outside of the Cluster Logging Custom Resource.

#### Create the `openshift-logging` namespace

OpenShift Logging will be run from within its own namespace `openshift-logging`. This namespace does not exist by default, and needs to be created before logging may be installed. The namespace is represented in yaml format as:

openshift_logging_namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-logging
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-logging: "true"
    openshift.io/cluster-monitoring: "true"
```

To create the namespace, run the following command:

```bash
oc create -f /opt/app-root/src/support/openshift_logging_namespace.yaml
```

#### Install the `Elasticsearch` and `Cluster Logging` Operators in the cluster

In order to install and configure the `EFK` stack into the cluster, additional operators need to be installed. These can be installed from the `Operator Hub` from within the cluster via the GUI.

When using operators in OpenShift, it is important to understand the basics of some of the underlying principles that make up the Operators. `CustomResourceDefinion (CRD)` and `CustomResource (CR)` are two Kubernetes objects that we will briefly describe.`CRDs` are generic pre-defined structures of data. The operator understands how to apply the data that is defined by the `CRD`. In terms of programming, `CRDs` can be thought as being similar to a class. `CustomResource (CR)` is an actual implementations of the `CRD`, where the structured data has actual values. These values are what the operator will use when configuring it’s service. Again, in programming terms, `CRs` would be similar to an instantiated object of the class.

The general pattern for using Operators is first, install the Operator, which will create the necessary `CRDs`. After the `CRDs` have been created, we can create the `CR` which will tell the operator how to act, what to install, and/or what to configure. For installing `openshift-logging`, we will follow this pattern.

To begin, log-in to the OpenShift Cluster’s GUI. `https://console-openshift-console.apps.odflab-55.container-contest.top:6443`

Then follow the following steps:

1. Install the Elasticsearch Operator:

   1. In the OpenShift console, click `Operators` → `OperatorHub`.

   2. Choose `Elasticsearch Operator` from the list of available Operators, and click `Install`.

   3. On the `Create Operator Subscription` page, **select Update Channel 4.6**, leave all other defaults and then click `Subscribe`.

      This makes the Operator available to all users and projects that use this OpenShift Container Platform cluster.

2. Install the Cluster Logging Operator:

   The `Cluster Logging` operator needs to be installed in the `openshift-logging` namespace. Please ensure that the `openshift-logging` namespace was created from the previous steps

   1. In the OpenShift console, click `Operators` → `OperatorHub`.
   2. Choose `Cluster Logging` from the list of available Operators, and click `Install`.
   3. On the `Create Operator Subscription` page, Under ensure `Installation Mode` that `A specific namespace on the cluster` is selected, and choose `openshift-logging`. In addition, **select Update Channel 4.6** and leave all other defaults and then click `Subscribe`.

3. Verify the operator installations:

   1. Switch to the `Operators` → `Installed Operators` page.

   2. Make sure the `openshift-logging` project is selected.

   3. In the *Status* column you should see green checks with either `InstallSucceeded` or `Copied` and the text *Up to date*.

      During installation an operator might display a `Failed` status. If the operator then installs with an `InstallSucceeded` message, you can safely ignore the `Failed` message.

4. Troubleshooting (optional)

   If either operator does not appear as installed, to troubleshoot further:

   - On the Copied tab of the Installed Operators page, if an operator show a Status of Copied, this indicates the installation is in process and is expected behavior.
   - Switch to the Catalog → Operator Management page and inspect the Operator Subscriptions and Install Plans tabs for any failure or errors under Status.
   - Switch to the Workloads → Pods page and check the logs in any Pods in the openshift-logging and openshift-operators projects that are reporting issues.

#### Create the Loggging `CustomResource (CR)` instance

Now that we have the operators installed, along with the `CRDs`, we can now kick off the logging install by creating a Logging `CR`. This will define how we want to install and configure logging.

1. In the OpenShift Console, switch to the the `Administration` → `Custom Resource Definitions` page.

2. On the `Custom Resource Definitions` page, click `ClusterLogging`.

3. On the `Custom Resource Definition Overview` page, select `View Instances` from the `Actions` menu.

   If you see a `404` error, don’t panic. While the operator installation succeeded, the operator itself has not finished installing and the `CustomResourceDefinition` may not have been created yet. Wait a few moments and then refresh the page.

4. On the `Cluster Loggings` page, click `Create Cluster Logging`.

   This step requires that you have completed the `Deploying and Managing OpenShift Container Storage` Module. If you have not completed the `OCS` module, you will need to substitute `storageClassName: ocs-storagecluster-ceph-rbd` with `storageClassName: gp2` in the `YAML` below before copying to the editor.

5. In the `YAML` editor, replace the code with the following:

openshift_logging_cr.yaml

```yaml
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance"
  namespace: "openshift-logging"
spec:
  managementState: "Managed"
  logStore:
    type: "elasticsearch"
    elasticsearch:
      nodeCount: 3
      storage:
         storageClassName: ocs-storagecluster-ceph-rbd
         size: 100Gi
      redundancyPolicy: "SingleRedundancy"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      resources:
        request:
          memory: 4G
  visualization:
    type: "kibana"
    kibana:
      replicas: 1
      nodeSelector:
        node-role.kubernetes.io/infra: ""
  curation:
    type: "curator"
    curator:
      schedule: "30 3 * * *"
      nodeSelector:
        node-role.kubernetes.io/infra: ""
  collection:
    logs:
      type: "fluentd"
      fluentd: {}
      nodeSelector:
        node-role.kubernetes.io/infra: ""
```

Then click `Create`.

#### Verify the Loggging install

Now that Logging has been created, let’s verify that things are working.

1. Switch to the `Workloads` → `Pods` page.
2. Select the `openshift-logging` project.

You should see pods for cluster logging (the operator itself), Elasticsearch, and Fluentd, and Kibana.

Alternatively, you can verify from the command line by using the following command:

```bash
oc get pods -n openshift-logging
```

You should eventually see something like:

```
NAME                                            READY   STATUS    RESTARTS   AGE
cluster-logging-operator-cb795f8dc-xkckc        1/1     Running   0          32m
elasticsearch-cdm-b3nqzchd-1-5c6797-67kfz       2/2     Running   0          14m
elasticsearch-cdm-b3nqzchd-2-6657f4-wtprv       2/2     Running   0          14m
elasticsearch-cdm-b3nqzchd-3-588c65-clg7g       2/2     Running   0          14m
fluentd-2c7dg                                   1/1     Running   0          14m
fluentd-9z7kk                                   1/1     Running   0          14m
fluentd-br7r2                                   1/1     Running   0          14m
fluentd-fn2sb                                   1/1     Running   0          14m
fluentd-pb2f8                                   1/1     Running   0          14m
fluentd-zqgqx                                   1/1     Running   0          14m
kibana-7fb4fd4cc9-bvt4p                         2/2     Running   0          14m
```

The *Fluentd* **Pods** are deployed as part of a **DaemonSet**, which is a mechanism to ensure that specific **Pods** run on specific **Nodes** in the cluster at all times:

```bash
oc get daemonset -n openshift-logging
```

You will see something like:

```
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
fluentd   9         9         9       9            9           kubernetes.io/os=linux   94s
```

You should expect 1 `fluentd` **Pod** for every **Node** in your cluster. Remember that **Masters** are still **Nodes** and `fluentd` will run there, too, to slurp the various logs.

You will also see the storage for ElasticSearch has been automatically provisioned. If you query the **PersistentVolumeClaim** objects in this project you will see the new storage.

```bash
oc get pvc -n openshift-logging
```

You will see something like:

```
NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS
MODES   STORAGECLASS                  AGE
elasticsearch-elasticsearch-cdm-ggzilasv-1   Bound    pvc-f3239564-389c-11ea-bab2-06ca7918708a   100Gi      RWO
        ocs-storagecluster-ceph-rbd   15m
elasticsearch-elasticsearch-cdm-ggzilasv-2   Bound    pvc-f324a252-389c-11ea-bab2-06ca7918708a   100Gi      RWO
        ocs-storagecluster-ceph-rbd   15m
elasticsearch-elasticsearch-cdm-ggzilasv-3   Bound    pvc-f326aa7d-389c-11ea-bab2-06ca7918708a   100Gi      RWO
        ocs-storagecluster-ceph-rbd   15m
```

Much like with the Metrics solution, we defined the appropriate `NodeSelector` in the Logging configuration (`CR`) to ensure that the Logging components only landed on the infra nodes. That being said, the `DaemonSet` ensures FluentD runs on **all** nodes. Otherwise we would not capture all of the container logs.

#### Accessing *Kibana*

As mentioned before, *Kibana* is the front end and the way that users and admins may access the OpenShift Logging stack. To reach the *Kibana* user interface, first determine its public access URL by querying the **Route** that got set up to expose Kibana’s **Service**:

To find and access the *Kibana* route:

1. In the OpenShift console, click on the `Networking` → `Routes` page.
2. Select the `openshift-logging` project.
3. Click on the `Kibana` route.
4. In the `Location` field, click on the URL presented.
5. Click through and accept the SSL certificates

Alternatively, this can be obtained from the command line:

```bash
oc get route -n openshift-logging
```

You will see something like:

```
NAME     HOST/PORT                                                           PATH   SERVICES   PORT    TERMINATION          WILDCARD
kibana   kibana-openshift-logging.apps.odflab-55.container-contest.top:6443          kibana     <all>   reencrypt/Redirect   None
```

Or, you can control+click the link:

[https://kibana-openshift-logging.apps.odflab-55.container-contest.top:6443](https://kibana-openshift-logging.apps.odflab-55.container-contest.top:6443/)

There is a special authentication proxy that is configured as part of the EFK installation that results in Kibana requiring OpenShift credentials for access.

Because you’ve already authenticated to the OpenShift Console as a cluster-admin user, you will see an administrative view of what Kibana has to show you (which you authorized by clicking the button).

#### Setting up Index Patterns

Once you open Kibana, before being able to view logs, we need to define an `index pattern` that will be used by Kibana to query ElasticSearch.

1. On the following screen, input `app*` as the index pattern, as shown below and hit `Next Step`.

   ![Kibana Index Pattern](https://preview.cloud.189.cn/image/imageAction?param=69DD9303E1722A1C260453F20D48FDA4087603FCEA7ED04D8623CBC6EB84528E67AAB36EAD3A4155E72BA398345CF08A6A69E8192D55F877A4899218FB34D508F5D39DD1A0F384D023E43441BF55921E3CEACF8BCF559A55974018E5DA3B9EAD21DB743A2D3EA495487A75343E9FDE37)

2. On the following screen, input `infra*` as the index pattern, as shown below and hit `Next Step`.

   ![Kibana Index Pattern](https://preview.cloud.189.cn/image/imageAction?param=226D767FFB6E08087F5DEC8FE3EEF41A394E9C52D4BAE86D86811E4EF6654D9ED0F5FC627D63313B9389F637CA1FECD08385DCB160F0EF68E29523B62C27F073CE58D855CC4A5730444F6E8E3437FC2FE12786610699022F186829042A0EF03A36308574719BDE108447632AC23E834D)

3. On the next screen, select `Timestamp` in the drop-down box, as shown below:

   ![Kibana Index Pattern](https://preview.cloud.189.cn/image/imageAction?param=E56E951264595832EBFE591A2ECB2D94A9F3A91171E75A366456DEAB857CED3AE7780B113A5DCA882B7A15E452D2021D5E532004033ED50EFEE89BB4E225BE364D5B11B9B1376287283569CA9B63D0062CDB6F6F756476748F543F1F57C88BFE8410532F74D6A044AE1669F9D6BD4D19)

4. Hit Create Index Pattern

#### Queries with *Kibana*

Once the *Kibana* web interface is up, we are now able to do queries. *Kibana* offers a the user a powerful interface to query all logs that come from the cluster.

By default, *Kibana* will show all logs that have been received within the the last 15 minutes. This time interval may be changed in the upper right hand corner. The log messages are shown in the middle of the page. All log messages that are received are indexed based on the log message content. Each message will have fields associated that are associated to that log message. To see the fields that make up an individual message, click on the arrow on the side of each message located in the center of the page. This will show the message fields that are contained.

First, set the default index pattern to `.all`. On the left hand side towards the top, in the drop down menu select the `.all` index pattern.

To select fields to show for messages, look on left hand side fore the `Available Fields` label. Below this are fields that can be selected and shown in the middle of the screen. Find the `hostname` field below the `Available Fields` and click `add`. Notice now, in the message pain, each message’s hostname is displayed. More fields may be added. Click the `add` button for `kubernetes.pod_name` and also for `message`.

To create a query for logs, the `Add a filter +` link right below the search box may be used. This will allow us to build queries using the fields of the messages. For example, if we wanted to see all log messages from the `openshift-logging` namespace, we can do the following:

1. Click on `Add a filter +`.
2. In the `Fields` input box, start typing `kubernetes.namespace_name`. Notice all of the available fields that we can use to build the query
3. Next, select `is`.
4. In the `Value` field, type in `openshift-logging`
5. Click the "Save" button

Now, in the center of the screen you will see all of the logs from all the pods in the `openshift-logging` namespace.

Of course, you may add more filters to refine the query.

One other neat option that Kibana allows you to do is save queries to use for later. To save a query do the following:

1. click on `Save` at the top of the screen.
2. Type in the name you would like to save it as. In this case, let’s type in `openshift-logging Namespace`

Once this has been saved, it can be used at a later time by hitting the `Open` button and selecting this query.

Please take time to explore the *Kibana* page and get experience by adding and doing more queries. This will be helpful when using a production cluster, you will be able to get the exact logs that you are looking for in a single place.

### Forwarding logs to external systems

In this section we will show you how to use a new feature in OpenShift 4.6 to ease forwarding logs to external log systems.

A new CustomResourceDefinition (CRD) named `ClusterLogForwarder` is used by the `Cluster Logging Operator` to create or modify internal Fluentd configmaps to forward logs to external (or internal) systems. Only one `ClusterLogForwarder` can exist in a cluster, and it combines all of the log forwarding rules.

Forwarding cluster logs to external third-party systems requires a combination of `outputs` and `pipelines` specified in a `ClusterLogForwarder` custom resource (CR) to send logs to specific endpoints inside and outside of your OpenShift Container Platform cluster. You can also use `inputs` to forward the application logs associated with a specific project to an endpoint. Let’s learn more about these concepts.

- An `output` is the destination for log data that you define, or where you want the logs sent. An output can be one of the following types:
  - `elasticsearch`: An external Elasticsearch v5.x or v6.x instance. The elasticsearch output can use a TLS connection.
  - `fluentdForward`: An external log aggregation solution that supports Fluentd. This option uses the Fluentd forward protocols. The fluentForward output can use a TCP or TLS connection and supports shared-key authentication by providing a shared_key field in a secret. Shared-key authentication can be used with or without TLS.
  - `syslog`: An external log aggregation solution that supports the syslog RFC3164 or RFC5424 protocols. The syslog output can use a UDP, TCP, or TLS connection.
  - `kafka`: A Kafka broker. The kafka output can use a TCP or TLS connection.
  - `default`: The internal OpenShift Container Platform Elasticsearch instance. You are not required to configure the default output. If you do configure a default output, you receive an error message because the default output is reserved for the Cluster Logging Operator.

If the output URL scheme requires TLS (HTTPS, TLS, or UDPS), then TLS server-side authentication is enabled. To also enable client authentication, the output must name a secret in the openshift-logging project. The secret must have keys of: tls.crt, tls.key, and ca-bundle.crt that point to the respective certificates that they represent.

- A `pipeline` defines simple routing from one log type to one or more outputs, or which logs you want to send. The log types are one of the following:
  - `application`: Container logs generated by user applications running in the cluster, except infrastructure container applications.
  - `infrastructure`: Container logs from pods that run in the openshift*, kube*, or default projects and journal logs sourced from node file system.
  - `audit`: Logs generated by the node audit system (auditd) and the audit logs from the Kubernetes API server and the OpenShift API server.

You can add labels to outbound log messages by using key:value pairs in the pipeline. For example, you might add a label to messages that are forwarded to others data centers or label the logs by type. Labels that are added to objects are also forwarded with the log message.

- An input forwards the application logs associated with a specific project to a pipeline.

More information may be found on the official OpenShift documentation site found here:

https://docs.openshift.com/container-platform/4.6/logging/cluster-logging-external.html

#### Sending logs to an external Syslog server

For the sake of simplification, we will emulate an external Syslog server by deploying a containerized Syslog server in a namespace called `external-logs`.

Since we also want to show how to separate application logs from infrastructure logs, we will deploy 2 'external' (containerized) Syslogs, one to receive forwarded application logs, and one to receive forwarded infrastructure logs.

First, let’s create a namespace called `external-logs` where we will deploy the Syslog server.

```bash
oc new-project external-logs
```

Now, let’s deploy the `Syslog` servers on that namespace. For that, we’ll be using a YAML file containing all the required resources:

```bash
oc create -f /opt/app-root/src/support/extlogs-syslog.yaml -n external-logs
```

Let’s check that everything is working fine, which can take a minute until the image is pulled for an external registry. When everything is OK, we should get an output similar to this:

```
NAME                              READY   STATUS    RESTARTS   AGE
syslog-ng-7cff7dd94-t5hh7         1/1     Running   0          1m
```

Now that our external Syslog server is available, let’s setup a log forwarding rule by creating a `ClusterLogForwarder`. First let’s look at the YAML file:

```YAML
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs: (1)
  - name: rsyslog-app
    syslog:
      facility: user
      payloadKey: message
      rfc: RFC3164
      severity: informational
    type: syslog (2)
    url: udp://syslog-ng.external-logs.svc:514 (3)
  - name: rsyslog-infra
    syslog:
      facility: user
      payloadKey: message
      rfc: RFC3164
      severity: informational
    type: syslog
    url: udp://syslog-ng-infra.external-logs.svc:514 (4)
  pipelines: (5)
  - inputRefs: (6)
    - application (7)
    labels:
      syslog: app
    name: syslog-app
    outputRefs:
    - rsyslog-app (8)
    - default
  - inputRefs:
    - infrastructure (8)
    labels:
      syslog: infra
    name: syslog-infra
    outputRefs:
    - rsyslog-infra (9)
    - default
```

In this YAML file, there are some notable fields:

```
The `outputs` (1) section defines all the remote log systems, in our case we have 2 Syslog servers:
```

- (3) This is the url for the one to store application-related logs. It is pointing to the service that is in the `external-logs` namespace.
- (4) This is the url one for infrastructure-related logs. It is pointing to the service that is in the `external-logs` namespace.

The `pipelines` section defines the sources and nature of logs that should be sent to the outputs defined before. `inputRefs` are used to describe the nature of the log to be sent, and as a reminder they can be either `application`, `infrastructure`, or `audit` for OpenShift audit logs (API access, etc).

We have 2 inputsRefs, (7) is for application logs and (8) is for infrastructure logs. Each inputRefs section contains an outRefs to tell where the logs should be sent, referring the outputs (1) defined in the beginning of the spec section.

Now let’s create the ClusterLogForwareder resource using the YAML file:

```bash
oc create -f /opt/app-root/src/support/extlogs-clusterlogforwarder.yaml
```

Once the CR is created, the Cluster Logging Operator redeploys the Fluentd pods. If the pods do not redeploy, you can delete the Fluentd pods to force them to redeploy.

```bash
oc delete pod --selector logging-infra=fluentd -n openshift-logging
```

Let’s check that all the Fluentd pods are now in Running state:

```bash
oc get pod --selector logging-infra=fluentd -n openshift-logging
```

You should see something like this in the output:

```
NAME            READY   STATUS        RESTARTS   AGE
fluentd-274bx   1/1     Running       0          1m
fluentd-5bqx8   1/1     Running       0          1m
fluentd-67qld   1/1     Running       0          1m
fluentd-m9v2f   1/1     Running       0          1m
fluentd-vs8bb   1/1     Running       0          1m
fluentd-x98gk   1/1     Running       0          1m
```

Now let’s check that the logs are being forwarded to the 2 Syslog servers. The Syslog server stores it’s logs in the /var/log/messages file within the container, so we need to check it’s content by doing an oc exec into the container. `Make sure you put the correct pod name in the command`:

We will be using the OpenShift Console Terminal to access the pod and check the `/var/log/messages` content.

1. Open the Administrator View and go to `workloads→Pods`

   ![Syslog Pods](https://preview.cloud.189.cn/image/imageAction?param=8301D6E1781B1EF09437092F1CFC71927EE978A4A78081F2D8DB390F3A6540B74D252769A18B032D4DEB97303593C212867FE11D51C8E412D0939DEA62769BB4108ED930FB75A8E3D0C6BE8A57DB2C76018947D1E59236F167F0FD4F10CAB0DC247DF0D06E2FC5DD69D4C95D640152F2)

2. Click on the syslog infrastructure pod, which name looks like `syslog-ng-infra-xyz`, and go the the `Terminal` tab

   ![Syslog Terminal](https://preview.cloud.189.cn/image/imageAction?param=B4157120F1B395E8B052DDA4E747B8EACB98626A83CD89FE74C4395E20B4BA3B8FEB793F3B82BE0F8668E0BAE59BF7011AB42690B4D12354D5BCB13A606723C66AC2C22E1F4710C921BB89FCF45B9E513AAA3AA325C6D869B571A3C8BE09F6763C554C06E1C0677932F428B5450DFFCB)

3. In the Terminal box, enter this command: `tail -f /var/log/messages`. The forwarded logs should then appear in the terminal.

   ![Syslog logs](https://preview.cloud.189.cn/image/imageAction?param=6CE6C182FCE18D8DDA734B662E4F2ADB2C230461677833DA173B6FC97D6A3E90EF210EC882F23ECBC9C1A885290B3DE0583A7963B606CC8267751D9589033E28874071E0994A316207C14252B8C2E11512CC79EF828968E83339AFCC619DA29E7D8E9571EF9FC7E505D8C9A8D26F3929)

   And voilà! You can repeat this procedure with the other pod to check that the application logs are correctly forwarded too.

   # External (LDAP) Authentication Providers, Users, and Groups

   ## Configuring External Authentication Providers

   OpenShift supports a number of different authentication providers, and you can find the complete list in the [understanding identity provider configuration](https://docs.openshift.com/container-platform/4.6/authentication/understanding-identity-provider.html). One of the most commonly used authentication providers is LDAP, whether provided by Microsoft Active Directory or by other sources.

   OpenShift can perform user authentication against an LDAP server, and can also configure group membership and certain RBAC attributes based on LDAP group membership.

   ### Background: LDAP Structure

   In this environment we are providing LDAP with the following user groups:

   - `ose-user`: Users with OpenShift access
     - Any users who should be able to log-in to OpenShift must be members of this group
     - All of the below mentioned users are in this group
   - `ose-normal-dev`: Normal OpenShift users
     - Regular users of OpenShift without special permissions
     - Contains: `normaluser1`, `teamuser1`, `teamuser2`
   - `ose-fancy-dev`: Fancy OpenShift users
     - Users of OpenShift that are granted some special privileges
     - Contains: `fancyuser1`, `fancyuser2`
   - `ose-teamed-app`: Teamed app users
     - A group of users that will have access to the same OpenShift **Project**
     - Contains: `teamuser1`, `teamuser2`

   #### Examine the OAuth configuration

   Since this is a pure, vanilla OpenShift 4 installation, it has the default OAuth resource. You can examine that OAuth configuration with the following:

   ```bash
   oc get oauth cluster -o yaml
   ```

   You will see something like:

   ```yaml
   apiVersion: config.openshift.io/v1
   kind: OAuth
   metadata:
     annotations:
       release.openshift.io/create-only: "true"
     creationTimestamp: "2021-01-04T18:09:16Z"
     generation: 1
     managedFields:
     - apiVersion: config.openshift.io/v1
       fieldsType: FieldsV1
       fieldsV1:
         f:metadata:
           f:annotations:
             .: {}
             f:release.openshift.io/create-only: {}
         f:spec: {}
       manager: cluster-version-operator
       operation: Update
       time: "2021-01-04T18:09:16Z"
     name: cluster
     resourceVersion: "1807"
     selfLink: /apis/config.openshift.io/v1/oauths/cluster
     uid: 8090316c-209e-425c-9ed6-39930337f557
   spec: {}
   ```

   There are a few things to note here. Firstly, there’s basically nothing here! How does the `kubeadmin` user work, then? The OpenShift OAuth system knows to look for a `kubeadmin` **Secret** in the `kube-system` **Namespace**. You can examine it with the following:

   ```bash
   oc get secret -n kube-system kubeadmin -o yaml
   ```

   You will see something like:

   ```yaml
   apiVersion: v1
   data:
     kubeadmin: JDJhJDEwJDNyLjhRNUdzdzExSFFyckFMeEF5NU9RT0hILzZYSEVpNHIxYURpNks3WVZPLnlxRGVnaHpx
   kind: Secret
   metadata:
     creationTimestamp: "2021-01-04T18:08:15Z"
     managedFields:
     - apiVersion: v1
       fieldsType: FieldsV1
       fieldsV1:
         f:data:
           .: {}
           f:kubeadmin: {}
         f:type: {}
       manager: cluster-bootstrap
       operation: Update
       time: "2021-01-04T18:08:15Z"
     name: kubeadmin
     namespace: kube-system
     resourceVersion: "98"
     selfLink: /api/v1/namespaces/kube-system/secrets/kubeadmin
     uid: b5177429-c0db-443e-bbed-8d0912779a96
   type: Opaque
   ```

   That **Secret** contains the encoded hash of the `kubeadmin` password. This account will continue to work even after we configure a new `OAuth`. If you want to disable it, you would need to delete the secret.

   In a real-world environment, you will likely want to integrate with your existing identity management solution. For this lab we are configuring LDAP as our `identityProvider`. Here’s an example of the OAuth configuration. Look for the element in `identityProviders` with `type: LDAP` like the following:

   ```yaml
   apiVersion: config.openshift.io/v1
   kind: OAuth
   metadata:
     name: cluster
   spec:
     identityProviders:
     - name: ldap (1)
       challenge: false
       login: true
       mappingMethod: claim (2)
       type: LDAP
       ldap:
         attributes: (3)
           id:
           - dn
           email:
           - mail
           name:
           - cn
           preferredUsername:
           - uid
         bindDN: "uid=openshiftworkshop,ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com" (4)
         bindPassword: (5)
           name: ldap-secret
         ca: (6)
           name: ca-config-map
         insecure: false
         url: "ldaps://ldap.jumpcloud.com/ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com?uid?sub?(memberOf=cn=ose-user,ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com)" (7)
     tokenConfig:
       accessTokenMaxAgeSeconds: 86400
   ```

   Some notable fields under `identityProviders:`:

   1. `name`: The unique ID of the identity provider. It is possible to have multiple authentication providers in an OpenShift environment, and OpenShift is able to distinguish between them.
   2. `mappingMethod: claim`: This section has to do with how usernames are assigned within an OpenShift cluster when multiple providers are configured. See the [Identity provider parameters](https://docs.openshift.com/container-platform/4.6/authentication/understanding-identity-provider.html#identity-provider-parameters_understanding-identity-provider) section for more information.
   3. `attributes`: This section defines the LDAP fields to iterate over and assign to the fields in the OpenShift user’s "account". If any attributes are not found / not populated when searching through the list, the entire authentication fails. In this case we are creating an identity that is associated with the LDAP `dn`, an email address from the LDAP `mail`, a name from the LDAP `cn`, and a username from the LDAP `uid`.
   4. `bindDN`: When searching LDAP, bind to the server as this user.
   5. `bindPassword`: Reference to the Secret that has the password to use when binding for searching.
   6. `ca`: Reference to the ConfigMap that contains the CA certificate to use for validating the SSL certificate of the LDAP server.
   7. `url`: Identifies the LDAP server and the search to perform.

   For more information on the specific details of LDAP authentication in OpenShift you can refer to the [Configuring an LDAP identity provider](https://docs.openshift.com/container-platform/4.6/authentication/identity_providers/configuring-ldap-identity-provider.html) documentation.

   To setup the LDAP identity provider we must:

   1. Create a `Secret` with the bind password.
   2. Create a `ConfigMap` with the CA certificate.
   3. Update the `cluster` `OAuth` object with the LDAP identity provider.

   As the `kubeadmin` user apply the OAuth configuration with `oc`.

   ```bash
   oc create secret generic ldap-secret --from-literal=bindPassword=b1ndP^ssword -n openshift-config
   wget https://ssl-ccp.godaddy.com/repository/gd-class2-root.crt -O /opt/app-root/src/support/ca.crt
   oc create configmap ca-config-map --from-file=/opt/app-root/src/support/ca.crt -n openshift-config
   oc apply -f /opt/app-root/src/support/oauth-cluster.yaml
   ```

   We use `apply` because there is an existing `OAuth` object. If you used `create` you would get an outright error that the object already exists. You still get a warning, but that’s OK.

   #### Syncing LDAP Groups to OpenShift Groups

   In OpenShift, groups can be used to manage users and control permissions for multiple users at once. There is a section in the documentation on how to [sync groups with LDAP](https://docs.openshift.com/container-platform/4.6/authentication/ldap-syncing.html). Syncing groups involves running a program called `groupsync` when logged into OpenShift as a user with `cluster-admin` privileges, and using a configuration file that tells OpenShift what to do with the users it finds in the various groups.

   We have provided a `groupsync` configuration file for you:

   View configuration file

   ```bash
   cat /opt/app-root/src/support/groupsync.yaml
   ```

   Without going into too much detail (you can look at the documentation), the `groupsync` config file does the following:

   - searches LDAP using the specified bind user and password
   - queries for any LDAP groups whose name begins with `ose-`
   - creates OpenShift groups with a name from the `cn` of the LDAP group
   - finds the members of the LDAP group and then puts them into the created OpenShift group
   - uses the `dn` and `uid` as the UID and name attributes, respectively, in OpenShift

   Execute the `groupsync`:

   ```bash
   oc adm groups sync --sync-config=/opt/app-root/src/support/groupsync.yaml --confirm
   ```

   You will see output like the following:

   ```
   group/ose-fancy-dev
   group/ose-user
   group/ose-normal-dev
   group/ose-teamed-app
   ```

   What you are seeing is the **Group** objects that have been created by the `groupsync` command. If you are curious about the `--confirm` flag, check the output of the help with `oc adm groups sync -h`.

   If you want to see the **Groups** that were created, execute the following:

   ```bash
   oc get groups
   ```

   You will see output like the following:

   ```
   NAME             USERS
   ose-fancy-dev    fancyuser1, fancyuser2
   ose-normal-dev   normaluser1, teamuser1, teamuser2
   ose-teamed-app   teamuser1, teamuser2
   ose-user         fancyuser1, fancyuser2, normaluser1, teamuser1, teamuser2
   ```

   Take a look at a specific group in YAML:

   ```bash
   oc get group ose-fancy-dev -o yaml
   ```

   The YAML looks like:

   ```yaml
   apiVersion: user.openshift.io/v1
   kind: Group
   metadata:
     annotations:
       openshift.io/ldap.sync-time: "2021-01-04T21:43:40Z"
       openshift.io/ldap.uid: cn=ose-fancy-dev,ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=co
   m
       openshift.io/ldap.url: ldap.jumpcloud.com:636
     creationTimestamp: "2021-01-04T21:43:40Z"
     labels:
       openshift.io/ldap.host: ldap.jumpcloud.com
     name: ose-fancy-dev
     resourceVersion: "80943"
     selfLink: /apis/user.openshift.io/v1/groups/ose-fancy-dev
     uid: 652f296a-b2ea-41ca-9b1e-5cfd98389438
   users:
   - fancyuser1
   - fancyuser2
   ```

   OpenShift has automatically associated some LDAP metadata with the **Group**, and has listed the users who are in the group.

   What happens if you list the **Users**?

   ```bash
   oc get user
   ```

   You will get:

   ```
   No resources found.
   ```

   Why would there be no **Users** found? They are clearly listed in the **Group** definition.

   **Users** are not actually created until the first time they try to log in. What you are seeing in the **Group** definition is simply a placeholder telling OpenShift that, if it encounters a **User** with that specific ID, that it should be associated with the **Group**.

   #### Change Group Policy

   In your environment, there is a special group of super developers called *ose-fancy-dev* who should have special `cluster-reader` privileges. This is a role that allows a user to view administrative-level information about the cluster. For example, they can see the list of all **Projects** in the cluster.

   Change the policy for the `ose-fancy-dev` **Group**:

   ```bash
   oc adm policy add-cluster-role-to-group cluster-reader ose-fancy-dev
   ```

   If you are interested in the different roles that come with OpenShift, you can learn more about them in the [role-based access control (RBAC)](https://docs.openshift.com/container-platform/4.6/authentication/using-rbac.html) documentation.

   #### Examine `cluster-reader` policy

   Go ahead and login as a regular user:

   ```bash
   oc login -u normaluser1 -p Op#nSh1ft
   ```

   Then, try to list **Projects**:

   ```bash
   oc get projects
   ```

   You will see:

   ```
   No resources found.
   ```

   Now, login as a member of `ose-fancy-dev`:

   ```bash
   oc login -u fancyuser1 -p Op#nSh1ft
   ```

   And then perform the same `oc get projects` command:

   ```bash
   oc get projects
   ```

   You will now see the list of all of the projects in the cluster:

   ```
   NAME                                                    DISPLAY NAME                        STATUS
       app-management
     * default
       kube-public
       kube-system
       labguide
       openshift
       openshift-apiserver
   ...
   ```

   You should now be starting to understand how RBAC in OpenShift Container Platform can work.

   #### Create Projects for Collaboration

   Make sure you login as the cluster administrator:

   ```bash
   oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
   ```

   Then, create several **Projects** for people to collaborate:

   ```bash
   oc adm new-project app-dev --display-name="Application Development"
   oc adm new-project app-test --display-name="Application Testing"
   oc adm new-project app-prod --display-name="Application Production"
   ```

   You have now created several **Projects** that represent a typical Software Development Lifecycle setup. Next, you will configure **Groups** to grant collaborative access to these projects.

   Creating projects with `oc adm new-project` does **not** use the project request process or the project request template. These projects will not have quotas or limitranges applied by default. A cluster administrator can "impersonate" other users, so there are several options if you wanted these projects to get quotas/limit ranges:

   1. use `--as` to specify impersonating a regular user with `oc new-project`
   2. use `oc process` and provide values for the project request template, piping into create (eg: `oc process … | oc create -f -`). This will create all of the objects in the project request template, which would include the quota and limit range.
   3. manually create/define the quota and limit ranges after creating the projects.

   For these exercises it is not important to have quotas or limit ranges on these projects.

   #### Map Groups to Projects

   As you saw earlier, there are several roles within OpenShift that are preconfigured. When it comes to **Projects**, you similarly can grant view, edit, or administrative access. Let’s give our `ose-teamed-app` users access to edit the development and testing projects:

   ```bash
   oc adm policy add-role-to-group edit ose-teamed-app -n app-dev
   oc adm policy add-role-to-group edit ose-teamed-app -n app-test
   ```

   And then give them access to view production:

   ```bash
   oc adm policy add-role-to-group view ose-teamed-app -n app-prod
   ```

   Now, give the `ose-fancy-dev` group edit access to the production project:

   ```bash
   oc adm policy add-role-to-group edit ose-fancy-dev -n app-prod
   ```

   #### Examine Group Access

   Log in as `normaluser1` and see what **Projects** you can see:

   ```bash
   oc login -u normaluser1 -p Op#nSh1ft
   oc get projects
   ```

   You should get:

   ```
   No resources found.
   ```

   Then, try `teamuser1` from the `ose-teamed-app` group:

   ```bash
   oc login -u teamuser1 -p Op#nSh1ft
   oc get projects
   ```

   You should get:

   ```
   NAME       DISPLAY NAME              STATUS
   app-dev    Application Development   Active
   app-prod   Application Production    Active
   app-test   Application Testing       Active
   ```

   You did not grant the team users edit access to the production project. Go ahead and try to create something in the production project as `teamuser1`:

   ```bash
   oc project app-prod
   oc new-app docker.io/siamaksade/mapit
   ```

   You will see that it will not work:

   ```
   error: can't lookup images: imagestreamimports.image.openshift.io is forbidden: User "teamuser1" cannot create resource "imagestreamimports" in API group "image.openshift.io" in the namespace "app-prod"
   error:  local file access failed with: stat docker.io/siamaksade/mapit: no such file or directory
   error: unable to locate any images in image streams, templates loaded in accessible projects, template files, local docker images with name "docker.io/siamaksade/mapit"
   
   Argument 'docker.io/siamaksade/mapit' was classified as an image, image~source, or loaded template reference.
   
   The 'oc new-app' command will match arguments to the following types:
   
     1. Images tagged into image streams in the current project or the 'openshift' project
        - if you don't specify a tag, we'll add ':latest'
     2. Images in the Docker Hub, on remote registries, or on the local Docker engine
     3. Templates in the current project or the 'openshift' project
     4. Git repository URLs or local paths that point to Git repositories
   
   --allow-missing-images can be used to point to an image that does not exist yet.
   
   See 'oc new-app -h' for examples.
   ```

   This failure is exactly what we wanted to see.

   #### Prometheus

   Now that you have a user with `cluster-reader` privileges (one that can see many administrative aspects of the cluster), we can revisit Prometheus and attempt to log-in to it.

   Login as a the user with `cluster-reader` privileges:

   ```bash
   oc login -u fancyuser1 -p Op#nSh1ft
   ```

   Find the `prometheus` `Route` with the following command:

   ```bash
   oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}{"\n"}'
   ```

   You will see something like the following:

   ```
   prometheus-k8s-openshift-monitoring.apps.odflab-55.container-contest.top:6443
   ```

   Before continuing, make sure to go to the OpenShift web console and log out by using the dropdown menu at the upper right where it says `kube:admin`. Otherwise Prometheus will try to use your `kubeadmin` user to pass through authentication. While it will work, it doesn’t demonstrate the `cluster-reader` role.

   The installer configured a `Route` for Prometheus by default. Go ahead and control+click the [Prometheus link](https://prometheus-k8s-openshift-monitoring.apps.odflab-55.container-contest.top:6443/) to open it in your browser. You’ll be greeted with a login screen. Click the **Log in with OpenShift** button, then select the `ldap` auth mechanism, and use the `fancyuser1` user that you gave `cluster-reader` privileges to earlier.

   More specifically, the `ose-fancy-dev` group has `cluster-reader` permissions, and `fancyuser1` is a member. Remember that the password for all of these users is `Op#nSh1ft`.

   You might get a certificate error because of a self-signed certificate (depending on how the cluster was installed). Make sure to accept it.

   After logging in, the first time you will be presented with an auth proxy permissions acknowledgement:

   ![prometheus auth proxy](https://preview.cloud.189.cn/image/imageAction?param=3BE7861B08079D29D5D3BD6CA8768DEC547BC5CFF06A5B4AF072FCB5A5A66E624F8BACF9360AF64902A88FEE03BD84EE8BF169BDFFDF76176F3B5ED762B694CCF1A4EA3CA30DBACD94186DF359846A1F9F2125DF718CEA958F4F619532799EC30C9CE54EF43481985905B8C71ADC7F0C)

   Figure 1. Auth Proxy Acceptance.

   There is actually an OAuth proxy that sits in the flow between you and the Prometheus container. This proxy is used to validate your AuthenticatioN (AuthN) as well as authorize (AuthZ) what is allowed to happen. Here you are explicitly authorizing the permissions from your `fancyuser1` account to be used as part of accessing Prometheus. Hit *Allow selected permissions*.

   At this point you are viewing Prometheus. There are no alerts configured. If you look at `Status` and then `Targets` you can see some interesting information about the current state of the cluster.

   After you are done, make sure to login again as the admin user:

   ```bash
   oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
   ```

# OpenShift Monitoring with Prometheus

## OpenShift Monitoring

In this lab you will explore various aspects of the builtin OpenShift Monitoring. This includes an overview of the OpenShift Alertmanager UI, accessing the Prometheus web console, running PromQL (Prometheuses Query Language) queries to inspect the cluster and finally looking at Grafana dashboards.

### OpenShift Monitoring

OpenShift Container Platform includes a pre-configured, pre-installed, and self-updating monitoring stack that is based on the Prometheus open source project and its wider eco-system. It provides monitoring of cluster components and includes a set of alerts to immediately notify the cluster administrator about any occurring problems and a set of Grafana dashboards. The cluster monitoring stack is only supported for monitoring OpenShift Container Platform clusters.

#### Examine Alerting Configuration

1. Login to the [OpenShift Web Console](https://console-openshift-console.apps.odflab-55.container-contest.top:6443/) with the kubeadmin credentials.

   ```
   kubeadmin
   ```

   ```
   VyYA4-KWyRT-9bFCf-2Frvz
   ```

   You will receive a self-signed certificate error in your browser when you first visit the OpenShift Web console. When OpenShift is installed, by default, a CA and SSL certificates are generated for all inter-component communication within OpenShift, including the web console.

2. On the left hand side, click on the "Monitoring" drop down.

3. Click on "Alerting". This is the OpenShift console’s view of the alert configuration.

The Alerting tab shows you information about currently configured and/or active alerts. You can see and do several things:

1. Filter alerts by their names.
2. Filter the alerts by their states. To fire, some alerts need a certain condition to be true for the duration of a timeout. If a condition of an alert is currently true, but the timeout has not been reached, such an alert is in the Pending state.
3. Alert name.
4. Description of an alert.
5. Current state of the alert and when the alert went into this state.
6. Value of the Severity label of the alert.
7. Actions you can do with the alert.

You’ll also note that you can click directly into the Alertmanager UI with a link that’s at the top of this page.

##### Metrics UI (Prometheus Console)

In addition to the Alerting screen, OpenShift’s built-in monitoring provides an interface to access metrics collected by Prometheus using the [Prometheus Query Language (PromQL)](https://prometheus.io/docs/prometheus/latest/querying/basics/).

1. On the left hand side of the OpenShift Console, under the "Monitoring" section, click the link for "Metrics".

##### Running Prometheus Queries

Let’s run a query to see the resources memory limit all pod definitions have defined.

1. Copy and paste the following query into the expression text box:

   ```
   sum(kube_pod_container_resource_limits_memory_bytes)/(1024^3)
   ```

2. Click the "Run Queries" button

3. You should now see a timeseries with a value in the list. This value is the latest gathered value for the timeseries as defined by this query. You will also see a graph showing the value over the last time period (defaults to 30m).

Now let’s run a query to see the cpu usage for the entire cluster.

1. Click "Add Query"

2. Copy and paste the following query into the query text box:

   ```
   cluster:cpu_usage_cores:sum
   ```

3. Click the "Run Queries" button

4. You should now see a timeseries with a value in the list. This value is the latest gathered value for the timeseries as defined by this query. You’ll also see that your new query got plotted on the same graph.

The metrics interface lets you run powerful queries on the information collected about your cluster.

You’ll also note that you can click directly into the Prometheus UI with a link that’s at the top of this page.

##### Dashboards UI (Grafana)

In addition to the Metrics UI, OpenShift monitoring provides a preconfigured "Dashboards" UI (aka "Grafana"). The purpose of these Dashboards is to show multiple metrics in easy to consume form graphed over time.

1. Click the "Dashboards" link under the "Monitoring" section on the left hand side.

   You will receive a self-signed certificate error in your browser when you first visit the Dashboards UI console. When OpenShift is installed, by default, a CA and SSL certificates are generated for all inter-component communication within OpenShift, including the Grafana dashboard.

2. Click "Log in with OpenShift" button.

3. Click "kube:admin" if the option is provided.

   ```
   kubeadmin
   ```

   ```
   VyYA4-KWyRT-9bFCf-2Frvz
   ```

4. On the "Authorize Access" page, click the "Allow selected permissions" button.

5. This is the Grafana "Home Dashboard" page.

6. Click on the "Home" dropdown at the top of the page

7. Select the "Kubernetes / Compute Resources / Cluster" dashboard. The "Kubernetes / Compute Resources / Cluster" dashbard shows an overview of the resources currently committed and actively being used by the cluster.

   ![grafana dashboard](https://preview.cloud.189.cn/image/imageAction?param=456F9096A34E5FECC549A9586226169B40981BDB341F6F7C771844267191AF84B13E45C6595D5A2CEC35C9AA4A470413EF5570454BF54C5CA4927738FE96BB144589556321CDEE9814805BEC14F4C5E37086FC9EAE9CBEC372A3FF65F3A864241FCBF82110DCD7761A18B055DA7400BC)

   Figure 1. Dashboards UI (Grafana)

# Project Template, Quota, and Limits

## Project Request Template, Quotas, Limits

In the Application Management Basics lab, you dealt with the fundamental building blocks of OpenShift, all contained within a **Project**.

Out of the box, OpenShift does not restrict the quantity of objects or amount of resources (eg: CPU, memory) that they can consume within a **Project**. Further, users may create an unlimited number of **Projects**. In a world with no constraints, or in a POC-type environment, that would be fine. But reality calls for a little restraint.

### Background: Project Request Template

When users invoke the `oc new-project` command, a new project request flow is initiated. During part of this this workflow, a default project **Template** is processed to create the newly requested project.

#### View the Default Project Request Template

To view the built-in default **Project Request Template**, execute the following command:

```bash
oc adm create-bootstrap-project-template -o yaml
```

Inspecting the the YAML output of this command, you will notice that there are various parameters associated with this **Template** displayed towards the bottom.

```bash
...
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
...
```

Next, let’s take a look at the help output for the `oc new-project` command:

```bash
oc new-project -h
```

Notice there is a `--display-name` directive on `oc new-project`? This option corresponds to the `PROJECT_DISPLAYNAME` parameter we saw in the output of the default **Template**.

The `new-project` workflow involves the user providing information to fulfill the project request. OpenShift decides if this request should be allowed (for example, are users allowed to create **Projects**? Does this user have too many **Projects**?) and, if so, processes the **Template**.

If you look at the objects defined in the **Template**, you will notice that there’s no mention of quota or limits. You’ll change that now.

**Templates** are a powerful tool that enable the user to easily create reusable sets of OpenShift objects with powerful parameterization. They can be used to quickly deploy more complicated and related components into OpenShift, and can be a useful part of your organization’s software development lifecycle (SDLC). More information can be found in the [template documentation](https://docs.openshift.com/container-platform/4.6/openshift_images/using-templates.html). We will not go into more detail on **Templates** in these exercises.

#### Modify the Project Request Template

You won’t actually have to make template changes in this lab — we’ve made them for you already. Use `cat`, `less`, or your favorite editor to view the modified **Project Request Template**:

```bash
cat /opt/app-root/src/support/project_request_template.yaml
```

Please take note that there are two new sections added: **ResourceQuota** and **LimitRange**.

### Background: ResourceQuota

The [quota documentation](https://docs.openshift.com/container-platform/4.6/applications/quotas/quotas-setting-per-project.html) provides a great description of **ResourceQuota**:

```
A resource quota, defined by a ResourceQuota object, provides constraints
that limit aggregate resource consumption per project. It can limit the
quantity of objects that can be created in a project by type, as well as
the total amount of compute resources and storage that may be consumed
by resources in that project.
```

In our case, we are setting a specific set of quota for CPU, memory, storage, volume claims, and **Pods**. Take a look at the `ResourceQuota` section from the `project_request_template.yaml` file:

```yaml
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: ${PROJECT_NAME}-quota (1)
  spec:
    hard:
      pods: 10 (2)
      requests.cpu: 4000m (3)
      requests.memory: 8Gi (4)
      resourcequotas: 1
      requests.storage: 50Gi (5)
      persistentvolumeclaims: 5 (6)
```

1. While only one quota can be defined in a **Project**, it still needs a unique name/id.
2. The total number of pods in a non-terminal state that can exist in the project.
3. CPUs are measured in "milicores". More information on how Kubernetes/OpenShift calculates cores can be found in the [upstream documentation](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/).
4. There is a system of both `limits` and `requests` that we will discuss more when we get to the **LimitRange** object.
5. Across all persistent volume claims in a project, the sum of storage requested cannot exceed this value.
6. The total number of persistent volume claims in a project.

For more details on the available quota options, refer back to the [quota documentation](https://docs.openshift.com/container-platform/4.6/applications/quotas/quotas-setting-per-project.html)

### Background: LimitRange

The [limit range documentation](https://docs.openshift.com/container-platform/4.6/nodes/clusters/nodes-cluster-limit-ranges.html) provides some good background:

```
A limit range, defined by a LimitRange object, enumerates compute resource
constraints in a project at the pod, container, image, image stream, and
persistent volume claim level, and specifies the amount of resources that a pod,
container, image, image stream, or persistent volume claim can consume.
```

While the quota sets an upper bound on the total resource consumption within a project, the `LimitRange` generally applies to individual resources. For example, setting how much CPU an individual **Pod** or container can use.

Take a look at the sample `LimitRange` definition that we have provided in the `project_request_template.yaml` file:

```yaml
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: ${PROJECT_NAME}-limits
    creationTimestamp: null
  spec:
    limits:
      -
        type: Container
        max: (1)
          cpu: 4000m
          memory: 1024Mi
        min: (2)
          cpu: 10m
          memory: 5Mi
        default: (3)
          cpu: 4000m
          memory: 1024Mi
        defaultRequest: (4)
          cpu: 100m
          memory: 512Mi
```

The difference between requests and default limits is important, and is covered in the [limit range documentation](https://docs.openshift.com/container-platform/4.6/nodes/clusters/nodes-cluster-limit-ranges.html). But, generally speaking:

1. `max` is the highest value that may be specified for limits and requests
2. `min` is the lowest value that may be specified for limits and requests
3. `default` is the maximum amount (limit) that the container may consume, when nothing is specified
4. `defaultRequest` is the minimum amount that the container may consume, when nothing is specified

In addition to these topics, there are things like **Quality of Service Tiers** as well as a **Limit** to **Request** ratio. There is additionally more information in the [limit range documentation](https://docs.openshift.com/container-platform/4.6/nodes/clusters/nodes-cluster-limit-ranges.html) section of the documentation.

For the sake of brevity, suffice it to say that there is a complex and powerful system of Quality of Service and resource management in OpenShift. Understanding the types of workloads that will be run in your cluster will be important to coming up with sensible values for all of these settings.

The settings we provide for you in these examples generally restrict projects to:

- A total CPU quota of 4 cores (`4000m`) where
  - Individual containers
    - must use 4 cores or less
    - cannot be defined with less than 10 milicores
    - will default to a request of 100 milicores (if not specified)
    - may burst up to a limit of 4 cores (if not specified)
- A total memory usage of 8 Gibibyte (8192 Megabytes) where
  - Individual containers
    - must use 1 Gi or less
    - cannot be defined with less than 5 Mi
    - will default to a request of 512 Mi
    - may burst up to a limit of 1024 Mi
- Total storage claims of 25 Gi or less
- A total number of 5 volume claims
- 10 or less **Pods**

In combination with quota, you can create very fine-grained controls, even across projects, for how users are allowed to request and utilize OpenShift’s various resources.

Remember that quotas and limits are applied at the **Project** level. **Users** may have access to multiple **Projects**, but quotas and limits do not apply directly to **Users**. If you want to apply one quota across multiple **Projects**, then you should look at the [multi-project quota](https://docs.openshift.com/container-platform/4.6/applications/quotas/quotas-setting-across-multiple-projects.html) documentation. We will not cover multi-project quota in these exercises.

### Installing the Project Request Template

With this background in place, let’s go ahead and actually tell OpenShift to use this new **Project Request Template**.

#### Create the Template

As we discussed earlier, a **Template** is just another type of OpenShift object. The `oc` command provides a `create` function that will take YAML/JSON as input and simply instantiate the objects provided.

Go ahead and execute the following:

```bash
oc create -f /opt/app-root/src/support/project_request_template.yaml -n openshift-config
```

This will create the **Template** object in the `openshift-config` **Project**. You can now see the **Templates** in the `openshift-config` project with the following:

```bash
oc get template -n openshift-config
```

You will see something like the following:

```
NAME              DESCRIPTION   PARAMETERS    OBJECTS
project-request                 5 (5 blank)   7
```

#### Setting the Default ProjectRequestTemplate

The default **projectRequestTemplate** is part of the OpenShift API Server configuration. This configuration is ultimately stored in a **ConfigMap** in the `openshift-apiserver` project. You can view the API Server configuration with the following command:

```bash
oc get cm config -n openshift-apiserver -o jsonpath --template="{.data.config\.yaml}" | jq
```

There is an OpenShift operator that looks at various **CustomResource** (CR) instances and ensures that the configurations they define are implemented in the cluster. In other words, the operator is ultimately responsible for creating/modifying the **ConfigMap**. You can see in the `jq` output that there is a `projectRequestMessage` but no `projectRequestTemplate` defined. There is currently no CR specifying anything, so the operator has configured the cluster with the "stock" settings. To add the default project request tempalate configuration, a CR needs to be created. The **CustomResource** will look like:

```yaml
apiVersion: "config.openshift.io/v1"
kind: "Project"
metadata:
  name: "cluster"
  namespace: ""
spec:
  projectRequestMessage: ""
  projectRequestTemplate:
    name: "project-request"
```

Notice the **projectRequestTemplate** name matches the name of the template we created earlier in the `openshift-config` project.

The next step is to create this **CustomResource**. Once this **CR** is created, the OpenShift operator will notice the **CR**, and apply the configuration changes. To create the **CustomResource**, issue this command:

```bash
oc apply -f /opt/app-root/src/support/cr_project_request.yaml -n openshift-config
```

Once this command is run, the OpenShift API Server configurations will be updated by the operator. This takes some time. You can monitor the rollout by waiting for the `apiserver` **Deployment** to finish:

```bash
sleep 30
oc rollout status deploy apiserver -n openshift-apiserver
```

This can now be verified by viewing the implemented configuration:

```bash
oc get cm config -n openshift-apiserver -o jsonpath --template="{.data.config\.yaml}" | jq
```

Notice the new **projectConfig** section:

```json
...
  "kind": "OpenShiftAPIServerConfig",
  "projectConfig": {
    "projectRequestMessage": "",
    "projectRequestTemplate": "openshift-config/project-request"
  },
...
```

#### Create a New Project

When creating a new project, you should see that a **Quota** and a **LimitRange** are created with it. First, create a new project called `template-test`:

```bash
oc new-project template-test
```

Then, use `describe` to look at some of this **Project’s** details:

```bash
oc describe project template-test
```

The output will look something like:

```
Name:           template-test
Created:        22 seconds ago
Labels:         <none>
Annotations:    openshift.io/description=
                openshift.io/display-name=
                openshift.io/requester=system:serviceaccount:lab-ocp-cns:dashboard-user
                openshift.io/sa.scc.mcs=s0:c24,c19
                openshift.io/sa.scc.supplemental-groups=1000590000/10000
                openshift.io/sa.scc.uid-range=1000590000/10000
Display Name:   <none>
Description:    <none>
Status:         Active
Node Selector:  <none>
Quota:
        Name:                   template-test-quota
        Resource                Used    Hard
        --------                ----    ----
        persistentvolumeclaims  0       5
        pods                    0       10
        requests.cpu            0       4
        requests.memory         0       8Gi
        requests.storage        0       50Gi
        resourcequotas          1       1
Resource limits:
        Name:           template-test-limits
        Type            Resource        Min     Max     Default Request Default Limit   Max Limit/Request Ratio
        ----            --------        ---     ---     --------------- -------------   -----------------------
        Container       cpu             10m     4       100m        4       -
        Container       memory          5Mi     1Gi     512Mi       1Gi     -
```

If you don’t see the Quota and Resource limits sections, you may have been too quick. Remember that the operator takes a moment to do everything it needs to, so it’s possible you created your project before the masters picked up the new configs. Go ahead and `oc delete project template-test` and then re-create it after a few moments.

You can also see that the **Quota** and **LimitRange** objects were created:

```bash
oc describe quota -n template-test
```

You will see:

```
Name:                   template-test-quota
Namespace:              template-test
Resource                Used  Hard
--------                ----  ----
persistentvolumeclaims  0     5
pods                    0     10
requests.cpu            0     4
requests.memory         0     8Gi
requests.storage        0     50Gi
resourcequotas          1     1
```

And:

```bash
oc get limitrange -n template-test
```

You will see:

```
NAME                   CREATED AT
template-test-limits   2020-12-16T00:16:39Z
```

Please make sure that the `project-request` template is created in the `openshift-config` project. Defining it in the OpenShift API server configurion without having the template in place will cause new projects to fail to create.

### Clean Up

If you wish, you can deploy the application from the Application Management Basics lab again inside this `template-test` project to observe how the **Quota** and **LimitRange** are applied. If you do, be sure to look at the JSON/YAML output (`oc get … -o yaml`) for things like the **DeploymentConfig** and the **Pod**.

Before you continue, you may wish to delete the **Project** you just created:

```bash
oc delete project template-test
```

# OpenShift Networking and NetworkPolicy

## OpenShift Network Policy Based SDN

OpenShift has a software defined network (SDN) inside the platform that is based on Open vSwitch. This SDN is used to provide connectivity between application components inside of the OpenShift environment. It comes with default network ranges pre-configured, although you can make changes to these should they conflict with your existing infrastructure, or for whatever other reason you may have.

The OpenShift Network Policy SDN plug-in allows projects to truly isolate their network infrastructure inside OpenShift’s software defined network. While you have seen projects isolate resources through OpenShift’s RBAC, the network policy SDN plugin is able to isolate pods in projects using pod and namespace label selectors.

The network policy SDN plugin was introduced in OpenShift 3.7, and more information about it and its configuration can be found in the [networking documentation](https://docs.openshift.com/container-platform/4.6/networking/understanding-networking.html). Additionally, other vendors are working with the upstream Kubernetes community to implement their own SDN plugins, and several of these are supported by the vendors for use with OpenShift. These plugin implementations make use of appc/CNI, which is outside the scope of this lab.

### Switch Your Project

Before continuing, make sure you are using a project that actually exists. If the last thing you did in the previous lab was delete a project, this will cause errors in the scripts in this lab.

```bash
oc project default
```

### Execute the Creation Script

Only users with project or cluster administration privileges can manipulate **Project** network policies.

Execute a script that we have prepared for you. It will create two **Projects** and then deploy a **DeploymentConfig** with a **Pod** for you:

```bash
bash /opt/app-root/src/support/create-net-projects.sh
```

### Examine the created infrastructure

Two **Projects** were created for you, `netproj-a` and `netproj-b`. Execute the following command to see the created resources:

```bash
oc get pods -n netproj-a
```

After a while you will see something like the following:

```
NAME           READY   STATUS              RESTARTS   AGE
ose-1-66dz2    0/1     ContainerCreating   0          7s
ose-1-deploy   1/1     Running             0          16s
```

Similarly:

```bash
oc get pods -n netproj-b
```

After a while you will see something like the following:

```
NAME           READY   STATUS      RESTARTS   AGE
ose-1-deploy   0/1     Completed   0          38s
ose-1-vj2gn    1/1     Running     0          30s
```

We will run commands inside the pod in the `netproj-a` **Project** that will connect to TCP port 5000 of the pod in the `netproj-b` **Project**.

### Test Connectivity (should work)

Now that you have some projects and pods, let’s test the connectivity between the pod in the `netproj-a` **Project** and the pod in the `netproj-b` **Project**.

To test connectivity between the two pods, run:

```bash
bash /opt/app-root/src/support/test-connectivity.sh
```

You will see something like the following:

```
Getting Pod B's IP... 10.129.0.180
Getting Pod A's Name... ose-1-66dz2
Checking connectivity between Pod A and Pod B... worked
```

Note that the last line says `worked`. This means that the pod in the `netproj-a` **Project** was able to connect to the pod in the `netproj-b` **Project**.

This worked because, by default, with the network policy SDN, all pods in all projects can connect to each other.

### Restricting Access

With the Network Policy based SDN, it’s possible to restrict access in a project by creating a `NetworkPolicy` custom resource (CR).

For example, the following restricts all access to all pods in a **Project** where this `NetworkPolicy` CR is applied. This is the equivalent of a `DenyAll` default rule on a firewall:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-by-default
spec:
  podSelector:
  ingress: []
```

Note that the `podSelector` is empty, which means that this will apply to all pods in this **Project**. Also note that the `ingress` list is empty, which means that there are no allowed `ingress` rules defined by this `NetworkPolicy` CR.

To restrict access to the pod in the `netproj-b` **Project** simply apply the above NetworkPolicy CR with:

```bash
oc create -n netproj-b -f /opt/app-root/src/support/network-policy-block-all.yaml
```

### Test Connectivity #2 (should fail)

Since the "block all by default" `NetworkPolicy` CR has been applied, connectivity between the pod in the `netproj-a` **Project** and the pod in the `netproj-b` **Project** should now be blocked.

Test by running:

```bash
bash /opt/app-root/src/support/test-connectivity.sh
```

You will see something like the following:

```
Getting Pod B's IP... 10.129.0.180
Getting Pod A's Name... ose-1-66dz2
Checking connectivity between Pod A and Pod B............ FAILED!
```

Note the last line that says `FAILED!`. This means that the pod in the `netproj-a` **Project** was unable to connect to the pod in the `netproj-b` **Project** (as expected).

To list all the `NetworkPolicy` objects, run the following:

```bash
oc get networkpolicy -n netproj-b
```

You should see something like the following:

```
NAME              POD-SELECTOR   AGE
deny-by-default   <none>         3m19s
```

### Allow Access

With the Network Policy based SDN, it’s possible to allow access to individual or groups of pods in a project by creating multiple `NetworkPolicy` CRs.

The following allows access to port 5000 on TCP for all pods in the project with the label `run: ose`. The pod in the `netproj-b` project has this label.

The ingress section specifically allows this access from all projects that have the label `name: netproj-a`.

```yaml
# allow access to TCP port 5000 for pods with the label "run: ose" specifically
# from projects with the label "name: netproj-a".
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-tcp-5000-from-netproj-a-namespace
spec:
  podSelector:
    matchLabels:
      run: ose
  ingress:
  - ports:
    - protocol: TCP
      port: 5000
    from:
    - namespaceSelector:
        matchLabels:
          name: netproj-a
```

Note that the `podSelector` is where the local project’s pods are matched using a specific label selector.

All `NetworkPolicy` CRs in a project are combined to create the allowed ingress access for the pods in the project. In this specific case the "deny all" policy is combined with the "allow TCP 5000" policy.

To allow access to the pod in the `netproj-b` **Project** from all pods in the `netproj-a` **Project**, simply apply the above NetworkPolicy CR with:

```bash
oc create -n netproj-b -f /opt/app-root/src/support/network-policy-allow-all-from-netproj-a.yaml
```

Listing the **NetworkPolicies**:

```bash
oc get networkpolicy -n netproj-b
```

This should show the new policy in place

```
NAME                                      POD-SELECTOR   AGE
allow-tcp-5000-from-netproj-a-namespace   run=ose        81s
deny-by-default                           <none>         7m11s
```

### Test Connectivity #3 (should work again)

Since the "allow access from `netproj-a` on port 5000" NetworkPolicy has been applied, connectivity between the pod in the `netproj-a` **Project** and the pod in the `netproj-b` **Project** should be allowed again.

Test by running:

```bash
bash /opt/app-root/src/support/test-connectivity.sh
```

You will see something like the following:

```
Getting Pod B's IP... 10.129.0.180
Getting Pod A's Name... ose-1-66dz2
Checking connectivity between Pod A and Pod B... worked
```

Note the last line that says `worked`. This means that the pod in the `netproj-a` **Project** was able to connect to the pod in the `netproj-b` **Project** (as expected).

# Disabling Project Self-Provisioning

## Disabling Project Self-Provisioning

Before continuing, make sure you’ve done the LDAP lab.

OpenShift, by default, allows authenticated users to create **Projects** to loggically house their applications. This feature enables admins to provide a "self service" feature that was popularized by the advent of the "PaaS" (Platform as a Service) model.

This feature may not be suitable for all sitations, and many administrators may want to have more control over Projects and who (if anyone) can create them. Some of these use cases may be:

- Environment Protection - Administrators may not want Projects getting created without their knowledge.
- Resource Allocation - Administrator may want fine grained control over resource allocation (i.e. "I don’t want to overcommit")
- Quota flexibility - Default quotas may be set, but Administrators may want to specify additional quotas (or fewer) depending on the Project scope.
- Accounting and Chargeback - In a multitenant system, there may be a need to do accounting and chargeback.
- General Administrative Control - Administrators may want total control in what goes in an environment.

There are other ways to gain this type of control besides disabling Project self-provisioning.

### Background: Projects

What are you disabling exactly? In a previous lab, you learned that a Project is a "bucket" of sorts; which holds all the resources for an application. You also learned that Projects can be assigned to a user or group for collaboration. But what’s the difference between a Project and a Kubernetes Namespace?

A Project directly maps to a Kubernetes Namespace. It also maps to the internal registry’s namespace as well as to a VXLAN VNID. So giving access to a Project does more than let users collaborate. You are allowing network communications, regestry namespace access, and access to objects within a Kubernetes Namespace.

#### Examine the Clusterrolebinding

In order to examine the `clusterrolebinding` for `self-provisioners`; you need to be **kubeadmin**.

You could also use the serviceaccount user that we’ve used in previous labs. Since `kubeadmin` and the service account user both have the `cluster-admin` role, it really doesn’t matter.

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

View the `self-provisioners` cluster role binding usage by running the `oc describe` command:

```bash
oc describe clusterrolebinding.rbac self-provisioners
```

You should see the following output:

```
Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
```

Here, it’s saying that the group `system:authenticated:oauth` (which is a group that every user that has authenticated is a part of, by default), is bound to the `self-provisioners` role. This role is a built-in role in OpenShift that allows the users to create projects.

If you are interested in the different roles that come with OpenShift, you can learn more about them in the [role-based access control (RBAC)](https://docs.openshift.com/container-platform/4.1/authentication/using-rbac.html) documentation.

To view the configuration itself, inspect the yaml by running the following:

```bash
oc get clusterrolebinding.rbac self-provisioners -o yaml
```

The configuration should look something like this:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2020-12-07T23:29:30Z"
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:rbac.authorization.kubernetes.io/autoupdate: {}
      f:roleRef:
        f:apiGroup: {}
        f:kind: {}
        f:name: {}
      f:subjects: {}
    manager: openshift-apiserver
    operation: Update
    time: "2020-12-07T23:29:30Z"
  name: self-provisioners
  resourceVersion: "10302"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/self-provisioners
  uid: 1bbfe87b-697c-4702-81d1-00e7f8e98207
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: self-provisioner
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated:oauth
```

#### Removing Self Provisioning Projects

To remove the `self-provisioner` cluster role from the group `system:authenticated:oauth` you need to remove that group from the role binding.

```bash
oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
```

Automatic updates reset the cluster roles to a default state. In order to disable this, you need to set the annotation `rbac.authorization.kubernetes.io/autoupdate` to `false` by running:

```bash
oc patch clusterrolebinding.rbac self-provisioners -p '{ "metadata": { "annotations": { "rbac.authorization.kubernetes.io/autoupdate": "false" } } }'
```

View the new configuration:

```bash
oc get clusterrolebinding.rbac self-provisioners -o yaml
```

It should now have no `subjects` in the YAML. It should look something like this:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "false"
  creationTimestamp: "2020-12-07T23:29:30Z"
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations: {}
      f:roleRef:
        f:apiGroup: {}
        f:kind: {}
        f:name: {}
    manager: openshift-apiserver
    operation: Update
    time: "2020-12-07T23:29:30Z"
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:rbac.authorization.kubernetes.io/autoupdate: {}
    manager: kubectl-patch
    operation: Update
    time: "2020-12-08T00:15:02Z"
  name: self-provisioners
  resourceVersion: "36456"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/self-provisioners
  uid: 1bbfe87b-697c-4702-81d1-00e7f8e98207
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: self-provisioner
```

Test this by logging in as `fancyuser1` and try to create a project.

```bash
oc login -u fancyuser1 -p Op#nSh1ft
oc new-project fancyuserproject
```

You should see an error message:

```
Error from server (Forbidden): You may not request a new project via this API.
```

Login as the `kubeadmin` user for the next exercise.

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

#### Customizing the request message

Now any time a user tries to create a project they will be greated with the same message `You may not request a new project via this API`. You can customize this message to give a more meaningful call to action.

For example, you can have the users submit a ticket requesting a project. We can do this by changing the text given, to include instructions:

```bash
oc patch --type=merge project.config.openshift.io cluster -p '{"spec":{"projectRequestMessage":"Please visit https://ticket.example.com to request a project"}}'
```

Here, you are adding the `projectRequestMessage` and the value `Please visit https://ticket.example.com to request a project` to the specification.

Before you can see this new message, you’ll need to wait for the `apiserver` application to rollout the changes. This can take some time to rollout, especially on a busy cluster.

```bash
oc rollout status -n  openshift-apiserver deploy/apiserver
```

Now, the user will get this message when trying to create a project. Test this by becoming the `fancyuser1` user.

```bash
oc login -u fancyuser1 -p Op#nSh1ft
```

And try to create a project.

```bash
oc new-project fancyuserproject
```

You should see the following message:

```
Error from server (Forbidden): Please visit https://ticket.example.com to request a project
```

#### Clean Up

Make sure you login as `kubeadmin` for the next lab.

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

Other labs may require the `self-provisioners` role, so let’s undo what we did:

```bash
oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects":[{"apiGroup":"rbac.authorization.k8s.io","kind":"Group","name":"system:authenticated:oauth"}]}'
oc patch clusterrolebinding.rbac self-provisioners -p '{"metadata":{"annotations":{"rbac.authorization.kubernetes.io/autoupdate":"true"}}}'
oc patch --type=json project.config.openshift.io cluster -p '[{"op": "remove", "path": "/spec/projectRequestMessage"}]'
```

# Cluster Resource Quotas

## Cluster resource quotas

Before continuing, make sure you’ve done the LDAP lab.

In a previous lab, you worked with quotas and saw how they could be applied to projects. You also set up default quotas, so anytime someone requests a new project; they get assigned the default quota. These project quotas are great for maintaining control over the resources in your cluster.

But what if you want to apply a quota not to individual projects, but accross projects?

### Use cases

There are two primary usecases where you would use `clusterresourcequota` instead of a project based `quota`. One of which, is when you want to set quotas on a specific user. This is useful when you want users to create as much projects as they need (thus achiving great multitenancy), but you want to limit the amount of resource they can consume.

The other use case is if you want to set a quota by application vertical. In this case, you want to set the quota on an application stack wholistically; as an application stack can span multiple OpenShift Projects.

In this lab, we will be exploring both use cases.

#### Setting quota per user

In order to set a `clusterresourcequota` to a user you need to be `kubeadmin`

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

Now, set a quota for the `normaluser1`. We will be using the `annotation` key of `openshift.io/requester=` to identify the projects that will have these quotas assigned to. For this exercise, we will set a hard quota against creating more than 10 pods.

```bash
oc create clusterquota for-user-normaluser1 \
    --project-annotation-selector openshift.io/requester=normaluser1 \
    --hard pods=10
```

The syntax is `openshift.io/requester=<username>`.

View the configuration.

```bash
oc get clusterresourcequotas for-user-normaluser1 -o yaml
```

The configuration should look something like this:

```yaml
piVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  creationTimestamp: "2020-12-02T22:48:38Z"
  generation: 1
  managedFields:
  - apiVersion: quota.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        .: {}
        f:quota:
          .: {}
          f:hard:
            .: {}
            f:pods: {}
        f:selector:
          .: {}
          f:annotations:
            .: {}
            f:openshift.io/requester: {}
          f:labels: {}
      f:status:
        .: {}
        f:namespaces: {}
        f:total: {}
    manager: oc
    operation: Update
    time: "2020-12-02T22:48:38Z"
  name: for-user-normaluser1
  resourceVersion: "55396"
  selfLink: /apis/quota.openshift.io/v1/clusterresourcequotas/for-user-normaluser1
  uid: 21670296-e0af-4572-8141-832217531cc3
spec:
  quota:
    hard:
      pods: "10"
  selector:
    annotations:
      openshift.io/requester: normaluser1
    labels: null
```

This user, `normaluser1`, can create no more than 10 pods accross all the projects he creates. This applies only to projects that he as created (based on the `openshift.io/requester: normaluser1` annotation), not any projects he has access to. More on this later.

Now, login as `normaluser1`

```bash
oc login -u normaluser1 -p Op#nSh1ft
```

List all your current projects

```bash
oc get projects
```

This user shouldn’t have any projects, and you should see output similar to this (don’t worry if you do though):

```
No resources found.
```

Create two projects `welcome1` and `welcome2`.

```bash
oc new-project welcome1
oc new-project welcome2
```

You’ll be creating two applications. One in the `welcome1` project and the other in the `welcome2` project.

```bash
oc new-app -n welcome1 --name=php1 quay.io/redhatworkshops/welcome-php:latest
oc new-app -n welcome2 --name=php2 quay.io/redhatworkshops/welcome-php:latest
```

After the deployment, you should have two running pods. One in each namespace. Check it with the `oc get pods` command (You may have to run this a few times before you see any output):

```bash
oc get pods -n welcome1 -l deployment=php1
oc get pods -n welcome2 -l deployment=php2
```

The output should look something like this:

```
[~] $ oc get pods -n welcome1 -l deployment=php1
NAME           READY   STATUS    RESTARTS   AGE
php1-1-nww4m   1/1     Running   0          4m20s
[~] $ oc get pods -n welcome2 -l deployment=php2
NAME           READY   STATUS    RESTARTS   AGE
php2-1-ljw9w   1/1     Running   0          4m20s
```

Now we can check the quota by first becoming `kubeadmin`:

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

Now run `oc describe clusterresourcequotas for-user-normaluser1` to see the status of the quota:

```bash
oc describe clusterresourcequotas for-user-normaluser1
```

You should see the following output:

```
Name:		for-user-normaluser1
Created:	22 minutes ago
Labels:		<none>
Annotations:	<none>
Namespace Selector: ["welcome1" "welcome2"]
Label Selector:
AnnotationSelector: map[openshift.io/requester:normaluser1]
Resource	Used	Hard
--------	----	----
pods		2	10
```

You see that not only that 2 out of 10 pods are being used, but that the namespaces the quota is being applied to. Check the namespace manifest for `welcome1` to see the annotation the quota is looking for:

```bash
oc get ns welcome1 -o yaml
```

The output should look something like this. Take special note of the annotations:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/description: ""
    openshift.io/display-name: ""
    openshift.io/requester: normaluser1
    openshift.io/sa.scc.mcs: s0:c26,c5
    openshift.io/sa.scc.supplemental-groups: 1000660000/10000
    openshift.io/sa.scc.uid-range: 1000660000/10000
  creationTimestamp: "2020-12-02T22:49:46Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:openshift.io/sa.scc.mcs: {}
          f:openshift.io/sa.scc.supplemental-groups: {}
          f:openshift.io/sa.scc.uid-range: {}
    manager: cluster-policy-controller
    operation: Update
    time: "2020-12-02T22:49:46Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:openshift.io/description: {}
          f:openshift.io/display-name: {}
          f:openshift.io/requester: {}
      f:status:
        f:phase: {}
    manager: openshift-apiserver
    operation: Update
    time: "2020-12-02T22:49:46Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:finalizers: {}
    manager: openshift-controller-manager
    operation: Update
    time: "2020-12-02T22:49:46Z"
  name: welcome1
  resourceVersion: "55712"
  selfLink: /api/v1/namespaces/welcome1
  uid: fe1ceda9-51aa-4222-b47b-e25181291f5e
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

Now as `normaluser1`, try to scale your apps beyond 10 pods:

```bash
oc login -u normaluser1 -p Op#nSh1ft
oc scale deploy/php1 -n welcome1 --replicas=5
oc scale deploy/php2 -n welcome2 --replicas=6
```

Take a note of how many pods are running:

```bash
oc get pods --no-headers -n welcome1 -l deployment=php1 | wc -l
oc get pods --no-headers -n welcome2 -l deployment=php2 | wc -l
```

Both of these commands should return no more than 10 added up together. Check the events to see the quota in action!

```bash
oc get events -n welcome1 | grep "quota" | head -1
oc get events -n welcome2 | grep "quota" | head -1
```

You should see a message like the following.

```
3m24s       Warning   FailedCreate        replicaset/php1-89fcb8d8b    Error creating: pods "php1-89fcb8d8b-spdw2" is forbid
den: exceeded quota: for-user-normaluser1, requested: pods=1, used: pods=10, limited: pods=10
```

To see the status, switch to the `kubeadmin` account and run the `describe` command from before:

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
oc describe clusterresourcequotas for-user-normaluser1
```

You should see that the hard pod limit has been reached

```
Name:           for-user-normaluser1
Created:        15 minutes ago
Labels:         <none>
Annotations:    <none>
Namespace Selector: ["welcome1" "welcome2"]
Label Selector:
AnnotationSelector: map[openshift.io/requester:normaluser1]
Resource        Used    Hard
--------        ----    ----
pods            10      10
```

#### Setting quota by label

In order to set a quota by application stacks that may span multiple projects, you’ll have to use labels to identify the project. First, make sure you’re `kubeadmin`

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

Now set a quota based on a label. For this lab we will use `appstack=pricelist` key/value based label to identify projects.

```bash
oc create clusterresourcequota for-pricelist \
    --project-label-selector=appstack=pricelist \
    --hard=pods=5
```

Now create two projects:

```bash
oc adm new-project pricelist-frontend
oc adm new-project pricelist-backend
```

Assign the `edit` role to the user `normaluser1` for these two projects:

```bash
oc adm policy add-role-to-user edit normaluser1 -n pricelist-frontend
oc adm policy add-role-to-user edit normaluser1 -n pricelist-backend
```

To identify these two projects to belonging to the `pricelist` application stack, you will need to label the corresponding namespace:

```bash
oc label ns pricelist-frontend appstack=pricelist
oc label ns pricelist-backend appstack=pricelist
```

Run the `oc describe` command for the `for-pricelist` cluster resource quota:

```bash
oc describe clusterresourcequotas for-pricelist
```

You should see that both of the projects are now being tracked:

```
Name:           for-pricelist
Created:        21 seconds ago
Labels:         <none>
Annotations:    <none>
Namespace Selector: ["pricelist-frontend" "pricelist-backend"]
Label Selector: appstack=pricelist
AnnotationSelector: map[]
Resource        Used    Hard
--------        ----    ----
pods            0       5
```

Login as `normaluser1` and create the applications in their respective projects:

```bash
oc login -u normaluser1 -p Op#nSh1ft
oc new-app -n pricelist-frontend --name frontend quay.io/redhatworkshops/pricelist:frontend
oc new-app -n pricelist-backend --name backend quay.io/redhatworkshops/pricelist:backend
```

Check the status of the quota by logging in as `kubeadmin` and running the `describe` command:

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
oc describe clusterresourcequotas for-pricelist
```

You should see that 2 out of 5 pods are being used against this quota:

```
Name:           for-pricelist
Created:        About a minute ago
Labels:         <none>
Annotations:    <none>
Namespace Selector: ["pricelist-frontend" "pricelist-backend"]
Label Selector: appstack=pricelist
AnnotationSelector: map[]
Resource        Used    Hard
--------        ----    ----
pods            2       5
```

The user `normaluser1` can create more pods because `pricelist-frontend` and `pricelist-backend` were assigned to the user by `kubeadmin`. They don’t have the `openshift.io/requester=normaluser1` annotation since `normaluser1` didn’t create them. You can already see how you can mix and match quota polices to fit your envrionment.

Test this by logging back in as `normaluser1` and try to scale the applications beyond 5 pods total.

```bash
oc login -u normaluser1 -p Op#nSh1ft
oc scale -n pricelist-frontend deploy/frontend --replicas=3
oc scale -n pricelist-backend deploy/backend --replicas=3
```

Just like before, you should see an error about not being able to scale:

```bash
oc get events -n pricelist-frontend | grep "quota" | head -1
oc get events -n pricelist-backend | grep "quota" | head -1
```

The output should be like the other exercise:

```
39s         Warning   FailedCreate        replicaset/backend-577cf89b68   Error creating: pods "backend-577cf89b68-l5svw" is
 forbidden: exceeded quota: for-pricelist, requested: pods=1, used: pods=5, limited: pods=5
```

#### Clean Up

Clean up the work you did by first becoming `kubeadmin`:

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

These quotas may interfere with other labs; so delete both of the `clusterresourcequota` we created in this lab:

```bash
oc delete clusterresourcequotas for-pricelist for-user-normaluser1
```

Also delete the projects we created for this lab:

```bash
oc delete projects pricelist-backend pricelist-frontend welcome1 welcome2
```

Make sure you login as `kubeadmin` in an existing project for the next lab.

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
oc project default
```

# Cluster Metering

## Cluster Metering

## Lab Overview

This hands-on workshop is for both system administrators and application developers interested in learning how to deploy and manage Metering operator and schedule reports.

### In this lab you will learn how to

- Deploy Metering Operator from OperatorHub
- Install the Metering stack
- Verify the installation
- Writing Reports
- Reading Reports

### Background

OpenShift Container Platform (OCP) 4.2 made Operator Metering generally available. Operator Metering is a chargeback and reporting tool to provide accountability for how resources are used across a cluster. Cluster admins can schedule reports based on historical usage data by Pod, Namespace, and Cluster wide. Operator Metering is part of the [Operator Framework](https://coreos.com/blog/introducing-operator-framework-metering).

This exercise is done with a mix of CLI and the OpenShift web console. All of the interactions with the web console are effectively creating or manipulating API objects in the background. It is possible to fully automate the process and/or do it using the CLI or other tools, but these methods are not covered in the exercise or documentation at this time.

### Installing Operator Metering

This lab assumes you have logged into an existing cluster with the `oc login` command

#### Examine cluster projects

View the `projects` in the existing cluster by running the `oc projects` command (this is analogous to `kubectl namespaces`). Search for openshift-metering in the output to make sure you’re not clobbering an existing installation:

```bash
oc projects | grep openshift-metering
```

The output should reflect that no Metering projects currently exist.

#### Login to the OpenShift Web Console

Login to the [OpenShift Web Console](https://console-openshift-console.apps.odflab-55.container-contest.top:6443/) with the kubeadmin credentials.

```
kubeadmin
VyYA4-KWyRT-9bFCf-2Frvz
```

You might receive a self-signed certificate error in your browser when you first visit the OpenShift Web console. When OpenShift is installed, by default, a CA and SSL certificates are generated for all inter-component communication within OpenShift, including the web console.

#### Add Metering Operator to cluster

To install the Metering Operator using the OpenShift Web Console first create an `openshift-metering` namespace, and then install the Metering Operator from OperatorHub.

1. In the OpenShift Container Platform web console, click **Administration** → **Namespaces** → **Create Namespace**.

2. Set the name to `openshift-metering` and make sure to label the namespace with

   ```
   openshift.io/cluster-monitoring=true
   ```

   Then click Create.

3. Next, click **Operators** → **OperatorHub**, and filter for `metering` to find the Metering Operator.

   ![Metering Operator Logo](https://preview.cloud.189.cn/image/imageAction?param=D0C84946D73406C2D455B7B4942A9FC40639F8C5D088223751DE10FC37667ECD40CFEBD3B048B347AD8EDEFB17D47BEC7A3D0C5C86A79594F868BE46FDF3FF56D56409407A1F7F77814142FE69961B7CED0E93B5864D721690B723B2D406130AE0B4BDCB6CA1623A92E119432362EEAB)

4. Click the **Metering** card, review the package description, and then click **Install**.

5. On the **Install Operator** screen, make sure the `4.6` **Update Channel** is selected. Also, verify that `A specific namespace on the cluster` is selected as the **Installation Mode**. The **Installed Namespace** should already be set to `Operator recommended namespace` with `openshift-metering` selected. Keep the **Approval Strategy** to `Automatic`.

   ![Metering Operator Install](https://preview.cloud.189.cn/image/imageAction?param=04112D7343B064F073606254AEADFD4AD8DEA1D99724E9CD3E76C0C663D8A83FBC3DF768F393303BB2D3A02CDF54AD310731BC2A7C344A16BC7B51D3D237332A444C0B8A38062D026A595FAFB29FCE7D4B9CCC34DB4EBDCB126DF846AFE1FE728C4E1129CAED4E7756F89597BA61499F)

   Once you’ve verified these options, click "Install".

6. This will take you the the "Installing Operator" progress page.

   ![Metering Operator Install Progress](https://preview.cloud.189.cn/image/imageAction?param=946D40FBF90CB1F27CC3270017C4E1947D1C60C7B9C2CA5AF562C899C295EB3C95210CBD9204CD5B5576E7FA08F20B89156CDC20ECBD947DA2911F3BDB702F0191939681AFC7DCCA7F217B7AD3F3218B218D8F9F0F575773A248174870C2B4665F6E83396E8F25432A9D153254BC43A4)

   Once the page displays **Installed operator - ready for use**, go ahead and click on `View Operator`.

7. This will take you to the **Operator Details** page.

   ![Metering Operator Install Progress](https://preview.cloud.189.cn/image/imageAction?param=47E19DCB77C78FE682D478BC665C86629D772F23023819CA7A8FC7C9CC5A2510A97F2ABBB17760D2F4DEDC83C10089E89CA23E154E8FAB29ACA9945915C6EAD7AEBDC0D2B9F3D68120611A73FD098A490DEE92A735BBDC3E4530A03F7A1988244B1BBF64325CF93386062B8FD83D55B1)

It may take a few minutes for the metering operator to install, be patient.

### Install the Metering stack

1. From the web console, ensure you are on the **Operator Details** page for the Metering Operator in the `openshift-metering` project. If you’ve navigated away from this page, you can get back to this page by clicking **Operators** → **Installed Operators**, then selecting the Metering Operator.

2. Under **Provided APIs**, click **Create Instance** on the **Metering Configuration** card.

   ![Metering Config Card](https://preview.cloud.189.cn/image/imageAction?param=57D5233B6280D2D3CC8E0FEB8E6F8D12D4968BF888E2FFB6D5FE91FBFDD3E426DD543139CAE3F5A477926126D1D8D21F792F53EAB14B51D63C3DE5ECB8328195DF7AE2621D4A7BCEEE1A445DE80CFBB149AA3AC4A38154DC1652C740B2194827B72EE4E9643C13F4CB3F1920768A4225)

3. Click on **YAML View** in the **Configure via** section; and replace the default configuration by copying and pasting the following MeteringConfig into the YAML editor and click `Create`:

   ```
   apiVersion: metering.openshift.io/v1
   kind: MeteringConfig
   metadata:
     name: "operator-metering"
   spec:
     unsupportedFeatures:
       enableHDFS: true
     storage:
       type: "hive"
       hive:
         type: "hdfs"
         hdfs:
           # Leave this value as-is.
           namenode: "hdfs-namenode-0.hdfs-namenode:9820"
   ```

   In order to not create a dependency on OpenShift Container Storage, this lab uses an unsupported storage configuration, HDFS, that the Metering operator deploys by itself.

### Verify the installation

To verify your metering installation, you can check that all the required Pods have been created, and check that your report data sources are beginning to import data.

1. Navigate to **Workloads** → **Pods** in the metering namespace and verify that Pods are being created. This can take several minutes after installing the metering stack.

   You can run the same check using the `oc` CLI:

   ```bash
   oc -n openshift-metering get pods
   ```

   You should see similiar output:

   ```
   NAME                                  READY   STATUS              RESTARTS   AGE
   hive-metastore-0                      1/2     Running             0          52s
   hive-server-0                         2/3     Running             0          52s
   metering-operator-68dd64cfb6-pxh8v    2/2     Running             0          2m49s
   presto-coordinator-0                  2/2     Running             0          31s
   reporting-operator-56c6c878fb-2zbhp   0/2     ContainerCreating   0          4s
   ```

2. Continue to check your Pods until they show `Ready`. This can take several minutes. Many Pods rely on other components to function before they themselves can be considered ready. Some Pods may restart if other Pods take too long to start, this is okay and can be expected during installation.

   You can follow the instantiation of the Pods by waiting for all the `StatefulSet` rollouts:

   ```bash
   until [[ $(oc get sts -n openshift-metering -o name | wc -l) -gt 4 ]]; do echo "waiting for statefulsets..." ; sleep 10 ; done
   oc rollout status sts/hdfs-datanode -n openshift-metering
   oc rollout status sts/hdfs-namenode -n openshift-metering
   oc rollout status sts/hive-metastore -n openshift-metering
   oc rollout status sts/hive-server -n openshift-metering
   oc rollout status sts/presto-coordinator -n openshift-metering
   oc rollout status sts/presto-worker -n openshift-metering
   ```

   Once done, you can use the `oc` CLI, to see them running:

   ```bash
   oc -n openshift-metering get pods
   ```

   ```
   NAME                                  READY   STATUS    RESTARTS   AGE
   hdfs-datanode-0                       1/1     Running   0          6m32s
   hdfs-namenode-0                       1/1     Running   0          6m32s
   hive-metastore-0                      2/2     Running   0          6m9s
   hive-server-0                         3/3     Running   0          6m9s
   metering-operator-6f7fb6f6fd-dfk6w    1/1     Running   0          22m
   presto-coordinator-0                  2/2     Running   0          5m43s
   reporting-operator-57c5b4d577-flsqb   2/2     Running   0          5m13s
   ```

3. Next, use the `oc` CLI to verify that the ReportDataSources are beginning to import data, indicated by a valid timestamp in the `EARLIEST METRIC` column (this may take a few minutes). We filter out the "-raw" ReportDataSources which do not import data:

   ```bash
   oc get reportdatasources -n openshift-metering | grep -v raw
   ```

After all Pods are ready and you have verified that data is being imported, you can begin using metering to collect data and report on your cluster.

### Writing Reports

The Report custom resource is used to manage the execution and status of reports. Metering produces reports derived from usage data sources, which can be used in further analysis and filtering.

A single Report resource represents a job that manages a database table and updates it with new information according to a schedule. The Report exposes the data in that table via the reporting-operator HTTP API. Reports with a `spec.schedule` field set are always running and track what time periods it has collected data for. This ensures that if metering is shutdown or unavailable for an extended period of time, it will backfill the data starting where it left off. If the schedule is unset, then the Report will run once for the time specified by the `reportingStart` and `reportingEnd`.

By default, reports wait for `ReportDataSources` to have fully imported any data covered in the reorting peroid. If the Report has a schedule, it will wait to run until the data in the period currently being processed has finished importing.

Use the `oc` CLI to get ReportQueries to see what reports are available:

```bash
oc get reportqueries -n openshift-metering | grep -v raw
```

ReportQueries with the `-raw` suffix are used by other ReportQueries to build more complex queries, and should not be used directly for reports. Therefore, we omitted them with the `grep -v raw` command.

#### Create Report with a Schedule

The following example Report will contain information on every Pod’s CPU requests, and will run every hour, adding the last hours worth of data each time it runs.

1. In the OpenShift Container Platform web console, click **Operators** → **Installed Operators**. On the **Installed Operators** click the Metering operator. This will bring you to the details page again.

   ![Metering Details Page](https://preview.cloud.189.cn/image/imageAction?param=0E5FAE8D95C3F82D0DCC367778424D69B8C2FBD633A80959D7B0B2CC2A18D541126F7A7411064716BF8104537F09CD04EC2EE245F6B5ABA6FC9468B8727CC9D5E17995A1F09BDB66913A1062A2FA3F6A7697501EA4F5DE1771B724D3A39033F797DA2B66DB9B98889CCD8D89D9038B2F)

2. Under the **Metering Report** card, click **Create Instance**.

   ![Metering Report card](https://preview.cloud.189.cn/image/imageAction?param=659623C8ACA1B9B39A8665D036BBEE24CE8672D35217A210515FD29876342AC802FBED6705B5A3B382845763FF32DD5DEDC4A0BF59A01ADB2790DBE9DC600BEFF99B0BF6B2DB1721B7CB132221E4A97CA44CD1922F474BF32292F88E180F44FF616099293388D206B9CFAA1A622FDB96)

   This opens the **Create Report** page. Click `YAML View` to get the YAML editor

3. Replace the default configuration by copying and pasting the following MeteringConfig into the YAML editor and click Create:

   ```
   apiVersion: metering.openshift.io/v1
   kind: Report
   metadata:
     name: cluster-cpu-usage-hourly
   spec:
     query: "cluster-cpu-usage"
     schedule:
       period: "hourly"
   ```

4. Next, use the `oc` CLI to verify that the report was created:

   ```bash
   oc get reports -n openshift-metering
   ```

   Using the `oc` CLI, it shows output similar to the following:

   ```
   NAME                       QUERY               SCHEDULE   RUNNING                  FAILED   LAST REPORT TIME   AGE
   cluster-cpu-usage-hourly   cluster-cpu-usage   hourly     ReportingPeriodWaiting                               7s
   ```

5. The alloted time will pass (one hour) and a report will be run. For the purpose of this workshop, let’s keep going.

#### Create One-Time Report

The following example Report will contain information on every Namespace’s CPU requests, and will run one time.

1. In the OpenShift Container Platform web console, click **Operators** → **Installed Operators**. On the **Installed Operators** click the Metering operator. This will, once again, bring you to the details page.

   ![Metering Details Page](https://preview.cloud.189.cn/image/imageAction?param=DC5413D9C5077208543FC757F354158357FAACEF593D991680ADFC28911BDC93C78B43A8B8B87C8A7B9845F5CAA9622692DD73271F664209B764B445620A269039726584A65754064C665FA562ED62A90BA348015DA389F2E4B3DEAA1742C0AA5BAE7E9E82B01422D7CF4BE6CFAFBF62)

2. Under the **Metering Report** card, click **Create Instance**.

   ![Metering Report card](https://preview.cloud.189.cn/image/imageAction?param=5F610817D9E194F50664F178DEFA6D7D292FAAF4AEC2D9DEECB55781B26AEDF2196FF5831B8C3C878981B057716A236DCAA2DE7B12E69F9EEE9383D0299B03B2782054A093599397A6E485367CAC2511FF13FCB9D2D445BD44C09A2EA117A8FA621DE2D408AF9B6DEBC45417B1D7C5E8)

   This opens the **Create Report** page. Click `YAML View` to get the YAML editor

3. Replace the default configuration by copying and pasting the following MeteringConfig into the YAML editor and click Create:

   ```
   apiVersion: metering.openshift.io/v1
   kind: Report
   metadata:
     name: namespace-cpu-request-2020
     namespace: openshift-metering
   spec:
     query: namespace-cpu-request
     reportingEnd: '2025-12-30T23:59:59Z'
     reportingStart: '2020-01-01T00:00:00Z'
     runImmediately: true
   ```

4. Next, use the `oc` CLI to verify that the report was created:

   ```bash
   oc get reports -n openshift-metering
   ```

   Using the `oc` CLI, it shows output similar to the following:

   ```
   NAME                         QUERY                   SCHEDULE   RUNNING                  FAILED   LAST REPORT TIME       AGE
   cluster-cpu-usage-hourly     cluster-cpu-usage       hourly     ReportingPeriodWaiting                                   4m37s
   namespace-cpu-request-2020   namespace-cpu-request              Finished                          2020-12-30T23:59:59Z   28s
   ```

### Reading Reports

To view reports complete the following:

1. In the OpenShift Container Platform web console, click **Administration** → **Chargeback**. This opens the `Chargeback Reporting` page.

   ![Chargeback Reporting](https://preview.cloud.189.cn/image/imageAction?param=063F8BAA91B65A9F3B2D6FEFA079BE1558C50EE8C616507350CCE7CF1B10089D3E1B8D2C8D240247DF7E5B466DA5F421F7A35A9E764CA6BD8D5072AC75DAF3272D6A17854F26F81D664A42884C28914BD3EE49BAC753499D02170EFBAE4D6C8E7A1260671789513F4B489D4572CC4638)

2. Select the one-time report created in the previous section titled `namespace-cpu-request-2020`

3. From this screen the report can be downloaded as a CSV file by scolling down, and clicking the Download button. The report is also displayed in the lower part of the screen.

   ![Download CSV](https://preview.cloud.189.cn/image/imageAction?param=44578171FFB4E7CA6733BDABFAEACBB65B60768201C57A3D4EE0C87722A8B5924758B92D37FEDC94F22A9E755AB222BD5469380CD5B595660447695F2AB419F93253E87700F92E559686CDD3B9A8E6EA6F670FAA2F8999E1F8DCC7AC2E6C9F6980141F7853AC444A90E3948FC3AE906D)

   This file can be imported to any metering application that accepts CSV format.

# Taints and Tolerations

## Taints and Tolerations

In a previous lab, you created "infra" nodes leveraging [nodeSelector](https://docs.openshift.com/container-platform/latest/nodes/scheduling/nodes-scheduler-node-selectors.html) to control where the workload lands. You saw how you can set nodes to perform certain tasks. The `nodeSelector` in the `nodePlacement` attracts pods to a set of nodes. But what if you want the opposite? What if you want to repel pods?

Having a taint in place will make sure a pod does NOT run on the tainted node.

Some usecases include:

- A node (or a group of nodes) in the cluster is reserved for special purposes
- Nodes having specialized hardware for specific tasks (like a GPU)
- Some nodes may be licensed for some software running on it
- Nodes can be in different network zone for compliance reasons
- You may want to troubleshoot a misbehaving node

In this lab we’ll explore how to use taints and tolerations to control where workloads land.

### Background

Taints and tolerations work in tandem to ensure that workloads are not scheduled onto certian nodes. Once a node is tainted, that node should not accept any workload that does not tolerate the taint(s). Tolerations that are applied to these workloads will allow (but do not require) the corresponding pods to schedule onto nodes with matching taints.

You will have to use `nodeSelector` along with taints and tolerations to have workloads ALWAYS land on a specific node.

#### Examine node labels

In order to examine the nodes, make sure you’re running as `kubeadmin`

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

List the current worker nodes, ignoring infra nodes and storage nodes.

```bash
oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/infra,!cluster.ocs.openshift.io/openshift-storage'
```

Depending on which labs you’ve done, you may have 3 more more nodes with the worker label set.

Now, let’s look to see if there are any taints added to these workers.

```bash
oc describe nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/infra,!cluster.ocs.openshift.io/openshift-storage' | grep Taint
```

There shouldn’t be any taints assinged to these workers.

#### Deploying test application

For this exercise we will be deploying an app to test with.

Before you deploy this app, you’ll need to label your nodes for the duration of this exercise. The application deploys on nodes labeled `appnode=welcome-php`.

```bash
oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/infra,!cluster.ocs.openshift.io/openshift-storage' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} oc label node {} appnode=welcome-php
```

To label a node, you pass the node name with a `key=value` pair. Example: `oc label node worker.example.com appnode=welcome-php`

Verify that the nodes were labeled, taking note of the name of the node(s).

```bash
oc get nodes -l appnode=welcome-php
```

You can now deploy the application using the YAML manifest that is provided in the home directory.

```bash
oc create -f ~/support/welcome-php.yaml
```

This creates the following objects

- The namespace `tt-lab`
- A deployment called `welcome-php` that’s running a sample php app
- A service called `welcome-php` that listens on port `8080`
- A route called `welcome-php`

We will be working on the `tt-lab` project, so switch to it now.

```bash
oc project tt-lab
```

Now scale the pods to equal to the number of nodes we have. The default scheduler will attempt to evenly distribute the app. Also we’ll wait for the rollout to complete.

```bash
NODE_NUMBER=$(oc get nodes --no-headers -l appnode=welcome-php | wc -l)
oc scale deploy welcome-php --replicas=${NODE_NUMBER}
oc rollout status deploy welcome-php
```

If the pods are not evenly distributed between nodes, you may have to run `oc delete pods --all` to let them rebalance.

Verify that the pods are spread evenly accross workers. There should be one pod for each node.

```bash
clear
oc get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
oc get nodes -l appnode=welcome-php
```

#### Tainting nodes

We will now taint a node, so it’ll reject workloads. First, we will go over what each taint does. There are 3 basic taints that you can set.

- `key=value:NoSchedule`
- `key=value:NoExecute`
- `key=value:PreferNoSchedule`

The `key` in this case is any string, up to 253 characters. The key must begin with a letter or number, and may contain letters, numbers, hyphens, dots, and underscores.

The `value` here is any string, up to 63 characters. The value must begin with a letter or number, and may contain letters, numbers, hyphens, dots, and underscores.

The 3 "effects" do the following:

- `NoSchedule` - New pods that do not match the taint are not scheduled onto that node. Existing pods on the node remain.
- `NoExecute` - New pods that do not match the taint cannot be scheduled onto that node. Existing pods on the node that do not have a matching toleration are removed.
- `PreferNoSchedule` - New pods that do not match the taint might be scheduled onto that node, but the scheduler tries not to. Existing pods on the node remain.

There is another component called the `operator`. We’ll go over the `operator` in detail in the "tolerations" section.

For this lab we will taint the first node that’s not an infra or storage node with `welcome-php=run:NoSchedule`. This makes it to where all new pods (without a proper toleration) NOT schedule on this node.

```bash
TTNODE=$(oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/infra,!cluster.ocs.openshift.io/openshift-storage' -o jsonpath='{range .items[0]}{.metadata.name}')
oc adm taint node ${TTNODE} welcome-php=run:NoSchedule
```

The syntax is: `oc adm taint node ${nodename} key=value:Effect`

Examine the nodes we deployed on and see that the taint is applied to one of the nodes.

```bash
oc describe nodes -l appnode=welcome-php | grep Taint
```

We used `NoSchedule` for the effect, so a pod should still be there. Remember that `NoSchedule` only stops new pods from spawning on the node (the command should return a `1`)

```bash
oc get pods -o wide | grep -c ${TTNODE}
```

Let’s delete the pods and wait for the `replicaSet` to redeploy them.

```bash
oc delete pods --all
oc rollout status deploy welcome-php
```

Since our deployment doesn’t have a toleration, the scheduler will deploy the pods on all nodes except the one with a taint. This command should return a `0`

```bash
oc get pods -o wide | grep -c ${TTNODE}
```

Examine where the pods are running.

```bash
clear
oc get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
oc get nodes -l appnode=welcome-php
```

#### Tolerations

A `toleration` is a way for pods to "tolerate" (or "ignore") a node’s taint during scheduling. Tolerations are applied in the `podSpec`, and is in the following form.

```yaml
tolerations:
- key: "welcome-php"
  operator: "Equal"
  value: "run"
  effect: "NoSchedule"
```

If the toleration "matches" then the scheduler will schedule the workload on this node (if need be…remember, it’s not a guarantee). Note that you have to match the `key`, `value`, and `effect`. There is also something called an `operator`.

The `operator` can be set to `Equal` or `Exists`, depending on the fuction you want.

- `Equal` - The `key`, `value`, and `effect` parameters must match. This is the default setting if nothing is provided.
- `Exists` - The `key` and the `effect` parameters must match. You **must** leave a blank value parameter, which matches any.

We’ll apply this toleration in the `spec.template.spec` section of the deployment.

```bash
oc patch deployment welcome-php --patch '{"spec":{"template":{"spec":{"tolerations":[{"key":"welcome-php","operator":"Equal","value":"run","effect":"NoSchedule"}]}}}}'
```

Patching triggers another deployment so we’ll wait for it to finish rolling out.

```bash
oc rollout status deploy welcome-php
```

You can see the toleration config by inspecting the deployment YAML

```bash
oc get deploy welcome-php -o yaml
```

Now, since we have the toleration in place, we should be running on the node with the taint (this should return `1`)

```bash
oc get pods -o wide | grep -c ${TTNODE}
```

Now when you list all pods, they should be now spread evenly.

```bash
clear
oc get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
oc get nodes -l appnode=welcome-php
```

To read more about taints and tolerations, you can take a look at the [Official Documentation](https://docs.openshift.com/container-platform/4.2/nodes/scheduling/nodes-scheduler-taints-tolerations.html).

#### Clean Up

Make sure you login as `kubeadmin` for the next lab.

```bash
oc login -u kubeadmin -p VyYA4-KWyRT-9bFCf-2Frvz
```

Other labs may be affected by taints, so let’s undo what we did:

```bash
oc delete project tt-lab
oc adm taint node ${TTNODE} welcome-php-
oc get nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/infra,!cluster.ocs.openshift.io/openshift-storage' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} oc label node {} appnode-
```

Make sure the nodes have that taint removed

```bash
oc describe nodes -l 'node-role.kubernetes.io/worker,!node-role.kubernetes.io/infra,!cluster.ocs.openshift.io/openshift-storage' | grep Taint
```

Also, verify that the label does not exist on the nodes we were working on. This command shouldn’t return any nodes.

```bash
oc get nodes -l appnode=welcome-php
```


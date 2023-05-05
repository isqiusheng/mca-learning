# Kubernetes集群Node管理

# 一、查看集群信息

~~~powershell
[root@k8s-master1 ~]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.10.100:6443
CoreDNS is running at https://192.168.10.100:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
~~~

# 二、查看节点信息

## 2.1 查看集群节点信息

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
k8s-master1   Ready    <none>   2d20h   v1.21.10
k8s-master2   Ready    <none>   2d20h   v1.21.10
k8s-master3   Ready    <none>   2d20h   v1.21.10
k8s-worker1   Ready    <none>   2d20h   v1.21.10
~~~

## 2.2 查看集群节点详细信息

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes -o wide
NAME          STATUS   ROLES    AGE     VERSION    INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master1   Ready    <none>   2d20h   v1.21.10   192.168.10.12   <none>        CentOS Linux 7 (Core)   5.17.0-1.el7.elrepo.x86_64   containerd://1.6.1
k8s-master2   Ready    <none>   2d20h   v1.21.10   192.168.10.13   <none>        CentOS Linux 7 (Core)   5.17.0-1.el7.elrepo.x86_64   containerd://1.6.1
k8s-master3   Ready    <none>   2d20h   v1.21.10   192.168.10.14   <none>        CentOS Linux 7 (Core)   5.17.0-1.el7.elrepo.x86_64   containerd://1.6.1
k8s-worker1   Ready    <none>   2d20h   v1.21.10   192.168.10.15   <none>        CentOS Linux 7 (Core)   5.17.0-1.el7.elrepo.x86_64   containerd://1.6.1
~~~

## 2.3 查看节点描述详细信息

~~~powershell
[root@k8s-master1 ~]# kubectl describe node k8s-master1
Name:               k8s-master1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s-master1
                    kubernetes.io/os=linux
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 192.168.10.12/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 10.244.159.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 22 Mar 2022 23:47:53 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  k8s-master1
  AcquireTime:     <unset>
  RenewTime:       Fri, 25 Mar 2022 20:38:38 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Wed, 23 Mar 2022 00:14:05 +0800   Wed, 23 Mar 2022 00:14:05 +0800   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Fri, 25 Mar 2022 20:36:09 +0800   Tue, 22 Mar 2022 23:47:53 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Fri, 25 Mar 2022 20:36:09 +0800   Tue, 22 Mar 2022 23:47:53 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Fri, 25 Mar 2022 20:36:09 +0800   Tue, 22 Mar 2022 23:47:53 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Fri, 25 Mar 2022 20:36:09 +0800   Fri, 25 Mar 2022 00:30:10 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.10.12
  Hostname:    k8s-master1
Capacity:
  cpu:                2
  ephemeral-storage:  51175Mi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3994696Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  48294789041
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3892296Ki
  pods:               110
System Info:
  Machine ID:                 a2c5254d78184027930ef5ba59f52d61
  System UUID:                e9dc4d56-4819-1544-2b93-21af423126d2
  Boot ID:                    e45fcd72-4fc2-45b5-be15-7d944a6b8bcd
  Kernel Version:             5.17.0-1.el7.elrepo.x86_64
  OS Image:                   CentOS Linux 7 (Core)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.1
  Kubelet Version:            v1.21.10
  Kube-Proxy Version:         v1.21.10
PodCIDR:                      10.244.2.0/24
PodCIDRs:                     10.244.2.0/24
Non-terminated Pods:          (3 in total)
  Namespace                   Name                              CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                              ------------  ----------  ---------------  -------------  ---
  default                     nginx-web-bbh48                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         2d20h
  kube-system                 calico-node-nkxrs                 250m (12%)    0 (0%)      0 (0%)           0 (0%)         2d20h
  kube-system                 metrics-server-8bb87844c-ptkxm    100m (5%)     0 (0%)      200Mi (5%)       0 (0%)         11h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                350m (17%)  0 (0%)
  memory             200Mi (5%)  0 (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:              <none>

~~~

# 三、worker node节点管理集群

* **如果是kubeasz安装，所有节点(包括master与node)都已经可以对集群进行管理**

* 如果是kubeadm安装，在node节点上管理时会报如下错误

~~~powershell
[root@k8s-worker1 ~]# kubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?
~~~

 只要把master上的管理文件`/etc/kubernetes/admin.conf`拷贝到node节点的`$HOME/.kube/config`就可以让node节点也可以实现kubectl命令管理

1, 在node节点的用户家目录创建`.kube`目录

~~~powershell
[root@k8s-worker1 ~]# mkdir /root/.kube
~~~

2, 在master节点做如下操作

~~~powershell
[root@k8s-worker1 ~]# scp /etc/kubernetes/admin.conf node1:/root/.kube/config
~~~

3, 在worker node节点验证

~~~powershell
[root@k8s-worker1 ~]# kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
k8s-master1   Ready    <none>   2d20h   v1.21.10
k8s-master2   Ready    <none>   2d20h   v1.21.10
k8s-master3   Ready    <none>   2d20h   v1.21.10
k8s-worker1   Ready    <none>   2d20h   v1.21.10
~~~

# 四、节点标签(label)

* k8s集群如果由大量节点组成，可将节点打上对应的标签，然后通过标签进行筛选及查看,更好的进行资源对象的相关选择与匹配

## 4.1 查看节点标签信息

~~~powershell
[root@k8s-master1 ~]# kubectl get node --show-labels
NAME          STATUS   ROLES    AGE     VERSION    LABELS
k8s-master1   Ready    <none>   2d20h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master1,kubernetes.io/os=linux
k8s-master2   Ready    <none>   2d20h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master2,kubernetes.io/os=linux
k8s-master3   Ready    <none>   2d20h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master3,kubernetes.io/os=linux
k8s-worker1   Ready    <none>   2d20h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-worker1,kubernetes.io/os=linux
~~~



## 4.2 设置节点标签信息

### 4.2.1 设置节点标签

为节点`k8s-worker1`打一个`region=huanai` 的标签

~~~powershell
[root@k8s-master1 ~]# kubectl label node k8s-worker1 region=huanai
node/k8s-worker1 labeled
~~~

### 4.2.2 查看所有节点标签

~~~powershell
[root@k8s-master1 ~]# kubectl get node --show-labels
NAME          STATUS   ROLES    AGE     VERSION    LABELS
k8s-master1   Ready    <none>   2d21h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master1,kubernetes.io/os=linux
k8s-master2   Ready    <none>   2d21h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master2,kubernetes.io/os=linux
k8s-master3   Ready    <none>   2d21h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master3,kubernetes.io/os=linux
k8s-worker1   Ready    <none>   2d21h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-worker1,kubernetes.io/os=linux,region=huanai
~~~



### 4.2.3 查看所有节点带region的标签

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes -L region
NAME          STATUS   ROLES    AGE     VERSION    REGION
k8s-master1   Ready    <none>   2d21h   v1.21.10
k8s-master2   Ready    <none>   2d21h   v1.21.10
k8s-master3   Ready    <none>   2d21h   v1.21.10
k8s-worker1   Ready    <none>   2d21h   v1.21.10   huanai
~~~



## 4.3 多维度标签

### 4.3.1 设置多维度标签

也可以加其它的多维度标签,用于不同的需要区分的场景

如把`k8s-master3`标签为华南区,A机房,测试环境,游戏业务

~~~powershell
[root@k8s-master1 ~]# kubectl label node k8s-master3 zone=A env=test bussiness=game
node/k8s-master3 labeled
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes k8s-master3 --show-labels
NAME          STATUS   ROLES    AGE     VERSION    LABELS
k8s-master3   Ready    <none>   2d21h   v1.21.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,bussiness=game,env=test,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master3,kubernetes.io/os=linux,zone=A
~~~

### 4.3.2 显示节点的相应标签

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes -L region,zone
NAME          STATUS   ROLES    AGE     VERSION    REGION   ZONE
k8s-master1   Ready    <none>   2d21h   v1.21.10
k8s-master2   Ready    <none>   2d21h   v1.21.10
k8s-master3   Ready    <none>   2d21h   v1.21.10            A
k8s-worker1   Ready    <none>   2d21h   v1.21.10   huanai
~~~

### 4.3.3 查找`region=huanai`的节点

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes -l region=huanai
NAME          STATUS   ROLES    AGE     VERSION
k8s-worker1   Ready    <none>   2d21h   v1.21.10
~~~



### 4.3.4 标签的修改

~~~powershell
[root@k8s-master1 ~]# kubectl label node k8s-master3 bussiness=ad --overwrite=true
node/k8s-master3 labeled
加上--overwrite=true覆盖原标签的value进行修改操作
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get nodes -L bussiness
NAME          STATUS   ROLES    AGE     VERSION    BUSSINESS
k8s-master1   Ready    <none>   2d21h   v1.21.10
k8s-master2   Ready    <none>   2d21h   v1.21.10
k8s-master3   Ready    <none>   2d21h   v1.21.10   ad
k8s-worker1   Ready    <none>   2d21h   v1.21.10
~~~



### 4.3.5  标签的删除

使用key加一个减号的写法来取消标签

~~~powershell
[root@k8s-master1 ~]# kubectl label node k8s-worker1 region-
node/k8s-worker1 labeled
~~~



### 4.3.6  标签选择器

标签选择器主要有2类:

* 等值关系: =, !=
* 集合关系: KEY in {VALUE1, VALUE2......}

~~~powershell
[root@k8s-master1 ~]# kubectl label node k8s-master2 env=test1
node/k8s-master2 labeled
[root@k8s-master1 ~]# kubectl label node k8s-master3 env=test2
node/k8s-master3 labeled
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get node -l "env in(test1,test2)"
NAME          STATUS   ROLES    AGE     VERSION
k8s-master2   Ready    <none>   2d21h   v1.21.10
k8s-master3   Ready    <none>   2d21h   v1.21.10
~~~








# kubernetes核心概念 Controller

# 一、pod控制器controller

## 1.1 Controller作用及分类

controller用于控制pod

参考: https://kubernetes.io/zh/docs/concepts/workloads/controllers/

![1564844640416](./img/work_load.png)

控制器主要分为:

* Deployments   部署无状态应用，控制pod升级,回退
* ReplicaSet       副本集,控制pod扩容,裁减
* ReplicationController(相当于ReplicaSet的老版本,现在建议使用Deployments加ReplicaSet替代RC)
* StatefulSets     部署有状态应用，结合Service、存储等实现对有状态应用部署
* DaemonSet     守护进程集，运行在所有集群节点(包括master), 比如使用filebeat,node_exporter
* Jobs                  一次性
* Cronjob            周期性

## 1.2 Deployment

### 1.2.1 Replicaset控制器的功能

- 支持新的基于集合的selector(以前的rc里没有这种功能)
- 通过改变Pod副本数量实现Pod的扩容和缩容

### 1.2.2 Deployment控制器的功能

- Deployment集成了上线部署、滚动升级、创建副本、回滚等功能
- Deployment里包含并使用了ReplicaSet

### 1.2.3 Deployment用于部署无状态应用

无状态应用的特点:

- 所有pod无差别
- 所有pod中容器运行同一个image
- 所有pod可以运行在集群中任意node上
- 所有pod无启动顺序先后之分
- 随意pod数量扩容或缩容
- 例如简单运行一个静态web程序

### 1.2.4 创建deployment类型应用

1, 准备YAML文件

~~~powershell
[root@k8s-master1 ~]# vim deployment-nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx			# deployment名
spec:				
  replicas: 1					# 副本集,deployment里使用了replicaset
  selector:
    matchLabels:
      app: nginx				# 匹配的pod标签,表示deployment和rs控制器控制带有此标签的pod
  template:					    # 代表pod的配置模板
    metadata:
      labels:
        app: nginx				# pod的标签
    spec:
      containers:				# 以下为pod里的容器定义
      - name: nginx
        image: nginx:1.15-alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
~~~

2, 应用YAML文件创建deployment

~~~powershell
[root@k8s-master1 ~]# kubectl apply -f deployment-nginx.yml
deployment.apps/deploy-nginx created
~~~

3, 查看验证

~~~powershell
[root@k8s-master1 ~]# kubectl get deployment				# deployment可简写成depoly
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
deploy-nginx   1/1     1            1           19s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
deploy-nginx-6c9764bb69-pbc2h   1/1     Running   0          75s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get replicasets				# replicasets可简写成rs
NAME                      DESIRED   CURRENT   READY   AGE
deploy-nginx-6c9764bb69   1         1         1       2m6s
~~~

### 1.2.5 访问deployment

1,查看pod的IP地址

~~~powershell
[root@k8s-master1 ~]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
deploy-nginx-6d9d558bb6-88nr8   1/1     Running   0          39s   10.244.159.155   k8s-master1   <none>           <none>

pod在k8s-master1节点,pod的IP为10.244.159.155
~~~

2, 查看所有集群节点的网卡

~~~powershell
[root@k8s-master1 ~]# ifconfig tunl0 |head -2
tunl0: flags=193<UP,RUNNING,NOARP>  mtu 1480
        inet 10.244.159.128  netmask 255.255.255.255
~~~

~~~powershell
[root@k8s-master2 ~]# ifconfig tunl0 |head -2
tunl0: flags=193<UP,RUNNING,NOARP>  mtu 1480
        inet 10.244.224.0  netmask 255.255.255.255
~~~

~~~powershell
[root@k8s-master3 ~]# ifconfig tunl0 |head -2
tunl0: flags=193<UP,RUNNING,NOARP>  mtu 1480
        inet 10.244.135.192  netmask 255.255.255.255
~~~

~~~powershell
[root@k8s-worker1 ~]# ifconfig tunl0 |head -2
tunl0: flags=193<UP,RUNNING,NOARP>  mtu 1480
        inet 10.244.194.64  netmask 255.255.255.255
~~~

* 可以看到所有集群节点的IP都为`10.244.0.0/16`这个大网段内的子网

3, 在任意集群节点上都可以访问此deploy里pod

~~~powershell
# curl 10.244.159.155

结果是任意集群节点都可以访问这个POD,但集群外部是不能访问的
~~~

### 1.2.6 删除deployment中的pod

1, 删除pod（**注意: 是删除deployment中的pod**）

~~~powershell
[root@k8s-master1 ~]# kubectl delete pod deploy-nginx-6c9764bb69-pbc2h
pod "deploy-nginx-6c9764bb69-pbc2h" deleted
~~~

2, 再次查看,发现又重新启动了一个pod(**节点由k8s-master1转为k8s-worker1 了,IP地址也变化了**)

~~~powershell
[root@k8s-master1 ~]# kubectl get pods  -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
deploy-nginx-6d9d558bb6-f2t6r   1/1     Running   0          28s   10.244.194.94   k8s-worker1   <none>           <none>

~~~

也就是说**==pod的IP不是固定的==**,比如把整个集群关闭再启动,pod也会自动启动,但是**IP地址也会变化**

**既然IP地址不是固定的,所以需要一个固定的访问endpoint给用户,那么这种方式就是service.**

### 1.2.7 pod版本升级

查看帮助

~~~powershell
[root@k8s-master1 ~]# kubectl set image -h
~~~

1, 升级前验证nginx版本

~~~powershell
[root@k8s-master1 ~]# kubectl describe pods deploy-nginx-6d9d558bb6-f2t6r | grep Image:
    Image:          nginx:1.15-alpine
    
[root@k8s-master1 ~]# kubectl exec deploy-nginx-6d9d558bb6-f2t6r -- nginx -v
nginx version: nginx/1.15.12
~~~

2, 升级为1.16版

~~~powershell
[root@k8s-master1 ~]# kubectl set image deployment deploy-nginx nginx=nginx:1.16-alpine --record
deployment.apps/deploy-nginx image updated
~~~

说明:

* `deployment deploy-nginx`代表名为deploy-nginx的deployment

* `nginx=nginx:1.16-alpine`前面的nginx为容器名

* --record  表示会记录

**容器名怎么查看?**

* `kubectl describe pod pod名`查看

* `kubectl edit deployment deployment名`来查看容器名

* `kubectl get deployment deployment名 -o yaml`来查看容器名

  

3, 验证

如果升级的pod数量较多，则需要一定时间，可通过下面命令查看是否已经成功

~~~powershell
[root@k8s-master1 ~]# kubectl rollout status deployment deploy-nginx	
deployment "deploy-nginx" successfully rolled out
~~~

验证 pod

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
deploy-nginx-5f4749c8c8-nskp9   1/1     Running   0          104s     更新后,后面的id变了
~~~

验证版本

~~~powershell
[root@k8s-master1 ~]# kubectl describe pod deploy-nginx-5f4749c8c8-nskp9 |grep Image:
    Image:          nginx:1.16-alpine							升级为1.16了
    
[root@k8s-master1 ~]# kubectl exec deploy-nginx-5f4749c8c8-nskp9 -- nginx -v
nginx version: nginx/1.16.1										升级为1.16了
~~~



**练习:** 再将nginx1升级为1.17版

~~~powershell
[root@k8s-master1 ~]# kubectl set image deployment deploy-nginx nginx=nginx:1.17-alpine --record
deployment.apps/deploy-nginx image updated
~~~

### 1.2.8 pod版本回退

1, 查看版本历史信息

~~~powershell
[root@k8s-master1 ~]# kubectl rollout history deployment deploy-nginx
deployment.apps/deploy-nginx
REVISION  CHANGE-CAUSE
1         <none>												原1.15版
2         kubectl set image deployment deploy-nginx nginx=nginx:1.16-alpine --record=true
3         kubectl set image deployment deploy-nginx nginx=nginx:1.17-alpine --record=true
~~~

2, 定义要回退的版本（还需要执行才是真的回退版本)

~~~powershell
[root@k8s-master1 ~]# kubectl rollout history deployment deploy-nginx --revision=1
deployment.apps/deploy-nginx with revision #1
Pod Template:
  Labels:       app=nginx
        pod-template-hash=6c9764bb69
  Containers:
   nginx:
    Image:      nginx:1.15-alpine				可以看到这是要回退的1.15版本
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
~~~

3, 执行回退

~~~powershell
[root@k8s-master1 ~]# kubectl rollout undo deployment deploy-nginx --to-revision=1
deployment.apps/deploy-nginx rolled back
~~~

4, 验证

~~~powershell
[root@k8s-master1 ~]# kubectl rollout history deployment deploy-nginx
deployment.apps/deploy-nginx
REVISION  CHANGE-CAUSE
2         kubectl set image deployment deploy-nginx nginx=nginx:1.16-alpine --record=true
3         kubectl set image deployment deploy-nginx nginx=nginx:1.17-alpine --record=true
4         <none>						回到了1.15版,但revision的ID变了
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
deploy-nginx-6c9764bb69-zgwpj   1/1     Running   0          54s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl describe pod deploy-nginx-6c9764bb69-zgwpj |grep Image:
    Image:          nginx:1.15-alpine				回到了1.15版

[root@k8s-master1 ~]# kubectl exec deploy-nginx-6c9764bb69-zgwpj -- nginx -v
nginx version: nginx/1.15.12						回到了1.15版
~~~

### 1.2.9 副本扩容

查看帮助

~~~powershell
[root@k8s-master1 ~]# kubectl scale -h
~~~

1, 扩容为2个副本

~~~powershell
[root@k8s-master1 ~]# kubectl scale deployment deploy-nginx --replicas=2
deployment.apps/deploy-nginx scaled
~~~

2, 查看

~~~powershell
[root@k8s-master1 ~]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
deploy-nginx-6d9d558bb6-4c64l   1/1     Running   0          27s   10.244.159.157   k8s-master1   <none>           <none>
deploy-nginx-6d9d558bb6-hkq2b   1/1     Running   0          71s   10.244.194.95    k8s-worker1   <none>           <none>

在两个node节点上各1个pod
~~~

3, 继续扩容(我们这里只有2个node,但是可以大于node节点数据)

~~~powershell
[root@master ~]# kubectl scale deployment deploy-nginx --replicas=4
deployment.extensions/nginx1 scaled
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
deploy-nginx-6d9d558bb6-4c64l   1/1     Running   0          87s     10.244.159.157   k8s-master1   <none>           <none>
deploy-nginx-6d9d558bb6-586dr   1/1     Running   0          31s     10.244.135.197   k8s-master3   <none>           <none>
deploy-nginx-6d9d558bb6-hkq2b   1/1     Running   0          2m11s   10.244.194.95    k8s-worker1   <none>           <none>
deploy-nginx-6d9d558bb6-kvgsc   1/1     Running   0          31s     10.244.224.13    k8s-master2   <none>           <none>
~~~

### 1.2.10 副本裁减

1, 指定副本数为1进行裁减

~~~powershell
[root@k8s-master1 ~]# kubectl scale deployment deploy-nginx --replicas=1
deployment.apps/deploy-nginx scaled
~~~

2, 查看验证

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
deploy-nginx-6d9d558bb6-hkq2b   1/1     Running   0          2m56s
~~~

### 1.2.11 多副本滚动更新

1, 先扩容多点副本

~~~powershell
[root@k8s-master1 ~]# kubectl scale deployment deploy-nginx --replicas=16
deployment.apps/deploy-nginx scaled
~~~

2, 验证

~~~powershell
[root@master ~]# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
nginx1-7d9b8757cf-2hd48   1/1     Running   0          61s
nginx1-7d9b8757cf-5m72n   1/1     Running   0          61s
nginx1-7d9b8757cf-5w2xr   1/1     Running   0          61s
nginx1-7d9b8757cf-5wmdh   1/1     Running   0          61s
nginx1-7d9b8757cf-6szjj   1/1     Running   0          61s
nginx1-7d9b8757cf-9dgsw   1/1     Running   0          61s
nginx1-7d9b8757cf-dc7qj   1/1     Running   0          61s
nginx1-7d9b8757cf-l52pr   1/1     Running   0          61s
nginx1-7d9b8757cf-m7rt4   1/1     Running   0          26m
nginx1-7d9b8757cf-mdkj2   1/1     Running   0          61s
nginx1-7d9b8757cf-s79kp   1/1     Running   0          61s
nginx1-7d9b8757cf-shhvk   1/1     Running   0          61s
nginx1-7d9b8757cf-sv8gb   1/1     Running   0          61s
nginx1-7d9b8757cf-xbhf4   1/1     Running   0          61s
nginx1-7d9b8757cf-zgdgd   1/1     Running   0          61s
nginx1-7d9b8757cf-zzljl   1/1     Running   0          61s
nginx2-559567f789-8hstz   1/1     Running   1          114m
~~~

3, 滚动更新

~~~powershell
[root@k8s-master1 ~]# kubectl set image deployment deploy-nginx nginx=nginx:1.17-alpine --record
deployment.apps/deploy-nginx image updated
~~~

4, 验证

~~~powershell
[root@k8s-master1 ~]# kubectl rollout status deployment deploy-nginx
......
Waiting for deployment "deploy-nginx" rollout to finish: 13 of 16 updated replicas are available...
Waiting for deployment "deploy-nginx" rollout to finish: 14 of 16 updated replicas are available...
Waiting for deployment "deploy-nginx" rollout to finish: 15 of 16 updated replicas are available...
deployment "deploy-nginx" successfully rolled out
~~~

### 1.2.12 删除deployment

如果使用 `kubectl delete deployment deploy-nginx `命令删除deployment,那么里面的pod也会被自动删除

## 1.3 Replicaset

1, 编写YAML文件

~~~powershell
[root@master ~]# vim rs-nginx.yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-nginx
  namespace: default
spec:                    # replicaset的spec
  replicas: 2            # 副本数
  selector:              # 标签选择器,对应pod的标签
    matchLabels:
      app: nginx         # 匹配的label
  template:
    metadata:
      name: nginx		# pod名
      labels:           # 对应上面定义的标签选择器selector里面的内容
        app: nginx
    spec:               # pod的spec
      containers:
      - name: nginx
        image: nginx:1.15-alpine
        ports:
        - name: http
          containerPort: 80
~~~

2, 应用YAML文件

~~~powershell
[root@k8s-master1 ~]# kubectl apply -f rs-nginx.yml
replicaset.apps/rs-nginx created
~~~

3, 验证

~~~powershell
[root@k8s-master1 ~]# kubectl get rs
NAME       DESIRED   CURRENT   READY   AGE
rs-nginx   2         2         2       26s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
rs-nginx-7j9hz   1/1     Running   0          44s
rs-nginx-pncsk   1/1     Running   0          43s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get deployment
No resources found.

找不到deployment,说明创建rs并没有创建deployment
~~~

# 二、pod控制器Controller进阶

## 2.1 DaemonSet

### 2.1.1 DaemonSet介绍

- DaemonSet能够让所有（或者特定）的节点运行同一个pod。
- 当节点加入到K8S集群中，pod会被（DaemonSet）调度到该节点上运行，当节点从K8S集群中被移除，被DaemonSet调度的pod会被移除
- 如果删除DaemonSet，所有跟这个DaemonSet相关的pods都会被删除。
- 如果一个DaemonSet的Pod被杀死、停止、或者崩溃，那么DaemonSet将会重新创建一个新的副本在这台计算节点上。
- DaemonSet一般应用于日志收集、监控采集、分布式存储守护进程等

### 2.1.2 DaemonSet应用案例

1, 编写YAML文件

~~~~powershell
[root@master ~]# vim daemonset-nginx.yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-nginx			
spec:
  selector:
    matchLabels:
      name: nginx-ds
  template:
    metadata:
      labels:
        name: nginx-ds
    spec:
      tolerations:						# tolerations代表容忍
      - key: node-role.kubernetes.io/master  # 能容忍的污点key
        effect: NoSchedule   # kubectl explain pod.spec.tolerations查看(能容忍的污点effect)
      containers:
      - name: nginx
        image: nginx:1.15-alpine
        imagePullPolicy: IfNotPresent
        resources:    # resources资源限制是为了防止master节点的资源被占太多(根据实际情况配置)
          limits:
            memory: 100Mi
          requests:
            memory: 100Mi
~~~~

2, apply应用YAML文件

~~~powershell
[root@k8s-master1 ~]# kubectl apply -f daemonset-nginx.yml
daemonset.apps/daemonset-nginx created
~~~

3, 验证

~~~powershell
[root@master ~]# kubectl get daemonset				# daemonset可简写为ds
[root@k8s-master1 ~]# kubectl get ds
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-nginx   4         4         4       4            4           <none>          114s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
daemonset-nginx-94z6d   1/1     Running   0          6s    10.244.194.104   k8s-worker1   <none>           <none>
daemonset-nginx-hs9mk   1/1     Running   0          6s    10.244.135.206   k8s-master3   <none>           <none>
daemonset-nginx-jrcf5   1/1     Running   0          6s    10.244.159.167   k8s-master1   <none>           <none>
daemonset-nginx-sslpl   1/1     Running   0          6s    10.244.224.22    k8s-master2   <none>           <none>

k8s集群中每个节点都会运行一个pod
~~~

## 2.2 Job

### 2.2.1 Job介绍

- 对于ReplicaSet而言，它希望pod保持预期数目、持久运行下去，除非用户明确删除，否则这些对象一直存在，它们针对的是耐久性任务，如web服务等。
- 对于非耐久性任务，比如压缩文件，任务完成后，pod需要结束运行，不需要pod继续保持在系统中，这个时候就要用到Job。
- Job负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束。

### 2.2.2 Job应用案例

#### 2.2.2.1 计算圆周率2000位

1, 编写YAML文件

~~~powershell
[root@master ~]# vim job1.yml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi			# job名
spec:
  template:
    metadata:
      name: pi		# pod名
    spec:
      containers:
      - name: pi	   # 容器名
        image: perl	   # 此镜像有800多M,可提前导入到所有节点,也可能指定导入到某一节点然后指定调度到此节点
        imagePullPolicy: IfNotPresent
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never   # 执行完后不再重启
~~~

2, 应用YAML文件创建job

~~~powershell
[root@master ~]# kubectl apply -f job1.yml
job.batch/pi created
~~~

3,  验证

~~~powershell
[root@k8s-master1 ~]# kubectl get jobs
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           11s        18s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME                         READY   STATUS             RESTARTS   AGE
pi-tjq9b                     0/1     Completed          0         27s

Completed状态,也不再是ready状态
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl logs pi-tjq9b
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989380952572010654858632788659361533818279682303019520353018529689957736225994138912497217752834791315155748572424541506959508295331168617278558890750983817546374649393192550604009277016711390098488240128583616035637076601047101819429555961989467678374494482553797747268471040475346462080466842590694912933136770289891521047521620569660240580381501935112533824300355876402474964732639141992726042699227967823547816360093417216412199245863150302861829745557067498385054945885869269956909272107975093029553211653449872027559602364806654991198818347977535663698074265425278625518184175746728909777727938000816470600161452491921732172147723501414419735685481613611573525521334757418494684385233239073941433345477624168625189835694855620992192221842725502542568876717904946016534668049886272327917860857843838279679766814541009538837863609506800642251252051173929848960841284886269456042419652850222106611863067442786220391949450471237137869609563643719172874677646575739624138908658326459958133904780275901

~~~

#### 2.2.2.2  创建固定次数job

1, 编写YAML文件

~~~powershell
[root@master ~]# vim job2.yml
apiVersion: batch/v1
kind: Job
metadata:
  name: busybox-job
spec:
  completions: 10                                               # 执行job的次数
  parallelism: 1                                                # 执行job的并发数
  template:
    metadata:
      name: busybox-job-pod
    spec:
      containers:
      - name: busybox
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["echo", "hello"]
      restartPolicy: Never
~~~

2, 应用YAML文件创建job

~~~powershell
[root@k8s-master1 ~]# kubectl apply -f job2.yml
job.batch/busybox-job created
~~~

3, 验证

~~~powershell
[root@k8s-master1 ~]# kubectl get job
NAME          COMPLETIONS   DURATION   AGE
busybox-job   2/10          9s         9s

[root@k8s-master1 ~]# kubectl get job
NAME          COMPLETIONS   DURATION   AGE
busybox-job   3/10          12s        12s

[root@k8s-master1 ~]# kubectl get job
NAME          COMPLETIONS   DURATION   AGE
busybox-job   4/10          15s        15s

[root@k8s-master1 ~]# kubectl get job
NAME          COMPLETIONS   DURATION   AGE
busybox-job   10/10         34s        48s

34秒左右结束
~~~



~~~powershell
[root@master ~]# kubectl get pods
NAME                         READY   STATUS             RESTARTS   AGE
busybox-job-5zn6l            0/1     Completed          0          34s
busybox-job-cm9kw            0/1     Completed          0          29s
busybox-job-fmpgt            0/1     Completed          0          38s
busybox-job-gjjvh            0/1     Completed          0          45s
busybox-job-krxpd            0/1     Completed          0          25s
busybox-job-m2vcq            0/1     Completed          0          41s
busybox-job-ncg78            0/1     Completed          0          47s
busybox-job-tbzz8            0/1     Completed          0          51s
busybox-job-vb99r            0/1     Completed          0          21s
busybox-job-wnch7            0/1     Completed          0          32s
~~~

#### 2.2.2.3 一次性备份MySQL数据库  

> 通过Job控制器创建应用备份MySQL数据库

##### 2.2.2.3.1  MySQL数据库准备

~~~powershell
[root@nginx jobcontroller]# cat 00_mysql.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-test
  namespace: default
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql-dump

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: default
spec:
  selector:
    matchLabels:
      app: mysql-dump
  serviceName: "mysql-test"
  template:
    metadata:
      labels:
        app: mysql-dump
    spec:
      nodeName: k8s-master3
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "abc123"
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: mysql-data
      volumes:
      - name: mysql-data
        hostPath:
          path: /opt/mysqldata
~~~



##### 2.2.2.3.2  创建用于实现任务的资源清单文件

~~~powershell
[root@nginx jobcontroller]# cat 03_job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mysql-dump
spec:
  template:
    metadata:
      name: mysql-dump
    spec:
      nodeName: k8s-master2
      containers:
      - name: mysql-dump
        image: mysql:5.7
        command: ["/bin/sh","-c","mysqldump --host=mysql-test -uroot -pabc123 --databases mysql > /root/mysql2022.sql"]
        volumeMounts:
        - mountPath: "/root"
          name: mysql-data
      restartPolicy: Never
      volumes:
      - name: mysql-data
        hostPath:
          path: /opt/mysqldump
~~~

## 2.3 CronJob

### 2.3.1 CronJob介绍

* 类似于Linux系统的crontab，在指定的时间周期运行相关的任务
* 时间格式：分时日月周

### 2.3.2 CronJob应用案例

#### 2.3.2.1 周期性输出字符

1, 编写YAML文件

~~~powershell
[root@k8s-master1 ~]# vim cronjob.yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob1
spec:
  schedule: "* * * * *"                 # 分时日月周
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo hello kubernetes
            imagePullPolicy: IfNotPresent
          restartPolicy: OnFailure
~~~

2, 应用YAML文件创建cronjob

~~~powershell
[root@k8s-master1 ~]# kubectl apply -f cronjob.yml
cronjob.batch/cronjob1 created
~~~

3, 查看验证

~~~powershell
[root@k8s-master1 ~]# kubectl get cronjob
NAME       SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob1   * * * * *   False     0        <none>          21s
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get pods
NAME                         READY   STATUS             RESTARTS   AGE
cronjob-1564993080-qlbgv     0/1     Completed          0          2m10s
cronjob-1564993140-zbv7f     0/1     Completed          0          70s
cronjob-1564993200-gx5xz     0/1     Completed          0          10s

看AGE时间,每分钟整点执行一次
~~~

#### 2.3.2.2 周期性备份MySQL数据库

##### 2.3.2.2.1 MySQL数据库准备

~~~powershell
[root@nginx jobcontroller]# cat 00_mysql.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-test
  namespace: default
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql-dump

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: default
spec:
  selector:
    matchLabels:
      app: mysql-dump
  serviceName: "mysql-test"
  template:
    metadata:
      labels:
        app: mysql-dump
    spec:
      nodeName: worker03
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "abc123"
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: "/var/lib/mysql"
          name: mysql-data
      volumes:
      - name: mysql-data
        hostPath:
          path: /opt/mysqldata
~~~

##### 2.3.2.2.2 Cronjob控制器类型应用资源清单文件

~~~powershell
[root@nginx jobcontroller]# cat 05_cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mysql-dump
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          nodeName: worker02
          containers:
          - name: c1
            image: mysql:5.7
            command: ["/bin/sh","-c","mysqldump --host=mysql-test -uroot -pabc123 --databases mysql > /root/mysql`date +%Y%m%d%H%M`.sql"]
            volumeMounts:
              - name: mysql-data
                mountPath: "/root"
          restartPolicy: Never
          volumes:
            - name: mysql-data
              hostPath:
                path: /opt/mysqldump
~~~

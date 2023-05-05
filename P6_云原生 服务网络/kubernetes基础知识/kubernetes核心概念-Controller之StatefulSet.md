# Kubernetes核心概念 Controller之StatefulSet控制器

# 一、StatefulSet控制器作用

- StatefulSet 是用来管理有状态应用的控制器。

- StatefulSet 用来管理某Pod集合的部署和扩缩， 并为这些 Pod 提供持久存储和持久标识符。

- 参考: https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/

# 二、无状态应用与有状态应用

## 2.1 无状态应用

- 如nginx

* 请求本身包含了响应端为响应这一请求所需的全部信息。每一个请求都像首次执行一样，不会依赖之前的数据进行响应。
* 不需要持久化的数据
* 无状态应用的多个实例之间互不依赖，可以无序的部署、删除或伸缩

## 2.2 有状态应用

- 如mysql

* 前后请求有关联与依赖
* 需要持久化的数据
* 有状态应用的多个实例之间有依赖，不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。

# 三、StatefulSet的特点

- 稳定的、唯一的网络标识符。		(通过headless服务实现)
- 稳定的、持久的存储。			       (通过PV，PVC，storageclass实现)
- 有序的、优雅的部署和缩放。       
- 有序的、自动的滚动更新。           

# 四、StatefulSet的YAML组成

需要三个组成部分:

1. headless service:                 实现稳定，唯一的网络标识
2. statefulset类型资源:            写法和deployment几乎一致，就是类型不一样
3. volumeClaimTemplate :     指定存储卷

# 五、创建StatefulSet应用

- 参考: https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/

## 5.1 编辑YAML资源清单文件

> 创建statelfulset应用来调用名为nfs-client的storageclass,以实现动态供给

```powershell
[root@k8s-master1 ~]# vim nginx-storageclass-nfs.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None								   # 无头服务
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web											# statefulset的名称
spec:
  serviceName: "nginx"
  replicas: 3										# 3个副本
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15-alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-client"		# 与前面定义的storageclass名称对应
      resources:
        requests:
          storage: 1Gi
```

```powershell
[root@k8s-master1 ~]# kubectl apply -f nginx-storageclass-nfs.yml
service/nginx created
statefulset.apps/web created
```

## 5.2 应用部署后验证

### 5.2.1 验证pod

> 产生了3个pod

```powershell
[root@k8s-master1 ~]# kubectl get pods |grep web
web-0                                     1/1     Running   0          1m15s
web-1                                     1/1     Running   0          1m7s
web-2                                     1/1     Running   0          57s
```

### 5.2.2 验证pv

> 自动产生了3个pv

```powershell
[root@k8s-master1 ~] # kubectl get pv
pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6  1Gi  RWO   Delete  Bound  default/www-web-0   nfs-client       3m
pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc  1Gi  RWO   Delete  Bound  default/www-web-1   nfs-client       2m
pvc-43afb71d-1d02-4699-b00c-71679fd75fc3  1Gi  RWO   Delete  ound   default/www-web-2   nfs-client       2m
```

### 5.2.3 验证pvc

> 自动产生了3个PVC

```powershell
[root@k8s-master1 ~]# kubectl get pvc |grep web
www-web-0  Bound   pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6   1Gi   RWO  nfs-client  3m
www-web-1  Bound   pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc   1Gi   RWO  nfs-client  2m
www-web-2  Bound   pvc-43afb71d-1d02-4699-b00c-71679fd75fc3   1Gi   RWO  nfs-client  2m
```

### 5.2.4 验证nfs服务目录

在nfs服务器（这里为hostos)的共享目录中发现自动产生了3个子目录

```powershell
[root@nfsserver ~]# ls /data/nfs/
default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6  
default-www-web-2-pvc-43afb71d-1d02-4699-b00c-71679fd75fc3
default-www-web-1-pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc  
```

3个子目录默认都为空目录

```powershell
[root@nfsserver ~]# tree /data/nfs/
/data/nfs/
├── default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6
├── default-www-web-1-pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc
└── default-www-web-2-pvc-43afb71d-1d02-4699-b00c-71679fd75fc3
```

### 5.2.5 验证存储持久性

在3个pod中其中一个创建一个主页文件

```powershell
[root@k8s-master1 ~]# kubectl exec -it web-0 -- /bin/sh
/ # echo "haha" >  /usr/share/nginx/html/index.html
/ # exit
```

在nfs服务器上发现文件被创建到了对应的目录中

```powershell
[root@nfsserver ~]# tree /data/nfs/
/data/nfs/
├── default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6
│   └── index.html								# 此目录里多了index.html文件，对应刚才在web-0的pod中的创建
├── default-www-web-1-pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc
└── default-www-web-2-pvc-43afb71d-1d02-4699-b00c-71679fd75fc3


[root@nfsserver ~]# cat /data/nfs/default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6/index.html
haha											# 文件内的内容也与web-0的pod中创建的一致

```

删除web-0这个pod,再验证

```powershell
[root@k8s-master1 ~]# kubectl delete pod web-0
pod "web-0" deleted


[root@k8s-master1 ~]# kubectl get pods |grep web			# 因为控制器的原因，会迅速再拉起web-0这个pod
web-0                                     1/1     Running   0          9s	  # 时间上看到是新拉起的pod
web-1                                     1/1     Running   0          37m
web-2                                     1/1     Running   0          37m

[root@k8s-master1 ~]# kubectl exec -it web-0 -- cat /usr/share/nginx/html/index.html
haha													# 新拉起的pod仍然是相同的存储数据

[root@nfsserver ~]# cat /data/nfs/default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6/index.html
haha													# nfs服务器上的数据还在
```

**结论: 说明数据可持久化**

### 5.2.6 访问验证

~~~powershell
验证Coredns是否可用
# kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   6d23h

# dig -t a www.baidu.com @10.96.0.10
~~~

~~~powershell
# dig -t a nginx.default.svc.cluster.local. @10.96.0.10

....
;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN  A       10.224.194.75
nginx.default.svc.cluster.local. 30 IN  A       10.224.159.141
nginx.default.svc.cluster.local. 30 IN  A       10.224.126.6

~~~

~~~powershell
# dig -t a web-0.nginx.default.svc.cluster.local. @10.96.0.10
~~~

~~~powershell
在kubernetes集群内创建pod访问
# kubectl run -it busybox --image=radial/busyboxplus
/ # curl nginx.default.svc.cluster.local.
web-0
/ # curl web-0.nginx.default.svc.cluster.local.
web-0
~~~

# 六、已部署应用滚动更新(含金丝雀发布)

>它将按照与 Pod 终止相同的顺序（从最大序号到最小序号）进行，每次更新一个 Pod。
>
>StatefulSet可以使用partition参数来实现金丝雀更新，partition参数可以控制StatefulSet控制器更新的Pod。下面，我们就进行StatefulSet控制器的金丝雀更新实战。

~~~powershell
kubectl patch sts web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
~~~

~~~powershell
说明：
使用patch参数来指定了StatefulSet控制器的partition参数为2，表示当更新时，只有Pod的编号大于等于2的才更新。
~~~

~~~powershell
# kubectl exec -it web-0 -- nginx -v
nginx version: nginx/1.15.12
~~~

~~~powershell
kubectl set image sts/web nginx=nginx:latest
~~~

~~~powershell
kubectl get pods -w
~~~

~~~powershell
# kubectl exec -it web-2 -- nginx -v
nginx version: nginx/1.21.6
~~~

~~~powershell
# kubectl get pods -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name                                     Image
web-0                                    nginx:1.15-alpine
web-1                                    nginx:1.15-alpine
web-2                                    nginx:latest
~~~

**如何实现全部更新呢？**

~~~powershell
kubectl patch sts web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
~~~

~~~powershell
kubectl set image sts/web nginx=nginx:latest
~~~

~~~powershell
# kubectl get pods -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
~~~

# 七、已部署应用扩容与缩容

在StatefulSet扩容时，会创建一个新的Pod，该Pod与之前的所有Pod都是有顺序的，并且新Pod的序号最大。在缩容时，StatefulSet控制器删除的也是序号最大的Pod。

~~~powershell
# kubectl scale sts web --replicas=4
~~~

~~~powershell
# kubectl get pods -w
~~~

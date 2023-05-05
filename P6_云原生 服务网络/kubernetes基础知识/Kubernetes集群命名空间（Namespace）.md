# Kubernetes集群命名空间（Namespace）

# 一、命名空间(namespace)作用

* Namespace是对一组资源和对象的抽象集合.
* 常见的 pod, service, deployment 等都是属于某一个namespace的（默认是 default）
* 不是所有资源都属于namespace，如nodes, persistent volume，namespace 等资源则不属于任何 namespace



# 二、查看namespace

~~~powershell
[root@k8s-master1 ~]# kubectl get namespaces       # namespaces可以简写为namespace或ns
NAME              STATUS   AGE
default           Active   130m			# 所有未指定Namespace的对象都会被默认分配在default命名空间
kube-node-lease   Active   130m			
kube-public       Active   130m			# 此命名空间下的资源可以被所有人访问
kube-system       Active   130m			# 所有由Kubernetes系统创建的资源都处于这个命名空间
~~~

# 三、查看namespace里的资源

**使用`kubectl get all --namespace=命名空间名称`可以查看此命名空间下的所有资源**

~~~powershell
[root@k8s-master1 ~]# kubectl get all --namespace=kube-system
NAME                                             READY   STATUS    RESTARTS   AGE
pod/calico-kube-controllers-7fdc86d8ff-cskfq     1/1     Running   3          5d1h
pod/calico-node-9dpc9                            1/1     Running   2          5d1h
pod/calico-node-jdmxw                            1/1     Running   3          5d1h
pod/calico-node-krwps                            1/1     Running   2          5d1h
pod/calico-node-tttlr                            1/1     Running   2          5d1h
pod/coredns-65dbdb44db-mm7cr                     1/1     Running   2          5d1h
pod/dashboard-metrics-scraper-545bbb8767-q66bc   1/1     Running   2          5d1h
pod/kubernetes-dashboard-65665f84db-nll6k        1/1     Running   4          5d1h
pod/metrics-server-869ffc99cd-8f4jd              1/1     Running   3          5d1h

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
service/dashboard-metrics-scraper   ClusterIP   10.2.246.128   <none>        8000/TCP                 5d1h
service/kube-dns                    ClusterIP   10.2.0.2       <none>        53/UDP,53/TCP,9153/TCP   5d1h
service/kubernetes-dashboard        NodePort    10.2.213.30    <none>        443:21351/TCP            5d1h
service/metrics-server              ClusterIP   10.2.232.121   <none>        443/TCP                  5d1h

NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/calico-node   4         4         4       4            4           beta.kubernetes.io/os=linux   5d1h

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calico-kube-controllers     1/1     1            1           5d1h
deployment.apps/coredns                     1/1     1            1           5d1h
deployment.apps/dashboard-metrics-scraper   1/1     1            1           5d1h
deployment.apps/kubernetes-dashboard        1/1     1            1           5d1h
deployment.apps/metrics-server              1/1     1            1           5d1h

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/calico-kube-controllers-7fdc86d8ff     1         1         1       5d1h
replicaset.apps/coredns-65dbdb44db                     1         1         1       5d1h
replicaset.apps/dashboard-metrics-scraper-545bbb8767   1         1         1       5d1h
replicaset.apps/kubernetes-dashboard-65665f84db        1         1         1       5d1h
replicaset.apps/metrics-server-869ffc99cd              1         1         1       5d1h

~~~

**使用`kubectl get 资源类型 --namespace=命名空间名称`可以查看此命名空间下的对应的资源**

~~~powershell
[root@k8s-master1 ~]# kubectl get pod --namespace=kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7fdc86d8ff-cskfq     1/1     Running   3          5d1h
calico-node-9dpc9                            1/1     Running   2          5d1h
calico-node-jdmxw                            1/1     Running   3          5d1h
calico-node-krwps                            1/1     Running   2          5d1h
calico-node-tttlr                            1/1     Running   2          5d1h
coredns-65dbdb44db-mm7cr                     1/1     Running   2          5d1h
dashboard-metrics-scraper-545bbb8767-q66bc   1/1     Running   2          5d1h
kubernetes-dashboard-65665f84db-nll6k        1/1     Running   4          5d1h
metrics-server-869ffc99cd-8f4jd              1/1     Running   3          5d1h
~~~



# 四、创建namespace 

## 4.1 命令创建

~~~powershell
[root@k8s-master1 ~]# kubectl create namespace ns1
namespace/ns1 created

[root@k8s-master1 ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   5d1h
kube-node-lease   Active   5d1h
kube-public       Active   5d1h
kube-system       Active   5d1h
ns1               Active   10s
~~~



## 4.2 YAML文件创建

* k8s中几乎所有的资源都可以通这YAML编排来创建
* 可以使用`kubectl edit 资源类型 资源名`编辑资源的YAML语法

~~~powershell
[root@k8s-master1 ~]# kubectl edit namespace ns1
......
~~~

* 也可使用`kubectl get 资源类型 资源名 -o yaml`来查看

~~~powershell
[root@k8s-master1 ~]# kubectl get ns ns1 -o yaml
......
~~~

* ==**还可通过`kubectl explain 资源类型`来查看语法文档**== 

~~~powershell
[root@k8s-master1 ~]# kubectl explain namespace				   # 查看namespace相关语法参数
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl explain namespace.metadata	   # 查看namespace下级metadata的相关语法参数
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl explain namespace.metadata.name  # 查看namespace下级metadata再下级name的相关语法参数
~~~



编写创建namespace的YAML文件

~~~powershell
[root@k8s-master1 ~]# vim create_ns2.yml
apiVersion: v1							# api版本号
kind: Namespace							# 类型为namespace
metadata:								# 定义namespace的元数据属性
  name: ns2					    		# 定义name属性为ns2
~~~

使用`kubctl apply -f`应用YAML文件

~~~powershell
[root@k8s-master1 ~]# kubectl apply -f create_ns2.yml
namespace/ns2 created
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   5d2h
kube-node-lease   Active   5d2h
kube-public       Active   5d2h
kube-system       Active   5d2h
ns1               Active   10m
ns2               Active   46s
~~~



# 五、删除namespace

**注意:** 

* 删除一个namespace会自动删除所有属于该namespace的资源(类似mysql中drop库会删除库里的所有表一样，请慎重操作)
* default,kube-system,kube-public命名空间不可删除

## 5.1 命令删除

~~~powershell
[root@k8s-master1 ~]# kubectl delete namespace ns1
namespace "ns1" deleted
~~~



## 5.2 YAML文件删除

~~~powershell
[root@k8s-master1 ~]# kubectl delete -f create_ns2.yml
namespace "ns2" deleted
~~~

~~~powershell
[root@k8s-master1 ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   5d2h
kube-node-lease   Active   5d2h
kube-public       Active   5d2h
kube-system       Active   5d2h
~~~


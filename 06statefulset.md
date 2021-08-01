## Stateful

### StatefulSet的基本概念

```
StatefulSet（有状态集，缩写为sts）常用于部署有状态的且需要有序启动的应用程序,主要用于管理有状态应用程序的工作负载API对象。
比如在生产环境中，可以部署ElasticSearch集群、MongoDB集群或者需要持久化的RabbitMQ集群、Redis集群、Kafka集群和ZooKeeper集群等。
和Deployment类似，一个StatefulSet也同样管理着基于相同容器规范的Pod。不同的是，StatefulSet为每个Pod维护了一个粘性标识。这些Pod是根据相同的规范创建的，但是不可互换，每个Pod都有一个持久的标识符，在重新调度时也会保留，一般格式为StatefulSetName-Number。比如定义一个名字是Redis-Sentinel的StatefulSet，指定创建三个Pod，那么创建出来的Pod名字就为Redis-Sentinel-0、Redis-Sentinel-1、Redis-Sentinel-2。

而StatefulSet创建的Pod一般使用Headless Service（无头服务）进行通信，和普通的Service的区别在于Headless Service没有ClusterIP，它使用的是Endpoint进行互相通信，Headless一般的格式为：
statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local。
说明：
	serviceName为Headless Service的名字，创建StatefulSet时，必须指定Headless Service名称；
	0..N-1为Pod所在的序号，从0开始到N-1；
	statefulSetName为StatefulSet的名字；
	namespace为服务所在的命名空间；
	.cluster.local为Cluster Domain（集群域）。
```

#### 简单案例

```
假如公司某个项目需要在Kubernetes中部署一个主从模式的Redis，此时使用StatefulSet部署就极为合适，因为StatefulSet启动时，只有当前一个容器完全启动时，后一个容器才会被调度，并且每个容器的标识符是固定的，那么就可以通过标识符来断定当前Pod的角色。
比如用一个名为redis-ms的StatefulSet部署主从架构的Redis，第一个容器启动时，它的标识符为redis-ms-0，并且Pod内主机名也为redis-ms-0，此时就可以根据主机名来判断，当主机名为redis-ms-0的容器作为Redis的主节点，其余从节点，那么Slave连接Master主机配置就可以使用不会更改的Master的Headless Service，此时Redis从节点（Slave）配置文件如下：

port 6379
slaveof redis-ms-0.redis-ms.public-service.svc.cluster.local 6379
tcp-backlog 511
timeout 0
tcp-keepalive 0

其中redis-ms-0.redis-ms.public-service.svc.cluster.local是Redis Master的Headless Service，在同一命名空间下只需要写redis-ms-0.redis-ms即可，后面的public-service.svc.cluster.local可以省略
```

### Stateful的注意事项

一般StatefulSet用于有以下一个或者多个需求的应用程序：

- 需要稳定的独一无二的网络标识符。

- 需要持久化数据。

- 需要有序的、优雅的部署和扩展。

- 需要有序的自动滚动更新。

```
如果应用程序不需要任何稳定的标识符或者有序的部署、删除或者扩展，应该使用无状态的控制器部署应用程序，比如Deployment或者ReplicaSet。
StatefulSet是Kubernetes 1.9版本之前的beta资源，在1.5版本之前的任何Kubernetes版本都没有。
Pod所用的存储必须由PersistentVolume Provisioner（持久化卷配置器）根据请求配置StorageClass，或者由管理员预先配置，当然也可以不配置存储。
为了确保数据安全，删除和缩放StatefulSet不会删除与StatefulSet关联的卷，可以手动选择性地删除PVC和PV
StatefulSet目前使用Headless Service（无头服务）负责Pod的网络身份和通信，需要提前创建此服务。
删除一个StatefulSet时，不保证对Pod的终止，要在StatefulSet中实现Pod的有序和正常终止，可以在删除之前将StatefulSet的副本缩减为0。
```

### 定义一个Stateful

```
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
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
        image: nginx
        ports:
        - containerPort: 80
          name: web
```

简单解释

- kind: Service定义了一个名字为Nginx的Headless Service，创建的Service格式为nginx-0.nginx.default.svc.cluster.local，其他的类似，因为没有指定Namespace（命名空间），所以默认部署在default。

- kind: StatefulSet定义了一个名字为web的StatefulSet，replicas表示部署Pod的副本数，本实例为2

```
在StatefulSet中必须设置Pod选择器（.spec.selector）用来匹配其标签（.spec.template.metadata.labels）。在1.8版本之前，如果未配置该字段（.spec.selector），将被设置为默认值，在1.8版本之后，如果未指定匹配Pod Selector，则会导致StatefulSet创建错误。
当StatefulSet控制器创建Pod时，它会添加一个标签statefulset.kubernetes.io/pod-name，该标签的值为Pod的名称，用于匹配Service
```

#### 创建

会等待所有的pod启动完才会进行其他的更新操作，web-0启动完才启动web-1

```
[root@k8s-master01 yaml]# kubectl create -f nginx-sts.yaml 
service/nginx created
statefulset.apps/web created
查看service
[root@k8s-master01 yaml]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   26d
nginx        ClusterIP   None         <none>        80/TCP    18m
查看pod，pod的格式为sts-0/1/2...
[root@k8s-master01 yaml]# kubectl get pod -owide
NAME      READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
busybox   1/1     Running   622        26d   172.18.195.2    k8s-master03   <none>           <none>
web-0     1/1     Running   0          19m   172.27.14.212   k8s-node02     <none>           <none>
web-1     1/1     Running   0          18m   172.17.125.26   k8s-node01     <none>           <none>
增加副本
[root@k8s-master01 yaml]# kubectl scale --replicas=3 sts web
statefulset.apps/web scaled
[root@k8s-master01 yaml]# kubectl get pod 
NAME      READY   STATUS              RESTARTS   AGE
busybox   1/1     Running             622        26d
web-0     1/1     Running             0          21m
web-1     1/1     Running             0          20m
web-2     0/1     ContainerCreating   0          6s

```

#### 解析无头service

```
创建busybox
cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
[root@k8s-master01 yaml]# kubectl exec -it busybox -n default -- sh
#格式为statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local
/ # nslookup web-0.nginx.default  
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx.default
Address 1: 172.27.14.212 web-0.nginx.default.svc.cluster.local
/ # ping web-2.nginx
PING web-2.nginx (172.17.125.27): 56 data bytes
64 bytes from 172.17.125.27: seq=0 ttl=62 time=0.491 ms
64 bytes from 172.17.125.27: seq=1 ttl=62 time=0.394 ms
^C
--- web-2.nginx ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.394/0.442/0.491 ms

```

###  StatefulSet扩容缩容

扩容更新主要是有顺序的，按照顺序上一个启动完成之后才会启动下一个

```
扩容
[root@k8s-master01 ~]#  kubectl scale --replicas=5 sts web
statefulset.apps/web scaled
更新策略是等待一个完全启动完成才会去启动下一个
[root@k8s-master01 ~]# kubectl get pod -w
NAME      READY   STATUS              RESTARTS   AGE
busybox   1/1     Running             635        26d
web-0     1/1     Running             0          13h
web-1     1/1     Running             0          13h
web-2     1/1     Running             0          12h
web-3     1/1     Running             0          23s
web-4     0/1     ContainerCreating   0          0s
web-4     1/1     Running             0          24s
```

缩容是按照从大到小的顺序进行的减少，一旦中途有小顺序的被删除(web-0),那么会停止缩容，创建那个被删除的，等待创建完毕之后继续删除。

```
缩容
[root@k8s-master01 ~]#  kubectl scale --replicas=2 sts web
statefulset.apps/web scaled
中途删除pod web-0
[root@k8s-master01 ~]# kubectl get pod -w
web-4     1/1     Terminating         0          4m37s
web-4     0/1     Terminating         0          4m38s
web-4     0/1     Terminating         0          4m46s
web-4     0/1     Terminating         0          4m46s
web-3     1/1     Terminating         0          5m9s
web-3     0/1     Terminating         0          5m10s
web-0     1/1     Terminating         0          13h
web-3     0/1     Terminating         0          5m11s
web-3     0/1     Terminating         0          5m11s
web-0     0/1     Terminating         0          13h
web-0     0/1     Terminating         0          13h
web-0     0/1     Terminating         0          13h
web-0     0/1     Pending             0          0s
web-0     0/1     Pending             0          0s
web-0     0/1     ContainerCreating   0          0s
web-0     1/1     Running             0          26s
web-2     1/1     Terminating         0          12h
web-2     0/1     Terminating         0          12h
web-2     0/1     Terminating         0          12h
web-2     0/1     Terminating         0          12h
查看
[root@k8s-master01 ~]# kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   635        26d
web-0     1/1     Running   0          2m42s
web-1     1/1     Running   0          13h

上边执行缩容，显示web-4先被删除，接着web-3,接着手动删除了pod web-0,这时候会停止删除web-2,先创建web-0,等web-0创建成功之后，接着删除web-2

```

###  Statefulset的更新策略

- On Delete策略

  ```
  OnDelete更新策略实现了传统（1.7版本之前）的行为，它也是默认的更新策略。当我们选择这个更新策略并修改StatefulSet的.spec.template字段时，StatefulSet控制器不会自动更新Pod，我们必须手动删除Pod才能使控制器创建新的Pod。
  ```

  

-  RollingUpdate策略

```
RollingUpdate（滚动更新）更新策略会更新一个StatefulSet中所有的Pod，采用与序号索引相反的顺序进行滚动更新。
```

我们先来看 RollingUpdate的策略

```
查看目前sts的策略(目前默认是)
[root@k8s-master01 yaml]# kubectl get sts web -oyaml|grep -C 3 -i str
            f:schedulerName: {}
            f:securityContext: {}
            f:terminationGracePeriodSeconds: {}
        f:updateStrategy:
          f:rollingUpdate:
            .: {}
            f:partition: {}
--
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
```

接着进行更新

```
[root@k8s-master01 yaml]# kubectl edit sts web
statefulset.apps/web edited
更新nginx镜像为1.9.1
查看更新过程，会从序号大的开始进行更新依次进行，在当前顺序变成Running和Ready状态之前，StatefulSet控制器不会更新下一个Pod，但它仍然会重建任何在更新过程中发生故障的Pod，使用它们当前的版本
[root@k8s-master01 yaml]# kubectl get pod -w
NAME      READY   STATUS              RESTARTS   AGE
busybox   1/1     Running             641        26d
web-0     1/1     Running             0          6h
web-1     0/1     ContainerCreating   0          7s
web-1     1/1     Running             0          29s
web-0     1/1     Terminating         0          6h
web-0     0/1     Terminating         0          6h
web-0     0/1     Terminating         0          6h
web-0     0/1     Terminating         0          6h
web-0     0/1     Pending             0          0s
web-0     0/1     Pending             0          0s
web-0     0/1     ContainerCreating   0          0s
web-0     1/1     Running             0          23s
或者这样查看更新过程
 kubectl rollout status sts web
```

ONDELETE策略

```
修改更新策略为ondelete
  updateStrategy:
    type: OnDelete
通过命令kubectl edit sts web修改镜像为最新的nginx:latest
查看pod变化(没有任何变化)
[root@k8s-master01 yaml]# kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   641        26d
web-0     1/1     Running   0          4m5s
web-1     1/1     Running   0          4m11s
web-2     1/1     Running   0          9s
web-3     1/1     Running   0          6s
删除一个pod
[root@k8s-master01 yaml]# kubectl delete pod web-1
pod "web-1" deleted
查看pod的镜像,为更新后的镜像
[root@k8s-master01 yaml]# kubectl get pod web-1 -oyaml |grep image
            f:image: {}
            f:imagePullPolicy: {}
  - image: nginx:latest
    imagePullPolicy: IfNotPresent
    image: nginx:latest
    imageID: docker-pullable://nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90

```

### 灰度发布分段更新

```
StatefulSet可以使用RollingUpdate更新策略的partition参数来分段更新一个StatefulSet。分段更新将会使StatefulSet中其余的所有Pod（序号小于分区）保持当前版本，只更新序号大于等于分区的Pod，利用此特性可以简单实现金丝雀发布（灰度发布）或者分阶段推出新功能等
```

修改更新策略

```
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'

目前设置partition为3，所有只有大于等于3的序号才会更新
设置镜像为image: nginx:1.9.1
查看更新
[root@k8s-master01 yaml]# kubectl get pod web-2 -oyaml|grep image
            f:image: {}
            f:imagePullPolicy: {}
  - image: nginx:latest
    imagePullPolicy: IfNotPresent
    image: nginx:latest
    imageID: docker-pullable://nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90
[root@k8s-master01 yaml]# kubectl get pod web-3 -oyaml|grep image
            f:image: {}
            f:imagePullPolicy: {}
  - image: nginx:1.9.1
    imagePullPolicy: IfNotPresent
    image: nginx:1.9.1
    imageID: docker-pullable://nginx@sha256:2f68b99bc0d6d25d0c56876b924ec20418544ff28e1fb89a4c27679a40da811b
[root@k8s-master01 yaml]# kubectl get pod web-4 -oyaml|grep image
            f:image: {}
            f:imagePullPolicy: {}
  - image: nginx:1.9.1
    imagePullPolicy: IfNotPresent
    image: nginx:1.9.1
    imageID: docker-pullable://nginx@sha256:2f68b99bc0d6d25d0c56876b924ec20418544ff28e1fb89a4c27679a40da811b
    
发现只有web-3 web-4镜像进行了更新
```

### 级联删除与非级联删除

使用非级联方式删除StatefulSet时，StatefulSet的Pod不会被删除；使用级联删除时，StatefulSet和它的Pod都会被删除

- 非级联删除

使用kubectldeletestsxxx删除StatefulSet时，只需提供--cascade=false参数，就会采用非级联删除，此时删除StatefulSet不会删除它的Pod

```
[root@k8s-master01 yaml]# kubectl delete sts web --cascade=false
warning: --cascade=false is deprecated (boolean value) and can be replaced with --cascade=orphan.
statefulset.apps "web" deleted
You have new mail in /var/spool/mail/root
[root@k8s-master01 yaml]# kubectl get po
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   642        26d
web-0     1/1     Running   0          36m
web-1     1/1     Running   0          49m
web-2     1/1     Running   0          36m
web-3     1/1     Running   0          36m
web-4     1/1     Running   0          36m
已经被删除
[root@k8s-master01 yaml]# kubectl get sts 
No resources found in default namespace.
由于此时删除了StatefulSet，因此单独删除Pod时，不会被重建：
[root@k8s-master01 yaml]# kubectl delete pod web-3
pod "web-3" deleted
[root@k8s-master01 yaml]# kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   642        26d
web-0     1/1     Running   0          38m
web-1     1/1     Running   0          51m
web-2     1/1     Running   0          39m
web-4     1/1     Running   0          38m
```

当再次创建此StatefulSet时，web-3会被重新创建，其他pod由于已经存在而不会被再次创建，因为最初此StatefulSet的replicas是4，所以web-4会被删除，如下（忽略AlreadyExists错误）：

```
[root@k8s-master01 yaml]# kubectl create -f nginx-sts.yaml 
statefulset.apps/web created
Error from server (AlreadyExists): error when creating "nginx-sts.yaml": services "nginx" already exists
[root@k8s-master01 yaml]# kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   642        26d
web-0     1/1     Running   0          42m
web-1     1/1     Running   0          54m
web-2     1/1     Running   0          42m
web-3     1/1     Running   0          7s

```

- 级联删除

省略--cascade=false参数即为级联删除：

```
[root@k8s-master01 2.2.7]# kubectl delete statefulset web
statefulset.apps "web" deleted
[root@k8s-master01 2.2.7]# kubectl get po
No resources found.
也可以这样删除
[root@k8s-master01 2.2.7]# kubectl delete -f sts-web.yaml 
service "nginx" deleted
```


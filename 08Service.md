## Service

### service简介

Service这样一种抽象概念：逻辑上的一组Pod，即一种可以访问Pod的策略——通常称为微服务。这一组Pod能够被Service访问到，通常是通过Label Selector（标签选择器）实现的。Service主要用于Pod之间的通信，对于Pod的IP地址而言，Service是提前定义好并且是不变的资源类型

### 创建一个service

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: nginx  #选择的pod
  ports:
  - protocol: TCP   #后端协议tcp udp stcp 默认tcp
    port: 80     # service的端口
    targetPort: 80   #应用的端口
     
     
-------------Deployment---------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
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
        image: nginx:latest
        ports:
        - containerPort: 80
        imagePullPolicy: IfNotPresent   
      
```

上述配置创建一个名为my-service的Service对象，它会将请求代理到TCP端口为80并且具有标签app=nginx的Pod上。这个Service会被分配一个IP地址，通常称为ClusterIP，它会被服务的代理使用。

需要注意的是，Service能够将一个接收端口映射到任意的targetPort。默认情况下，targetPort将被设置为与Port字段相同的值。targetPort可以设置为一个字符串，引用backend Pod的一个端口的名称

**对于Kubernetes集群中的应用，Kubernetes提供了简单的Endpoints API，只要Service中的一组Pod发生变更，应用程序就会被更新** ,创建service的同时会创建同名Endpoints 

```
[root@k8s-master01 yaml]# kubectl get ep
NAME         ENDPOINTS                                                     AGE
kubernetes   192.168.89.250:6443,192.168.89.251:6443,192.168.89.252:6443   30d
my-service   172.17.125.41:80,172.25.92.84:80  
删除pod之后，ep会随之变化
[root@k8s-master01 yaml]# kubectl delete pod  nginx-deployment-75b69bd684-x2g5r
pod "nginx-deployment-75b69bd684-x2g5r" deleted
[root@k8s-master01 yaml]# kubectl get ep
NAME         ENDPOINTS                                                     AGE
kubernetes   192.168.89.250:6443,192.168.89.251:6443,192.168.89.252:6443   30d
my-service   172.25.92.84:80,172.27.14.223:80                              18m
查看service
[root@k8s-master01 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    32d
my-service   ClusterIP   10.106.72.25   <none>        8080/TCP   47h
访问service  curl  10.106.72.25:8080   得到的结果跟访问pod一样
curl  172.25.92.84:80
```

### 定义没有Selector的Service

#### 使用Service代理k8s外部应用

- 希望在生产环境中访问外部的数据库集群。

- 希望Service指向另一个NameSpace中或其他集群中的服务。

-  正在将工作负载转移到Kubernetes集群，和运行在Kubernetes集群之外的backend。

定义如下

```
kind: Service
apiVersion: v1
metadata:
  metadata:
  labels:
    app: nginx-svc-external
  name: nginx-svc-external
spec:
  ports:
  - name: http 
    protocol: TCP   
    port: 8080    
    targetPort: 80
  type: ClusterIP  
      
```

service启动之后不会创建同名Endpoints 

```
[root@k8s-master01 yaml]# kubectl create -f nginx-svc-exteral.yaml 
service/nginx-svc-external created
[root@k8s-master01 yaml]# kubectl get svc
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes           ClusterIP   10.96.0.1       <none>        443/TCP    32d
my-service           ClusterIP   10.106.72.25    <none>        8080/TCP   47h
nginx-svc-external   ClusterIP   10.107.208.65   <none>        8080/TCP   6s
[root@k8s-master01 yaml]# kubectl get ep
NAME         ENDPOINTS                                                     AGE
kubernetes   192.168.89.250:6443,192.168.89.251:6443,192.168.89.252:6443   32d
my-service   172.25.92.84:80,172.27.14.223:80                              47h

```

自己创建ep

```
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: nginx-svc-external  #这个必须跟service一样
  name: nginx-svc-external    #这个必须跟service一样
  namespace: default
subsets:
- addresses:
  - ip: 220.181.38.251    #代理的ip地址  这里使用的本机解析的百度的ip地址
  ports:
  - name: http   #更service一样
    port: 80
    protocol: TCP  #更service一样
创建
[root@k8s-master01 yaml]# kubectl create -f nginx-ep-external.yaml 
endpoints/nginx-svc-external created
[root@k8s-master01 yaml]# kubectl get ep
NAME                 ENDPOINTS                                                     AGE
kubernetes           192.168.89.250:6443,192.168.89.251:6443,192.168.89.252:6443   32d
my-service           172.25.92.84:80,172.27.14.223:80                              47h
nginx-svc-external   220.181.38.251:80                                             5s
```

访问测试

```
[root@k8s-master01 yaml]# curl 220.181.38.251:80  -I
HTTP/1.1 200 OK
Date: Fri, 06 Aug 2021 13:47:46 GMT
Server: Apache
Last-Modified: Tue, 12 Jan 2010 13:48:00 GMT
ETag: "51-47cf7e6ee8400"
Accept-Ranges: bytes
Content-Length: 81
Cache-Control: max-age=86400
Expires: Sat, 07 Aug 2021 13:47:46 GMT
Connection: Keep-Alive
Content-Type: text/html

访问service   结果一样
[root@k8s-master01 yaml]# curl 10.96.104.244:8080 -I
HTTP/1.1 200 OK
Date: Fri, 06 Aug 2021 13:49:16 GMT
Server: Apache
Last-Modified: Tue, 12 Jan 2010 13:48:00 GMT
ETag: "51-47cf7e6ee8400"
Accept-Ranges: bytes
Content-Length: 81
Cache-Control: max-age=86400
Expires: Sat, 07 Aug 2021 13:49:16 GMT
Connection: Keep-Alive
Content-Type: text/html
```

### 使用Service反代域名

ExternalName Service是Service的特例，它没有Selector，也没有定义任何端口和Endpoint，它通过返回该外部服务的别名来提供服务。

比如当查询主机my-service.prod.svc时，集群的DNS服务将返回一个值为my.database.example.com的CNAME记录：

```
# cat nginx-externalName.yaml 

apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-externalname
  name: nginx-externalname
spec:
  type: ExternalName    #类型不一样
  externalName: www.baidu.com
  
创建
[root@k8s-master01 yaml]# kubectl create -f nginx-externalName.yaml 
service/nginx-externalname created
查看
[root@k8s-master01 yaml]# kubectl get svc nginx-externalname
NAME                 TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)   AGE
nginx-externalname   ExternalName   <none>       www.baidu.com   <none>    20s

简单验证
[root@k8s-master01 yaml]# kubectl exec -it busybox -- sh
/ # nslookup nginx-externalname
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      nginx-externalname
Address 1: 110.242.68.3   解析出百度的访问地址
Address 2: 110.242.68.4
/ # wget nginx-externalname
Connecting to nginx-externalname (180.101.49.11:80)
wget: server returned error: HTTP/1.1 403 Forbidden  #跨域导致的
/ # You have ne外部w mail in /var/spool/mail/root

```

### 外部访问service

使用NodePort

```
NodePort：在所有安装了kube-proxy的节点上打开一个端口，此端口可以代理至后端Pod，然后集群外部可以使用节点的IP地址和NodePort的端口号访问到集群Pod的服务。NodePort端口范围默认是30000-32767  apiserver的配置文件定义的cat /usr/lib/systemd/system/kube-apiserver.service
```

```
创建svc与pod并且关联
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: nginx  #选择的pod
  ports:
  - protocol: TCP   #后端协议tcp udp stcp 默认tcp
    port: 80     # service的端口
    targetPort: 80   #应用的端口
     
     
-------------Deployment---------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
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
        image: nginx:latest
        ports:
        - containerPort: 80
        imagePullPolicy: IfNotPresent   
```

查看

```
[root@k8s-master01 yaml]# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP    19h
my-service   ClusterIP   10.96.55.33   <none>        8080/TCP   2m5s
[root@k8s-master01 yaml]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
busybox                             1/1     Running   18         18h
nginx-deployment-75b69bd684-j5l9d   1/1     Running   0          8m5s
nginx-deployment-75b69bd684-stksx   1/1     Running   0          8m5s
[root@k8s-master01 yaml]# curl 10.96.55.33:8080 -I
HTTP/1.1 200 OK
Server: nginx/1.21.1
Date: Sun, 05 Sep 2021 08:31:39 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 06 Jul 2021 14:59:17 GMT
Connection: keep-alive
ETag: "60e46fc5-264"
Accept-Ranges: bytes

修改为NodePort并且指定端口
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
    nodePort: 31000
  selector:
    app: nginx
  sessionAffinity: None
  type: NodePort
  
  查看并且访问测试
  [root@k8s-master01 yaml]# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP          19h
my-service   NodePort    10.96.55.33   <none>        8080:31000/TCP   7m51s
[root@k8s-master01 yaml]# hostname -I
192.168.10.180 192.168.10.60 172.25.244.192 172.17.0.1 
[root@k8s-master01 yaml]# curl 192.168.10.180:31000 -I
HTTP/1.1 200 OK
Server: nginx/1.21.1
Date: Sun, 05 Sep 2021 08:38:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 06 Jul 2021 14:59:17 GMT
Connection: keep-alive
ETag: "60e46fc5-264"
Accept-Ranges: bytes

```



### 总结

#### service的类型

```
ClusterIP：在集群内部使用，也是默认值。
ExternalName：通过返回定义的CNAME别名。
NodePort：在所有安装了kube-proxy的节点上打开一个端口，此端口可以代理至后端Pod，然后集群外部可以使用节点的IP地址和NodePort的端口号访问到集群Pod的服务。NodePort端口范围默认是30000-32767。
LoadBalancer：使用云提供商的负载均衡器公开服务。

```




















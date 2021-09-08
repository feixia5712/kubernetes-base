## Secret

### 命令行创建secret

```
[root@k8s-master01 secret]# ll
总用量 8
-rw-r--r-- 1 root root 7 9月   7 21:04 pasword.txt
-rw-r--r-- 1 root root 6 9月   7 21:04 username.txt
[root@k8s-master01 secret]# cat pasword.txt 
123456
您在 /var/spool/mail/root 中有新邮件
[root@k8s-master01 secret]# cat username.txt 
admin
kubectl create secret generic mysecret01 --from-file=./username.txt --from-file=./password.txt
[root@k8s-master01 secret]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
mysecret01            Opaque                                2      8s
账号密码被使用base64加密了
[root@k8s-master01 secret]# kubectl get secret mysecret01 -oyaml
apiVersion: v1
data:
  password.txt: MTIzNDU2Cg==
  username.txt: YWRtaW4K
kind: Secret
metadata:
  creationTimestamp: "2021-09-07T13:06:12Z"
  name: mysecret01
  namespace: default
  resourceVersion: "615468"
  uid: 08ec984f-b556-49ed-aa6e-aad80d2136da
type: Opaque
[root@k8s-master01 secret]# echo "MTIzNDU2Cg=="|base64 -d
123456

```

### 直接创建

```
kubectl create secret generic db-user-pass --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='  #必须使用单引号
```

### 基于yaml文件创建

```
[root@k8s-master01 secret]# echo -n 'admin' | base64
YWRtaW4=
[root@k8s-master01 secret]# echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
[root@k8s-master01 secret]# kubectl create -f secret.yaml 
[root@k8s-master01 secret]# cat secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

r如果不想加密之后写入yaml，直接写入明文也是可以的,使用stringData

```
[root@k8s-master01 secret]# cat secret01.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysecret08
type: Opaque
stringData:
  username: administrator
[root@k8s-master01 secret]# kubectl create -f secret01.yaml 
查看username被加密了
[root@k8s-master01 secret]# kubectl get secret mysecret08 -oyaml
apiVersion: v1
data:
  username: YWRtaW5pc3RyYXRvcg==
kind: Secret
metadata:
  creationTimestamp: "2021-09-07T13:18:10Z"
  name: mysecret08
  namespace: default
  resourceVersion: "617205"
  uid: 519fd49c-d732-4ec6-8283-406aac9f8af2
type: Opaque
```

### docker下载镜像配置密码

```
[root@k8s-master01 secret]# kubectl create secret docker-registry secret-tiger-docker --docker-username=admin --docker-password='123456'  --docker-email=admin@jovision.com --docker-server=http://192.168.10.11:8009
[root@k8s-master01 secret]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
secret-tiger-docker   kubernetes.io/dockerconfigjson        1      37s
配置pod引入
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dp-cm
  name: dp-cm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dp-cm
  template:
    metadata:
      labels:
        app: dp-cm
    spec:
      imagePullSecrets:
      - name: secret-tiger-docker  #引入账号密码
      containers:
      - image: nginx:latest
        name: nginx
```

### 证书secret创建

```
kubectl create secret tls my-tls-secret --cert=path/to/cert/file --key=path/to/key/file
Ingress-nginx配置引入证书
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  namespace: default
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 8080
  tls:#属于spec下边的
  - secretName: my-tls-secret

一般生产环境情况下，证书会配置在slb上，一般不会配置到ingress
```

### subPath解决挂载覆盖问题

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dp-cm
  name: dp-cm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dp-cm
  template:
    metadata:
      labels:
        app: dp-cm
    spec:
      imagePullSecrets:
      - name: secret-tiger-docker
      containers:
      - image: nginx:latest
        name: nginx
        volumeMounts:
          - name: nginx-conf  #必须跟volumes的名字一样
            mountPath: /etc/nginx/nginx.conf  #写清楚具体挂载的路径文件
            subPath: nginx.conf  #以文件的形式挂载,只是挂载一个文件,这样避免覆盖问题
        env:
        - name: TEST_ENV
          value: testenv
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-conf
          
 以上执行后，只是会替换nginx.conf         
```

### 热更新ConfigMap或者secret

```
第一种是yaml文件生成的，直接编辑replace即可
第二种文件形式创建的configMap
例如
game.conf被修改了，我们如何进行热更新
如下
kubectl create cm cmfromfile --from-file=game.conf --dry-run=client -oyaml|kubectl replace -f -
查看已经被更新
[root@k8s-master01 conf]# kubectl get cm cmfromfile -oyaml
apiVersion: v1
data:
  game.conf: |
    name='zdk'
    code.status=true
    auth=jvs
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-06T12:51:20Z"
  name: cmfromfile
  namespace: default
  resourceVersion: "624518"
  uid: 9d6810de-7d7c-403c-84e0-48a3c9692af3

```

### ConfigMap OR Secret使用限制

```
提前场景ConfigMap和Secret
引用Key必须存在
envFrom、valueFrom无法热更新环境变量
envFrom配置环境变量，如果key是无效的，它会忽略掉无效的key
ConfigMap和Secret必须要和Pod或者是引用它资源在同一个命名空间
subPath也是无法热更新的
ConfigMap和Secret最好不要太大
```
### v1.19之后新增不可变ConfigMap/Secret

```
新增了一个参数immutable: true 加上这个参数后，保证文件不能被修改
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable: true
```

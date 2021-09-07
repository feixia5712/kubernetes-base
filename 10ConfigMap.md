## ConfigMap

### 简介

ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中。使用时， [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。ConfigMap 在设计上不是用来保存大量数据的。在 ConfigMap 中保存的数据不可超过 1 MiB，pod不能跨命名空间去引用ConfigMap，是有隔离性的。

### ConfigMap创建的几种形式

- 基于目录创建

```
mkdir conf
创建两个配置文件 (key:value形式)
[root@k8s-master01 conf]# cat game.conf 
name='zdk'
code.status=true
[root@k8s-master01 conf]# cat game.conf 
name='zdk'
code.status=true
直接创建
kubectl create cm cmfromdir --from-file=conf -n default
查看
kubectl create cm cmfromdir --from-file=conf
[root@k8s-master01 conf]# kubectl get cm cmfromdir -oyaml
apiVersion: v1
data:
  game.conf: |
    name='zdk'
    code.status=true
  game01.conf: |
    color.good='purple'
    core.core=0
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-06T12:22:35Z"
  name: cmfromdir
  namespace: default
  resourceVersion: "400630"
  uid: b8898c0e-c06a-444f-8b53-0835543a3822

```

- 基于文件创建

```
[root@k8s-master01 conf]# kubectl create cm cmfromfile --from-file=game.conf
[root@k8s-master01 conf]# kubectl get cm cmfromfile -oyaml
apiVersion: v1
data:
  game.conf: |
    name='zdk'
    code.status=true
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-06T12:51:20Z"
  name: cmfromfile
  namespace: default
  resourceVersion: "404787"
  uid: 9d6810de-7d7c-403c-84e0-48a3c9692af3

自定义名称
kubectl create cm cmfromfile01 --from-file=game-conf=game.conf
[root@k8s-master01 conf]# kubectl get cm cmfromfile01 -oyaml
apiVersion: v1
data:
  game-conf: |   #名称变了
    name='zdk'
    code.status=true
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-06T12:56:28Z"
  name: cmfromfile01
  namespace: default
  resourceVersion: "405531"
  uid: 4f9b64b1-18bb-4e7c-8bb0-fe9ce117d5cd
```

- 基于环境变量创建

```
[root@k8s-master01 conf]# kubectl create cm gamecmenv --from-env-file=game.conf 
configmap/gamecmenv created
[root@k8s-master01 conf]# kubectl get cm gamecmenv -oyaml
apiVersion: v1
data:
  code.status: "true"  #注意跟基于文件创建区别
  name: '''zdk'''
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-06T12:59:44Z"
  name: gamecmenv
  namespace: default
  resourceVersion: "406007"
  uid: 064a80e6-62d9-4c25-8ee5-765c6f88dbb0

```

- 基于literal创建

```
[root@k8s-master01 conf]# kubectl create cm cmliteral --from-literal=Passwd=123 --from-literal=Name=alex
configmap/cmliteral created
[root@k8s-master01 conf]# kubectl get cm cmliteral -oyaml
apiVersion: v1
data:
  Name: alex
  Passwd: "123"
kind: ConfigMap
metadata:
  creationTimestamp: "2021-09-06T13:03:05Z"
  name: cmliteral
  namespace: default
  resourceVersion: "406488"
  uid: 81f6a1a6-8c1f-4810-b914-abbe26dda6ac
```

- 基于yaml文件创建

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true   
    
kubectl create cm -f xx.yaml
```

### ConfigMap的使用

- valueFrom 引入ConfigMap

```
kubectl create deployment dp-cm --image=nginx:latest --dry-run=client -oyaml > dp-cm.yaml
[root@k8s-master01 conf]# cat dp-cm.yaml 
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
      containers:
      - image: nginx:latest
        name: nginx
        env:
        - name: TEST_ENV  #自己定义的环境变量
          value: testenv
        - name: NAME  #自定义名称，引用configmap
          valueFrom:
            configMapKeyRef:
              name: gamecmenv #confimap名称
              key: name  # comfigmap里面的变量name的值,会赋值给上边定义的NAMAE
```

登录容器查看

```
[root@k8s-master01 conf]# kubectl exec dp-cm-7f9888874d-26r7b  it -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=dp-cm-7f9888874d-26r7b
NAME='zdk'   #获取到
TEST_ENV=testenv  #自己定义的
MY_SERVICE_PORT=tcp://10.96.55.33:8080
KUBERNETES_SERVICE_HOST=10.96.0.1
```

- 多个变量引入envForm

```
[root@k8s-master01 conf]# cat dp-cm.yaml 
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
      containers:
      - image: nginx:latest
        name: nginx
        envFrom:
        - configMapRef:
            name: gamecmenv
          prefix: fromCm_  #会在gamecmenv的变量的前边加上这个标识
        env:
        - name: TEST_ENV
          value: testenv
这样会把gamecmenv的configMap里面的key:value都引入到pod中

```

查看

```
[root@k8s-master01 conf]# kubectl exec dp-cm-5f548bc59d-ftp9m it -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=dp-cm-5f548bc59d-ftp9m
fromCm_code.status=true   #有刚才定义的前缀
fromCm_name='zdk'
TEST_ENV=testenv
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_PORT=443
MY_SERVICE_PORT=tcp://10.96.55.33:8080
MY_SERVICE_PORT_8080_TCP_PORT=8080
MY_SERVICE_SERVICE_HOST=10.96.55.33
MY_SERVICE_SERVICE_PORT=8080
MY_SERVICE_PORT_8080_TCP=tcp://10.96.55.33:8080
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
MY_SERVICE_PORT_8080_TCP_PROTO=tcp

```

- 以文件形式挂载

```
kubectl create cm redis-conf --from-file=redis.conf

[root@k8s-master01 conf]# cat dp-cm.yaml 
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
      containers:
      - image: nginx:latest
        name: nginx
        volumeMounts:
          - name: redisconf  #必须跟volumes的名字一样
            mountPath: /etc/config
        env:
        - name: TEST_ENV
          value: testenv
      volumes:
      - name: redisconf
        configMap:
          name: redis-conf
[root@k8s-master01 conf]# 
[root@k8s-master01 conf]# kubectl get cm
NAME               DATA   AGE
redis-conf         1      4m55s
查看
[root@k8s-master01 conf]# kubectl exec -it dp-cm-6f9955dc8f-gc5pt -- ls /etc/config/
redis.conf
修改的化，可以直接编辑cm，修改后会直接生效的
[root@k8s-master01 conf]# kubectl edit cm redis-conf

```

- 自定义挂载权限以及名称

```
自定义挂载的文件名称以及权限
[root@k8s-master01 conf]# cat dp-cm.yaml 
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
      containers:
      - image: nginx:latest
        name: nginx
        volumeMounts:
          - name: redisconf  #必须跟volumes的名字一样
            mountPath: /etc/config
          - name: cmfromfile
            mountPath: /etc/config2
        env:
        - name: TEST_ENV
          value: testenv
      volumes:
      - name: redisconf
        configMap:
          name: redis-conf
      - name: cmfromfile
        configMap:
          name: cmfromfile
          items:
          - key: game.conf #文件名称,创建configMap时指定的文件名称--from-file
            path: game.conf_bak #自定义挂载后容器内的文件名称
            mode: 0644 #定义的权限
          - key: game.conf
            path: game.conf_bak01
          defaultMode: 0666 #默认的权限
查看

[root@k8s-master01 conf]# kubectl exec -it dp-cm-78756b96c9-zpgrh -- bash
root@dp-cm-78756b96c9-zpgrh:/# cd /etc/config2
root@dp-cm-78756b96c9-zpgrh:/etc/config2# ls -l
total 0
lrwxrwxrwx 1 root root 20 Sep  7 12:17 game.conf_bak -> ..data/game.conf_bak
lrwxrwxrwx 1 root root 22 Sep  7 12:17 game.conf_bak01 -> ..data/game.conf_bak01
root@dp-cm-78756b96c9-zpgrh:/etc/config2# cd ..data
root@dp-cm-78756b96c9-zpgrh:/etc/config2/..data# ls -l
total 8
-rw-r--r-- 1 root root 28 Sep  7 12:17 game.conf_bak
-rw-rw-rw- 1 root root 28 Sep  7 12:17 game.conf_bak01
root@dp-cm-78756b96c9-zpgrh:/etc/config2/..data# 
```


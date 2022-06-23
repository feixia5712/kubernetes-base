## Deployment

###  简介

用于部署无状态的服务，这个最常用的控制器。一般用于管理维护企业内部无状态的微服务，比如configserver、zuul、springboot。他可以管理多个副本的Pod实现无缝迁移、自动扩容缩容、自动灾难恢复、一键回滚等功能。

### 创建一个Deployment

命令行直接创建

```
kubectl create deployment nginx --image=nginx:1.15.2
```

利用文件创建

```
# cat nginx-deploy.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2020-09-19T02:41:11Z"
  generation: 1
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 3 #副本数
  revisionHistoryLimit: 10 # 历史记录保留的个数
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:  #更新策略
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

```
kubectl create -f nginx-deploy.yaml --record
```

### 查看相关状态

```
查看deploy状态
# kubectl get deploy -owide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
nginx   2/2     2            2           9m29s   nginx        nginx:1.15.2   app=nginx	
```

字段解释

- `NAME` 列出了集群中 Deployment 的名称。
- `READY` 显示应用程序的可用的 *副本* 数。显示的模式是“就绪个数/期望个数”。
- `UP-TO-DATE` 显示为了达到期望状态已经更新的副本数。
- `AVAILABLE` 显示应用可供用户使用的副本数。
- `AGE` 显示应用程序运行的时间。
-  CONTAINERS：容器名称
-  IMAGES：容器的镜像
-  SELECTOR：管理的Pod的标签

```
要查看 Deployment 创建的 ReplicaSet
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s
```

- `NAME` 列出名字空间中 ReplicaSet 的名称；
- `DESIRED` 显示应用的期望副本个数，即在创建 Deployment 时所定义的值。 此为期望状态；
- `CURRENT` 显示当前运行状态中的副本个数；
- `READY` 显示应用中有多少副本可以为用户提供服务；
- `AGE` 显示应用已经运行的时间长度。

注意 ReplicaSet 的名称始终被格式化为`[Deployment名称]-[随机字符串]`。 其中的随机字符串是使用 pod-template-hash 作为种子随机生成的。

```
要查看每个 Pod 自动生成的标签
kubectl get pods --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
```

要查看 Deployment 上线状态

```
kubectl rollout status deploy nginx
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

### 更新deployment

```
更改deployment的镜像并记录：
kubectl set image deploy nginx nginx=nginx:1.15.3 –record
--record会记录下每次更新的状态以便回滚使用
保留份数是由yaml文件中的参数  revisionHistoryLimit: 10 # 历史记录保留的个数
```

查看更新过程

```
 kubectl rollout status deploy nginx
 或者使用describe查看
 kubectl describe deploy nginx 
```

### Deployment回滚

```
# 执行更新操作
[root@k8s-master01 ~]# kubectl set image deploy nginx nginx=nginx:787977da --record
deployment.apps/nginx image updated
[root@k8s-master01 ~]# kubectl get po 
NAME                     READY   STATUS              RESTARTS   AGE
nginx-6cdd5dd489-lv28z   1/1     Running             0          7m12s
nginx-6cdd5dd489-nqqz7   1/1     Running             0          6m37s
nginx-7d79b96f68-x7t67   0/1     ContainerCreating   0          19s
查看更新历史
[root@k8s-master01 ~]# kubectl rollout history deploy nginx
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deploy nginx nginx=nginx:1.15.3 --record=true
3         kubectl set image deploy nginx nginx=nginx:1.15.4 --record=true
4         kubectl set image deploy nginx nginx=nginx:787977da --record=true
回滚到上个版本
kubectl rollout undo deploy nginx
回滚到某个版本
 kubectl rollout undo deploy nginx --to-revision=5
查看指定版本的详细信息
 kubectl rollout history deploy nginx --revision=5
```

###  Deployment的暂停和恢复

```
暂停的话，之后进行的任何变更都不会立即执行
[root@k8s-master01 ~]# kubectl rollout pause deployment nginx 
deployment.apps/nginx paused
多次进行更新
[root@k8s-master01 ~]# kubectl set image deploy nginx nginx=nginx:1.15.3 --record
deployment.apps/nginx image updated
[root@k8s-master01 ~]# # 进行第二次配置变更
[root@k8s-master01 ~]# # 添加内存CPU配置
[root@k8s-master01 ~]# kubectl set -h
Configure application resources

 These commands help you make changes to existing application resources.

Available Commands:
  env            Update environment variables on a pod template
  image          Update image of a pod template
  resources      Update resource requests/limits on objects with pod templates
  selector       Set the selector on a resource
  serviceaccount Update ServiceAccount of a resource
  subject        Update User, Group or ServiceAccount in a RoleBinding/ClusterRoleBinding

Usage:
  kubectl set SUBCOMMAND [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
[root@k8s-master01 ~]# kubectl set resources deploy nginx -c nginx --limits=cpu=200m,memory=128Mi --requests=cpu=10m,memory=16Mi
deployment.apps/nginx resource requirements updated
[root@k8s-master01 ~]# kubectl get deploy nginx -oyaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "11"
    kubernetes.io/change-cause: kubectl set image deploy nginx nginx=nginx:1.15.3
      --record=true
  creationTimestamp: "2020-09-19T02:41:11Z"
  generation: 18
  labels:
    app: nginx
  name: nginx
  namespace: default
  resourceVersion: "2660534"
  selfLink: /apis/apps/v1/namespaces/default/deployments/nginx
  uid: 1d9253a5-a36c-48cc-aefe-56f95967db66
spec:
  paused: true
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.15.3
        imagePullPolicy: IfNotPresent
        name: nginx
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 10m
            memory: 16Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2020-09-19T03:26:50Z"
    lastUpdateTime: "2020-09-19T03:26:50Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2020-09-19T03:30:15Z"
    lastUpdateTime: "2020-09-19T03:30:15Z"
    message: Deployment is paused
    reason: DeploymentPaused
    status: Unknown
    type: Progressing
  observedGeneration: 18
  readyReplicas: 2
  replicas: 2

```

查看pod是否被更新

```
[root@k8s-master01 ~]# kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6cdd5dd489-lv28z   1/1     Running   0          30m
nginx-6cdd5dd489-nqqz7   1/1     Running   0          30m
pod没有被更新

恢复暂停状态，pod被更新
[root@k8s-master01 ~]# kubectl rollout resume deploy nginx 
deployment.apps/nginx resumed
[root@k8s-master01 ~]# kubectl get rs
NAME               DESIRED   CURRENT   READY   AGE
nginx-5475c49ffb   0         0         0       21m
nginx-5dfc8689c6   0         0         0       33m
nginx-66bbc9fdc5   0         0         0       52m
nginx-68db656dd8   1         1         0       15s
nginx-6cdd5dd489   2         2         2       31m
nginx-799b8478d4   0         0         0       21m
nginx-7d79b96f68   0         0         0       24m
nginx-f974656f7    0         0         0       21m

```

### 重要参数配置

常见deployment

```
 kubectl get deploy nginx -oyaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "12"
    kubernetes.io/change-cause: kubectl set image deploy nginx nginx=nginx:1.15.3
      --record=true
  creationTimestamp: "2020-09-19T02:41:11Z"
  generation: 19
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.15.3
        imagePullPolicy: IfNotPresent
        name: nginx
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 10m
            memory: 16Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

重要参数

```
.spec.revisionHistoryLimit：设置保留RS旧的revision的个数，设置为0的话，不保留历史数据
.spec.minReadySeconds：可选参数，指定新创建的Pod在没有任何容器崩溃的情况下视为Ready最小的秒数，默认为0，即一旦被创建就视为可用。
滚动更新的策略：
	.spec.strategy.type：更新deployment的方式，默认是RollingUpdate
RollingUpdate：滚动更新，可以指定maxSurge和maxUnavailable
     maxUnavailable：指定在回滚或更新时最大不可用的Pod的数量，可选字段，默认25%，可以设置成数字或百分比，如果该值为0，那么maxSurge就不能0
     maxSurge：可以超过期望值的最大Pod数，可选字段，默认为25%，可以设置成数字或百分比，如果该值为0，那么maxUnavailable不能为0
maxSurge
此参数控制滚动更新过程中副本总数的超过 DESIRED 的上限。maxSurge 可以是具体的整数（比如 3），也可以是百分百，向上取整。maxSurge 默认值为 25%。
比如，DESIRED 为 10，那么副本总数的最大值为：roundUp(10 + 10 25%) = 13
所以我们看到 CURRENT 就是 13。
maxUnavailable
此参数控制滚动更新过程中，不可用的副本相占 DESIRED 的最大比例。 maxUnavailable 可以是具体的整数（比如 3），也可以是百分百，向下取整。maxUnavailable 默认值为 25%。
在上面的例子中，DESIRED 为 10，那么可用的副本数至少要为：
10 - roundDown(10 25%) = 8
所以我们看到 AVAILABLE 就是 8。
maxSurge 值越大，初始创建的新副本数量就越多；maxUnavailable 值越大，初始销毁的旧副本数量就越多。

Recreate：重建，先删除旧的Pod，在创建新的Pod

```

参考有状态与无状态的区别:https://blog.csdn.net/qq_50119948/article/details/121609072

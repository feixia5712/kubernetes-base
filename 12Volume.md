## Volumes

### 简介

```
Container（容器）中的磁盘文件是短暂的，当容器崩溃时，kubelet会重新启动容器，但最初的文件将丢失，Container会以最干净的状态启动。另外，当一个Pod运行多个Container时，各个容器可能需要共享一些文件。Kubernetes Volume可以解决这两个问题。
一些需要持久化数据的程序才会用到Volumes，或者一些需要共享数据的容器需要volumes。
Redis-Cluster：nodes.conf
日志收集的需求：需要在应用程序的容器里面加一个sidecar，这个容器是一个收集日志的容器，比如filebeat，它通过volumes共享应用程序的日志文件目录。

```

###  emptyDir

```
和上述volume不同的是，如果删除Pod，emptyDir卷中的数据也将被删除，一般emptyDir卷用于Pod中的不同Container共享数据。它可以被挂载到相同或不同的路径上。
默认情况下，emptyDir卷支持节点上的任何介质，可能是SSD、磁盘或网络存储，具体取决于自身的环境。可以将emptyDir.medium字段设置为Memory，让Kubernetes使用tmpfs（内存支持的文件系统），虽然tmpfs非常快，但是tmpfs在节点重启时，数据同样会被清除，并且设置的大小会被计入到Container的内存限制当中。
使用emptyDir卷的示例，直接指定emptyDir为{}即可：
```

创建语句如下

```
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
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt
          name: share-volume
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx2
        command:
        - sh
        - -c
        - sleep 3600
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /mnt
          name: share-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: share-volume
        emptyDir: {}
          #medium: Memory
```

验证

```
[root@k8s-master01 conf]# kubectl exec -it nginx-9d87b9dd6-4w48q -- bash
Defaulted container "nginx" out of: nginx, nginx2
root@nginx-9d87b9dd6-4w48q:/# df -h
Filesystem               Size  Used Avail Use% Mounted on
overlay                   98G  4.2G   94G   5% /
tmpfs                     64M     0   64M   0% /dev
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   98G  4.2G   94G   5% /opt     #挂载了
shm                       64M     0   64M   0% /dev/shm
tmpfs                    3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                    3.9G     0  3.9G   0% /proc/acpi
tmpfs                    3.9G     0  3.9G   0% /proc/scsi
tmpfs                    3.9G     0  3.9G   0% /sys/firmware
root@nginx-9d87b9dd6-4w48q:/# cd /opt/
root@nginx-9d87b9dd6-4w48q:/opt# touch 123
root@nginx-9d87b9dd6-4w48q:/opt# exit
[root@k8s-master01 conf]# kubectl exec -it nginx-9d87b9dd6-4w48q -c nginx2 -- bash
root@nginx-9d87b9dd6-4w48q:/# df -h
Filesystem               Size  Used Avail Use% Mounted on
overlay                   98G  4.2G   94G   5% /
tmpfs                     64M     0   64M   0% /dev
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   98G  4.2G   94G   5% /mnt   #挂载了
shm                       64M     0   64M   0% /dev/shm
tmpfs                    3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                    3.9G     0  3.9G   0% /proc/acpi
tmpfs                    3.9G     0  3.9G   0% /proc/scsi
tmpfs                    3.9G     0  3.9G   0% /sys/firmware
root@nginx-9d87b9dd6-4w48q:/# cd /mnt/
root@nginx-9d87b9dd6-4w48q:/mnt# ls -l
total 0
-rw-r--r-- 1 root root 0 Sep  8 13:47 123
root@nginx-9d87b9dd6-4w48q:/mnt# 

上个容器nginx的/opt目录里面的内容，跟另外容器nginx2的/mnt是共享的,一样的
```

### hostPath

```
hostPath卷可将节点上的文件或目录挂载到Pod上，用于Pod自定义日志输出或访问Docker内部的容器等。
使用hostPath卷的示例。将主机的/data目录挂载到Pod的/test-pd目录：

```

yaml文件实例

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
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt
          name: share-volume
        - mountPath: /etc/timezone
          name: timezone
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx2
        command:
        - sh
        - -c
        - sleep 3600
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /mnt
          name: share-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: share-volume
        emptyDir: {}
          #medium: Memory

      - name: timezone
        hostPath:
          path: /etc/timezone   #挂载宿主机的这个目录文件到容器内
          type: File
```

- hostPath卷常用的type（类型）如下：

- type为空字符串：默认选项，意味着挂载hostPath卷之前不会执行任何检查。

- DirectoryOrCreate：如果给定的path不存在任何东西，那么将根据需要创建一个权限为0755的空目录，和Kubelet具有相同的组和权限。

- Directory：目录必须存在于给定的路径下。

- FileOrCreate：如果给定的路径不存储任何内容，则会根据需要创建一个空文件，权限设置为0644，和Kubelet具有相同的组和所有权。

- File：文件，必须存在于给定路径中。

- Socket：UNIX套接字，必须存在于给定路径中。

- CharDevice：字符设备，必须存在于给定路径中。

- BlockDevice：块设备，必须存在于给定路径中。

### NFS挂载

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
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt
          name: share-volume
        - mountPath: /etc/timezone
          name: timezone
      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx2
        command:
        - sh
        - -c
        - sleep 3600
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /mnt
          name: share-volume
        - mountPath: /opt  #挂载到这里，在这里引用
          name: nfs-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: share-volume
        emptyDir: {}
          #medium: Memory

      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File
      - name: nfs-volume
        nfs:  #nfs挂载配置
          server: 192.168.0.204
          path: /data/nfs/test-dp
```
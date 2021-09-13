## Pv与Pvc

### 简介

```
PersistentVolume：简称PV，是由Kubernetes管理员设置的存储，可以配置Ceph、NFS、GlusterFS等常用存储配置，相对于Volume配置，提供了更多的功能，比如生命周期的管理、大小的限制。PV分为静态和动态。
PersistentVolumeClaim：简称PVC，是对存储PV的请求，表示需要什么类型的PV，需要存储的技术人员只需要配置PVC即可使用存储，或者Volume配置PVC的名称即可
先创建pv,然后配置pvc去链接pv，然后pod去调用即可
PV 持久卷和普通的 Volume 一样，也是使用 卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期
持久卷申领（PersistentVolumeClaim，PVC）表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）；同样 PVC 申领也可以请求特定的大小和访问模式 
```

### PV回收策略

```
Retain：保留，该策略允许手动回收资源，当删除PVC时，PV仍然存在，PV被视为已释放，管理员可以手动回收卷。
Recycle：回收，如果Volume插件支持，Recycle策略会对卷执行rm -rf清理该PV，并使其可用于下一个新的PVC，但是本策略将来会被弃用，目前只有NFS和HostPath支持该策略。
Delete：删除，如果Volume插件支持，删除PVC时会同时删除PV，动态卷默认为Delete，目前支持Delete的存储后端包括AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder等。
可以通过persistentVolumeReclaimPolicy: Recycle字段配置
```

### PV访问策略

```
ReadWriteOnce：可以被单节点以读写模式挂载，命令行中可以被缩写为RWO。
ReadOnlyMany：可以被多个节点以只读模式挂载，命令行中可以被缩写为ROX。
ReadWriteMany：可以被多个节点以读写模式挂载，命令行中可以被缩写为RWX。
ReadWriteOncePod ：只允许被单个Pod访问，需要K8s 1.22+以上版本，并且是CSI创建的PV才可使用
```

### 存储分类

```
文件存储：一些数据可能需要被多个节点使用，比如用户的头像、用户上传的文件等，实现方式：NFS、NAS、FTP、CephFS等。
块存储：一些数据只能被一个节点使用，或者是需要将一块裸盘整个挂载使用，比如数据库、Redis等，实现方式：Ceph、GlusterFS、公有云。
对象存储：由程序代码直接实现的一种存储方式，云原生应用无状态化常用的实现方式，实现方式：一般是符合S3协议的云存储，比如AWS的S3存储、Minio、七牛云等。
```

### PV配置示例-NFS

安装nfs服务

```
node02作为nfs的server端
yum install nfs* rpcbind -y
其他节点作为客户端
yum install nfs-utils -y
NFS服务端：mkdir /data/k8s -p
NFS服务器创建共享目录：vim /etc/exports
/data/k8s/ *(rw,sync,no_subtree_check,no_root_squash)
exportfs -r
systemctl restart nfs rpcbind
挂载测试：mount -t nfs nfs-serverIP:/data/k8s /mnt/
mount -t nfs 192.168.10.184:/data/k8s /mnt/
```

创建pv

```
[root@k8s-master01 pv]# cat nfs-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs   #pv的名称
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs-slow
  nfs:
    path: /data/k8s
    server: 192.168.10.184
  
capacity：容量配置
volumeMode：卷的模式，目前支持Filesystem（文件系统） 和 Block（块），其中Block类型需要后端存储支持，默认为文件系统
accessModes：该PV的访问模式
storageClassName：PV的类，一个特定类型的PV只能绑定到特定类别的PVC；绑定时需要用到
persistentVolumeReclaimPolicy：回收策略
mountOptions：非必须，新版本中已弃用
nfs：NFS服务配置，包括以下两个选项
path：NFS上的共享目录
server：NFS的IP地址
```

查看

```
[root@k8s-master01 pv]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-nfs   5Gi        RWX            Delete           Available           nfs-slow                6s
```

pv状态简介

```
Available：可用，没有被PVC绑定的空闲资源。
Bound：已绑定，已经被PVC绑定。
Released：已释放，PVC被删除，但是资源还未被重新使用。
Failed：失败，自动回收失败。
```

### PV配置示例-hostpath

```
跟上边的差不多，主要是hostpath这一点不一样
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: hostpath
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

### pvs绑定到pv，pod去使用pvc

```
[root@k8s-master01 pv]# cat pvc-nfs.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-pvc-claim
spec:
  storageClassName: nfs-slow #需要绑定的pv的storageClass的名称
  accessModes:
    - ReadWriteMany  #这个模式必须跟需要绑定的pv的模式一样
  resources:
    requests:
      storage: 3Gi  #这个必须比绑定的pv的大小要小
```

创建

```
[root@k8s-master01 pv]# kubectl create -f pvc-nfs.yaml 
persistentvolumeclaim/nfs-pvc-claim created
[root@k8s-master01 pv]# kubectl get pvc
NAME            STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc-claim   Bound    pv-nfs   5Gi        RWX            nfs-slow       6s
查看pv的状态以及是bound状态
[root@k8s-master01 pv]# kubectl get pv pv-nfs
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pv-nfs   5Gi        RWX            Delete           Bound    default/nfs-pvc-claim   nfs-slow                21h
```

pod去配置pvc

```
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage #volume的名称
      persistentVolumeClaim:
       claimName: task-pvc-claim  #链接的pvc的名称
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

### 创建

```
kubectl create -f pod-pvc.yaml
设置
[root@k8s-master01 pv]# kubectl exec -it task-pv-pod -- sh
# cd /usr/share/nginx/html
在nfs的server的机器上导入一点数据
[root@k8s-node02 k8s]#touch index.html 
[root@k8s-node02 k8s]# echo "xxx" >index.html 
[root@k8s-node02 k8s]# pwd
/data/k8s
# cat index.html 
xxx
在pod机器上查看
[root@k8s-master01 pv]# kubectl exec -it task-pv-pod -- sh
# cd /usr/share/nginx/html
# cat index.html
xxx
```

### deployment形式

```
[root@k8s-master01 pv]# cat deploy-pvc.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-nfs
  name: nginx-nfs
  namespace: default
spec:
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx-nfs
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-nfs
    spec:
      volumes:
      - name: task-pv-storage
        persistentVolumeClaim:
          claimName: nfs-pvc-claim  #pvc的名字
      containers:
      - env:
        - name: TZ
          value: Asia/Shanghai
        - name: LANG
          value: C.UTF-8
        image: nginx
        imagePullPolicy: IfNotPresent
        name: nginx-nfs
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

验证查看

```
[root@k8s-master01 pv]# kubectl get pod
nginx-nfs-5cb6f678d5-p2vql          1/1     Running   0          5s
nginx-nfs-5cb6f678d5-qlx4f          1/1     Running   0          5s

登录第一个pod进去查看
[root@k8s-master01 pv]# kubectl exec -it nginx-nfs-5cb6f678d5-p2vql -- sh  
# cd /usr/share/
# ls
X11	     bash-completion  debconf	   doc-base    gcc-8  keyrings	man    pam	    polkit-1	    terminfo
adduser      bug	      debianutils  dpkg        gdb    libc-bin	menu   pam-configs  readline	    xml
base-files   ca-certificates  dict	   fontconfig  info   lintian	misc   perl5	    sensible-utils  zoneinfo
base-passwd  common-licenses  doc	   fonts       java   locale	nginx  pixmaps	    tabset	    zsh
# cd nginx
# cd html
# pwd
/usr/share/nginx/html
# ls
index.html
# cat index.html
xxx
# echo "gg" >index.html  #修改下，然后登录第二个pod看看是否可以查看到改变
# 
您在 /var/spool/mail/root 中有新邮件
登录第二个pod查看
[root@k8s-master01 pv]# kubectl exec -it nginx-nfs-5cb6f678d5-qlx4f -- sh
# cd /usr/share/nginx/html
# cat index.html
gg
#
```

### 注意事项

```
PVC一直Pending的原因：
PVC的空间申请大小大于PV的大小
PVC的StorageClassName没有和PV的一致
PVC的accessModes和PV的不一致

挂载PVC的Pod一直处于Pending：
PVC没有创建成功/PVC不存在
PVC和Pod不在同一个Namespace
```


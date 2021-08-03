## Label与Slector

### 简介

*标签（Labels）* 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的

而Selector（标签选择器）则是针对匹配对象的查询方法。注：键-值对就是key-value pair。

例如，常用的标签tier可用于区分容器的属性，如frontend、backend；或者一个release_track用于区分容器的环境，如canary、production等。

### 定义一个label

```
设置label给node
[root@k8s-master01 ~]# kubectl label node k8s-node01 ds=true
node/k8s-node02 labeled
查看
[root@k8s-master01 yaml]# kubectl get nodes -l "ds=true"
NAME         STATUS   ROLES    AGE   VERSION
k8s-node01   Ready    <none>   29d   v1.20.8
部署pod到这个节点
        app: nginx
    spec:
      nodeSelector:  #选择节点
        ds: "true"
      containers:
      - image: nginx:latest
```

### Selector条件匹配

Selector主要用于资源的匹配，只有符合条件的资源才会被调用或使用，可以使用该方式对集群中的各类资源进行分配。

假如对Selector进行条件匹配，目前已有的Label如下：

*基于集合* 的标签需求允许你通过一组值来过滤键。 支持三种操作符：`in`、`notin` 和 `exists` (只可以用在键标识符上)

```
[root@k8s-master01 yaml]# kubectl get pod --show-labels
NAME          READY   STATUS    RESTARTS   AGE   LABELS
busybox       1/1     Running   690        28d   app=busybox-1
nginx-2dst6   1/1     Running   0          44h   app=nginx,controller-revision-hash=6b7f98c455,pod-template-generation=2
nginx-dcn54   1/1     Running   0          44h   app=nginx,controller-revision-hash=6b7f98c455,pod-template-generation=2
[root@k8s-master01 yaml]# kubectl get pod -l "app in (nginx,busybox)"
NAME          READY   STATUS    RESTARTS   AGE
nginx-2dst6   1/1     Running   0          44h
nginx-dcn54   1/1     Running   0          44h
You have new mail in /var/spool/mail/root
[root@k8s-master01 yaml]# kubectl get pod -l "app in (nginx,busybox-1)"
NAME          READY   STATUS    RESTARTS   AGE
busybox       1/1     Running   690        28d
nginx-2dst6   1/1     Running   0          44h
nginx-dcn54   1/1     Running   0          44h
[root@k8s-master01 yaml]# kubectl get pod -l "app!=nginx"
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   690        28d
```

### 修改标签

```
修改标签可以使用overwrite参数修改标签
[root@k8s-master01 yaml]# kubectl label node k8s-node01 ds=tt --overwrite
node/k8s-node01 labeled
```



### 删除标签

```
删除标签ds=true
[root@k8s-master01 yaml]# kubectl label node k8s-node01 ds-
node/k8s-node01 labeled
查看标签没了
[root@k8s-master01 yaml]# kubectl get nodes -l "ds=true"
NAME         STATUS   ROLES    AGE   VERSION
k8s-node02   Ready    <none>   29d   v1.20.8
```


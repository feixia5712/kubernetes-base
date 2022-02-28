## glusterFs

### 简介

GlusterFS是集群式NAS存储系统，分布式文件系统（POSIX兼容），Tcp/Ip方式互联的一个并行的网络文件系统，通过原生 GlusterFS 协议访问数据，也可以通过 NFS/CIFS 协议访问数据，没有元数据服务器，实现整个系统的性能、可靠性和稳定性。

### 常用术语

- Brick  最基本的存储单元，表示为trusted storage pool中输出的目录，供客户端挂载用。
- Volume 一个卷。在逻辑上由N个bricks组成。
- FUSE  Unix-like OS上的可动态加载的模块，允许用户不用修改内核即可创建自己的文件系统。
- Glustered  Gluster management daemon，要在trusted storage pool中所有的服务器上运行。
- POSIX 一个标准，GlusterFS兼容。

### GlusterFS卷类型

基本卷：

```
(1) distribute volume：分布式卷
(2) stripe volume：条带卷
(3) replica volume：复制卷
```

复合卷：

```
(4) distribute stripe volume：分布式条带卷
(5) distribute replica volume：分布式复制卷
(6) stripe replica volume：条带复制卷
(7) distribute stripe replicavolume：分布式条带复制卷
```

### 安装

目前是三台机器复用k8s机器

```
k8s-master03  192.168.89.182
k8s-node01    192.168.89.183
k8s-node02    192.168.89.184
```

开始安装

```
添加源开始安装(3台服务器都需要执行)
yum install centos-release-gluster glusterfs-server glusterfs -y
服务端：yum  install   glusterfs-server glusterfs glusterfs-cli   glusterfs-fuse  glusterfs-libs glusterfs-api 
客户端安装(需要挂载的服务器执行)  yum  install    glusterfs    glusterfs-fuse  glusterfs-libs 
最好不要使用操作系统的磁盘,单独挂载磁盘 (3台服务器都需要进行操作)
分区(fdisk /dev/sdb) 格式化(mkfs.xfs) 挂载 
mount /dev/sdb1  /data/brick1
```

启动服务

```
service glusterd start
service glusterd status
```

添加节点服务器(k8s-master03上执行)

```
gluster peer probe k8s-node01
gluster peer probe k8s-node02
```

在三台服务器上执行查看命令

```
[root@k8s-master03 ~]# gluster  peer  status
Number of Peers: 2

Hostname: k8s-node01
Uuid: 4d4306df-3ba4-4e1d-85db-3df2173c7d25
State: Peer in Cluster (Connected)

Hostname: k8s-node02
Uuid: 1e633d47-c9e2-4365-85d7-32bdb0095203
State: Peer in Cluster (Connected)
```

创建数据卷

参考:https://www.cnblogs.com/xiexun/p/14600702.html

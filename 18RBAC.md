## RBAC权限认证

### serviceaccount讲解

```
Service Account 用来访问 kubernetes API，通过 kubernetes API 创建和管理，每个 account 只能在一个 namespace 上生效，存储在 kubernetes API 中的 Secrets 资源。kubernetes 会默认创建，并且会自动挂载到 Pod 中的 /run/secrets/kubernetes.io/serviceaccount 目录下。
Service account是为了方便Pod里面的进程调用Kubernetes API或其他外部服务而设计的,service account则是仅局限它所在的namespace
每个namespace都会自动创建一个default service account
```

案例讲解

```
[root@k8s-master01 ~]# kubectl create namespace qiangungun  #创建一个名称空间
namespace "qiangungun" created
[root@k8s-master01 ~]# kubectl get sa -n qiangungun  #名称空间创建完成后会自动创建一个sa
NAME      SECRETS   AGE
default   1         11s
[root@k8s-master01 ~]# kubectl get secret -n qiangungun  #同时也会自动创建一个secret
NAME                  TYPE                                  DATA      AGE
default-token-5jtz2   kubernetes.io/service-account-token   3         19s
```

**在创建的名称空间中新建一个pod**

```
[root@k8s-master01 pod-example]# cat pod_demo.yaml 
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
  namespace: qiangungun
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
     - containerPort: 80
       name: www
```

查看验证

```
[root@k8s-master01 pod-example]# kubectl apply -f  pod_demo.yaml 
pod "task-pv-pod" created
[root@k8s-master01 pod-example]# kubectl get pod -n qiangungun 
NAME          READY     STATUS    RESTARTS   AGE
task-pv-pod   1/1       Running   0          13s
[root@k8s-master01 pod-example]# kubectl get  pod task-pv-pod -o yaml   -n qiangungun 
......
volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-5jtz2
......
volumes:  #挂载sa的secret
  - name: default-token-5jtz2
    secret:
      defaultMode: 420
      secretName: default-token-5jtz2 
......
```

**#名称空间新建的pod如果不指定sa，会自动挂载当前名称空间中默认的sa(default)**

另外一种情况就是,当我们收到创建sa的时候, #会自动创建一个secret(admin-token-rxtrc),用于当前sa连接至当前API server时使用的认证信息

```
[root@k8s-master01 ~]# kubectl create sa test -n qiangungun
serviceaccount/test created
您在 /var/spool/mail/root 中有新邮件
[root@k8s-master01 ~]# kubectl get sa -n qiangungun
NAME      SECRETS   AGE
default   1         9m54s
test      1         9s
[root@k8s-master01 ~]# kubectl get secret -n qiangungun
NAME                  TYPE                                  DATA   AGE
default-token-qb9ws   kubernetes.io/service-account-token   3      10m
test-token-2v6kl      kubernetes.io/service-account-token   3      40s
```

参考:https://www.cnblogs.com/panwenbin-logs/p/10029834.html

### RBAC介绍

```
在Kubernetes中，授权有ABAC（基于属性的访问控制）、RBAC（基于角色的访问控制）、Webhook、Node、AlwaysDeny（一直拒绝）和AlwaysAllow（一直允许）这6种模式。从1.6版本起，Kubernetes 默认启用RBAC访问控制策略。从1.8开始，RBAC已作为稳定的功能。通过设置–authorization-mode=RBAC，启用RABC。在RABC API中，通过如下的步骤进行授权：
1）定义角色：在定义角色时会指定此角色对于资源的访问控制的规则；
2）绑定角色：将主体与角色进行绑定，对用户进行访问授权。
```

![img](https://www.kubernetes.org.cn/img/2018/05/image.png)

#### 角色与集群角色

```
在RBAC API中，角色包含代表权限集合的规则。在这里，权限只有被授予，而没有被拒绝的设置。在Kubernetes中有两类角色，即普通角色和集群角色。
可以通过Role定义在一个命名空间中的角色，或者可以使用ClusterRole定义集群范围的角色。
一个角色只能被用来授予访问单一命令空间中的资源。
集群角色(ClusterRole)能够被授予如下资源的权限：
    集群范围的资源（类似于Node）
    非资源端点（类似于”/healthz”）
    集群中所有命名空间的资源（类似Pod）
```

#### **角色绑定和集群角色绑定**

```
角色绑定用于将角色与一个或一组用户进行绑定，从而实现将对用户进行授权的目的。主体分为用户、组和服务帐户。
角色绑定也分为角色普通角色绑定和集群角色绑定。
角色绑定只能引用同一个命名空间下的角色。
角色绑定也可以通过引用集群角色授予访问权限，当主体对资源的访问仅限与本命名空间，这就允许管理员定义整个集群的公共角色集合，然后在多个命名空间中进行复用。
集群角色可以被用来在集群层面和整个命名空间进行授权。
```

#### **资源**

```
在Kubernets中，主要的资源包括：Pods、Nodes、Services、Deployment、Replicasets、Statefulsets、Namespace、Persistents、Secrets和ConfigMaps等。另外，有些资源下面存在子资源，例如：Pod下就存在log子资源
```

#### **主体**

```
RBAC授权中的主体可以是组，用户或者服务帐户。用户通过字符串表示，比如“alice”、 “bob@example.com”等，具体的形式取决于管理员在认证模块中所配置的用户名。system:被保留作为用来Kubernetes系统使用，因此不能作为用户的前缀。组也有认证模块提供，格式与用户类似。
```

#### 案例参考

【一个 RoleBinding 可以引用同一的名字空间中的任何 Role。 或者，一个 RoleBinding 可以引用某 ClusterRole 并将该 ClusterRole 绑定到 RoleBinding 所在的名字空间。 如果你希望将某 ClusterRole 绑定到集群中所有名字空间，你要使用 ClusterRoleBinding。】

### 使用RoleBanding将用户绑定到Role上

```
kubectl create role pods-reader --verb=get,list,watch --resource=pods --dry-run=client -o yaml  #通过 -o参数导出yaml格式，可以大致看到Role是如何定义的
[root@k8s-master01 ~]# kubectl create role pods-reader --verb=get,list,watch --resource=pods --dry-run=client -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: pods-reader
  namespace: ingress-nginx  #指定命名空间
rules:
- apiGroups: # "" 标明 core API 组
  - ""
  resources: #包含哪些资源
  - pods
  verbs: #对以上资源允许进行哪些操作
  - get
  - list
  - watch
 [root@k8s-master01 ~]# kubectl create -f daem.yaml 
role.rbac.authorization.k8s.io/pods-reader created
[root@k8s-master01 ~]# kubectl get role -n ingress-nginx
NAME          CREATED AT
pods-reader   2021-09-26T14:05:54Z
[root@k8s-master01 ~]# kubectl get role 
NAME          CREATED AT
pods-reader   2021-09-26T14:05:54Z
[root@k8s-master01 ~]# kubectl describe role pods-reader
Name:         pods-reader
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  pods       []                 []              [get list watch]
 
```

绑定[role跟rolebinding必须在一个命名空间]

```
[root@k8s-master01 ~]# kubectl create sa qiangungun
serviceaccount/qiangungun created
[root@k8s-master01 rbac]# kubectl create -f roband.yaml 
rolebinding.rbac.authorization.k8s.io/qiangungun-read-pods created
[root@k8s-master01 rbac]# kubectl get rolebinding
NAME                   ROLE               AGE
qiangungun-read-pods   Role/pods-reader   13s
[root@k8s-master01 rbac]# cat roband.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: qiangungun-read-pods
  namespace: ingress-nginx #权限在这个命名空间下
roleRef:   #role引用，表示引用哪个role
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pods-reader
subjects:  #动作的执行主题
- kind: ServiceAccount
  name: qiangungun
  namespace: default
```

借助dashboard进行验证

当我们创建一个serviceaccount的时候,就相当于创建了一个登录用户，借助secret里的token就可以登录dashboard,默认是没有权限的

```
查看登录的token
第一种方式使用describe,这个是不加密的,直接利用获取的token进行加密
[root@k8s-master01 rbac]# kubectl describe secret qiangungun-token-lcvk5
Name:         qiangungun-token-lcvk5
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: qiangungun
              kubernetes.io/service-account.uid: 1efdec6c-bf2d-49a1-9ae0-d93ee6e0662b

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1411 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlRhZUNOeW5Iay1TZ3RrUF9DbFBybV9iLUhRSHFDT3NVdVFjSjBma0tNbXMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InFpYW5ndW5ndW4tdG9rZW4tbGN2azUiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicWlhbmd1bmd1biIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjFlZmRlYzZjLWJmMmQtNDlhMS05YWUwLWQ5M2VlNmUwNjYyYiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnFpYW5ndW5ndW4ifQ.cj3bglqXQJSjaRdNfjNivydY0BZU8OZjnFcCYSekjO4_qT9mDhKcBIH-xGzMCRuZF7P1Ubdq63nTWjumpRhWfR5XIHxYinB-Cu1EmV8EjrzeZGeWZ-Xa18jYih2zUbvqDnQ16Gzf4jp3gqT7bkXDAgTZmxcSQKR5SFxlc2G1M2XbdDlF4WPMN2M7tNBg4b9yLma5Gn6kFZ0ljqFtkJE0E5w1ywpNlTaZVb50Wj8Pu6qg9Dh4-ItwCw2i4O19-74GK17WXrgwi3bdO1thitzg5IZiNX4JNNho_GJfC6LMwRqRXFXQA3KyrOfpiCDBngU6XmUqYbqS6woJnO678OkxCA
第二种使用-oyaml的方式查看到是加密的需要使用base64 -d进行解密
[root@k8s-master01 rbac]# kubectl get secret qiangungun-token-lcvk5 -oyaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUQ1RENDQXN5Z0F3SUJBZ0lVWGR3OVJ0RzEyL25qdXIyYnF3SUczMjBrTzdrd0RRWUpLb1pJaHZjTkFRRUwKQlFBd2R6RUxNQWtHQTFVRUJoTUNRMDR4RURBT0JnTlZCQWdUQjBKbGFXcHBibWN4RURBT0JnTlZCQWNUQjBKbAphV3BwYm1jeEV6QVJCZ05WQkFvVENrdDFZbVZ5Ym1WMFpYTXhHakFZQmdOVkJBc1RFVXQxWW1WeWJtVjBaWE10CmJXRnVkV0ZzTVJNd0VRWURWUVFERXdwcmRXSmxjbTVsZEdWek1DQVhEVEl4TURrd016QXhNemN3TUZvWUR6SXgKTWpFd09ERXdNREV6TnpBd1dqQjNNUXN3Q1FZRFZRUUdFd0pEVGpFUU1BNEdBMVVFQ0JNSFFtVnBhbWx1WnpFUQpNQTRHQTFVRUJ4TUhRbVZwYW1sdVp6RVRNQkVHQTFVRUNoTUtTM1ZpWlhKdVpYUmxjekVhTUJnR0ExVUVDeE1SClMzVmlaWEp1WlhSbGN5MXRZVzUxWVd3eEV6QVJCZ05WQkFNVENtdDFZbVZ5Ym1WMFpYTXdnZ0VpTUEwR0NTcUcKU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRQ3huaG1la1FwcUtqd2pJTXMwSllLNDlHL2xyb3c4VVhEVAowcXVVVWhIclpVZ1J5c2VRVFBlTjIvQ3BNTEl1cGRhWXd1bExCZFE0eFgzNm5lZnhveitSZ2NOSmNPL0FoclIwCmdtWFRwOGpmbEFvcDVGZDV2dE1UMXQrYklrbkJCQW5vMnRHQ2RCRTdoVlFjZGlCZzZ1cTlxZ04vYUZOVEhpY0QKeDRkOUptakRTcjNhOUdjY1J3MERkM1BYTjRnQjdPbE5Qa1ZyUUtVUjVxUUVDNGpCT3FhbEdKRDJiUktZSUVMWApaWDVoelNYSU4xeDNSN3g2OWJ5N1BPZS9MbGplWW9heU1NQk5kRWkybDJ0SENKUHZKM20vMzhLdk9XRWpRcldICjlMZDd2VmN2S1gxOWVYZllIeUtuWGtEbkdLUnFHYWh3ejAvN0FJWUdIRWJsMjI1aTF6VzNBZ01CQUFHalpqQmsKTUE0R0ExVWREd0VCL3dRRUF3SUJCakFTQmdOVkhSTUJBZjhFQ0RBR0FRSC9BZ0VDTUIwR0ExVWREZ1FXQkJSeAptVVRPd2ZHNVNFZkpZcE90SEszZUk0UkJiakFmQmdOVkhTTUVHREFXZ0JSeG1VVE93Zkc1U0VmSllwT3RISzNlCkk0UkJiakFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBVUhjaUQ1WGtyN2Z1T1J6b2FGQ0JUTm5OQWxzU0hMbloKZzBKUXNndS9KTGtVTWdmenlzZVFyeUpiNFpENFdteHRHSmtLNUpHNlROSktIeG1Mck53Y1gzMGp1V1Nrc0FVOAo0TjFLVi9XSGlzN1JKR2FWSi9YRmViSWt4aGg1UGpPTTJ2VzEzc1BXUlZNUTEzeDN6NFJtN0FxTUNZTkFSUUQyCm9uUXdhQU5FRlU3bGU5S01rVWJzbGNSdVptSElXNU1xbEFpMW9ab09iN1l5WXFrR214K3lOL1dlaFlQaGdvamMKNWZJZGZwM1lqUWQxelFSUHd5Zi9yU05iaWtOMFBWaUs5TTFwZUVDSWF1ZVFXOHBGWEJpV29iN1R6NU1LaFNWUQpySkJFeHZrRWxCQkduZy9Fc2d0QjI0WENsbDRJRmxYNmoxRnpsaVJLWm1MdWtUTmpSR1ZWSEE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklsUmhaVU5PZVc1SWF5MVRaM1JyVUY5RGJGQnliVjlpTFVoUlNIRkRUM05WZFZGalNqQm1hMHROYlhNaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbkZwWVc1bmRXNW5kVzR0ZEc5clpXNHRiR04yYXpVaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNXVZVzFsSWpvaWNXbGhibWQxYm1kMWJpSXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMblZwWkNJNklqRmxabVJsWXpaakxXSm1NbVF0TkRsaE1TMDVZV1V3TFdRNU0yVmxObVV3TmpZeVlpSXNJbk4xWWlJNkluTjVjM1JsYlRwelpYSjJhV05sWVdOamIzVnVkRHBrWldaaGRXeDBPbkZwWVc1bmRXNW5kVzRpZlEuY2ozYmdscVhRSlNqYVJkTmZqTml2eWRZMEJaVThPWmpuRmNDWVNla2pPNF9xVDltRGhLY0JJSC14R3pNQ1J1WkY3UDFVYmRxNjNuVFdqdW1wUmhXZlI1WElIeFlpbkItQ3UxRW1WOEVqcnplWkdlV1otWGExOGpZaWgyelVidnFEblExNkd6ZjRqcDNncVQ3YmtYREFnVFpteGNTUUtSNVNGeGxjMkcxTTJYYmREbEY0V1BNTjJNN3ROQmc0Yjl5TG1hNUduNmtGWjBsanFGdGtKRTBFNXcxeXdwTmxUYVpWYjUwV2o4UHU2cWc5RGg0LUl0d0N3Mmk0TzE5LTc0R0sxN1dYcmd3aTNiZE8xdGhpdHpnNUlaaU5YNEpOTmhvX0dKZkM2TE13UnFSWEZYUUEzS3lyT2ZwaUNEQm5nVTZYbVVxWWJxUzZ3b0puTzY3OE9reENB
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: qiangungun
    kubernetes.io/service-account.uid: 1efdec6c-bf2d-49a1-9ae0-d93ee6e0662b
  creationTimestamp: "2021-09-27T12:11:20Z"
  name: qiangungun-token-lcvk5
  namespace: default
  resourceVersion: "4809998"
  uid: 8a0c7906-22a4-40c4-8515-1f8e2e504ace
type: kubernetes.io/service-account-token
解密
[root@k8s-master01 rbac]# echo "ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklsUmhaVU5PZVc1SWF5MVRaM1JyVUY5RGJGQnliVjlpTFVoUlNIRkRUM05WZFZGalNqQm1hMHROYlhNaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbkZwWVc1bmRXNW5kVzR0ZEc5clpXNHRiR04yYXpVaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNXVZVzFsSWpvaWNXbGhibWQxYm1kMWJpSXNJbXQxWW1WeWJtVjBaWE11YVc4dmMyVnlkbWxqWldGalkyOTFiblF2YzJWeWRtbGpaUzFoWTJOdmRXNTBMblZwWkNJNklqRmxabVJsWXpaakxXSm1NbVF0TkRsaE1TMDVZV1V3TFdRNU0yVmxObVV3TmpZeVlpSXNJbk4xWWlJNkluTjVjM1JsYlRwelpYSjJhV05sWVdOamIzVnVkRHBrWldaaGRXeDBPbkZwWVc1bmRXNW5kVzRpZlEuY2ozYmdscVhRSlNqYVJkTmZqTml2eWRZMEJaVThPWmpuRmNDWVNla2pPNF9xVDltRGhLY0JJSC14R3pNQ1J1WkY3UDFVYmRxNjNuVFdqdW1wUmhXZlI1WElIeFlpbkItQ3UxRW1WOEVqcnplWkdlV1otWGExOGpZaWgyelVidnFEblExNkd6ZjRqcDNncVQ3YmtYREFnVFpteGNTUUtSNVNGeGxjMkcxTTJYYmREbEY0V1BNTjJNN3ROQmc0Yjl5TG1hNUduNmtGWjBsanFGdGtKRTBFNXcxeXdwTmxUYVpWYjUwV2o4UHU2cWc5RGg0LUl0d0N3Mmk0TzE5LTc0R0sxN1dYcmd3aTNiZE8xdGhpdHpnNUlaaU5YNEpOTmhvX0dKZkM2TE13UnFSWEZYUUEzS3lyT2ZwaUNEQm5nVTZYbVVxWWJxUzZ3b0puTzY3OE9reENB"|base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IlRhZUNOeW5Iay1TZ3RrUF9DbFBybV9iLUhRSHFDT3NVdVFjSjBma0tNbXMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InFpYW5ndW5ndW4tdG9rZW4tbGN2azUiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicWlhbmd1bmd1biIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjFlZmRlYzZjLWJmMmQtNDlhMS05YWUwLWQ5M2VlNmUwNjYyYiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnFpYW5ndW5ndW4ifQ.cj3bglqXQJSjaRdNfjNivydY0BZU8OZjnFcCYSekjO4_qT9mDhKcBIH-xGzMCRuZF7P1Ubdq63nTWjumpRhWfR5XIHxYinB-Cu1EmV8EjrzeZGeWZ-Xa18jYih2zUbvqDnQ16Gzf4jp3gqT7bkXDAgTZmxcSQKR5SFxlc2G1M2XbdDlF4WPMN2M7tNBg4b9yLma5Gn6kFZ0ljqFtkJE0E5w1ywpNlTaZVb50Wj8Pu6qg9Dh4-ItwCw2i4O19-74GK17WXrgwi3bdO1thitzg5IZiNX4JNNho_GJfC6LMwRqRXFXQA3KyrOfpiCDBngU6XmUqYbqS6woJnO678OkxCA[root@k8s-master01 rbac]#
得到的结果是一样的
```

### 使用ClusterRoleBanding将用户绑定到ClusterRole上

```
[root@k8s-master01 rbac]# kubectl create clusterrole cluster-reader --verb=get,list,watch --resource=pods --dry-run=client -oyaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: cluster-reader #ClusterRole属于集群级别，所有不可以定义namespace
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
  [root@k8s-master01 rbac]# kubectl create sa cluster-role 
serviceaccount/cluster-role created
[root@k8s-master01 rbac]# kubectl get sa
NAME           SECRETS   AGE
cluster-role   1         10s
default        1         23d
qiangungun     1         85m
[root@k8s-master01 rbac]# cat clusterrolwband01.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: cluster-read-all-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-reader
subjects:
- kind: ServiceAccount
  name: cluster-role
  namespace: default  #这个需要加命名空间是为了找到这个账户
  验证的化还是跟上边一样
```

聚合rolebinding

案例2

```
不同用户不同权限
需求：
    用户dotbalo可以查看default、kube-system下Pod的日志
    用户dukuan可以在default下的Pod中执行命令，并且可以删除Pod
```




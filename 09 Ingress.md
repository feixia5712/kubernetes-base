## Ingress

### 简介

Ingress为Kubernetes集群中的服务提供了入口，可以提供负载均衡、SSL终止和基于名称的虚拟主机，在生产环境中常用的Ingress有Treafik、Nginx、HAProxy、Istio等。

#### 基本概念

在Kubernetesv 1.1版中添加的Ingress用于从集群外部到集群内部Service的HTTP和HTTPS路由，流量从Internet到Ingress再到Services最后到Pod上，通常情况下，Ingress部署在所有的Node节点上。
Ingress可以配置提供服务外部访问的URL、负载均衡、终止SSL，并提供基于域名的虚拟主机。但Ingress不会暴露任意端口或协议。

### 安装

#### 先安装helm

参考：https://helm.sh/zh/docs/intro/install/
下载需要的版本:https://github.com/helm/helm/releases

```
下载
wget https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz
解压
tar -zxvf helm-v3.6.3-linux-amd64.tar.gz
移动
mv linux-amd64/helm /usr/local/bin/helm
查看版本
[root@k8s-master01 yaml]# helm version
version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"clean", GoVersion:"go1.16.5"}

```

#### 安装Ingress-nginx

参考:https://kubernetes.github.io/ingress-nginx/deploy/

下载k8s.gcr.io的镜像地址参考https://hub.docker.com/u/willdockerhub

https://github.com/willzhang/image-syncer

```
添加源
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
[root@k8s-master01 yaml]# helm search repo ingress-nginx
NAME                       	CHART VERSION	APP VERSION	DESCRIPTION                                       
ingress-nginx/ingress-nginx	4.0.1        	1.0.0      	Ingress controller for Kubernetes using NGINX a...
下载
[root@k8s-master01 yaml]# helm pull ingress-nginx/ingress-nginx
[root@k8s-master01 yaml]# ll
总用量 13420
-rw-r--r-- 1 root root 13702117 7月  15 03:00 helm-v3.6.3-linux-amd64.tar.gz
-rw-r--r-- 1 root root    25940 9月   5 17:30 ingress-nginx-4.0.1.tgz
解压
tar -zxvf ingres-nginx-4.0.1.tar.gz
cd ingress-nginx
```

涉及配置文件的修改(涉及的镜像可以去docker镜像官方查找替换掉)

```
修改 value.yaml
需要修改的位置
a)	Controller和admissionWebhook的镜像地址，需要将公网镜像同步至公司内网镜像仓库（和课程不一致的版本，需要自行同步gcr镜像的，可以百度查一下使用阿里云同步gcr的镜像，也可以参考这个连接https://blog.csdn.net/weixin_39961559/article/details/80739352，或者参考这个连接： https://blog.csdn.net/sinat_35543900/article/details/103290782）
b)	镜像的digest值注释 
c)	hostNetwork设置为true  设备为host网络便于宿主机直接访问
d)	dnsPolicy设置为 ClusterFirstWithHostNet
e)	NodeSelector添加ingress: "true"部署至指定节点  部署的时候选择节点使用
f)	类型更改为kind: DaemonSet  便于部署到某些指定节点,以后扩容只是需要在需要部署ingress-ngix的节点上做个标签即可
```

修改后的配置文件参考

```
[root@k8s-master01 ingress-nginx]# cat values.yaml |grep -v ^$|grep -v ^#
controller:
  name: controller
  image:
    registry: willdockerhub
    image: ingress-nginx-controller
    # for backwards compatibility consider setting the full image url via the repository value below
    # use *either* current default registry/image or repository format or installing chart by providing the values.yaml will fail
    # repository:
    tag: "v1.0.0"
    #digest: sha256:0851b34f69f69352bf168e6ccf30e1e20714a264ab1ecd1933e4d8c0fc3215c6
    pullPolicy: IfNotPresent
    # www-data -> uid 101
    runAsUser: 101
    allowPrivilegeEscalation: true
  # Use an existing PSP instead of creating one
  existingPsp: ""
  # Configures the controller container name
  containerName: controller
  # Configures the ports the nginx-controller listens on
  containerPort:
    http: 80
    https: 443
  # Will add custom configuration options to Nginx https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
  config: {}
  ## Annotations to be added to the controller config configuration configmap
  ##
  configAnnotations: {}
  # Will add custom headers before sending traffic to backends according to https://github.com/kubernetes/ingress-nginx/tree/main/docs/examples/customization/custom-headers
  proxySetHeaders: {}
  # Will add custom headers before sending response traffic to the client according to: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#add-headers
  addHeaders: {}
  # Optionally customize the pod dnsConfig.
  dnsConfig: {}
  # Optionally customize the pod hostname.
  hostname: {}
  # Optionally change this to ClusterFirstWithHostNet in case you have 'hostNetwork: true'.
  # By default, while using host network, name resolution uses the host's DNS. If you wish nginx-controller
  # to keep resolving names inside the k8s network, use ClusterFirstWithHostNet.
  dnsPolicy: ClusterFirstWithHostNet
  # Bare-metal considerations via the host network https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#via-the-host-network
  # Ingress status was blank because there is no Service exposing the NGINX Ingress controller in a configuration using the host network, the default --publish-service flag used in standard cloud setups does not apply
  reportNodeInternalIp: false
  # Process Ingress objects without ingressClass annotation/ingressClassName field
  # Overrides value for --watch-ingress-without-class flag of the controller binary
  # Defaults to false
  watchIngressWithoutClass: false
  # Required for use with CNI based kubernetes installations (such as ones set up by kubeadm),
  # since CNI and hostport don't mix yet. Can be deprecated once https://github.com/kubernetes/kubernetes/issues/23920
  # is merged
  hostNetwork: true
  ## Use host ports 80 and 443
  ## Disabled by default
  ##
  hostPort:
    enabled: false
    ports:
      http: 80
      https: 443
  ## Election ID to use for status update
  ##
  electionID: ingress-controller-leader
  # This section refers to the creation of the IngressClass resource
  # IngressClass resources are supported since k8s >= 1.18 and required since k8s >= 1.19
  ingressClassResource:
    name: nginx
    enabled: true
    default: false
    controllerValue: "k8s.io/ingress-nginx"
    # Parameters is a link to a custom resource containing additional
    # configuration for the controller. This is optional if the controller
    # does not require extra parameters.
    parameters: {}
  # labels to add to the pod container metadata
  podLabels: {}
  #  key: value
  ## Security Context policies for controller pods
  ##
  podSecurityContext: {}
  ## See https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/ for
  ## notes on enabling and using sysctls
  ###
  sysctls: {}
  # sysctls:
  #   "net.core.somaxconn": "8192"
  ## Allows customization of the source of the IP address or FQDN to report
  ## in the ingress status field. By default, it reads the information provided
  ## by the service. If disable, the status field reports the IP address of the
  ## node or nodes where an ingress controller pod is running.
  publishService:
    enabled: true
    ## Allows overriding of the publish service to bind to
    ## Must be <namespace>/<service_name>
    ##
    pathOverride: ""
  ## Limit the scope of the controller
  ##
  scope:
    enabled: false
    namespace: ""   # defaults to $(POD_NAMESPACE)
  ## Allows customization of the configmap / nginx-configmap namespace
  ##
  configMapNamespace: ""   # defaults to $(POD_NAMESPACE)
  ## Allows customization of the tcp-services-configmap
  ##
  tcp:
    configMapNamespace: ""   # defaults to $(POD_NAMESPACE)
    ## Annotations to be added to the tcp config configmap
    annotations: {}
  ## Allows customization of the udp-services-configmap
  ##
  udp:
    configMapNamespace: ""   # defaults to $(POD_NAMESPACE)
    ## Annotations to be added to the udp config configmap
    annotations: {}
  # Maxmind license key to download GeoLite2 Databases
  # https://blog.maxmind.com/2019/12/18/significant-changes-to-accessing-and-using-geolite2-databases
  maxmindLicenseKey: ""
  ## Additional command line arguments to pass to nginx-ingress-controller
  ## E.g. to specify the default SSL certificate you can use
  ## extraArgs:
  ##   default-ssl-certificate: "<namespace>/<secret_name>"
  extraArgs: {}
  ## Additional environment variables to set
  extraEnvs: []
  # extraEnvs:
  #   - name: FOO
  #     valueFrom:
  #       secretKeyRef:
  #         key: FOO
  #         name: secret-resource
  ## DaemonSet or Deployment
  ##
  kind: DaemonSet
  ## Annotations to be added to the controller Deployment or DaemonSet
  ##
  annotations: {}
  #  keel.sh/pollSchedule: "@every 60m"
  ## Labels to be added to the controller Deployment or DaemonSet
  ##
  labels: {}
  #  keel.sh/policy: patch
  #  keel.sh/trigger: poll
  # The update strategy to apply to the Deployment or DaemonSet
  ##
  updateStrategy: {}
  #  rollingUpdate:
  #    maxUnavailable: 1
  #  type: RollingUpdate
  # minReadySeconds to avoid killing pods before we are ready
  ##
  minReadySeconds: 0
  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
  #  - key: "key"
  #    operator: "Equal|Exists"
  #    value: "value"
  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"
  ## Affinity and anti-affinity
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
  ##
  affinity: {}
    # # An example of preferred pod anti-affinity, weight is in the range 1-100
    # podAntiAffinity:
    #   preferredDuringSchedulingIgnoredDuringExecution:
    #   - weight: 100
    #     podAffinityTerm:
    #       labelSelector:
    #         matchExpressions:
    #         - key: app.kubernetes.io/name
    #           operator: In
    #           values:
    #           - ingress-nginx
    #         - key: app.kubernetes.io/instance
    #           operator: In
    #           values:
    #           - ingress-nginx
    #         - key: app.kubernetes.io/component
    #           operator: In
    #           values:
    #           - controller
    #       topologyKey: kubernetes.io/hostname
    # # An example of required pod anti-affinity
    # podAntiAffinity:
    #   requiredDuringSchedulingIgnoredDuringExecution:
    #   - labelSelector:
    #       matchExpressions:
    #       - key: app.kubernetes.io/name
    #         operator: In
    #         values:
    #         - ingress-nginx
    #       - key: app.kubernetes.io/instance
    #         operator: In
    #         values:
    #         - ingress-nginx
    #       - key: app.kubernetes.io/component
    #         operator: In
    #         values:
    #         - controller
    #     topologyKey: "kubernetes.io/hostname"
  ## Topology spread constraints rely on node labels to identify the topology domain(s) that each Node is in.
  ## Ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
  ##
  topologySpreadConstraints: []
    # - maxSkew: 1
    #   topologyKey: failure-domain.beta.kubernetes.io/zone
    #   whenUnsatisfiable: DoNotSchedule
    #   labelSelector:
    #     matchLabels:
    #       app.kubernetes.io/instance: ingress-nginx-internal
  ## terminationGracePeriodSeconds
  ## wait up to five minutes for the drain of connections
  ##
  terminationGracePeriodSeconds: 300
  ## Node labels for controller pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector:
    kubernetes.io/os: linux
    ingress: "true"  
  ## Liveness and readiness probe values
  ## Ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  ##
  # startupProbe:
  #   httpGet:
  #     # should match container.healthCheckPath
  #     path: "/healthz"
  #     port: 10254
  #     scheme: HTTP
  #   initialDelaySeconds: 5
  #   periodSeconds: 5
  #   timeoutSeconds: 2
  #   successThreshold: 1
  #   failureThreshold: 5
  livenessProbe:
    httpGet:
      # should match container.healthCheckPath
      path: "/healthz"
      port: 10254
      scheme: HTTP
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 5
  readinessProbe:
    httpGet:
      # should match container.healthCheckPath
      path: "/healthz"
      port: 10254
      scheme: HTTP
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 3
  # Path of the health check endpoint. All requests received on the port defined by
  # the healthz-port parameter are forwarded internally to this path.
  healthCheckPath: "/healthz"
  ## Annotations to be added to controller pods
  ##
  podAnnotations: {}
  replicaCount: 1
  minAvailable: 1
  # Define requests resources to avoid probe issues due to CPU utilization in busy nodes
  # ref: https://github.com/kubernetes/ingress-nginx/issues/4735#issuecomment-551204903
  # Ideally, there should be no limits.
  # https://engineering.indeedblog.com/blog/2019/12/cpu-throttling-regression-fix/
  resources:
  #  limits:
  #    cpu: 100m
  #    memory: 90Mi
    requests:
      cpu: 100m
      memory: 90Mi
  # Mutually exclusive with keda autoscaling
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 11
    targetCPUUtilizationPercentage: 50
    targetMemoryUtilizationPercentage: 50
    behavior: {}
      # scaleDown:
      #   stabilizationWindowSeconds: 300
      #  policies:
      #   - type: Pods
      #     value: 1
      #     periodSeconds: 180
      # scaleUp:
      #   stabilizationWindowSeconds: 300
      #   policies:
      #   - type: Pods
      #     value: 2
      #     periodSeconds: 60
  autoscalingTemplate: []
  # Custom or additional autoscaling metrics
  # ref: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-custom-metrics
  # - type: Pods
  #   pods:
  #     metric:
  #       name: nginx_ingress_controller_nginx_process_requests_total
  #     target:
  #       type: AverageValue
  #       averageValue: 10000m
  # Mutually exclusive with hpa autoscaling
  keda:
    apiVersion: "keda.sh/v1alpha1"
  # apiVersion changes with keda 1.x vs 2.x
  # 2.x = keda.sh/v1alpha1
  # 1.x = keda.k8s.io/v1alpha1
    enabled: false
    minReplicas: 1
    maxReplicas: 11
    pollingInterval: 30
    cooldownPeriod: 300
    restoreToOriginalReplicaCount: false
    scaledObject:
      annotations: {}
      # Custom annotations for ScaledObject resource
      #  annotations:
      # key: value
    triggers: []
 #     - type: prometheus
 #       metadata:
 #         serverAddress: http://<prometheus-host>:9090
 #         metricName: http_requests_total
 #         threshold: '100'
 #         query: sum(rate(http_requests_total{deployment="my-deployment"}[2m]))
    behavior: {}
 #     scaleDown:
 #       stabilizationWindowSeconds: 300
 #       policies:
 #       - type: Pods
 #         value: 1
 #         periodSeconds: 180
 #     scaleUp:
 #       stabilizationWindowSeconds: 300
 #       policies:
 #       - type: Pods
 #         value: 2
 #         periodSeconds: 60
  ## Enable mimalloc as a drop-in replacement for malloc.
  ## ref: https://github.com/microsoft/mimalloc
  ##
  enableMimalloc: true
  ## Override NGINX template
  customTemplate:
    configMapName: ""
    configMapKey: ""
  service:
    enabled: true
    annotations: {}
    labels: {}
    # clusterIP: ""
    ## List of IP addresses at which the controller services are available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []
    # loadBalancerIP: ""
    loadBalancerSourceRanges: []
    enableHttp: true
    enableHttps: true
    ## Set external traffic policy to: "Local" to preserve source IP on
    ## providers supporting it
    ## Ref: https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer
    # externalTrafficPolicy: ""
    # Must be either "None" or "ClientIP" if set. Kubernetes will default to "None".
    # Ref: https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
    # sessionAffinity: ""
    # specifies the health check node port (numeric port number) for the service. If healthCheckNodePort isn’t specified,
    # the service controller allocates a port from your cluster’s NodePort range.
    # Ref: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
    # healthCheckNodePort: 0
    ports:
      http: 80
      https: 443
    targetPorts:
      http: http
      https: https
    type: ClusterIP
    # type: NodePort
    # nodePorts:
    #   http: 32080
    #   https: 32443
    #   tcp:
    #     8080: 32808
    nodePorts:
      http: ""
      https: ""
      tcp: {}
      udp: {}
    ## Enables an additional internal load balancer (besides the external one).
    ## Annotations are mandatory for the load balancer to come up. Varies with the cloud service.
    internal:
      enabled: false
      annotations: {}
      # loadBalancerIP: ""
      ## Restrict access For LoadBalancer service. Defaults to 0.0.0.0/0.
      loadBalancerSourceRanges: []
      ## Set external traffic policy to: "Local" to preserve source IP on
      ## providers supporting it
      ## Ref: https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer
      # externalTrafficPolicy: ""
  extraContainers: []
  ## Additional containers to be added to the controller pod.
  ## See https://github.com/lemonldap-ng-controller/lemonldap-ng-controller as example.
  #  - name: my-sidecar
  #    image: nginx:latest
  #  - name: lemonldap-ng-controller
  #    image: lemonldapng/lemonldap-ng-controller:0.2.0
  #    args:
  #      - /lemonldap-ng-controller
  #      - --alsologtostderr
  #      - --configmap=$(POD_NAMESPACE)/lemonldap-ng-configuration
  #    env:
  #      - name: POD_NAME
  #        valueFrom:
  #          fieldRef:
  #            fieldPath: metadata.name
  #      - name: POD_NAMESPACE
  #        valueFrom:
  #          fieldRef:
  #            fieldPath: metadata.namespace
  #    volumeMounts:
  #    - name: copy-portal-skins
  #      mountPath: /srv/var/lib/lemonldap-ng/portal/skins
  extraVolumeMounts: []
  ## Additional volumeMounts to the controller main container.
  #  - name: copy-portal-skins
  #   mountPath: /var/lib/lemonldap-ng/portal/skins
  extraVolumes: []
  ## Additional volumes to the controller pod.
  #  - name: copy-portal-skins
  #    emptyDir: {}
  extraInitContainers: []
  ## Containers, which are run before the app containers are started.
  # - name: init-myservice
  #   image: busybox
  #   command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  admissionWebhooks:
    annotations: {}
    enabled: true
    failurePolicy: Fail
    # timeoutSeconds: 10
    port: 8443
    certificate: "/usr/local/certificates/cert"
    key: "/usr/local/certificates/key"
    namespaceSelector: {}
    objectSelector: {}
    # Use an existing PSP instead of creating one
    existingPsp: ""
    service:
      annotations: {}
      # clusterIP: ""
      externalIPs: []
      # loadBalancerIP: ""
      loadBalancerSourceRanges: []
      servicePort: 443
      type: ClusterIP
    createSecretJob:
      resources: {}
        # limits:
        #   cpu: 10m
        #   memory: 20Mi
        # requests:
        #   cpu: 10m
        #   memory: 20Mi
    patchWebhookJob:
      resources: {}
    patch:
      enabled: true
      image:
        registry: liangjw
        image: kube-webhook-certgen
        # for backwards compatibility consider setting the full image url via the repository value below
        # use *either* current default registry/image or repository format or installing chart by providing the values.yaml will fail
        # repository:
        tag: v1.0
        #digest: sha256:f3b6b39a6062328c095337b4cadcefd1612348fdd5190b1dcbcb9b9e90bd8068
        pullPolicy: IfNotPresent
      ## Provide a priority class name to the webhook patching job
      ##
      priorityClassName: ""
      podAnnotations: {}
      nodeSelector:
        kubernetes.io/os: linux
      tolerations: []
      runAsUser: 2000
  metrics:
    port: 10254
    # if this port is changed, change healthz-port: in extraArgs: accordingly
    enabled: false
    service:
      annotations: {}
      # prometheus.io/scrape: "true"
      # prometheus.io/port: "10254"
      # clusterIP: ""
      ## List of IP addresses at which the stats-exporter service is available
      ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
      ##
      externalIPs: []
      # loadBalancerIP: ""
      loadBalancerSourceRanges: []
      servicePort: 10254
      type: ClusterIP
      # externalTrafficPolicy: ""
      # nodePort: ""
    serviceMonitor:
      enabled: false
      additionalLabels: {}
      # The label to use to retrieve the job name from.
      # jobLabel: "app.kubernetes.io/name"
      namespace: ""
      namespaceSelector: {}
      # Default: scrape .Release.Namespace only
      # To scrape all, use the following:
      # namespaceSelector:
      #   any: true
      scrapeInterval: 30s
      # honorLabels: true
      targetLabels: []
      metricRelabelings: []
    prometheusRule:
      enabled: false
      additionalLabels: {}
      # namespace: ""
      rules: []
        # # These are just examples rules, please adapt them to your needs
        # - alert: NGINXConfigFailed
        #   expr: count(nginx_ingress_controller_config_last_reload_successful == 0) > 0
        #   for: 1s
        #   labels:
        #     severity: critical
        #   annotations:
        #     description: bad ingress config - nginx config test failed
        #     summary: uninstall the latest ingress changes to allow config reloads to resume
        # - alert: NGINXCertificateExpiry
        #   expr: (avg(nginx_ingress_controller_ssl_expire_time_seconds) by (host) - time()) < 604800
        #   for: 1s
        #   labels:
        #     severity: critical
        #   annotations:
        #     description: ssl certificate(s) will expire in less then a week
        #     summary: renew expiring certificates to avoid downtime
        # - alert: NGINXTooMany500s
        #   expr: 100 * ( sum( nginx_ingress_controller_requests{status=~"5.+"} ) / sum(nginx_ingress_controller_requests) ) > 5
        #   for: 1m
        #   labels:
        #     severity: warning
        #   annotations:
        #     description: Too many 5XXs
        #     summary: More than 5% of all requests returned 5XX, this requires your attention
        # - alert: NGINXTooMany400s
        #   expr: 100 * ( sum( nginx_ingress_controller_requests{status=~"4.+"} ) / sum(nginx_ingress_controller_requests) ) > 5
        #   for: 1m
        #   labels:
        #     severity: warning
        #   annotations:
        #     description: Too many 4XXs
        #     summary: More than 5% of all requests returned 4XX, this requires your attention
  ## Improve connection draining when ingress controller pod is deleted using a lifecycle hook:
  ## With this new hook, we increased the default terminationGracePeriodSeconds from 30 seconds
  ## to 300, allowing the draining of connections up to five minutes.
  ## If the active connections end before that, the pod will terminate gracefully at that time.
  ## To effectively take advantage of this feature, the Configmap feature
  ## worker-shutdown-timeout new value is 240s instead of 10s.
  ##
  lifecycle:
    preStop:
      exec:
        command:
          - /wait-shutdown
  priorityClassName: ""
revisionHistoryLimit: 10
defaultBackend:
  ##
  enabled: false
  name: defaultbackend
  image:
    registry: mirrorgooglecontainers
    image: defaultbackend-amd64
    # for backwards compatibility consider setting the full image url via the repository value below
    # use *either* current default registry/image or repository format or installing chart by providing the values.yaml will fail
    # repository:
    tag: "1.5"
    pullPolicy: IfNotPresent
    # nobody user -> uid 65534
    runAsUser: 65534
    runAsNonRoot: true
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
  # Use an existing PSP instead of creating one
  existingPsp: ""
  extraArgs: {}
  serviceAccount:
    create: true
    name: ""
    automountServiceAccountToken: true
  ## Additional environment variables to set for defaultBackend pods
  extraEnvs: []
  port: 8080
  ## Readiness and liveness probes for default backend
  ## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
  ##
  livenessProbe:
    failureThreshold: 3
    initialDelaySeconds: 30
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 5
  readinessProbe:
    failureThreshold: 6
    initialDelaySeconds: 0
    periodSeconds: 5
    successThreshold: 1
    timeoutSeconds: 5
  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
  #  - key: "key"
  #    operator: "Equal|Exists"
  #    value: "value"
  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"
  affinity: {}
  ## Security Context policies for controller pods
  ## See https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/ for
  ## notes on enabling and using sysctls
  ##
  podSecurityContext: {}
  # labels to add to the pod container metadata
  podLabels: {}
  #  key: value
  ## Node labels for default backend pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector:
    kubernetes.io/os: linux
  ## Annotations to be added to default backend pods
  ##
  podAnnotations: {}
  replicaCount: 1
  minAvailable: 1
  resources: {}
  # limits:
  #   cpu: 10m
  #   memory: 20Mi
  # requests:
  #   cpu: 10m
  #   memory: 20Mi
  extraVolumeMounts: []
  ## Additional volumeMounts to the default backend container.
  #  - name: copy-portal-skins
  #   mountPath: /var/lib/lemonldap-ng/portal/skins
  extraVolumes: []
  ## Additional volumes to the default backend pod.
  #  - name: copy-portal-skins
  #    emptyDir: {}
  autoscaling:
    annotations: {}
    enabled: false
    minReplicas: 1
    maxReplicas: 2
    targetCPUUtilizationPercentage: 50
    targetMemoryUtilizationPercentage: 50
  service:
    annotations: {}
    # clusterIP: ""
    ## List of IP addresses at which the default backend service is available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []
    # loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    type: ClusterIP
  priorityClassName: ""
rbac:
  create: true
  scope: false
podSecurityPolicy:
  enabled: false
serviceAccount:
  create: true
  name: ""
  automountServiceAccountToken: true
imagePullSecrets: []
tcp: {}
udp: {}
dhParam:
[root@k8s-master01 ingress-nginx]# 

```

#### 开始部署

```
kubectl label node k8s-node01 ingress=true
kubectl create ns ingress-nginx
helm install ingress-nginx -n ingress-nginx .
查看：
[root@k8s-master01 ingress-nginx]# kubectl get po -n ingress-nginx -owide
NAME                             READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
ingress-nginx-controller-klpbc   1/1     Running   0          33m   192.168.10.183   k8s-node01   <none>           <none>
```

***注意***

 将ingress controller部署至Node节点（ingress controller不能部署在master节点，需要将ingress controller部署至Node节点，生产环境最少三个ingress controller，并且最好是独立的节点）

想要某个节点部署直接对应节点打label即可

```
kubectl label node k8s-node02 ingress=true
```

### Ingress-nginx代理域名

```
4.0.1版本语法与3.x版本语法有些稍微区别   ingress有namespace的区别
[root@k8s-master01 yaml]# cat ingress.yaml 
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
  - host: foo.bar.com  #域名 域名配置，可以不写，匹配*， *.bar.com
    http:
      paths:
      - path: /   # 相当于nginx的location配合，同一个host可以配置多个path / /abc
        pathType: Prefix
        backend:
          service:
            name: my-service  #SVC的名称
            port:
              number: 8080  #svc的port
              
[root@k8s-master01 yaml]# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP          46h
my-service   NodePort    10.96.55.33   <none>        8080:31000/TCP   27h
```

创建

```
kubectl create -f ingress.yaml
查看
[root@k8s-master01 yaml]# kubectl get ingress
NAME      CLASS    HOSTS         ADDRESS        PORTS   AGE
example   <none>   foo.bar.com   10.96.145.18   80      21h
```

本地做host访问，ip是ingress部署机器的ip

多域名访问

```
只需要在刚才的配置文件中在加入一个host即可
- host: foo.bar01.com  #域名
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service01  #SVC的名称
            port:
              number: 8081  #svc的port
```




## Job Or CronJob简单使用

### JObs

```
Job 会创建一个或者多个 Pods，并将继续重试 Pods 的执行，直到指定数量的 Pods 成功终止。 随着 Pods 成功结束，Job 跟踪记录成功完成的 Pods 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job 的操作会清除所创建的全部 Pods。 挂起 Job 的操作会删除 Job 的所有活跃 Pod，直到 Job 被再次恢复执行。
类似一次性任务
```

例如

```
[root@k8s-master01 yaml]# cat job.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    job-name: echo
  name: echo
  namespace: default
spec:
  suspend: true # 1.21+
  ttlSecondsAfterFinished: 100
  backoffLimit: 4
  completions: 5
  parallelism: 3
  template:
    spec:
      containers:
      - command:
        - echo
        - Hello, Job
        image: registry.cn-beijing.aliyuncs.com/dotbalo/busybox
        imagePullPolicy: IfNotPresent
        name: echo
        resources: {}
      restartPolicy: Never
```

解释

```
backoffLimit: 如果任务执行失败，失败多少次后不再执行
completions：有多少个Pod执行成功，认为任务是成功的
为空默认和parallelism数值一样
parallelism：并行执行任务的数量
如果parallelism数值大于未完成任务数，只会创建未完成的数量；
     比如completions是4，并发是3，第一次会创建3个Pod执行任务，
     第二次只会创建一个Pod执行任务
ttlSecondsAfterFinished：Job在执行结束之后（状态为completed或Failed）自动清理。设置为0表示执行结束立即删除，不设置则不会清除，需要开启TTLAfterFinished特性
```

执行验证

```
kubectl create -f job.yaml
[root@k8s-master01 yaml]# kubectl get pod  很快执行了5次执行完成
NAME                                READY   STATUS      RESTARTS   AGE
echo-6kpxq                          0/1     Completed   0          3s
echo-bxq2m                          0/1     Completed   0          4s
echo-gpthq                          0/1     Completed   0          4s
echo-q229k                          0/1     Completed   0          4s
echo-qsclh                          0/1     Completed   0          3s
[root@k8s-master01 yaml]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
echo   5/5           3s         9s
```

### CronJob

```
一个 CronJob 对象就像 crontab (cron table) 文件中的一行。 它用 Cron 格式进行编写， 并周期性地在给定的调度时间执行 Job。
注意：
所有 CronJob 的 schedule: 时间都是基于 kube-controller-manager. 的时区
CronJobs 对于创建周期性的、反复重复的任务很有用，例如执行数据备份或者发送邮件。 CronJobs 也可以用来计划在指定时间来执行的独立任务，例如计划当集群看起来很空闲时 执行某个 Job。
 CronJob 控制器每 10 秒钟执行一次检查
对于每个 CronJob，CronJob 控制器（Controller） 检查从上一次调度的时间点到现在所错过了调度次数。如果错过的调度次数超过 100 次， 那么它就不会启动这个任务
```

案例

```
[root@k8s-master01 yaml]# cat cronjob.yaml 
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  labels:
    run: hello
  name: hello
  namespace: default
spec:
  concurrencyPolicy: Allow
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
    spec:
      template:
        metadata:
          labels:
            run: hello
        spec:
          containers:
          - args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
            image: registry.cn-beijing.aliyuncs.com/dotbalo/busybox
            imagePullPolicy: IfNotPresent
            name: hello
            resources: {}
          restartPolicy: OnFailure
          securityContext: {}
  schedule: '*/1 * * * *'
  successfulJobsHistoryLimit: 3
  suspend: false
```

说明

```
apiVersion: batch/v1beta1 #1.21+ batch/v1
schedule：调度周期，和Linux一致，分别是分时日月周。
restartPolicy：重启策略，和Pod一致。
concurrencyPolicy：并发调度策略。可选参数如下：
         Allow：允许同时运行多个任务。
         Forbid：不允许并发运行，如果之前的任务尚未完成，新的任务不会被创建。
         Replace：如果之前的任务尚未完成，新的任务会替换的之前的任务。
suspend：如果设置为true，则暂停后续的任务，默认为false。停止计划任务执行
successfulJobsHistoryLimit：保留多少已完成的任务，按需配置。默认3
failedJobsHistoryLimit：保留多少失败的任务。默认1
```

查看

```
[root@k8s-master01 yaml]# kubectl get cj
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        38s             2m57s
[root@k8s-master01 yaml]# kubectl get job
NAME             COMPLETIONS   DURATION   AGE
hello-27193816   1/1           1s         2m45s
hello-27193817   1/1           1s         105s
hello-27193818   1/1           1s         45s
[root@k8s-master01 yaml]# kubectl get pod
NAME                                READY   STATUS      RESTARTS   AGE
hello-27193816-fb5fb                0/1     Completed   0          2m48s
hello-27193817-xgnf5                0/1     Completed   0          108s
hello-27193818-qb55s                0/1     Completed   0          48s
```


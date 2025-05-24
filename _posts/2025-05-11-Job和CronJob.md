---
title: Job和CronJob
date: 2025-05-11 22:34:00 +0800
categories: [k8s,Job和CronJob]
tags: [k8s,Job和CronJob]
---

Job
负责处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。
CronJob 
就是在 Job 上加上了时间调度。

##### 1、job示例
```yaml
apiVersion: batch/v1            #api版本
kind: Job                       #资源类型
metadata:                       
  name: job-demo
spec:
  parallelism: 2                #并行执行任务的数量
  completions: 4                #有多少个Pod执行成功，认为任务是成功的
  template:
    spec:
      restartPolicy: Never      #Pod重启策略，必须填写，job只能为Never和OnFailure
      containers:
      - name: counter
        image: busybox
        command:
        - "bin/sh"
        - "-c"
        - "for i in 9 8 7 6 5 4 3 2 1; do echo $i; done"
  backoffLimit: 4                   ##如果任务执行失败，失败多少次后不再执行，默认值为6    
  activeDeadlineSeconds: 100        #最长运行时间，一旦运行超过100s，这个job的所有pod都会被终止
```
```shell
[root@k8smaster job]# kubectl get job
NAME       COMPLETIONS   DURATION   AGE
job-demo   3/4           25s        25s
[root@k8smaster job]# kubectl get pods
NAME                             READY   STATUS      RESTARTS   AGE
job-demo-fm2wt                   0/1     Completed   0          20s
job-demo-fs9zc                   0/1     Completed   0          26s
job-demo-g6swq                   0/1     Completed   0          26s
job-demo-qbsfq                   0/1     Completed   0          23s
```
查看某一个pod的日志输出
```shell
[root@k8smaster job]# kubectl logs job-demo-qbsfq
9
8
7
6
5
4
3
2
1
```
删除这个job
```shell
[root@k8smaster job]# kubectl delete jobs job-demo 
job.batch "job-demo" deleted
```
如果的任务执行失败了，我们这里定义了 restartPolicy=Never，那么任务在执行失败后 Job 控制器就会不断地尝试创建一个新 Pod，当然，这个尝试肯定不能无限进行下去。我们可以通过 Job 对象的 spec.backoffLimit 字段来定义重试次数，另外需要注意的是 Job 控制器重新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10s、20s、40s… 后。

如果我们定义的 restartPolicy=OnFailure，那么任务执行失败后，Job 控制器就不会去尝试创建新的 Pod了，它会不断地尝试重启 Pod 里的容器。

job任务对应的pod运行结束后，会变成Completed状态

##### 2、cronjob示例
cronjob其实就是job的Controller，和Deployment 与 ReplicaSet 的关系一样
一个 CronJob 对象其实就对应中 crontab 文件中的一行，它根据配置的时间格式周期性地运行一个Job，格式和 crontab 也是一样的。
```yaml
apiVersion: batch/v1
kind: CronJob                       #资源类型
metadata:
  name: cronjob-demo
spec:
  schedule: "*/1 * * * *"           #调度周期，和Linux一致，分别是分时日月周。
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: counter
            image: busybox
            command:
            - "bin/sh"
            - "-c"
            - "for i in 9 8 7 6 5 4 3 2 1; do echo $i; done"
```
查看cronjib
```shell
[root@k8smaster job]# kubectl get cronjobs.batch cronjob-demo 
NAME           SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob-demo   */1 * * * *   False     0        <none>          10s
```
运行一段时间后，查看多出来几个pod，每分钟运行一次
```shell
[root@k8smaster nodeSelector]# kubectl get pods
NAME                             READY   STATUS      RESTARTS   AGE
cronjob-demo-1663756800-62zz2    0/1     Completed   0          112s
cronjob-demo-1663756860-j87f7    0/1     Completed   0          52s
```
删除这个cronjob
```shell
[root@k8smaster nodeSelector]# kubectl delete cronjobs cronjob-demo 
cronjob.batch "cronjob-demo" delete
```
由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。这时候，你可以通过 spec.concurrencyPolicy 字段来定义具体的处理策略。
比如：
    concurrencyPolicy=Allow,这也是默认情况，这意味着这些 Job 可以同时存在；
    concurrencyPolicy=Forbid,如果之前的任务尚未完成，新的任务不会被创建
    concurrencyPolicy=Replace,这意味着新产生的 Job 会替换旧的、没有执行完的 Job。
cronjob yaml文件详解
```yaml
apiVersion: batch/v1beta1  #api版本，1.21+为batch/v1
kind: CronJob              #资源类型
metadata:                  #源数据定义
  name: hello              #名称
  namespace: default       #命名空间
spec:                      #具体定义
  concurrencyPolicy: Allow #并发调度策略。
  failedJobsHistoryLimit: 1 #保留多少失败的任务 按需配置。
  schedule: '*/1 * * * *'   #调度周期，和Linux一致，分别是分时日月周。
  successfulJobsHistoryLimit: 3 #保留多少已完成的任务，按需配置。
  suspend: false          #如果设置为true，则暂停后续的任务，默认为false。
  jobTemplate:            #这个为job数据定义，可以设置metadata.labels标签  
    spec:                 #以下为Pod设置
      template:
        spec:
          containers:    
          - name: hello
          ...
          restartPolicy: OnFailure  #必须设置，重启策略，和Pod一致。
```
---
title: Deployment
date: 2025-05-11 22:34:00 +0800
categories: [k8s,Deployment]
tags: [k8s,Deployment]
toc: true
---

Deployment 实现了k8s中一个非常重要的功能：Pod 的“水平扩展 / 收缩”（horizontal scaling out/in）
如果你更新了 Deployment 的 Pod 模板（比如，修改了容器的镜像），那么 Deployment 就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。
而这个能力的实现，依赖的是 Kubernetes 项目中的一个非常重要的概念（API 对象）：ReplicaSet。

一个 ReplicaSet 对象就是由副本数目的定义和一个 Pod 模板组成的。ReplicaSet是 Deployment 的一个子集。Deployment 控制的是ReplicaSet 对象，而不是 Pod 对象。
Deployment 实际上是一个两层控制器。首先，它通过 ReplicaSet 的个数来描述应用的版本；然后，它再通过 ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量。
Deployment 控制 ReplicaSet（版本），ReplicaSet 控制 Pod（副本数）。
Deployment-->ReplicaSet-->Pod
其中，ReplicaSet 负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数（比如，3 个）。这也正是 Deployment 只允许容器的 restartPolicy=Always 的主要原因：只有在容器能保证自己始终是 Running 状态的前提下，ReplicaSet 调整 Pod 的个数才有意义。
而在此基础上，Deployment 同样通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作。
其中，“水平扩展 / 收缩”非常容易实现，Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以了。
```yaml
apiVersion: apps/v1  #必选，api版本号
kind: ReplicaSet     #必选，资源类型
metadata:            #必选，元数据
  name: nginx-set    #必选，名称
  namespace: default #可选，命名空间
  labels:            #可选，标签
    app: nginx
spec:                       #必选，定义控制器信息
  replicas: 3               #必选，期望副本数
  revisionHistoryLimit: 10  #可选，保留历史rs控制器数据，默认为10
  strategy:                 #可选，更新升级策略，默认值如下
    rollingUpdate:          #滚动更新的配置参数
      maxSurge: 25%         #允许更新时pod副本超出期望值的百分比，默认25,也可以写个数maxSurge: 1
      maxUnavailable: 25%   #更新过程中不可用的pod副本的百分比，默认25,也可以写个数maxUnavailable: 1
    type: RollingUpdate     #更新升级策略，RollingUpdate滚动更新，Recreate重建，默认滚动更新
  selector:                 #必选，选择器
    matchLabels:            #标签选择器，主要用于选择下面template所定义的标签信息
      app: nginx            #匹配的pod标签
  template:                 #必选，pod模板信息,请查看pod详解
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```

###### 0、 kubectl get deployments.apps -owide详解
```shell
NAME     READY  UP-TO-DATE  AVAILABLE   AGE    CONTAINERS     IMAGES                                SELECTOR
nginx-1  1/1    1           1           7m39s  nginx          10.122.6.81:5000/image/nginx:v1      app=nginx-1
```

NAME：      Deployment名称
READY：     Pod的状态，已经Ready的个数
UP-TO-DATE：已经达到期望状态的被更新的副本数
AVAILABLE： 已经可以用的副本数
AGE：       显示应用程序运行的时间
CONTAINERS：容器名称
IMAGES：    容器的镜像
SELECTOR：  管理的Pod的标签
    
###### 1、deployment的水平扩展
把副本数改为2，原本为3，改为4个副本只需要将2改为4
```shell
[root@k8smaster build-yaml]# kubectl scale replicaset nginx-set --replicas=2
replicaset.extensions/nginx-set scaled
```
再将副本改为3
```shell
[root@k8smaster build-yaml]# kubectl scale deployment nginx-deployment --replicas=3
[root@k8smaster build-yaml]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5754944d6c   0         0         0       18h
nginx-deployment-6f655f5d99   0         0         0       18h
nginx-deployment-6f859b4555   3         3         2       18h
nginx-set                     0         0         0       18m
```
DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；
CURRENT：当前处于 Running 状态的 Pod 的个数；

###### 2、Deployment的更新
Deployment的更新可以使用yaml文件进行修改之后使用kubectl apply -f进行更新，也可以直接使用kubectl edit直接进行修改，也可以kubectl set进行更新

`kubectl edit deployment/nginx-deployment`
`kubectl set image deploy <deployname> xxx --record`
--record #会记录更新的内容，之后使用kubectl rollout history可以看到更新的参数

1. 更新容器镜像
`kubectl set image deploy nginx-1 nginx-1=10.122.6.81:5000/image/nginx:v2 --record`
2. 更新容器资源限制
`kubectl set resources deploy/deploy名称 --containers=容器名称 --limits=cpu=40m,memory=100Mi --requests=cpu=40m,memory=100Mi --record`
  --limits #容器最大可使用资源
  --requests #容器启动时申请的资源
  --containers #pod中如果存在多个容器，可以选择容器进行更新
  --all    #pod中所以容器全部设置，不写--containers参数默认all
3. 更新容器变量
`kubectl set env deploy/deploy名称 --containers=容器名称 --env=环境变量=值 --record`
#查看deploy中容器所有变量
`kubectl set env deploy/deploy名称 --list`
#修改变量
`kubectl set env deploy/nginx-1 --containers=centos type=centos7 name=bxw`
  --list #查看pod中所有容器变量
  --containers #选项pod中容器，不写默认所有

###### 3、Deployment的回滚
deployment每次更新时会创建一个新的rs控制器用来运行更新后的pod，旧的rs控制器副本数会变为0，所以我们可以使用kubectl rollout命令进行回滚操作，并且还可以设置停止更新，之后在恢复更新。
1. 查看deploy历史版本信息，默认会保存最近的10个，由deploy的.deploy.spce.revisionHistoryLimit设置,如果为0，就再也不能进行回滚操作了
`kubectl rollout history deploy/<deployname>`
```shell
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```

2. 查看指定版本详细信息
`kubectl rollout history deployment nginx-1  --revision=1`
--revisin  #指定版本号

3. 验证deploy升级状态
`kubectl rollout status deployment nginx-1 --timeout=1s`
--timeout  #检测超时时间，默认为一直监听状态直到完成或失败

4. 回滚deploy
注意：回滚并不会一直回滚之前的版本，如果当前版本为7，回滚后会把6变成8(就没有6版本了，版本信息为5 7 8)，如果再次回滚还会回滚到7，把7变为9。以此类推。

1) 回滚到指定版本
`kubectl rollout undo deployment nginx-1 --to-revision=6`
--ro-revision #指定回滚的版本，不指定默认为上一个版本

5. 暂停更新与恢复更新
如果使用命令进行更新deploy,需要更新的内容较多的话，需要使用到暂停更新来避免每次修改会生成一个更新，而是先暂停等所以更新操作完成后，在启用更新生成一次更新信息，以方便后续有问题的回滚操作。
1) 暂停更新
`kubectl rollout pause deployment <deployname>`
2) 启用更新
`kubectl rollout resume deployment <deployname>`

#示例
`kubectl rollout history deployment nginx-1`  #查看记录现在的版本号
`kubectl rollout pause deployment nginx-1`  #暂停更新
`kubectl set resources deploy/nginx-1 --containers=nginx --limits=cpu=101m,memory=100Mi --requests=cpu=101m,memory=100Mi`
`kubectl set env deploy/nginx-1 --containers=centos type=centos10 name=zhangzhuo zhangzhuo=xxxx`
`kubectl rollout resume deployment nginx-1` #启用更新
`kubectl rollout history deployment nginx-1`  #再次验证版本号是否只增加1

###### 4、滚动更新过程
查看deployment/nginx-deployment的滚动更新状态\
```shell
[root@k8smaster build-yaml]# kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 2 of 3 updated replicas are available...
```
在这个返回结果中，“2 out of 3 new replicas have been updated”意味着已经有 2 个 Pod 进入了 UP-TO-DATE 状态
修改 Deployment 有很多方法。比如，我可以直接使用 kubectl edit 指令编辑 Etcd 里的 API 对象。
```shell
[root@k8smaster build-yaml]# kubectl edit deployments/nginx-deployment
deployment.extensions/nginx-deployment edited
```
这个 kubectl edit 指令，会帮你直接打开 nginx-deployment 的 API 对象。然后，你就可以修改这里的 Pod 模板部分了。比如，在这里，我将 nginx 镜像的版本升级到了 1.9
kubectl edit 并不神秘，它不过是把 API 对象的内容下载到了本地文件，让你修改完成后再提交上去。
kubectl edit 指令编辑完成后，保存退出，Kubernetes 就会立刻触发“滚动更新”的过程。你还可以通过 kubectl rollout status 指令查看 nginx-deployment 的状态变化：
此时可用使用如下命令查看
```shell
[root@k8smaster build-yaml]# kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
```
也可以通过events查看滚动更新过程
```shell
[root@k8smaster build-yaml]# kubectl describe deployment nginx-deployment
Events:
  Type    Reason             Age                  From                   Message
  ----    ------             ----                 ----                   -------
  Normal  ScalingReplicaSet  28m (x2 over 32m)    deployment-controller  Scaled down replica set nginx-set to 1
  Normal  ScalingReplicaSet  17m (x2 over 27m)    deployment-controller  Scaled down replica set nginx-set to 0
  Normal  ScalingReplicaSet  16m (x2 over 38m)    deployment-controller  Scaled up replica set nginx-deployment-6f859b4555 to 3
  Normal  ScalingReplicaSet  9m7s                 deployment-controller  Scaled up replica set nginx-deployment-585d66c554 to 1
  Normal  ScalingReplicaSet  7m40s (x2 over 27m)  deployment-controller  Scaled down replica set nginx-deployment-6f859b4555 to 2
  Normal  ScalingReplicaSet  7m40s                deployment-controller  Scaled up replica set nginx-deployment-585d66c554 to 2
  Normal  ScalingReplicaSet  6m8s                 deployment-controller  Scaled down replica set nginx-deployment-6f859b4555 to 1
  Normal  ScalingReplicaSet  6m8s                 deployment-controller  Scaled up replica set nginx-deployment-585d66c554 to 3
```
如此交替进行，新 ReplicaSet 管理的 Pod 副本数，从 0 个变成 1 个，再变成 2 个，最后变成 3 个。而旧的 ReplicaSet 管理的 Pod 副本数则从 3 个变成 2 个，再变成 1 个，最后变成 0 个。这样，就完成了这一组 Pod 的版本升级过程。
像这样，将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”。
使用如下命令查看新旧两个replicaset
```shell
[root@k8smaster build-yaml]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5754944d6c   0         0         0       19h
nginx-deployment-585d66c554   3         3         2       14m
nginx-deployment-6f655f5d99   0         0         0       19h
nginx-deployment-6f859b4555   1         1         1       19h
nginx-set                     0         0         0       38m
```
这种滚动更新的好处
在升级刚开始的时候，集群里只有 1 个新版本的 Pod。如果这时，新版本 Pod 有问题启动不起来，那么“滚动更新”就会停止，
这也就要求你一定要使用 Pod 的 Health Check 机制检查应用的运行状态，而不是简单地依赖于容器的 Running 状态。要不然的话，虽然容器已经变成 Running 了，但服务很有可能尚未启动，“滚动更新”的效果也就达不到了。
这里使用的是rediniess健康检查
而为了进一步保证服务的连续性，Deployment Controller 还会确保，在任何时间窗口内，只有指定比例的 Pod 处于离线状态。同时，它也会确保，在任何时间窗口内，只有指定比例的新 Pod 被创建出来。这两个比例的值都是可以配置的，默认都是 DESIRED 值的 25%。

在上面这个 Deployment 的例子中，它有 3 个 Pod 副本，那么控制器在“滚动更新”的过程中永远都会确保至少有 2 个 Pod 处于可用状态，至多只有 4 个 Pod 同时存在于集群中。这个策略，是 Deployment 对象的一个字段，名叫 RollingUpdateStrategy，
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
在上面这个 RollingUpdateStrategy 的配置中，maxSurge 指定的是除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod；而 maxUnavailable 指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod。
这两个配置还可以用前面我们介绍的百分比形式来表示，比如：maxUnavailable=50%，指的是我们最多可以一次删除“50%*DESIRED 数量”个 Pod。
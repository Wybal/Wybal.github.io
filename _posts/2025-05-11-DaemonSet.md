---
title: DaemonSet
date: 2025-05-11 22:34:00 +0800
categories: [k8s,DaemonSet]
tags: [k8s,DaemonSet]
---

DaemonSet 的主要作用，是让你在 Kubernetes 集群里，运行一个 Daemon Pod。 所以，这个 Pod 有如下三个特征：
1、这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
2、每个节点上只有一个这样的 Pod 实例；
3、当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

常用在以下几种例子
1、各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
2、各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；
3、各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。

DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早。
这个时候，整个 Kubernetes 集群里还没有可用的容器网络，所有 Worker 节点的状态都是 NotReady（NetworkReady=false）。这种情况下，普通的 Pod 肯定不能运行在这个集群上。
[root@k8smaster DaemonSet]# cat daemonset.yml 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: registry.aliyuncs.com/google_containers/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
这个 DaemonSet，管理的是一个 fluentd-elasticsearch 镜像的 Pod。这个镜像的功能非常实用：通过 fluentd 将 Docker 容器里的日志转发到 ElasticSearch 中

DaemonSet 又是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？
DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 name=fluentd-elasticsearch 标签的 Pod 在运行。
而检查的结果，可能有这么三种情况
1、没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
2、有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；
3、正好只有一个这种 Pod，那说明这个节点是正常的。

如何在指定的node上创建新pod呢？
nodeAffinity
nodeAffinity 相较于 nodeSelector 的优势在于: 1. 语言更具表现力（不仅仅是“对完全匹配规则的 AND”） 2. 你可以发现规则是“软需求”/“偏好”，而不是硬性要求，因此， 如果调度器无法满足该要求，仍然调度该 Pod 3. 你可以使用节点上（或其他拓扑域中）的 Pod 的标签来约束，而不是使用 节点本身的标签，来允许哪些 pod 可以或者不可以被放置在一起。 

apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime

1、requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个 nodeAffinity 必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个 nodeAffinity；
2、这个 Pod，将来只允许运行在“metadata.name”是“node-geektime”的节点上

operator: In（即：部分匹配；如果你定义 operator: Equal，就是完全匹配）
DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 nodeAffinity 定义。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。
DaemonSet 并不需要修改用户提交的 YAML 文件里的 Pod 模板，而是在向 Kubernetes 发起请求之前，直接修改根据模板生成的 Pod 对象
此外，DaemonSet 还会给这个 Pod 自动加上另外一个与调度相关的字段，叫作 tolerations。这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）
而 DaemonSet 自动加上的 tolerations 字段，格式如下所示：

apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
这个 Toleration 的含义是：“容忍”所有被标记为 unschedulable“污点”的 Node；“容忍”的效果是允许调度。
通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。
这种机制，正是我们在部署 Kubernetes 集群的时候，能够先部署 Kubernetes 本身、再部署网络插件的根本原因：因为当时我们所创建的 flannel 的 YAML，实际上就是一个 DaemonSet。
在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个 Toleration，从而忽略节点的 unschedulable“污点”。
当然，你也可以在 Pod 模板里加上更多种类的 Toleration，从而利用 DaemonSet 达到自己的目的。比如，在这个 fluentd-elasticsearch DaemonSet 里，我就给它加上了这样的 Toleration：

tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule

实践：
在 DaemonSet 上，我们一般都应该加上 resources 字段，来限制它的 CPU 和内存使用，防止它占用过多的宿主机资源。
创建
kubectl create -f fluentd-elasticsearch.yaml
查看pod
[root@k8smaster DaemonSet]# kubectl get pod -n kube-system -l name=fluentd-elasticsearch  -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE        NOMINATED NODE   READINESS GATES
fluentd-elasticsearch-9nf2p   1/1     Running   0          105m   10.244.1.33   k8snode1    <none>           <none>
fluentd-elasticsearch-b4vzr   1/1     Running   0          105m   10.244.0.5    k8smaster   <none>           <none>
fluentd-elasticsearch-kgpnz   1/1     Running   0          105m   10.244.2.36   k8snode2    <none> 
查看daemonset对象
[root@k8smaster DaemonSet]# kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         3       3            3           <none>          106m
查看kubectl rollout history
[root@k8smaster DaemonSet]# kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonset.extensions/fluentd-elasticsearch 
REVISION  CHANGE-CAUSE
1         <none>
在升级命令后面加上--record参数，升级所用指令就会出现在history里

ControllerRevision其实是一个通用的版本管理对象。StatefulSet和DaemonSet都是使用这个进行版本管理
  Deployment没使用这个版本管理api对象管理版本。原因：第一，它比较老。第二，它要用replicaset。
ControllerRevision API对象，专门用来记录某种 Controller 对象的版本。
 查看ControllerRevision
[root@k8smaster DaemonSet]# kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-587b6cd698   daemonset.apps/fluentd-elasticsearch   1          111m
使用如下命令回滚到revision=1时的状态
kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
DaemonSet Controller 就可以使用这个历史 API 对象，对现有的 DaemonSet 做一次 PATCH 操作（等价于执行一次 kubectl apply -f “旧的 DaemonSet 对象”），从而把这个 DaemonSet“更新”到一个旧版本。
所以回滚完成以后并不会回退到1，而是Revision=3
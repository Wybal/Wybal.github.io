---
title: pod的resources limit
date: 2025-05-04 22:39:00 +0800
categories: [k8s,pod的resources limit]
tags: [k8s,pod的resources limit]
---

作为一个容器集群编排与管理项目，还包括了对应用的资源管理和调度的处理。

在k8s中，pod是最小的原子调度单位，这也就意味着，所有跟调度和资源管理相关的属性都应该是属于 Pod 对象的字段。

这其中最重要的部分就是pod的cpu和内存配置，如下：
cpu这样的资源被称为“可压缩资源”，它的特点就是，当可压缩资源不足时，pod只会“饥饿”，不会退出
内存这样的资源称为“不可压缩资源”，当不可压缩资源不足时，pod就会因为oom被内核杀掉
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
由于pod可以由多个container组成，所以cpu和内存资源的限额，是要配置到每个container的定义上的。
这样，pod整体的资源配置，就是由container的配置值累加得到。
其中，k8s里为cpu设置的单位是“cpu的个数”。比如 cpu=1指的就是这个pod的cpu限额是1个cpu
此外，k8s允许你将cpu限额设置为分数，比如在上面的例子里，cpu limits的值就是500m。
500m指的就是500millicpu，也就是0.5个cpu的意思。这样这个pod就会被分配到1个cpu一半的计算能力
也可以写成cpu=0.5.实际使用中还是推荐使用500m的写法
对应内存来说，它的单位是bytes。k8s支持Ei、Pi、Ti、Gi、Mi、Ki(或者E、P、T、G、M、K)的方式作为bytes的值
在上面的例子中，memory requests的值为64MiB
注意：Mi和M的区别 1Mi=1024*1024  1M=1000*1000

cpu和内存资源还要分为limits和requests两种情况
两者的区别：在调度的时候,kube-scheduler只会按照requests的值进行计算。而真正设置cgroups限制的时候，kubelet则按照limits的值来进行设置
当你指定了requests.cpu=250m之后，相当于cgroups的cpu.shares的值设置为(250/1000)*1024
而当没有设置requests.cpu的时候，cpu.shares默认则是1024.
这样,k8s就通过cpu.shares完成了对cpu时间的按比例分配

如果你指定了 limits.cpu=500m 之后，则相当于将 Cgroups 的 cpu.cfs_quota_us 的值设置为 (500/1000)*100ms，而 cpu.cfs_period_us 的值始终是 100ms。这样，Kubernetes 就为你设置了这个容器只能用到 CPU 的 50%。

而对于内存来说，当你指定了 limits.memory=128Mi 之后，相当于将 Cgroups 的 memory.limit_in_bytes 设置为 128 * 1024 * 1024。而需要注意的是，在调度的时候，调度器只会使用 requests.memory=64Mi 来进行判断。

容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，这是因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额。
用户在提交 Pod 时，可以声明一个相对较小的 requests 值供调度器使用，而 Kubernetes 真正设置给容器 Cgroups 的，则是相对较大的 limits 值。

k8s的三种Qos模型
1. 当pod里的每个container都同时设置了requests和limits，并且limit和requests值相等时，这个pod就属于Guaranteed类型
  当 Pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值，所以，这也属于 Guaranteed. 况。
2. 而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别
3. 而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort

k8s设置三种Qos类别是为了当宿主机资源紧张的时候，Kubelet对pod进行Eviction(即资源回收)时需要用到的
具体地说，当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发 Eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等。
目前，Kubernetes 为你设置的 Eviction 的默认阈值如下所示：
memory.available<100Mi
nodefs.available<10%
nodefs.inodesFree<5%
imagefs.available<15%
上述各个触发条件在kubelet里是可配置的，如下:
`kubelet --eviction-hard=imagefs.available<10%,memory.available<500Mi,nodefs.available<5%,nodefs.inodesFree<5% --eviction-soft=imagefs.available<30%,nodefs.available<10% --eviction-soft-grace-period=imagefs.available=2m,nodefs.available=2m --eviction-max-pod-grace-period=600`
在这个配置可以看到Evction在k8s里分为soft和hard两种模式
一、Soft Eviction 允许你为 Eviction 过程设置一段“优雅时间”，比如上面例子里的 imagefs.available=2m，就意味着当 imagefs 不足的阈值达到 2 分钟之后，kubelet 才会开始 Eviction 的过程。
二、Hard Eviction 模式下，Eviction 过程就会在阈值达到之后立刻开始。

当发生Eviction发生的时候，kubelet删除pod的优先级
一、是BestEffort类别
二、Burstable类别，并且发生“饥饿”的资源使用量已超出requests的pod
三、Guaranteed类别，并且只有此类别的pod的资源使用量超过limits的限制，或者宿主机本身处于memory pressure状态时，guaranteed的pod才可能被选中进行Eviction操作
对应同Qos类别的pod来说，k8s会根据pod的优先级进一步排序和选择

cpuset的设置
你可以通过设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力。这种情况下，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。
在k8s里使用如下实现：
1、pod必须是Guaranteed的Qos模型
2、pod的cpu资源的limits和requests设置为同一相等的整数值

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "200Mi"
        cpu: "2"
```
该pod就会被绑定在2个独占的cpu核上

当宿主机的Eviction阙值达到后，就会进入memorypressure或者diskpressure状态，从而避免新的pid被调度到这台宿主机
其实就是给宿主机打了污点标记
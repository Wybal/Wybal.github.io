---
title: nodename、nodeSeletor、Taint与Toleration、Affinity
date: 2025-05-13 22:34:00 +0800
categories: [k8s,nodename,nodeSeletor,Taint与Toleration,Affinity]
tags: [k8s,nodename,nodeSeletor,Taint与Toleration,Affinity]
toc: true
---



容忍度（Toleration）是应用于 Pod 上的，允许（但并不要求）Pod 调度到带有与之匹配的污点的节点上。
设计理念:Taint在一类服务器上打上污点，让不能容忍这个污点的Pod不能部署在打了污点的服务器上。Toleration是让Pod容忍节点上配置的污点，可以让一些需要特殊配置的Pod能够调度到具有污点的特殊配置的节点上

容忍与nodeSelector区别
taint被标记，如果没有Toleration不能调度到该节点，调度到有taint的node，需要有Toleration才能正常调度，但pod也有可能没有被调度到有taint的node
nodeSelector固定调度某些Pod到指定的一些节点，他的作用是强制性的，如果集群中没有符合的node，Pod会一直处于等待调度的阶段。
##### 一、直接指定nodename
.spec.nodeName将Pod直接调度到指定的Node节点上，会【跳过Scheduler的调度策略,可以越过Taints污点进行调度】，该匹配规则是【强制】匹配。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: foo-node # 调度 Pod 到特定的节点
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

使用此配置文件来创建一个 Pod，该 Pod 将只能被调度到 foo-node 节点。

##### 二、nodeSelector
强制指定pod运行在某些节点
###### 1.查看node的labels
```shell
[root@k8smaster volume]# kubectl get nodes  --show-labels
NAME        STATUS   ROLES    AGE   VERSION   LABELS
k8smaster   Ready    master   20d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,cm=test,kubernetes.io
/arch=amd64,kubernetes.io/hostname=k8smaster,kubernetes.io/os=linux,node-role.kubernetes.io/master=
k8snode1    Ready    <none>   20d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=am
d64,kubernetes.io/hostname=k8snode1,kubernetes.io/os=linux
k8snode2    Ready    <none>   20d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=am
d64,kubernetes.io/hostname=k8snode2,kubernetes.io/os=linux
```

###### 2.node添加labels
`kubectl label nodes <nodename> <key>=<value>`
```shell
[root@k8smaster volume]# kubectl label nodes k8smaster drc=test
node/k8smaster labeled
```
###### 3.node删除labels
`kubectl label nodes <nodename> <key>-`
```shell
[root@k8smaster volume]# kubectl label nodes k8smaster drc-
node/k8smaster labeled
```
###### 4.pod匹配node的labels
给k8smaster节点打上cm=test的labels
```shell
[root@k8smaster nodeSelector]# kubectl get nodes k8smaster --show-labels
NAME        STATUS   ROLES    AGE   VERSION   LABELS
k8smaster   Ready    master   20d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,cm=test,kubernetes.io
/arch=amd64,kubernetes.io/hostname=k8smaster,kubernetes.io/os=linux,node-role.kubernetes.io/master=
```
指定pod运行在携带cm=test labels的node上
```shell
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: busybox-pod
  name: test-busybox
spec:
  containers:
  - command:
    - sleep
    - "2400"
    image: busybox
    imagePullPolicy: Always
    name: test-busybox
  nodeSelector:
    cm: test
#    disk: ssd              当有多个kv时，node必须同时满足多个lables
```
查看是否运行在k8smaster节点上
```shell
[root@k8smaster nodeSelector]# kubectl get pods -o wide
test-busybox                        1/1     Running   0          18s     10.244.0.4    k8smaster   <none> 
```

**当pod已经运行在节点上，即使删除掉节点上的labels，也不影响被调度到当前节点已经运行的pod**

##### 二、Taint和Toleration
污点在k8s中相当于给node设置了一个锁，容忍相当于这个锁的钥匙，调度到这个节点的Pod需要使用容忍来解开这个锁，但是污点与容忍并不会固定的把某些Pod调度到这个节点。如果Pod第一次调度被调度到了没有污点的节点，他就会被分配到这个节点，容忍的配置就无效了，如果这个Pod正好被调度到有污点的node，他的容忍配置会被解析，验证这个Pod是否可以容忍node的污点。

打上Taint的node，除非pod配置了Toleration这个Taint，否则不会调度到该node上

###### 1.查看node的taint
`kubectl get nodes master -o yaml | jq '.spec.taints'`  #使用jq命令解析输出
`kubectl get nodes master -o go-template --template '{{range .spec.taints}}{{.}} {{end}}'`  #使用go-template模板输出并打印
[map[effect:NoSchedule key:node-role.kubernetes.io/master] map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]
```shell
[root@k8smaster ~]# kubectl describe nodes k8smaster | grep  -iA5  Taints
Taints:             <none>
```

###### 2.node添加taint
`kubectl taint node \<nodename> \<key>=\<value>:\<effect>`
污点配置策略详解

-    NoSchedule:       禁止调度到该节点，已经在该节点上的Pod不受影响
-    NoExecute:        禁止调度到该节点，如果不符合这个污点，会立马被驱逐（如果配置了tolerationSeconds,会在指定时间后驱逐）
-    PreferNoSchedule: 尽量避免将Pod调度到指定的节点上，如果没有更合适的节点，可以部署到该节点

```shell
[root@k8smaster ~]# kubectl taint node k8snode1 ssd=true:NoExecute
node/k8snode1 tainted
[root@k8smaster nodeSelector]# kubectl describe nodes k8snode1 | grep Ta
Taints:             ssd=true:NoExecute
```

###### 3.node删除taint
`kubectl taint node \<nodename> \<key>=\<value>:\<effect>-`
```shell
[root@k8smaster ~]# kubectl taint node k8snode1 ssd=true:NoExecute-
node/k8snode1 untainted
[root@k8smaster ~]# kubectl taint node k8snode1 ssd-
node/k8snode1 untainted
```

###### 4.k8s内置的taint
node.kubernetes.io/not-ready:节点未准备好，相当于节点状态Ready的值为False。
node.kubernetes.io/unreachable:Node Controller访问不到节点，相当于节点状态Ready的值为Unknown。node.kubernetes.io/out-of-disk:磁盘耗尽。
node.kubernetes.io/memory-pressure:节点存在内存压力。
node.kubernetes.io/disk-pressure:节点存在磁盘压力。
node.kubernetes.io/network-unavailable:节点网络不可达。
node.kubernetes.io/unschedulable:节点不可调度。
node.cloudprovider.kubernetes.io/uninitialized:如果Kubelet启动时指定了一个外部的cloudprovider，它将给当前节点添加一个Taint将其标记为不可用。在cloud-controller-manager的一个controller初始化这个节点后，Kubelet将删除这个Taint。

###### 5.查看pod的Toleration
容忍作用与Pod资源配置在tolerations字段，用于容忍配置在node节点的Taint污点，可以让Pod部署在有污点的node节点。

k8s中部署pod会自动添加一些内置的容忍，容忍的默认时间的300秒，表示在node节点出现了内置污点后，会容忍300秒的时间之后会进行迁移
```
    Tolerations:             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
```

查看kube-proxy的Tolerations
```shell
[root@k8smaster ~]# kubectl describe pod kube-proxy-9mspw -n kube-system
Tolerations:     
                 CriticalAddonsOnly
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/pid-pressure:NoSchedule
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
```

###### 6.pod添加Toleration
**容忍匹配污点规则**
当有多个taint和toleration时，除去taint和toleration能匹配的部分，剩下的部分就是对pod的效果
如果剩余的 Taint 中存在 effect=NoSchedule，则调度器不会把该 pod 调度到这一节点上。
如果剩余的 Taint 中没有 NoSchedule 的效果，但是有 PreferNoSchedule 效果，则调度器会尝试不把这个pod指派给这个节点
如果剩余 Taint 的效果有 NoExecute 的，并且这个 pod已经在该节点运行，则会被驱逐；如果没有在该节点运行，也不会再被调度到该节点上
```shell
[root@master kube]# kubectl explain pod.spec.tolerations
  effect            <string>  #污点策略，为空表示可以匹配任一策略，枚举值[NoSchedule, PreferNoSchedule,NoExecute]
  key	              <string>  #容忍度匹配的污点key，可以为空，当为空时operator必须为Exists
  operator	        <string>  #运算符，枚举值[Equal,Exists],默认为Equal
  value   	        <string>  #容忍度匹配的污点值，如果运算符为Exists必须为空
  tolerationSeconds	<integer> #容忍的时间(单位为秒)，表示这个容忍只容忍一定的时间，之后会进行迁移。当有这一项时,effect必须为NoExecute，默认不用设置,0或负值表示立即驱逐pod
```

* 方式一:完全匹配

```shell
此时taint和toleration的k，v，effect要完全相等，任意一个不匹配就不能容忍
tolerations:
- key: "taintKey"      #污点的key名称
  operator: "Equal"    #匹配类型，Equal表示匹配污点的所有值
  value: "taintValue"  #污点key的值
  effect: "NoSchedule" #污点类型
```

* 方式二:不完全匹配

```shell
注意不完全匹配，设置可以是一个可以是俩个，由自己定义比如的tolerations只配置了key和effect,则只匹配key，effect
tolerations:
- key: "taintKey"      #污点的key值
  operator: "Exists"   #匹配类型，只要符合污点设置的值即可，配置哪些就表示只匹配哪些
  effect: "NoSchedule" #污点的类型
```

* 方式三:大范围匹配（不推荐key为内置Taint）

```shell
此时只匹配key
tolerations:
- key: "taintKey"      #污点的key值
  operator: "Exists"   #匹配类型，只要符合污点设置的值即可，配置那些就表示只匹配那些
只匹配effect
tolerations:
- effect: NoExecute
  operator: Exists
```

* 方式四:匹配所有（不推荐）

```shell
表示匹配所有污点，不管什么污点都匹配。在k8s中的daemonsets资源默认情况下是容忍所有污点的
tolerations:
- operator: "Exists"  #匹配所有，没有任何条件
```

方式四是Daemonset资源默认容忍，kube-proxy的tolerations 也包括 operator: Exists
```shell
kubectl  get daemonsets.apps -n kube-system cilium -oyaml | grep tolerations -A 10
      tolerations:
      - operator: Exists
```

如果在容忍中添加tolerationSeconds参数，表示这个容忍只容忍一定的时间，之后会进行迁移。如果设置为0或者负值系统将立即驱赶当前Pod。(单位为秒)
当有tolerationSeconds时，effect必须为"NoExecute"
```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 120  #容忍时间
```


给k8snode1打上污点ssd=true:NoSchedule
```shell
[root@k8smaster ~]# kubectl taint node k8snode1 ssd=true:NoSchedule
node/k8snode1 tainted
[root@k8smaster ~]# kubectl describe nodes k8snode1 | grep Ta
Taints:             ssd=true:NoSchedule
```
给k8snode1打上labels ssh=true
```shell
[root@k8smaster ~]# kubectl label node k8snode1 ssh=true
node/node1 labeled
[root@k8smaster ~]# kubectl get nodes k8snode1 --show-labels
NAME       STATUS   ROLES    AGE   VERSION   LABELS
k8snode1   Ready    <none>   21d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8snode1,kubernetes.io/os=linux,ssh=true
```
创建Deployment，容忍污点key: "ssd"的所有污点，并且指定运行在有ssh: "true" 标签的node上
```yaml
apiVersion: apps/v1    
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      tolerations:   #容忍配置
      - key: "ssd"
        operator: "Exists"
      containers:
      - name: nginx
        image: nginx:1.8
      nodeSelector:  #指定调度的节点
        ssh: "true
```

查看pod运行在携带key: "ssd"污点的k8snode1上
```shell
[root@k8smaster tolerations]# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE        NOMINATED NODE   READINESS GATES
nginx-7cc999dbd-xgw4l               1/1     Running   0          5m30s   10.244.1.61   k8snode1    <none> 
"
```

设置只能容忍120s，AGE超过2m的就会被Terminating，重新进行调度，因为我的环境只有一个节点符合要求所有就会在node1上一直kill超时的pod，再重新调度到node1上start pod，当调度到没有taint的node上超时也不会被kill掉

```shell
[root@master kube]# kubectl get po -owide
NAME                           READY   STATUS              RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES
taint-deploy-f4648d77f-6kqm2   1/1     Terminating         0          2m26s   10.244.0.8    node1   <none>           <none>
taint-deploy-f4648d77f-fzc9s   0/1     ContainerCreating   0          2s      <none>        node1   <none>           <none>
taint-deploy-f4648d77f-m8s6l   1/1     Running             0          81s     10.244.0.10   node1   <none>           <none>
taint-deploy-f4648d77f-nxz2v   1/1     Running             0          26s     10.244.0.11   node1   <none>           <none>
taint-deploy-f4648d77f-qhr8c   1/1     Terminating         0          2m2s    10.244.0.9    node1   <none>           <none>
```

##### 三、 Affinity

调度器在使用的时候，经过了 predicates 和 priorities 两个阶段，
但是在实际的生产环境中，往往我们需要根据自己的一些实际需求来控制 Pod 的调度，
亲和力作用于俩方面，分别是Pod与node之间的关系，Pod与Pod之间的关系。

-    NodeAffinity:节点亲和力/反亲和力
-    PodAffinity:Pod亲和力
-    PodAntiAffinity:Pod反亲和力


###### 1.nodeAffinity
节点亲和性概念上类似于 nodeSelector， 它使你可以根据节点上的标签来约束 Pod 可以调度到哪些节点上。 节点亲和性有两种：
   
-   1、preferredDuringSchedulingIgnoredDuringExecution软策略,调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod
-   2、requiredDuringSchedulingIgnoredDuringExecution硬策略,调度器只有在规则被满足的时候才能执行调度。此功能类似于 nodeSelector， 但其语法表达能力更强

当同时指定nodeSelector和nodeAffinity时，必须同时满足才能正常调度pod

```shell
$ kubectl explain pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution    #硬亲和[nodeSelectorTerms]
    nodeSelectorTerms                               #节点选择器，nodeSelectorTerms下面有多个选项的话，满足任何一个条件即可

  preferredDuringSchedulingIgnoredDuringExecution   #软亲和[weight,preference]
  - weight                                          #权重1-100，和preference关联
    preference                                      #节点选择器项，和weight关联

                                                    #以下共用，匹配方式2种[matchExpressions/matchFields]
      matchExpressions	<[]Object>                  #匹配node的label表达式，如果matchExpressions有多个表达式的话，则必须同时满足这些条件才能正常调度 Pod
      matchFields	<[]Objec                          #匹配字段，字段包括node的名称
      - key	<string> -required-                     #匹配label的key
        operator	<string> -required-               #匹配label的方式，以下6种[DoesNotExist/Exists/Gt/In/Lt/NotIn]
          "DoesNotExist"                            #节点不存在label的key为指定的值即可,不能配置values字段
          "Exists"                                  #节点存在label的key为指定的值即可,不能配置values字段
          "Gt"                                      #label 的值大于某个值,大于value指定的值
          "In"                                      #label 的值在某个列表中,相当于key = value的形式
          "Lt"                                      #label 的值小于某个值,小于value指定的值
          "NotIn"                                   #label 的值不在某个列表中,相当于key != value的形式
        values	<[]string>                          #匹配label的value，可以为多个。如果operator是In或NotIn必须非空；如果是Exists或DoesNotExist必须为空，如果是Gt或Lt必须为一个整数
        - ""
        - ""   
```

- 例子1


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodeaffinity-pod
spec:
  containers:
  - image: busybox:latest
    name: nodeselector-containers
    command: [ "/bin/sh", "-c", "tail -f /etc/passwd" ]
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: ssd
            operator: In
            values:
            - "true"
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
```        
```shell
[root@master ~]# kubectl get po -owide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
nodeaffinity-pod   1/1     Running   0          49s   10.244.2.34   node2   <none>           <none>
[root@master ~]# kubectl get no --show-labels
NAME     STATUS   ROLES           AGE    VERSION   LABELS
node2    Ready    <none>          121d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gpu=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux
```
可以看到node2只有gpu=true的label也调度成功了，当有多个matchExpressions时满足任一一个即可调度。现在将配置文件更改为如下，删除pod再重新创建
```yaml
        nodeSelectorTerms:
        - matchExpressions:
          - key: ssd
            operator: In
            values:
            - "true"
          - key: gpu
            operator: In
            values:
            - "true"
```
```shell
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  28s   default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/master: }, 3 node(s) didn't match Pod's node affinity/selector. preemptio
n: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.[root@master kube]# kubectl get po -owide -w
NAME               READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
nodeaffinity-pod   0/1     Pending   0          37s   <none>   <none>   <none>           <none>
```
pod一直处于pending状态，查看事件，调度失败，给node再添加一个label ssd=true立即调度成功。一个matchExpressions下有多个表达式时需要都满足才能调度
```shell
[root@master ~]# kubectl label no node2 ssd=true
node/node2 labeled
[root@master ~]# kubectl get po -owide -w
NAME               READY   STATUS              RESTARTS   AGE     IP       NODE    NOMINATED NODE   READINESS GATES
nodeaffinity-pod   0/1     ContainerCreating   0          3m49s   <none>   node2   <none>           <none>
```

- 例子2


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-affinity
  labels:
    app: node-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-affinity
  template:
    metadata:
      labels:
        app: node-affinity
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
          name: nginxweb
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
            nodeSelectorTerms:                    #节点选择器配置，只能配置一个
            - matchExpressions:                   #匹配条件设置，可以配置多个，如果是多个他们是或的关系，所有条件都可以被匹配
              - key: kubernetes.io/hostname       #node的labels，匹配的node的key设置,可以配置多个，如果配置多个key他们的关系为and，即需要满足所有的条件才会被匹配
                operator: NotIn                   #匹配方式，有多种
                values:                           #value值，可以写多个
                - "k8snode2"
          preferredDuringSchedulingIgnoredDuringExecution:  # 软策略
          - weight: 1                             #软亲和力的权重，权重越高优先级越大，范围1-100
            preference:                           #软亲和力配置项，和weight同级
              matchExpressions:                   #匹配条件设置
              - key: cm                           #匹配的node的key设置,与硬亲和力一致
                operator: In
                values:
                - "test"
```
这个pod不能运行在k8snode2这个节点上，尽量运行在携带cm=test的labels的node上，有Taint的node不能调度
查看node的labels，k8snode1携带了cm=test的labels，kubernetes.io/hostname的labels每个node默认携带
```shell
[root@k8smaster affinity]# kubectl get nodes  --show-labels
NAME        STATUS   ROLES    AGE   VERSION   LABELS
k8smaster   Ready    master   21d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kube
rnetes.io/hostname=k8smaster,kubernetes.io/os=linux,node-role.kubernetes.io/master=
k8snode1    Ready    <none>   21d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,cm=test,kubernetes.io/arch=am
d64,kubernetes.io/hostname=k8snode1,kubernetes.io/os=linux
k8snode2    Ready    <none>   21d   v1.15.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kube
rnetes.io/hostname=k8snode2,kubernetes.io/os=linux
```
k8snode1携带了cm=test的labels，可以看到全部调度到了k8snode1节点上
```shell
[root@k8smaster affinity]# kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
node-affinity-84d4dc5cc5-29lzj   1/1     Running   0          9s    10.244.1.69   k8snode1    <none>           <none>
node-affinity-84d4dc5cc5-7jdl2   1/1     Running   0          9s    10.244.1.71   k8snode1    <none>           <none>
node-affinity-84d4dc5cc5-ktbqg   1/1     Running   0          9s    10.244.1.70   k8snode1    <none>           <none>
```

###### 2. podAffinity
Pod 亲和性（podAffinity）主要解决 Pod 可以和哪些 Pod 部署在同一个拓扑域中的问题（其中拓扑域用主机标签实现，可以是单个主机，也可以是多个主机组成的 cluster、zone 等等），而 Pod 反亲和性主要是解决 Pod 不能和哪些 Pod 部署在同一个拓扑域中的问题，它们都是处理的 Pod 与 Pod 之间的关系，比如一个 Pod 在一个节点上了，那么我这个也得在这个节点，或者你这个 Pod 在节点上了，那么我就不想和你待在同一个节点上。

注意: 
topologyKey指定label的key必须有节点存在，如果集群内没有节点存在这个label的key，则会一直调度失败
当调度成功后，即使节点和topologyKey相匹配label被删除也不影响已运行的节点
当调度成功后，即使亲和的pod被删除，也不影响当前pod的运行
```yaml
[root@master ~]# kubectl explain pod.spec.affinity.podAffinity
  preferredDuringSchedulingIgnoredDuringExecution         #软亲和
    weight	<integer> -required                           #权重1-100，配合podAffinityTerm使用
    podAffinityTerm	<Object> -required                    #配合weight使用
      namespaces	<[]string>                              #匹配哪个namespace下的pod标签，为空表示匹配当前pod的命名空间
      namespaceSelector	<Object>                          #选择多个namespaces，为空表示当前pod的命名空间，{}表示匹配所有的命名空间
      topologyKey	<string> -required                      #必须项不能为空，亲和或者反亲和的域范围，匹配的拓扑域的key，也就是节点上label的key，key和value相同的为同一个域，可以用于标注不同的机房和地区，如:kubernetes.io/hostname,zone
      labelSelector	<Object>                              #pod标签选择器,有2个选择方式[matchExpressions,matchLabels]
        matchLabels                                   #匹配单个label
          key: value
        matchExpressions                              #匹配表达式
        - key	<string> -required-                     #匹配label的key
          operator	<string> -required-               #匹配label的方式
            "DoesNotExist"                            #节点不存在label的key为指定的值即可,不能配置values字段
            "Exists"                                  #节点存在label的key为指定的值即可,不能配置values字段
            "In"                                      #label 的值在某个列表中,相当于key = value的形式
            "NotIn"                                   #label 的值不在某个列表中,相当于key != value的形式
          values	<[]string>                          #匹配label的value，可以为多个。如果operator是In或NotIn必须非空；如果是Exists或DoesNotExist必须为空
          - ""
            ""
  requiredDuringSchedulingIgnoredDuringExecution          #硬亲和，与软亲和只能选择其中一种，硬亲和如果找不到匹配的kv会一直处于Pending状态
    namespaces	<[]string>                                #匹配哪个namespace下的pod标签，为空表示匹配当前pod的命名空间
    namespaceSelector	<Object>                            #选择多个namespaces，为空表示当前pod的命名空间，{}表示匹配所有的命名空间
    topologyKey	<string> -required                        #必须项不能为空，亲和或者反亲和的域范围，匹配的拓扑域的key，也就是节点上label的key，key和value相同的为同一个域，可以用于标注不同的机房和地区，如:kubernetes.io/hostname,zone
    labelSelector	<Object>                                #pod标签选择器,有2个选择方式[matchExpressions,matchLabels]
      matchLabels                                   #匹配单个label
        key: value
      matchExpressions                              #匹配表达式
      - key	<string> -required-                     #匹配label的key
        operator	<string> -required-               #匹配label的方式
          "DoesNotExist"                            #节点不存在label的key为指定的值即可,不能配置values字段
          "Exists"                                  #节点存在label的key为指定的值即可,不能配置values字段
          "In"                                      #label 的值在某个列表中,相当于key = value的形式
          "NotIn"                                   #label 的值不在某个列表中,相当于key != value的形式
        values	<[]string>                          #匹配label的value，可以为多个。如果operator是In或NotIn必须非空；如果是Exists或DoesNotExist必须为空
        - ""
          ""
```

- 示例1


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: podaffinity-required-pod
spec:
  containers:
  - name: nginx-containers
    image: nginx:latest
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - { key: app, operator: In, values: ["proxy","web"] }
        topologyKey: zone
```

将pod调度到有app=web或app=proxy的pod运行的节点，且节点有zone为key的label
查看有没有zone为key的label的node
```shell
[root@master kube]# kubectl get no --show-labels
node1    Ready    <none>          132d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,ssh=true,zone=beijing
```
查看有app=web或app=proxy的pod运行的节点是哪个
```shell
[root@master kube]# kubectl get po -owide -l "app in (proxy,web)"
NAME            READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
app-proxy-pod   1/1     Running   0          9m21s   10.244.1.5   node1   <none>           <none>
```
创建pod，查看是否运行在node1节点上
```shell
[root@master ~]# kubectl apply -f pod-proxy.yml 
[root@master kube]# kubectl get po -owide
NAME                       READY   STATUS    RESTARTS       AGE   IP            NODE    NOMINATED NODE   READINESS GATES
app-proxy-pod              1/1     Running   0              11m   10.244.1.5    node1   <none>           <none>
podaffinity-required-pod   1/1     Running   0              13m   10.244.1.6    node1   <none>           <none>
```
此时，如果删掉了携带app=proxy的pod,已调度的pod不受影响
```shell
[root@k8smaster affinity]# kubectl delete pods app-proxy-pod
pod "test-busybox" deleted
```
如果再删掉已调度的pod,重新调度,因为使用的是硬亲和，pod就会一直处于pending状态,这是因为现在没有一个节点上面拥有 app=proxy这个标签的 Pod，
```shell
[root@k8smaster affinity]# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
podaffinity-required-pod    0/1     Pending   0          109s
```
再将携带app=proxy的pod启起来，pod-affinity就会运行在同一节点上
```shell
[root@master kube]# kubectl get po -owide
NAME                       READY   STATUS    RESTARTS       AGE   IP            NODE    NOMINATED NODE   READINESS GATES
app-proxy-pod              1/1     Running   0              11m   10.244.1.5    node1   <none>           <none>
podaffinity-required-pod   1/1     Running   0              13m   10.244.1.6    node1   <none>           <none>
```
如果将node1 zone为key的label删除，已调度的pod不受影响
如果删除已调度的pod，再重新创建，pod也会一直处于pending状态

- 例子2


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podaffinity-perferred-pod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - { key: app, operator: In, values: ["cache"] }
              topologyKey: zone
          - weight: 20
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - { key: app, operator: In, values: ["db"] }
              topologyKey: zone
      containers:
      - name: myapp
        image: busybox:latest
        command: ["/bin/sh", "-c", "tail -f /etc/passwd" ]
```
创建这个deployment，尽量满足以key为zone的node，且节点上运行有app=cache或app=db的pod
查看node2节点有zone的label，node1没有
```shell
[root@master kube]# kubectl get no --show-labels
NAME     STATUS   ROLES           AGE    VERSION   LABELS
node1    Ready    <none>          132d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,ssh=true
node2    Ready    <none>          124d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gpu=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,ssd=true,zone=xian
```
查看有没有app=cache或app=db的pod在运行，没有这样的pod在运行
```shell
[root@master kube]# kubectl get po -l "app in (cache,db)"
No resources found in default namespace.
```
查看pod还是running了，说明在pod软亲和策略下，即使有1个条件不满足或者条件都不满足，都可以被调度
```shell
[root@master kube]# kubectl get po  -owide
NAME                                        READY   STATUS    RESTARTS        AGE    IP            NODE    NOMINATED NODE   READINESS GATES
podaffinity-perferred-pod-698b9fdcf-crhgk   1/1     Running   0               3m5s   10.244.2.38   node2   <none>           <none>
podaffinity-perferred-pod-698b9fdcf-q5kgw   1/1     Running   0               3m5s   10.244.1.7    node1   <none>           <none>
podaffinity-perferred-pod-698b9fdcf-qgxjj   1/1     Running   0               3m5s   10.244.2.39   node2   <none>           <none>
```
查看pod的调度过程，虽然没有满足条件的node，但是还是将pod分配给了node2
```shell
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  4m19s  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/master: }, 2 node(s) had untolerated taint {node.kubernetes.io/unreachable: }. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.  
  Normal   Scheduled         4m19s  default-scheduler  Successfully assigned default/podaffinity-perferred-pod-698b9fdcf-crhgk to node2
  Normal   Pulling           4m18s  kubelet            Pulling image "busybox:latest"
  Normal   Pulled            3m31s  kubelet            Successfully pulled image "busybox:latest" in 47.374819602s
  Normal   Created           3m31s  kubelet            Created container myapp
  Normal   Started           3m31s  kubelet            Started container myapp
```

###### 3. Pod 反亲和性（podAntiAffinity）
为了保证分布式，让deployment的pod运行在不同的节点上，反亲和pod本身的label
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podantiaffinity-perferred-pod
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      name: myapp
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - { key: app, operator: In, values: ["myapp"] }
            topologyKey: zone
      containers:
      - name: myapp
        image: busybox:latest
        command: ["/bin/sh", "-c", "tail -f /etc/passwd" ]
```

```shell
[root@master kube]# kubectl get no --show-labels
NAME     STATUS   ROLES           AGE    VERSION   LABELS
master   Ready    control-plane   133d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    <none>          133d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,ssh=true,zone=beijing
node2    Ready    <none>          125d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gpu=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,ssd=true,zone=xian
```

node1和node2分别只运行了一个pod。master节点没有key为zone的label，pod的反亲和性没有在master节点上生效，运行了2个相同的pod，如果有key为zone的label应该只运行一个
```shell
[root@master kube]# kubectl get po -owide
NAME                                            READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
podantiaffinity-perferred-pod-94fcf7f4f-74fxc   1/1     Running   0          51s   10.244.0.20   master   <none>           <none>
podantiaffinity-perferred-pod-94fcf7f4f-7r7ss   1/1     Running   0          51s   10.244.2.43   node2    <none>           <none>
podantiaffinity-perferred-pod-94fcf7f4f-gzcdd   1/1     Running   0          51s   10.244.0.21   master   <none>           <none>
podantiaffinity-perferred-pod-94fcf7f4f-h27pk   1/1     Running   0          51s   10.244.1.16   node1    <none>           <none>
```
给master节点添加key为zone的label
```shell
[root@master kube]# kubectl label no master zone=hebei
node/master labeled
[root@master kube]# kubectl get no --show-labels
NAME     STATUS   ROLES           AGE    VERSION   LABELS
master   Ready    control-plane   133d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=,zone=hebei
node1    Ready    <none>          133d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,ssh=true,zone=beijing
node2    Ready    <none>          125d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gpu=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,ssd=true,zone=xian
```
删除deploy重新创建，再次查看pod的调度
```shell
[root@master kube]# kubectl get po -owide 
NAME                                            READY   STATUS    RESTARTS   AGE    IP            NODE     NOMINATED NODE   READINESS GATES
podantiaffinity-perferred-pod-94fcf7f4f-fzh75   1/1     Running   0          8m3s   10.244.1.17   node1    <none>           <none>
podantiaffinity-perferred-pod-94fcf7f4f-k9lf8   0/1     Pending   0          8m3s   <none>        <none>   <none>           <none>
podantiaffinity-perferred-pod-94fcf7f4f-lh6pq   1/1     Running   0          8m3s   10.244.0.22   master   <none>           <none>
podantiaffinity-perferred-pod-94fcf7f4f-pkjqk   1/1     Running   0          8m3s   10.244.2.44   node2    <none>           <none>
```
有一个pod处于pending状态，查看描述，调度失败，因为每个节点都已经有了一个app=myapp的label的pod，因为pod的反亲和性，无法进行调度
```shell
[root@master kube]# kubectl describe po podantiaffinity-perferred-pod-94fcf7f4f-k9lf8
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  8m24s               default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/master: }, 2 node(s) had untolerated taint {node.kubernetes
.io/unreachable: }. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.  Warning  FailedScheduling  3m (x3 over 8m24s)  default-scheduler  0/3 nodes are available: 3 node(s) didn't match pod anti-affinity rules. preemption: 0/3 nodes are available: 3 No preemption victims found f
or incoming pod.
```
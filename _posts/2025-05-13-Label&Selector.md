---
title: Label&Selector
date: 2025-05-13 22:34:00 +0800
categories: [k8s,Label,Selector]
tags: [k8s,Label,Selector]
---

Label：对k8s中各种资源进行分类、分组，添加一个具有特别属性的一个标签。

Selector：通过一个过滤的语法进行查找到对应标签的资源
一、Label & Selector
当Kubernetes对系统的任何API对象如Pod和节点进行“分组”时，会对其添加Label（key=value形式的“键-值对”）用以精准地选择对应的API对象。而Selector（标签选择器）则是针对匹配对象的查询方法。注：键-值对就是key-value pair。

例如，常用的标签tier可用于区分容器的属性，如frontend、backend；或者一个release_track用于区分容器的环境，如canary、production等。
1.1 Label的介绍
label可以给k8s中大多数资源进行标签的定义，主要作用为用于指定对用户有意义且相关的对象的标识属性，但并不对资源工作参数任何影响。

标签是键值对。有效的标签键有两个段：可选的前缀和名称，用斜杠（/）分隔。 名称段是必需的，必须小于等于 63 个字符，以字母数字字符（[a-z0-9A-Z]）开头和结尾， 带有破折号（-），下划线（_），点（ .）和之间的字母数字。
kubernetes.io/ 和 k8s.io/ 前缀是为 Kubernetes 核心组件保留的。

##### 一、标签的crud
大多数kubernetes中的资源都是可以进行打标签的，命令是kubectl label。
可以给指定资源添加多个标签
###### 1.添加labels
kubectl labels <resourcesclass> <resources> <key>=<value>
>给node添加label
kubectl labels node matser zone=1
>给指定pod添加label
kubectl label deployments.apps nginx-1 app=nginx 
>给某一类型的pod添加label
kubectl label deployments.apps app=test --all
yaml文件中添加labels
```yaml
metadata:
  labels:
    app: my-app
    env: production
```

###### 2.删除labels
kubecrl labels <resourcesclass> <resources> <key>-
>删除node的label
kubectl label node master zone-
>删除指定资源的label
kubectl label deployments nginx-1 app-
>删除某一类型所有资源的label
kubectl label deployments app- --all

###### 3.修改标签
kubecrl labels <resourcesclass> <resources> <key>=<value> --overwrite
kubectl label deployments nginx-1 app=nginx-1 --overwrite

###### 4.查看资源标签
kubecrl labels <resourcesclass> --show-labels
kubectl get deployments --show-labels
kubectl get node --show-labels
kubectl get node master --show-labels

##### 二、yaml中匹配labels
资源支持基于集合的需求。例如 Job、 Deployment、 ReplicaSet 和 DaemonSet

    matchLabels - 卷必须包含带有此值的标签
    matchExpressions - 通过设定键（key）、值列表和操作符（operator） 来构造的需求。合法的操作符有 In、NotIn、Exists 和 DoesNotExist。

来自 matchLabels 和 matchExpressions 的所有需求都按逻辑与的方式组合在一起。 这些需求都必须被满足才被视为匹配

selector:
  matchLabels:
    component: redis
  matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: Exists }

matchLabels 是由 {key,value} 对组成的映射。 matchLabels 映射中的单个 {key,value} 等同于 matchExpressions 的元素， 其 key 字段为 "key"，operator 为 "In"，而 values 数组仅包含 "value"。 matchExpressions 是 Pod 选择算符需求的列表。 有效的运算符包括 In、NotIn、Exists 和 DoesNotExist。 在 In 和 NotIn 的情况下，设置的值必须是非空的。 

##### 三、利用label查找资源
kubectl get可以打印资源列表，并且可以使用以下参数利用label进行资源筛选
    -l: 匹配kv对
    -L: 只匹配k
匹配规则,只针对-l
    ==,=:等于
    !=:不等于
    in:包含
    notin:不包含
###### 1. -L: 只匹配key 
>注意:
命令中的key区分大小写，输出的key列统一为大写。输出所有指定类型的资源，即使没有匹配到指定的key。并会输出指定的key列和对应的value，即使value为空也会显示为空

匹配node，匹配一个key。HOSTNAME为指定的key，下面为对应的value
[root@master yaml]# kubectl get node -Lkubernetes.io/hostname
NAME     STATUS   ROLES           AGE   VERSION   HOSTNAME
master   Ready    control-plane   62d   v1.24.1   master
node1    Ready    <none>          62d   v1.24.1   node1
node2    Ready    control-plane   61d   v1.24.1   node2

匹配pod，匹配多个key。app和controller-revision-hash为key
[root@master yaml]# kubectl get pods -Lapp -Lcontroller-revision-hash -A 
NAMESPACE      NAME                             READY   STATUS    RESTARTS        AGE    APP       CONTROLLER-REVISION-HASH
dev1           webapp                           1/1     Running   0               58m    webapp    
kube-flannel   kube-flannel-ds-d86gj            1/1     Running   12 (7h6m ago)   60d    flannel   5d454f6775
kube-flannel   kube-flannel-ds-fglgm            1/1     Running   13 (7h6m ago)   60d    flannel   5d454f6775
kube-flannel   kube-flannel-ds-w5tl5            1/1     Running   14 (7h6m ago)   60d    flannel   5d454f6775

 
###### 2. -l: 匹配kv对(只输出匹配到kv的资源)
>匹配一个kv对
kubectl get po -l app=nginx
kubectl get po -l app!=nginx

>匹配多个kv对，kv对是and的关系
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production),tier in (frontend)'

>匹配一个k多个v
kubectl get pods -l 'environment in (production, qa)'
kubectl get pods -l 'environment notin (production, qa)'

>匹配多个k，一个v
kubectl get pods -l 'environment,environment notin (frontend)'

查看labels
[root@master ~]# kubectl get node --show-labels 
NAME     STATUS   ROLES           AGE    VERSION   LABELS
master   Ready    control-plane   134d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
node1    Ready    <none>          134d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux,ssh=true,zone=beijing,gpu=true
node2    Ready    <none>          126d   v1.24.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gpu=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,ssd=true,zone=xian

匹配key为zone，value为xian的label的node
[root@master ~]# kubectl get no -lzone=xian
NAME    STATUS   ROLES    AGE    VERSION
node2   Ready    <none>   126d   v1.24.1

匹配key不为zone,vaule不为xian的node
[root@master ~]# kubectl get no -l "zone notin (xian)"
NAME     STATUS   ROLES           AGE    VERSION
master   Ready    control-plane   134d   v1.24.1
node1    Ready    <none>          134d   v1.24.1
master节点没有zone的label也被匹配到了

匹配zone不等于xian，ssh等于xian的node
[root@master ~]# kubectl get no -l "zone notin (xian),ssh in (true)"
NAME    STATUS   ROLES    AGE    VERSION
node1   Ready    <none>   134d   v1.24.1

匹配gpu和ssh都为true的node
[root@master ~]# kubectl get no -l "gpu,ssh in (true)"
NAME    STATUS   ROLES    AGE    VERSION
node1   Ready    <none>   134d   v1.24.1


###### 3. 即-L:匹配key，也-l:匹配kv
当有l时，就只会输出匹配到l指定的kv对的资源，l指定的kv对不会在输出中打印，L指定的key会作为一列输出对应的value
[root@master ~]# kubectl get no -lzone=xian -Lgpu
NAME    STATUS   ROLES    AGE    VERSION   GPU
node2   Ready    <none>   126d   v1.24.1   
[root@master ~]# kubectl get pods -lapp=webapp -Lcontroller-revision-hash -A 
NAMESPACE   NAME     READY   STATUS    RESTARTS   AGE   CONTROLLER-REVISION-HASH
dev1        webapp   1/1     Running   0          61m  
---
title: StatefulSet
date: 2025-05-11 22:34:00 +0800
categories: [k8s,StatefulSet]
tags: [k8s,StatefulSet]
---

StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。
1、拓扑状态。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。
2、存储状态。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。
所以，StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。


正是 Headless Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。
可以看到，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。
Headless一般的格式为：statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local
```yaml
Headless Service示例文件：
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None #主要是这里为None
  selector:
    app: nginx
```
可以看到，所谓的 Headless Service，其实仍是一个标准 Service 的 YAML 文件。只不过，它的 clusterIP 字段的值是：None，即：这个 Service，没有一个 VIP 作为“头”。这也就是 Headless 的含义。所以，这个 Service 被创建后并不会被分配一个 VIP，而是会以 DNS 记录的方式暴露出它所代理的 Pod。
这个sevice所代理的pod依然是Label selector机制选择出来的，即所有app=nginx标签的pod，都会被这个service代理起来
你按照这样的方式创建了一个 Headless Service 之后，它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录，如下所示：
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
这个dns正是k8s为pod分配的唯一的可解析身份

```yaml
apiVersion: apps/v1         #必须，api版本
kind: StatefulSet           #必须，资源类型
metadata:                   #必须，元数据信息 
  name: web                 #必须，名称
spec:                       #必须，定义控制器信息
  serviceName: "nginx"      #必须，管理这个StatefulSet的service名称(无头服务)
  replicas: 2               #必须，副本数
  revisionHistoryLimit: 10  #可选，历史版本保存的最大数量，默认10
  updateStrategy:           #可选，更新策略配置
    rollingUpdate:          #RollingUpdate更新策略才需要
      partition: 0          #表示在滚动更新时保留的pod编号，为0即表示保留，小于0的不进行更新,不更新小于 N 的pod
    type: RollingUpdate     #类型默认为RollingUpdate滚动更新，其他参数Ondelete在更新pod后只有删除pod才会触发更新
  selector:                 #必须，选择器配置，选择pod
    matchLabels:            #必须，标签选择器
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```
##### 0、StatefulSet和Headless Service
    对于包含 N 个 副本的 StatefulSet，当部署 Pod 时，它们是依次创建的，顺序为 0..N-1。 
    当缩容、删除、更新 Pod 时，它们是逆序的，顺序为 N-1..0。
    在将操作应用到 Pod 之前，它前面的所有 Pod 必须是 Running 和 Ready 状态。否则会暂停新的pod的操作
```shell
[root@k8smaster statefulset]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   25h
nginx        ClusterIP   None         <none>        80/TCP    95m
[root@k8smaster statefulset]# kubectl get statefulset web
NAME   READY   AGE
web    2/2     97m
[root@k8smaster statefulset]# kubectl get pod -o wide
NAME       READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
web-0      1/1     Running   0          100m   10.244.1.20   k8snode1   <none>           <none>
web-1      1/1     Running   0          100m   10.244.2.25   k8snode2   <none>           <none>
```
创建一个一次性的pod，--rm意味pod退出就会被删除
```shell
[root@k8smaster statefulset]# kubectl run -it --image centos:7 dns-test --restart=Never --rm /bin/bash
```
使用nslookup解析就可以解析到对应pod的ip地址
```shell
[root@dns-test yum.repos.d]# nslookup web-0.nginx.default.svc.cluster.local
Server:		10.1.0.10
Address:	10.1.0.10#53

Name:	web-0.nginx.default.svc.cluster.local
Address: 10.244.1.20

[root@dns-test yum.repos.d]# nslookup web-1.nginx.default.svc.cluster.local
Server:		10.1.0.10
Address:	10.1.0.10#53

Name:	web-1.nginx.default.svc.cluster.local
Address: 10.244.2.25
```
##### 1、StatefulSet的扩缩
将app=nginx的2个pod删掉
```shell
[root@k8smaster statefulset]# kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```
查看新创建的两个pod，k8s为它们分配了和原来一样的网络身份”web-0.nginx"和”web-1.nginx“
```shell
[root@k8smaster statefulset]# kubectl get pod -w -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          65s
web-1   1/1     Running   0          62s
```
当只删掉web-0的pod
```shell
[root@k8smaster statefulset]# kubectl delete pod web-0
pod "web-0" deleted
[root@k8smaster statefulset]# kubectl get pod -w -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          3s
web-1   1/1     Running   0          3m15s
查看新创建的pod，k8s为它们分配了和原来一样的网络身份”web-0.nginx”
```

##### 2、StatefulSet的存储
创建一个pvc
```yaml
[root@k8smaster volume]# cat pvc.yml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```
创建pv
```yaml
[root@k8smaster volume]# cat pv.yml 
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-test
  labels:
    name: pv-test
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /test-volume
```
pod配置文件如下：
```yaml
[root@k8smaster volume]# cat sts.yml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web1
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: manual
      resources:
        requests:
          storage: 1Gi
```
创建这个pv
```shell
[root@k8smaster volume]# kubectl create -f pv.yml 
persistentvolume/pv-test created
[root@k8smaster volume]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-test   1Gi        RWO            Recycle          Available           manual                  4s
```
创建sts.yml配置的这个pod
```shell
[root@k8smaster volume]# kubectl create -f sts.yml 
statefulset.apps/web1 created
```
查看pvc和pv的bound关系
```shell
[root@k8smaster volume]# kubectl get pvc
NAME         STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web1-0   Bound     pv-test   1Gi        RWO            manual         34m
www-web1-1   Pending                                       manual         34m
```
查看创建的pod的状态
```shell
[root@k8smaster volume]# kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
web1-0   1/1     Running   0          35m   10.244.1.31   k8snode1   <none>           <none>
web1-1   0/1     Pending   0          35m   <none>        <none>     <none>           <none>
```
exec到web1-0容器内
```shell
[root@k8smaster volume]# kubectl exec -it  web1-0 -- /bin/bash
root@web1-0:/# ls
root@web1-0:/usr/share/nginx/html# echo "test" > index.html
```
查看k8snode1节点下/test-volume目录下的文件
```shell
[root@k8snode1 test-volume]# pwd
/test-volume
[root@k8snode1 test-volume]# cat index.html 
test
```
##### 3、 StatefulSet实现灰度发布
StatefulSet在使用rollingUpdate滚动更新策略时，可以设置partition参数实现简单的灰度发布

#配置如下
```yaml
  updateStrategy: 
    rollingUpdate:  
      partition: 1      #表示在滚动更新时保留的pod编号，为1即表示保留小于1的pod不进行更新
    type: RollingUpdate 
```
#演示
```shell
kubectl set env sts/web test=1 zhang=1 test1=sss #更新
get pod -l app=nginx-1 -w
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          14s
web-1   1/1     Running   0          53s
web-2   1/1     Running   0          57s
web-2   1/1     Terminating   0          80s   #更新web-2
web-2   0/1     Pending       0          0s
web-2   0/1     ContainerCreating   0          0s
web-2   1/1     Running             0          2s
web-1   1/1     Terminating         0          84s #更新web-1
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          0s
web-1   1/1     Running             0          2s
```
由于0小于1所以没有进行更新

使用ondelete更新策略也可以实现简单的灰度发布
将更新策略设置为ondelete后，手动删除指定的pod,只要被手动删除的pod才会更新，没有执行删除操作的不会更新

##### 4、 StatefulSet级联删除与非级联删除
级联删除(默认)删除StatefulSet时同时删除pod，非级联删除删除StatefulSet时不会删除pod，此时的pod会变成孤儿pod，再次删除pod时pod不会被重建。

#非级联删除
```shell
kubectl delete sts web --cascade=false
statefulset.apps "web" deleted
```
--cascade=false  #关闭级联删除
```shell
[root@k8smaster statefulset]# kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
web-0                              1/1     Running   0          6s
web-1                              1/1     Running   0          5s
[root@k8smaster statefulset]# kubectl delete statefulsets.apps web 
statefulset.apps "web" deleted
[root@k8smaster statefulset]# kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
[root@k8smaster statefulset]# kubectl delete statefulsets.apps web --cascade=false
statefulset.apps "web" deleted
[root@k8smaster statefulset]# kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
web-0                              1/1     Running   0          22s
web-1                              1/1     Running   0          20s
再删除pod
[root@k8smaster statefulset]# kubectl delete pods web-0
pod "web-0" deleted
[root@k8smaster statefulset]# kubectl delete pods web-1
pod "web-1" deleted
```

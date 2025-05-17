---
title: Pv、Pvc、StorageClass
date: 2025-05-05 22:41:00 +0800
categories: [k8s,Pv,Pvc,StorageClass]
tags: [k8s,Pv,Pvc,StorageClass]
toc: true
---

**关于namespace**
PersistentVolume 卷的绑定是排他性的。 由于pvc是namespace作用域的对象，使用 "Many" 模式（ROX、RWX）来挂载申领的操作只能在同一namespace内进行。
storageclass不是namespace作用域的对象，pv也不是namespace作用域的对象但是它会和namespace作用域的pvc进行绑定
#### 一、PV
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
    readOnly: false
```
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```
##### 1、pv的访问模式(accessModes)
每个volume只能同时以一种访问模式挂载
ReadWriteOnce(RWO)
    卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式仍然可以在同一节点上运行的多个 Pod 访问该卷。 
ReadOnlyMany(ROX)
    卷可以被多个节点以只读方式挂载。
ReadWriteMany(RWX)
    卷可以被多个节点以读写方式挂载。
ReadWriteOncePod(RWOP)
    特性状态： Kubernetes v1.29 [stable]
    卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用 ReadWriteOncePod 访问模式。
##### 2、pv的回收策略(persistentVolumeReclaimPolicy)
数据卷可以被 Retained（保留）、Recycled（回收）或 Deleted（删除）。
* 保留（Retain）
保留策略 Retain 使得用户可以手动回收资源。当 PersistentVolumeClaim 对象被删除时，PersistentVolume 卷仍然存在，对应的数据卷被视为"已释放（released）"。 由于卷上仍然存在这前一申领人的数据，该卷还不能用于其他申领。 管理员可以通过下面的步骤来手动回收该卷：

    删除 PersistentVolume 对象。与之相关的、位于外部基础设施中的存储资产在 PV 删除之后仍然存在。
    根据情况，手动清除所关联的存储资产上的数据。
    手动删除所关联的存储资产。

如果你希望重用该存储资产，可以基于存储资产的定义创建新的 PersistentVolume 卷对象。
* 删除（Delete）
对于支持 Delete 回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施中移除所关联的存储资产。 动态制备的卷会继承其 StorageClass 中设置的回收策略， 该策略默认为 Delete。管理员需要根据用户的期望来配置 StorageClass； 否则 PV 卷被创建之后必须要被编辑或者修补。 参阅更改 PV 卷的回收策略。
* 回收（Recycle）
警告：
删除数据，回收策略 Recycle 已被废弃。取而代之的建议方案是使用动态制备。
如果下层的卷插件支持，回收策略 Recycle 会在卷上执行一些基本的擦除 （rm -rf /thevolume/*）操作，之后允许该卷用于新的 PVC 申领
修改pv的回收策略。
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

##### 3、卷模式(volumeMode)
特性状态： Kubernetes v1.18 [stable]

针对 PV 持久卷，Kubernetes 支持两种卷模式（volumeModes）：Filesystem（文件系统） 和 Block（块）。 volumeMode 是一个可选的 API 参数。 如果该参数被省略，默认的卷模式是 Filesystem。

volumeMode 属性设置为 Filesystem 的卷会被 Pod 挂载（Mount） 到某个目录。 

volumeMode 设置为 Block，以便将卷作为原始块设备来使用。 这类卷以块设备的方式交给 Pod 使用，其上没有任何文件系统。如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前在设备上创建文件系统。Pod 中运行的应用必须知道如何处理原始块设备。

##### 4、挂载选项(mountOptions)
指定pv被挂载在节点上使用的附加挂载选项，支持以下卷类型
*   azureFile
*   cephfs（于 v1.28 中弃用）
*   cinder（于 v1.18 中弃用）
*   iscsi
*   nfs
*   rbd（于 v1.28 中弃用）
*   vsphereVolume
Kubernetes 不对挂载选项执行合法性检查。如果挂载选项是非法的，挂载就会失败。
##### 5、容量(capacity)
定义 PV 的存储容量
##### 6、类型
存储类型，fc,hostpath,ceph,glusterfs，nfs...
##### 7、存储类(storageClassName)
定义 PV 的存储类别名称
##### 8、节点亲和性(nodeAffinity)
对大多数卷类型而言，不需要设置节点亲和性字段。 只需要为 local pv显式地设置此属性。

每个 PV 卷可以通过设置节点亲和性来定义一些约束，进而限制从哪些节点上可以访问此卷。 使用这些卷的 Pod 只会被调度到节点亲和性规则所选择的节点上执行。 要设置节点亲和性，配置 PV 卷 .spec 中的 nodeAffinity

##### 9、pv的状态
Available
    卷是一个空闲资源，尚未绑定到任何pvc
Bound
    该卷已经绑定到pvc
Released
    所绑定的pvc已被删除，但是关联存储资源尚未被集群回收
Failed
    卷的自动回收操作失败

#### 二、PVC
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```
##### 1、pvc的访问模式(accessModes)
与pv相同，使用与pv相同的约定
##### 2、pvc的卷模式(volumeMode)
与pv相同，使用与pv相同的约定
##### 3、pvc的资源请求(resources)
使用与pv相同的约定
##### 4、pvc的存储类(storageClassName)
与pv相同，使用与pv相同的约定
PVC 申领不必一定要请求某个类。如果 PVC 的 storageClassName 属性值设置为 ""，这一 PVC 申领只能绑定到未设置存储类的 PV 卷
##### 5、选择器(selector)
目前，设置了非空 selector 的 PVC 对象无法让集群为其动态制备 PV 卷。
pvc可以设置selector来进一步过滤pv，只有标签与选择算符相匹配的卷能够绑定到申领上。 选择算符包含两个字段：
    matchLabels - 卷必须包含带有此值的标签
    matchExpressions - 通过设定键（key）、值列表和操作符（operator） 来构造的需求。合法的操作符有 In、NotIn、Exists 和 DoesNotExist。

来自 matchLabels 和 matchExpressions 的所有需求都按逻辑与的方式组合在一起。 这些需求都必须被满足才被视为匹配。

#### 三、pod使用pvc
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

#### 四、storageclass
当没有对应的storageclass，pv和pvc指定的storageclass相同，俩者就会绑定
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```
##### 1、回收策略(reclaimPolicy)

  Delete 删除，默认策略
  Retain 保留
通过storageclass创建并管理的pv会使用storageclass指定的回收策略
##### 2、挂载选项(mountOptions)
有storageclass创建的pv被挂载在节点上使用的附加挂载选项
Kubernetes 不对挂载选项执行合法性检查。如果挂载选项是非法的，挂载就会失败。
##### 3、卷绑定模式(volumeBindingMode)
volumeBindingMode指定了卷绑定和动态制备发生在什么时候

  Immediate 默认模式，创建了pvc就完成了pv的创建和绑定，即使pod没有创建
  WaitForFirstConsumer 延迟pvc和pv的创建和绑定，直到pod的创建。只有Local 插件支持，如果pod使用此策略的storageclass，不能使用节点亲和性会导致pvc pending。可以使用nodeSelecter
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
##### 4、允许卷扩展(allowVolumeExpansion)
pv可配置为可扩展，当storageclass的allowVolumeExpansion设置为true时，允许用户通过编辑对应的pvc来调整卷大小
以下类型支持卷扩展
卷类型	Kubernetes 版本要求
rbd	1.11
Azure File	1.11
Portworx	1.11
FlexVolume	1.13
CSI	1.14 (alpha), 1.16 (beta)
##### 5、选项参数(parameters)
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-nfs
provisioner: example.com/external-nfs
parameters:
  server: nfs-server.example.com
  path: /share
  readOnly: "false"
```
    server：NFS 服务器的主机名或 IP 地址。
    path：NFS 服务器导出的路径。
    readOnly：是否将存储挂载为只读的标志（默认为 false）。
##### 6、默认storageClass
设置默认的storageClass，将annotations注解storageclass.kubernetes.io/is-default-class设置为true
当集群中存在默认的 StorageClass 并且用户创建了一个未指定 storageClassName 的 PersistentVolumeClaim 时， DefaultStorageClass 准入控制器会自动向其中添加指向默认存储类的 storageClassName 字段。
在集群的多个 StorageClass 设置 storageclass.kubernetes.io/is-default-class 注解为 true， 并之后创建了未指定 storageClassName 的 PersistentVolumeClaim， Kubernetes 会使用最新创建的默认 StorageClass。
```yaml
[root@master yaml]# cat nfs-storage.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  namespace: kube-system
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  ## 是否设置为默认的storageclass
provisioner: nfs-client                                   ## 动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
mountOptions:
  - hard                                                  ## 指定为硬挂载方式
  - nfsvers=4                                             ## 指定NFS版本,这个需要根据NFS Server版本号设置
```
```
[root@master yaml]# kubectl get sc
NAME                    PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   nfs-client    Delete          Immediate           false                  18h
```

PV描述的是持久化存储数据卷，这个 API 对象主要定义的是一个持久化存储在宿主机上的目录，比如一个 NFS 的挂载目录。
PVC描述的是pod所希望的持久化存储的属性，比如，volume存储的大小，可读权限等
定义一个NFS类型的PV，
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
```
声明一个1G大小的PVC
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```
用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定。这里要检查的条件，包括两部分：
1、第一个条件，当然是 PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
2、而第二个条件，则是 PV 和 PVC 的 storageClassName 字段必须一样。
PV和PVC是一对一的关系
pod使用上面声明的PVC
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```
pod在volume字段声明自己要使用的PVC的名字，等这个pod被创建之后，kubelet就会把这个PVC所对应的PV，挂载在这个pod容器内的目录上
PVC和PV的设计，其实和“面向对象”的思想完全一致，PVC是持久化存储的“接口”，提供对某种持久化存储的描述；持久化存储的具体实现部分由PV负责完成

在k8s中，存在一个专门处理持久化存储的控制器，Volume Controller
这个Contruller维护着多个控制循环，其中一个循环，扮演的就是撮合 PV 和 PVC 的“红娘”的角色。它的名字叫作 PersistentVolumeController
PersistentVolumeController 会不断地查看当前每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个“单身”的 PVC 进行绑定。这样，Kubernetes 就可以保证用户提交的每一个 PVC，只要有合适的 PV 出现，它就能够很快进入绑定状态
所谓将一个 PV 与 PVC 进行“绑定”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 spec.volumeName 字段上。
```
[root@k8smaster volume]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pv-test   3Gi        RWX            Recycle          Bound    default/www-web1-0   manual                  2d1
9h[root@k8smaster volume]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pv-test   3Gi        RWX            Recycle          Bound    default/www-web1-0   manual                  2d19h
[root@k8smaster volume]# kubectl get pvc
NAME         STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web1-0   Bound     pv-test   1Gi        RWX            manual         2d19h
```
在这个www-web1-0的pvc的yaml配置信息里可以看到volumename的字段就是pv-test
```
[root@k8smaster volume]# kubectl get pvc www-web1-0  -o yaml | grep -i  volumename
  volumeName: pv-test
```
持久化 Volume 的实现，往往依赖于一个远程存储服务，比如：远程文件存储（比如，NFS、GlusterFS）、远程块存储（比如，公有云提供的远程磁盘）等等。
而 Kubernetes 需要做的工作，就是使用这些存储服务，来为容器准备一个持久化的宿主机目录，以供将来进行绑定挂载时使用。而所谓“持久化”，指的是容器在这个目录里写入的文件，都会保存在远程存储中，从而使得这个目录具备了“持久性”。
这个准备“持久化”宿主机目录的过程，我们可以形象地称为“两阶段处理”
1、“第一阶段”（Attach），
由volume controller负责维护，控制循环的名字叫AttachDetachController。作用就是不断的检查每一个pod对应的pv，和这个pod所在宿主机之间的挂载情况，从而决定是否对这个PV进行Attach或Dettach操作。
volume控制器是kube-controller-manager的一部分，所以一定是运行在master节点上。
Kubernetes 提供的可用参数是 nodeName，即宿主机的名字
2、而对于“第二阶段”（Mount）；
第二阶段的mount或者umount操作，必须发生在pod对应的宿主机上，所以它是kubelet组件的一部分，这个控制循环的名字，叫VolumeManagerReconciler，运行起来后是一个独立于kubelet的goroutime
Kubernetes 提供的可用参数是 dir，即 Volume 的宿主机目录

删除一个PV时，k8s需要umount和dettch两个阶段，执行反向操作即可

二、StorageClass
在大规模集群中PVC数量很多，如果手动创建PV，耗时耗力
k8s提供了一套可以自动创建PV的机制，即：Dynamic Provisioning。
人工管理 PV 的方式就叫作 Static Provisioning
Dynamic Provisioning 机制工作的核心，在于一个名叫 StorageClass 的 API 对象。
StorageClass 对象的作用，其实就是创建 PV 的模板。
第一，PV 的属性。比如，存储类型、Volume 的大小等等。
第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。
有了着两个就可以根据用户提交的PVC找到对应的StorageClass，然后调用storageclass声明的存储插件，创建PV
使用ROOK存储服务的话，storageclass使用如下的yml文件定义

apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  #The value of "clusterNamespace" MUST be the same as the one in which your rook cluster exist
  clusterNamespace: rook-ceph

定义了一个block-service的storageclass，存储插件是rook-ceph
创建这个storageclas
$ kubectl create -f sc.yaml
开发者，只需要在PVC指定storageclass名字即可

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: block-service
  resources:
    requests:
      storage: 30Gi


在这个PVC里添加了一个storageclassname的字段:block-service
当我们通过 kubectl create 创建上述 PVC 对象之后，Kubernetes 就会调用ROOK-ceph 的 API，创建出一块Persistent Disk。然后，再使用这个 Persistent Disk 的信息，自动创建出一个对应的 PV 对象。

PVC 描述的，是 Pod 想要使用的持久化存储的属性，比如存储的大小、读写权限等。
PV 描述的，则是一个具体的 Volume 的属性，比如 Volume 的类型、挂载目录、远程存储服务器地址等。
而 StorageClass 的作用，则是充当 PV 的模板。并且，只有同属于一个 StorageClass 的 PV 和 PVC，才可以绑定在一起。
StorageClass 的另一个重要作用，是指定 PV 的 Provisioner（存储插件）。这时候，如果你的存储插件支持 Dynamic Provisioning 的话，Kubernetes 就可以自动为你创建 PV 了。

总结：
用户提交请求创建pod，Kubernetes发现这个pod声明使用了PVC，那就靠PersistentVolumeController帮它找一个PV配对。
没有现成的PV，就去找对应的StorageClass，帮它新创建一个PV，然后和PVC完成绑定。
新创建的PV，还只是一个API 对象，需要经过“两阶段处理”变成宿主机上的“持久化 Volume”才真正有用：
第一阶段由运行在master上的AttachDetachController负责，为这个PV完成 Attach 操作，为宿主机挂载远程磁盘；
第二阶段是运行在每个节点上kubelet组件的内部，把第一步attach的远程磁盘 mount 到宿主机目录。这个控制循环叫VolumeManagerReconciler，运行在独立的Goroutine，不会阻塞kubelet主循环。
完成这两步，PV对应的“持久化 Volume”就准备好了，POD可以正常启动，将“持久化 Volume”挂载在容器内指定的路径。
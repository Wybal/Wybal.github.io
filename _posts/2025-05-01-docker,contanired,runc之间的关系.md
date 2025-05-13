---
title: docker,contanired,runc之间的关系
date: 2025-05-01 22:34:00 +0800
categories: [容器,runc]
tags: [容器,runc]
---


### 1、什么是runc
RunC 是从 Docker 的 libcontainer 中迁移而来的，实现了容器启停、资源隔离等功能。Docker将RunC捐赠给 OCI 作为OCI 容器运行时标准的参考实现。Docker 默认提供了 docker-runc 实现。事实上，通过 containerd 的封装，可以在 Docker Daemon 启动的时候指定 RunC的实现。

OCI(Open Container Initiative)主要有2个标准，容器运行时标准（runtime spec）和 容器镜像标准（image spec）
实现OCI容器运行时标准有:runc,runv,runsc，gvisor

当你运行一个 Docker 容器时，这些是 Docker 实际经历的步骤：
1. 下载镜像
2. 将镜像文件解开为bundle文件，将一个文件系统拆分成多层
3. 从bundle文件运行容器
   
runc遵守OCI容器运行时标准，与内核交互创建并运行容器,runc是最广泛的容器运行时,也称为Low-level容器运行时。
实现了容器的创建，启动，运行，停止，重启...也就是上述步骤的第3步
**怎么使用runc**
```shell
create the bundle
$ mkdir -p /mycontainer/rootfs

# [ab]use Docker to copy a root fs into the bundle
$ docker export $(docker create busybox) | tar -C /mycontainer/rootfs -xvf -

# create the specification, by default sh will be the entrypoint of the container
$ cd /mycontainer
$ runc spec

# launch the container
$ sudo -i
$ cd /mycontainer
$ runc run mycontainerid

# list containers
$ runc list

# stop the container
$ runc kill mycontainerid

# cleanup
$ runc delete mycontainerid
```

### 2、container，和cri-o，dockershim
CRI（容器运行时接口）是 Kubernetes 用来控制创建和管理容器的不同运行时的 API，它使 Kubernetes 更容易使用不同的容器运行时。它是一个插件接口，这意味着任何符合该标准实现的容器运行时都可以被 Kubernetes 所使用。
Kubernetes 项目不必手动添加对每个运行时的支持，CRI API 描述了 Kubernetes 如何与每个运行时进行交互，由运行时决定如何实际管理容器，因此只要它遵守 CRI 的 API 即可。
![Alt text](images/image-3.png)
在早期的 Kubernetes 中包括一个名为 dockershim 的组件，使它能够支持 Docker。但 Docker 由于比 Kubernetes 更早，没有实现 CRI，所以这就是 dockershim 存在的原因，它支持将 Docker 被硬编码到 Kubernetes 中。从 Kubernetes 1.24 中已经完全移除dockershim组件。

container、cri-o都是遵循CRI的容器运行时，也是High-Level容器运行时，不是`真正意义`的容器运行时，将容器运行的实现交给了runc

既然runc就可以实现容器的启动，停止，重启，运行...操作，为什么还需要containerd，cri-o...这类所谓的High-Level容器运行时呢？
runc只是一个命令行工具，containerd是一个守护进程，runc的实例不能超过底层容器进程。通常它在create调用时开始它的生命，然后只是在容器的 rootfs 中的指定文件去运行。containerd 可以管理超过数千个runc容器。它更像是一个服务器，它侦听传入请求以启动、停止或报告容器的状态。在引擎盖下containerd使用runC。然而，containerd不仅仅是一个容器生命周期管理器。它还负责镜像管理（从注册中心拉取和推送镜像，在本地存储镜像等）、跨容器网络管理和其他一些功能。
![Alt text](images/image.png)
containerd主要负责以下功能：
1. 管理容器的生命周期（从创建容器到销毁容器）
2. 拉取/推送容器镜像
3. 存储管理（管理镜像及容器数据的存储）
4. 调用 runc 运行容器（与 runc 等容器运行时交互）
5. 管理容器网络接口及网络

![Alt text](images/image.jpg)
上图是 Containerd 整体的架构。构筑在 Containerd 组件之上以及跟这些组件做交互的都是 Containerd 的 client，Kubernetes 跟 Containerd 通过 CRI 做交互时，本身也作为 Containerd 的一个 client。Containerd 本身有提供了一个 CRI，叫 ctr；在这些clinet上面就是Google Cloud、Docker、IBM、阿里云、等容器云平台
从 k8s 的角度看，选择 containerd作为运行时的组件，它调用链更短，组件更少，更稳定，占用节点资源更少。
![Alt text](images/image-1.png)

### 3、docker
Docker 于 2013 年发布，解决了开发人员在端到端运行容器时遇到的许多问题。这里是他包含的所有东西：

容器镜像格式
一种构建容器镜像的方法（Dockerfile/docker build）；
一种管理容器镜像（docker image、docker rm等）；
一种管理容器实例的方法（docker ps, docker rm 等）；
一种共享容器镜像的方法（docker push/pull）；
一种运行容器的方式（docker run）；
当时，Docker 是一个单体系统。但是，这些功能中没有一个是真正相互依赖的。这些中的每一个都可以在可以一起使用的更小、更集中的工具中实现。每个工具都可以通过使用一种通用格式、一种容器标准来协同工作。从 Docker 1.11 之后，Docker Daemon 被分成了多个模块以适应 OCI 标准。拆分之后，结构分成了以下几个部分。
![Alt text](images/image-2.png)
其中，containerd 独立负责容器运行时和生命周期（如创建、启动、停止、中止、信号处理、删除等），其他一些如镜像构建、卷管理、日志等由 Docker Daemon 的其他模块处理。
Docker 自己在内部使用 containerd，当你安装 Docker 时也会安装 containerd
>现在创建一个docker容器的时候，Docker Daemon 并不能直接帮我们创建了，而是请求 containerd 来创建一个容器。当containerd 收到请求后，也不会直接去操作容器，而是创建一个叫做 containerd-shim 的进程。让这个进程去操作容器，我们指定容器进程是需要一个父进程来做状态收集、维持 stdin 等 fd 打开等工作的，假如这个父进程就是 containerd，那如果 containerd 挂掉的话，整个宿主机上所有的容器都得退出了，而引入 containerd-shim 这个垫片就可以来规避这个问题了，就是提供的live-restore的功能。这里需要注意systemd的
MountFlags=slave。
然后创建容器需要做一些 namespaces 和 cgroups 的配置，以及挂载 root 文件系统等操作。runc 就可以按照这个 OCI 文档来创建一个符合规范的容器。
真正启动容器是通过 containerd-shim 去调用 runc 来启动容器的，runc 启动完容器后本身会直接退出，containerd-shim 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。containerd，containerd-shim和容器进程(即容器主进程)三个进程，是有依赖关系的。

**reference**
https://zhuanlan.zhihu.com/p/438352784
https://zhuanlan.zhihu.com/p/490585683
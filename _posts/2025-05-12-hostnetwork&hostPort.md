---
title: hostnetwork&hostPort
date: 2025-05-12 22:34:00 +0800
categories: [k8s,hostnetwork,hostPort]
tags: [k8s,hostnetwork,hostPort]
toc: true
---

##### 1、hostNetwork:true
在Pod的yaml定义文件中配置该选项后，Pod就使用宿主机的网络栈，即pod ip就是宿主机的ip，pod直接使用宿主机的网络根namespace。这样和直接访问运行在宿主机上的服务没有什么区别，使用“宿主机IP+端口”的方式访问运行在Pod中的服务，比如下面的Pod定义文件，
如果Pod重启，可能会被调度到别的宿主机，访问IP也需跟着变动，不能在一台宿主机上运行同样的Pod，还要防止端口冲突，如果已经有服务占用了宿主机的端口，新的Pod将不能启动。
```yaml
[root@k8smaster hostport]# cat hostnetworkpod.yml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  containers:
    - name: nginx
      image: nginx:1.23.1
  nodeName: node2
```
查看pod的ip地址为node2的ip地址
[root@node2 ~]# kubectl get po -owide
NAME             READY   STATUS    RESTARTS   AGE   IP                NODE    NOMINATED NODE   READINESS GATES
nginx            1/1     Running   0          19m   192.168.189.202   node2   <none>           <none>

node2节点上查看端口80被监听
[root@node2 ~]# netstat -tunlp | grep 80         
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      34261/nginx: master 
tcp6       0      0 :::80                   :::*                    LISTEN      34261/nginx: master 

访问测试:
[root@k8smaster hostport]# curl 192.168.189.202
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

进入pod查看网络，看到的就是宿主机的网络
[root@k8smaster hostport]# kubectl exec -it nginx -- /bin/bash
root@k8snode1:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a7:24:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.189.202/24 brd 192.168.189.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ac64:b26:e745:a829/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever


##### 2、hostport:
hostport就是暴露pod所在节点ip+port给外部访问，使用DNAT机制将hostport指定的端口映射到容器的端口之上
这种方式和hostnetwork有相同的缺点，容器重启可能会被调度到其他的主机上。不能在一台宿主机上运行同样的Pod，除非人为调整Pod对外暴露的端口。
```yaml
[root@node2 ~]# cat hostport.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.23.1
      ports:
       - containerPort: 80
         hostPort: 8080
```
[root@node2 ~]# kubectl get po -owide
NAME             READY   STATUS    RESTARTS   AGE   IP                NODE    NOMINATED NODE   READINESS GATES
nginx-hostport   1/1     Running   0          10s   10.244.1.19       node1   <none>           <none>

访问测试
通过pod ip+containerport访问，80是默认端口可以不添加
[root@node2 ~]# curl 10.244.1.19
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

通过node节点ip+hostport访问
[root@node2 ~]# curl 192.168.189.201:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

但是调度节点上并没有监听8080端口
[root@node1 ~]# netstat -tunlp | grep 8080
查看防火墙规则
[root@node1 ~]# iptables -t nat -L
Chain CNI-DN-7657dae84bc8ea41d90c1 (1 references)
target     prot opt source               destination         
CNI-HOSTPORT-SETMARK  tcp  --  node1/24             anywhere             tcp dpt:webcache
CNI-HOSTPORT-SETMARK  tcp  --  localhost            anywhere             tcp dpt:webcache
DNAT       tcp  --  anywhere             anywhere             tcp dpt:webcache to:10.244.1.19:80


---
title: ansible-role
date: 2024-05-07 22:26:00 +0800
categories: [ansible,role]
tags: [ansible]
---

一、role角色
角色是ansible自1.2版本引入的新特性，用于层次性、结构化地组织playbook。roles能够根据层次型结构自动装载变量文件、tasks以及handlers等。要使用roles只需要在playbook中使用include指令即可。简单来讲，roles就是通过分别将变量、文件、任务、模板及处理器放置于单独的目录中，并可以便 捷地include它们的一种机制。角色一般用于基于主机构建服务的场景中，但也可以是用于构建守护进程等场景中

运维复杂的场景：建议使用 roles，代码复用度高

roles：多个角色的集合目录， 可以将多个的role，分别放至roles目录下的独立子目录中
示例
```
roles/
  mysql/
  nginx/
  tomcat/
  redis/
```
默认roles存放路径
/root/.ansible/roles
/usr/share/ansible/roles
/etc/ansible/roles
2、 ansible Roles目录编排

roles目录结构
```
playbook1.yml
playbook2.yml
roles/
 project1/
   tasks/
   files/
   vars/       
   templates/
   handlers/
   default/    
   meta/       
 project2/
   tasks/
   files/
   vars/       
   templates/
   handlers/
   default/    
   meta/
```  
Roles各目录作用

roles/project/ :项目名称,有以下子目录

    files/ ：存放由copy或script模块等调用的文件
    templates/：template模块查找所需要模板文件的目录
    tasks/：定义task,role的基本元素，至少应该包含一个名为main.yml的文件；其它的文件需要在 此文件中通过include进行包含
    handlers/：至少应该包含一个名为main.yml的文件；此目录下的其它的文件需要在此文件中通过 include进行包含
    vars/：定义变量，至少应该包含一个名为main.yml的文件；此目录下的其它的变量文件需要在此 文件中通过include进行包含
    meta/：定义当前角色的特殊设定及其依赖关系,至少应该包含一个名为main.yml的文件，其它文 件需在此文件中通过include进行包含
    default/：设定默认变量时使用此目录中的main.yml文件，比vars的优先级低
二、 创建role
创建role的步骤
1. 创建以roles命名的目录
2. 在roles目录中分别创建以各角色名称命名的目录，如mysql等
3. 在每个角色命名的目录中分别创建files、handlers、tasks、templates和vars等目录；用不到的目录可以创建为空目录，也可以不创建
4. 在每个角色相关的子目录中创建相应的文件,如 tasks/main.yml,templates/nginx.conf.j2
5. 在playbook文件中，调用需要的角色

范例：roles的目录结构
```shell
[20:48:37 root@web1 roles]#tree 
.
├── nginx-role.yaml
└── roles
    ├── files
    │   └── nginx.conf
    ├── tasks
    │   ├── groupadd.yaml
    │   ├── install.yaml
    │   ├── main.yaml
    │   ├── restart.yaml
    │   └── useradd.yaml
    └── vars
        └── main.yaml
```

三、实现docker roles
```shell
[root@k8smaster roles]# mkdir -p docker/{tasks,handlers,files，templates}
[14:20:19 root@ansible docker]#cat tasks/main.yml 
- include: docker_install.yaml   #docker安装
- include: docker_service.yaml   #生成docker的service文件
- include: swap_off.yaml         #关闭swap交换分区
- include: docker_etc.yaml       #生成docker的etc文件
- include: docker_start.yaml     #启动docker
```
#各tasks内容

```shell
[14:20:24 root@ansible docker]#cat tasks/docker_install.yaml 
- name: install docker
  unarchive: src=docker-19.03.15.tgz dest=/tmp/
- name: cpoy docker is bin
  shell: cp -rf /tmp/docker/* /usr/bin/
- name: remove docker-bin
  shell: rm -rf /tmp/docker
```
```shell
[14:33:26 root@ansible docker]#cat tasks/docker_service.yaml 
- name: add containerd_service
  template: src=containerd.service.j2 dest=/etc/systemd/system/containerd.service
- name: add docker_service
  template: src=docker.service.j2 dest=/etc/systemd/system/docker.service 
- name: add docker socket
  template: src=docker.socket.j2 dest=/etc/systemd/system/docker.socket
```
```shell 
[14:33:48 root@ansible docker]#cat tasks/swap_off.yaml 
- name: swap off is fstab
  lineinfile: path=/etc/fstab regexp="swap" state=absent
- name: swap is off
  shell: swapoff -a
```
```shell  
[14:34:06 root@ansible docker]#cat tasks/docker_etc.yaml 
- name: mkdir /etc/docker
  file: path=/etc/docker state=directory
- name: docker etc is file
  template: src=daemon.json.j2 dest=/etc/docker/daemon.json
```
```shell 
[14:34:17 root@ansible docker]#cat tasks/docker_start.yaml 
- name: start containerd
  service: name=containerd state=restarted enabled=yes
- name: start docker.socket
  service: name=docker.socket state=restarted enabled=yes
- name: start docker
  service: name=docker state=restarted enabled=yes
```
files目录下准备docker二进制文件
```shell
[14:34:28 root@ansible docker]#ls files/
docker-19.03.15.tgz
```
template目录准备service文件与etc配置文件
```shell
[14:36:20 root@ansible docker]#ls templates/
containerd.service.j2  daemon.json.j2  docker.service.j2  docker.socket.j2
```
在playbook中调用角色
```shell
[14:37:49 root@ansible roles]#cat role_docker.yml 
- hosts: all
  remote_user: root
  roles:
    - docker
```   
最终的目录结构
```shell
[14:37:47 root@ansible roles]#tree 
.
├── docker
│   ├── files
│   │   └── docker-19.03.15.tgz
│   ├── handlers
│   ├── tasks
│   │   ├── docker_etc.yaml
│   │   ├── docker_install.yaml
│   │   ├── docker_service.yaml
│   │   ├── docker_start.yaml
│   │   ├── main.yml
│   │   └── swap_off.yaml
│   └── templates
│       ├── containerd.service.j2
│       ├── daemon.json.j2
│       ├── docker.service.j2
│       └── docker.socket.j2
└── role_docker.yml
```
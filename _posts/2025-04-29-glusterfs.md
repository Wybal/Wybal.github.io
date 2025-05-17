---
title: glusterfs
date: 2025-04-29 22:26:00 +0800
categories: [分布式存储,glusterfs]
tags: [分布式存储]
pin: true 
[toc]
---

### 一、glusterfs服务端部署

初装配置

| 主机名           |     Ip地址      |     盘符 |
| ---------------- | :-------------: | -------: |
| glusterfs-node1  | 192.168.189.131 | /dev/sdb |
| glusterfs-node2  | 192.168.189.132 | /dev/sdb |
| glusterfs-client | 192.168.189.150 |

#### 1、安装常见依赖包
```
yum install -y vim net-tools ntpdate
```
#### 2、配置yum源，安装glusterfs相关包
 ```
yum install epel-release -y 
yum install -y centos-release-gluster6.noarch
yum	install glusterfs-server  glusterfs glusterfs-fuse glusterfs-rdma -y
 ```
#### 3、启动glusterd服务并设置开机自启
```
systemctl enable --now glusterd
```
#### 4、设置主机名并写入hosts文件
```
hostnamectl set-hostname glusterfs-node1
su
hostnamectl set-hostname glusterfs-node2
su

[root@glusterfs-node1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.189.131 glusterfs-node1
192.168.189.132 glusterfs-node2
scp -rp /etc/hosts glusterfs-node2:/etc/
```
#### 5、关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/enforcing/disabled/g' /etc/selinux/config
```
#### 6、配置ssh互信
```
[root@glusterfs-node1 ~]# ssh-keygen
[root@glusterfs-node1 ~]# ssh-copy-id glusterfs-node2
[root@glusterfs-node2 ~]# ssh-keygen
[root@glusterfs-node2 ~]# ssh-copy-id glusterfs-node1
```
#### 7、配置时间同步
###### 1)更改当前时区
```
[root@glusterfs-node1 ~]# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
[root@glusterfs-node1 ~]# ntpdate cn.pool.ntp.org
[root@glusterfs-node2 ~]# cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
[root@glusterfs-node2 ~]# ntpdate cn.pool.ntp.org
```
###### 2)配置时间同步服务端
修改`glusterfs-node1`的`/etc/chrony.conf`文件如下：
```
[root@glusterfs-node1 ~]# grep -v "#" /etc/chrony.conf
server cn.pool.ntp.org  iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 192.168.189.0/24
local stratum 10
logdir /var/log/chrony
```
###### 3)配置时间同步客户端
修改`glusterfs-node2`的`/etc/chrony.conf`文件如下：
```
[root@glusterfs-node2 ~]# grep -v "#" /etc/chrony.conf
server 192.168.189.131 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```
###### 4)两台分别重启chrony服务并设置开机自启
```
Systemctl restart chronyd
Systemctl enable chronyd
```
###### 5)查看当前时间同步状态
```
[root@glusterfs-node1 ~]# chronyc sources -v
[root@glusterfs-node2 ~]# chronyc sources -v
```
#### 8、添加节点
```
[root@glusterfs-node1 ~]# gluster peer probe glusterfs-node2
peer probe: success. 
[root@glusterfs-node1 ~]# gluster peer status
Number of Peers: 1

Hostname: glusterfs-node2
Uuid: 1131bc82-350e-4da0-9ec9-a4f25f768dc8
State: Peer in Cluster (Connected)
```
#### 9、创建卷
###### 1)格式化磁盘并挂载
```
[root@glusterfs-node1 ~]# mkfs.xfs /dev/sdb
[root@glusterfs-node2 ~]# mkfs.xfs /dev/sdb
[root@glusterfs-node1 ~]# mount /dev/sdb /gluster
[root@glusterfs-node2 ~]# mount /dev/sdb /gluster
```
写入/etc/fstab文件开机自动挂载
###### 2)创建卷，此处创建了一个复制卷
```
gluster v create fuzhi transport tcp replica 2  glusterfs-node1:/gluster/brick1/ glusterfs-node2:/gluster/brick1/
```
###### 3)启动卷
```
gluster v start  fuzhi
```
### 二、客户端部署及使用
#### 1、安装客户端相关依赖包
```
yum -y install glusterfs glusterfs-fuse glusterfs-cli glusterfs-libs glusterfs-client-xlator
```
#### 2、挂载卷
配置`/etc/hosts`文件
```
[root@localhost ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.189.131 glusterfs-node1
192.168.189.132 glusterfs-node2
```
挂载server提供的卷，支持高可用，即使挂载点down机，只要集群正常，仍然可以正常使用
```
mount -t glusterfs  <server>:/<volume>  <mountdir>
```
```
[root@localhost ~]# mount.glusterfs 192.168.189.132:/replcia2 /media/
```
#### 3、以nfs-ganesha方式挂载
**服务端**
gluster默认没有打开nfs挂载方式，需要在服务端打开
打开nfs挂载
```
gluster volume set replica2  nfs.disable  off
```
打开后，需要重启下volume才能生效,如果stop前已挂载volume，stop volume后client会卸载volume，start volume会自动挂载上volume
```
gluster v stop replica2
gluster v start replica2
```
安装nfs-ganesha依赖包
```
yum install -y nfs-ganesha nfs-ganesha-gluster
```
修改配置文件
```
[root@glusterfs-node1 /]# grep -Ev "#|^$" /etc/ganesha/ganesha.conf 
EXPORT
{
	Export_Id=1 ;
	Path = "/glu";			   #指定nfs共享目录的位置，该目录在volume中必须存在，客户端挂载使用的就是glusterfs volume下的该目录
	Pseudo = "/glu-pseudo"; #共享给nfs客户端的虚拟目录
	Disable_ACL =True;      #是否禁用ACL
	Protocols = "3","4";    #nfs协议版本号
	Access_Type = RW;       #访问权限类型，rw，ro
	Squash = root_squash;   #客户端访问时是否匿名,所有用户不匿名（no_squash），root用户匿名（root_squash），所有用户匿名（all_squash）
	Sectype="sys";
	Transports = "UDP","TCP";
	FSAL {						#定义的是准备导出的gluster的volume
		Name = "GLUSTER";		#是应该导出卷的卷格式GLUSTER
		Hostname="glusterfs-node1";	#是主机名 ，不同主机是不同的
		Volume = "replica2";	#是gluster的volume名
	}
}
```
重启nfs-ganesha服务
`systemctl restart nfs-ganesha`
查看gluster volume下的目录，如果没有通过glusterfs-client创建该目录
```
[root@glusterfs-client ~]# df -h
glusterfs-node2:/replica2  5.0G  584M  4.5G  12% /media
[root@glusterfs-client ~]# mkdir /media/glu
[root@glusterfs-client ~]# ls -l /media/
drwxr-xr-x 2 root root         6 Aug 14 16:31 glu
```
**客户端**
安装nfs相关依赖包
`yum install -y nfs-utils`
查看nfs共享目录
```
[root@glusterfs-client nfs]# showmount -e glusterfs-node1
Export list for glusterfs-node1:
//glu-pseudo (everyone)
```
使用暴露的虚拟目录进行挂载
```
[root@glusterfs-client ~]# mount.nfs 192.168.189.131:/glu-pseudo /nfs/  
[root@glusterfs-client ~]# df -h
Filesystem                 Size  Used Avail Use% Mounted on
192.168.189.131:/glu       5.0G  583M  4.5G  12% /nfs
```
***
创建卷之前先了解一下glusterfs卷的常见模式
### 三、glusterfs卷的常见模式
#### 1、复制卷（replica）
双复制类似于raid1，可以选择三复制，四复制....支持多副本,每一个brick都是一个独立完整的副本。

```
 gluster volume create NEW-VOLNAME [replica COUNT] [transport tcp | rdma | tcp, rdma] NEW-BRICK
```
 eg:
```
gluster volume create test-volume3 replica 2 transport tcp server1:/data/brick1/test1/ server2:/data/brick1/test1/
```
#### 2、哈希卷（Distribute）
每一个brick只存储副本的一部分，以文件为单位。
```
 gluster volume create NEW-VOLNAME [transport tcp | rdma | tcp, rdma] NEW-BRICK
 ```
 eg:
 ```
gluster	volume	create	new-volume server1:/data/brick1/exp2 server2:/data/brick1/exp2
 ```
 当客户端挂载分布式卷时，创建多个文件，在每个server端只能看到一部分文件
 ```
 [root@glusterfs-client]# ls
testfile1  testfile10  testfile2  testfile3  testfile4  testfile5  testfile6  testfile7  testfile8  testfile9
```
server端查看
```
[root@glusterfs-node1 ~]#  ls /data0/gluster/data1/
testfile1  testfile10  testfile2  testfile3  testfile4  testfile6  testfile8
[root@glusterfs-node2 ~]#  ls /data0/gluster/data1/
testfile5  testfile7  testfile9
```
#### 3、条带卷（Strip）
条带卷将数据条带化，切割成若干等份，分别存储在不同的brick里面。减少了负载加速了文件存取的速度
 ```
 gluster volume create NEW-VOLNAME [stripe COUNT] [transport tcp | rdma | tcp, rdma] NEW-BRICK...
 ```
eg：
```
gluster volume create test-volume3 stripe 2 transport tcp server1:/data/brick1/test1/ server2:/data/brick1/test1/
```
当客户端挂载条带式卷时，每个server端只能看到条带后的一部分文件
```
[root@glusterfs-client ~]#  du -sh /root/root.tar.gz
140M    /root/root.tar.gz
```
server端查看
```
[root@lvs ~]# du -sh /data0/gluster/data3/root.tar.gz
70M     /data0/gluster/data3/root.tar.gz
[root@lvs ~]# du -sh /data0/gluster/data3/root.tar.gz
70M     /data0/gluster/data3/root.tar.gz
```
#### 4、分布式复制卷
 ```
 gluster volume create NEW-VOLNAME [replica COUNT] [transport tcp | rdma | tcp, rdma] NEW-BRICK...
 ```
#### 5、分布式条带卷
 ```
gluster volume create NEW-VOLNAME [stripe COUNT] [transport tcp | rdma | tcp, rdma] NEW-BRICK...
 ```
#### 6、条带复制卷
 ```
gluster volume create NEW-VOLNAME [stripe COUNT] [replica COUNT] [transport tcp | rdma | tcp, rdma] NEW-BRICK...
 ```
#### 7、仲裁卷
  如果是副本为3的仲裁器卷，其中第三个brick 充当仲裁器brick。 该配置具有防止发生裂脑的机制。
```
# gluster volume create  <VOLNAME>  replica 3 arbiter 1 host1:brick1 host2:brick2 host3:brick3`
```
#### 8、冗余卷(EC卷)
冗余卷，会把一份数据切分成多份，然后计算得到冗余码，并存储到各节点中，在损坏一定的比例数据下，数据也不会损坏。类似于raid5，6。
```
# gluster volume create <VOLUME> disperse 3  host1:brick1 host2:brick2 host3:brick3
gluster v info
Volume Name: test-disperse  
Type: Disperse  
Volume ID: 0390d729-b6d8-4edd-bb72-bd28c3ec7472  
Status: Started  
Snapshot Count: 0  
Number of Bricks: 1 x (2 + 1) = 3  
```
### 四、glusterfs常用命令及操作
#### 1、启停/查看glusterd服务,设置开机自启
 ```
systemctl start glusterd
systemctl stop glusterd
systemctl status glusterd
systemctl enable glusterd
systemctl list-unit-files | grep glusterd
 ```
#### 2、查看卷信息
```
gluster volume list    #列出集群中的所有卷：
gluster volume info [all] <VOLNAME> #查看集群中的卷信息：
gluster volume status [all]    #查看集群中的卷状态：
gluster volume status <VOLNAME> [detail| clients | mem | inode | fd] 
```

#### 3、为存储池添加/移除服务器节点
```
gluster peer probe <node>		添加服务器节点
gluster peer detach <node>		移除服务器节点 移除节点前必须将该节点上的brick删除
gluster peer status		        查看所有节点的基本状态
```
#### 4、创建/启动/停止/删除卷
```
gluster volume start <volname>
gluster volume stop <volname>
gluster volume delete <volname>  删除卷之前必须先停止卷，最后可清空brick server节点对应目录下的内容
```
eg：
```
[root@glusterfs-node1 brick]# gluster v stop test-replica4
Stopping volume will make its data inaccessible. Do you want to continue? (y/n) y
volume stop: test-replica4: success
[root@glusterfs-node1 brick]# gluster v delete test-replica4
Deleting volume will erase all information about the volume. Do you want to continue? (y/n) y
volume delete: test-replica4: success
```
#### 5、扩容/缩容volume
###### 1)缩容
缩容，remove-brick之后会自动均衡数据
```
gluster v remove-brick <VOLNAME> [replica <COUNT>] <BRICK> ... <start|stop|status|commit|force>  
```
eg：
如果是复制卷或者条带卷，则每次移除的brick数必须是replica或者stripe的整数倍，并属于同一subvolume
```
[root@glusterfs-node1 brick]# gluster v remove-brick replica2 glusterfs-node1:/test-replica{3..4}/brick start
[root@glusterfs-node1 brick]# gluster v remove-brick replica2 glusterfs-node1:/test-replica{3..4}/brick status
 Node Rebalanced-files          size       scanned      failures       skipped               status  run time in h:m:s
 ---------      -----------   -----------   -----------   -----------   -----------         ------------     --------------
localhost                0        0Bytes             9             0             0          in progress        0:00:10
The estimated time for rebalance to complete will be unavailable for the first 10 minutes.
[root@glusterfs-node1 brick]# gluster v remove-brick replica2 glusterfs-node1:/test-replica{3..4}/brick commit
```
如果是`distribute` volume可以直接remove brick
>remove-brick前，node1 801M，node2 1.4G
```
[root@glusterfs-node1 brick1]# du -sh 
801M	.
[root@glusterfs-node2 brick1]# du -sh 
1.4G	.
```
>remove-brick start
```
[root@glusterfs-node1 ~]# gluster v remove-brick haxi glusterfs-node2:/glusterfs/replica/brick1/ start
Running remove-brick with cluster.force-migration enabled can result in data corruption. It is safer to disable this option so that files that receive writes during migration are not migrated.
Files that are not migrated can then be manually copied after the remove-brick commit operation.
Do you want to continue with your current cluster.force-migration settings? (y/n) y
volume remove-brick start: success
ID: d8cc074f-ac59-439e-8b7d-dbbbef279045
```
>remove-brick后,node1 2.2G，node2上的数据自动迁移到node1上
```
[root@glusterfs-node1 brick1]# du -sh 
2.2G	.
```
>commit前
```
[root@glusterfs-node1 ~]# gluster v info
Brick1: glusterfs-node1:/glusterfs/replica/brick1
Brick2: glusterfs-node2:/glusterfs/replica/brick1
```
```
[root@glusterfs-node1 ~]# gluster v remove-brick haxi glusterfs-node2:/glusterfs/replica/brick1/ commit
volume remove-brick commit: success
Check the removed bricks to ensure all files are migrated.
If files with data are found on the brick path, copy them via a gluster mount point before re-purposing the removed brick. 
```
>commit后
```
[root@glusterfs-node1 ~]# gluster v info
Brick1: glusterfs-node1:/glusterfs/replica/brick1
```
###### 2)扩容
```
gluster volume add-brick <VOLNAME> [<stripe|replica> <COUNT> [arbiter <COUNT>]] <NEW-BRICK> ... [force]
```
eg:
add-brick前
```
[root@glusterfs-node1 brick1]# du -sh 
2.2G	.
[root@glusterfs-node1 ~]# gluster v info
Brick1: glusterfs-node1:/glusterfs/replica/brick1
```
add-brick start
```
[root@glusterfs-node1 ~]# gluster v add-brick haxi glusterfs-node2:/glusterfs/replica/brick1/
volume add-brick: success
```
add-brick之后
```
[root@glusterfs-node2 replica]# gluster v info
Brick1: glusterfs-node1:/glusterfs/replica/brick1
Brick2: glusterfs-node2:/glusterfs/replica/brick1
```
**扩容完后最好均衡下数据，见第7节**
如果是复制卷或者条带卷，则每次添加的Brick数必须是replica的整数倍。

#### 6、替换brick
**替换前如果故障brick或源brick有进程，需要先kill掉再进行替换**
**使用一个新的brick替换旧的brick，2个brick路径不能一致，使用replace**
>on distribute only volumes仅哈希卷不允许迁移替换
```
gluster volume replace-brick <VOLNAME> <SOURCE-BRICK> <NEW-BRICK> {commit force}
```
>注意：new-brick要求未被其他gluster使用并且和源brick大小一致
```
gluster volume replace-brick replica2 192.168.189.132:/gluster1/brick1 192.168.189.132:/gluster/brick1 commit force
迁移完成后，源brick就不会再存储新写入的数据
```
当dest-brick和src-brick路径一致时，使用replace会failed，此时应使用reset-brick，如果此时使用replace-brick会报错，如下：
```
$ gluster v replace-brick replica2 glusterfs-node1:/glusterfs/replica/brick3 glusterfs-node1:/glusterfs/replica/brick3 commit force
volume replace-brick: failed: Brick: glusterfs-node1:/glusterfs/replica/brick3 not available. Brick may be containing or be contained by an existing brick
```
**当硬盘坏了后，替换硬盘后并仍将新硬盘挂载在原有brick目录下，需要使用reset-brick**
```
gluster volume reset-brick <VOLNAME> <SOURCE-BRICK> {start} | {<NEW-BRICK> commit force} 
gluster volume reset-brick VOLNAME HOSTNAME:BRICKPATH start #终止指定brick的进程，开始reset
gluster volume reset-brick <VOLNAME> <SOURCE-BRICK> {<NEW-BRICK> commit force} #进程被kill后，使用相同brick-path替换src-brick时执行
```
```
$ gluster v reset-brick replica2 glusterfs-node1:/glusterfs/replica/brick3 glusterfs-node1:/glusterfs/replica/brick3 commit force
volume reset-brick: success: reset-brick commit force operation successful
```
当src-brick和dest-brick不一致时，使用reset-brick会failed，并提示使用replace-brick
```
[root@glusterfs-node1 replica]# gluster v reset-brick replica2 glusterfs-node1:/glusterfs/replica/brick1 glusterfs-node1:/glusterfs/replica/brick3 commit force
volume reset-brick: failed: When destination brick is new, please use gluster volume replace-brick <volname> <src-brick> <dst-brick> commit force
```

**openEuler替换brick**

* 在服务端查看故障brick进程id，并结束该进程
* 客户端卸载包含故障brick的卷
* 客户端重新挂载卷到新的挂载点<客户端挂载点>
* 查询故障节点的备份节点的扩展属性getfattr -d -m. -e hex BRICK
* 客户端新建目录并删除
   cd <客户端挂载点>
   mkdir testDir
   rm -rf testDir
* 设置扩展属性触发自愈
   setfattr -n trusted.non-existent-key -v abc <客户端挂载点>
   setfattr -x trusted.non-existent-key <客户端挂载点>
* 强制替换卷
   gluster volume replace-brick VOLUME BAD-BRICK NEW-BRICK commit force

#### 7、均衡volume
>一般add-brick扩容后才需要均衡卷

当add new brick后，新的brick里面都会创建已存在的目录，但是内容为空
当新增文件时，glusterfs会根据各个brick的剩余空间大小来决定写到那个brick里
当新增brick后，新增brick下会同步创建以前存在的目录，但目录下为空
新增brick以前创建的旧目录下新增文件默认不会在新brick下存储
volume根下的文件也不会在新的brick下存储
只有新创建的目录及新创建目录下的文件才会在新的brick下存储

```
gluster volume rebalance <VOLNAME> fix-layout start   重新哈希分布目录，不会迁移数据
gluster volume rebalance <VOLNAME> start              开始迁移数据
gluster volume rebalance <VOLNAME> start force
gluster volume rebalance <VOLNAME> status
gluster volume rebalance <VOLNAME> stop
```
eg:
1. fix-layout重新哈希分布目录，
   不会迁移数据，但可以让旧目录下的新创建的文件存储在新brick下。
```
[root@glusterfs-node2 kylin]# gluster v rebalance replica2 fix-layout start
volume rebalance: replica2: success: Rebalance on replica2 has been started successfully. Use rebalance status command to check status of the rebalance process.
ID: e3116dbe-60b9-4867-8f32-8f4a37d6c23f
当rebalance后在旧目录下或者volume根目录下创建新文件，会在新的brick下存储
```
2. 均衡数据，会把新旧brick下的数据进行迁移均衡,创建的新文件也会在new brick下存储

>rebalance前

```
[root@glusterfs-node2 replica]# gluster v info
Brick1: glusterfs-node1:/glusterfs/replica/brick1
Brick2: glusterfs-node2:/glusterfs/replica/brick1
[root@glusterfs-node2 brick1]# du -sh 
4.0K	.
[root@glusterfs-node1 brick1]# du -sh 
2.2G	.
```
>rebalance start

```
[root@glusterfs-node1 ~]# gluster volume rebalance haxi start
volume rebalance: haxi: success: Rebalance on haxi has been started successfully. Use rebalance status command to check status of the rebalance process.
ID: 7c67dbcf-48d1-45ee-ad64-c122f7018bb5
```
```
正在均衡
# gluster volume rebalance test-volume status
                                  Node  Rebalanced-files  size  scanned       status
                             ---------  ----------------  ----  -------  -----------
  617c923e-6450-4065-8e33-865e28d9428f               416  1463      312  in progress
均衡完成
 # gluster volume rebalance test-volume status
                                  Node  Rebalanced-files  size  scanned       status
                             ---------  ----------------  ----  -------  -----------
  617c923e-6450-4065-8e33-865e28d9428f               502  1873      334   completed
暂停均衡
  # gluster volume rebalance test-volume stop
                                  Node  Rebalanced-files  size  scanned       status
                             ---------  ----------------  ----  -------  -----------
  617c923e-6450-4065-8e33-865e28d9428f               59   590      244       stopped
  Stopped rebalance process on volume test-volume
```
>rebalance后

```
[root@glusterfs-node1 brick1]# du -sh 
801M	.
[root@glusterfs-node2 brick1]# du -sh 
1.4G	.
```
**设置迁移速度**
`gluster volume set <VOLUME> rebal-throttle lazy|normal|aggressive`

默认normal
* Lazy:慢速模式，较少线程迁移
* Normal：正常模式线程数量适中
* Aggressive：激进模式，较多线程迁移

**均衡前的注意事项**

* 如果有文件损坏，先修复
* 均衡前，确认集群没有自修复在进行否则会影响数据正确率和迁移效率
* 先执行fix-layout再进行数据迁移，可以提高迁移效率
* 如果可以，停止客户端使用，再迁移，可以提高迁移效率
* 迁移过程中，使用status时刻查看迁移状态
* 当集群较大时，可能会出现某个节点均衡失败的问题，一般重新开始执行均衡即可
* 如果迁移对程序影响较大，可以只执行fix-layout，只修复目录哈希分布，不会实际迁移数据，此时新文件会存储在新节点或者brick上

#### 8、限额quota

开启/关闭系统配额：
```
gluster volume quota <VOLNAME> enable | disable
```
设置/修改目录配额，此次的DIR为volume的相对路径：
```
gluster volume quota <VOLNAME> limit-usage <DIR> <HARD_LIMIT>
```
查看配额：
```
gluster volume quota <VOLNAME> list [<DIR>]
```
eg：
开启限额quota
```
[root@glusterfs-node2 ~]# gluster v quota replica2 enable
volume quota : success
[root@glusterfs-node2 ~]# gluster v info
Volume Name: replica2
Type: Replicate
Volume ID: e041e77e-0b24-4290-a451-ba14acaded92
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: glusterfs-node1:/glusterfs/replica/brick
Brick2: glusterfs-node2:/glusterfs/replica/brick
Options Reconfigured:
---
features.quota-deem-statfs: on   #打开此设置，当执行df -h时返回的是硬限制的大小，而不是volume的大小，比如设置/目录配额为1GB，volume为10GB，当打开此选项时df查看为1GB大小
features.inode-quota: on         #inode的配额
features.quota: on               #容量的配额
---
cluster.quorum-type: auto
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```
设置目录配额,默认单位为Bytes,volume下必须存在此目录，此处的/为volume的/，不是操作系统的/
```
[root@glusterfs-node2 brick]# gluster volume quota replica2 limit-usage /test1 100
volume quota : success
[root@glusterfs-node2 brick]# gluster v quota replica2 list /test1
Path                   Hard-limit  Soft-limit      Used  Available  Soft-limit exceeded? Hard-limit exceeded?
-------------------------------------------------------------------------------------------------------------------------------
/test1                 100Bytes     80%(80Bytes)   0Bytes 100Bytes
```
修改目录配额，KB，MB，GB...
```
[root@glusterfs-node2 brick]# gluster volume quota replica2 limit-usage /test1 1GB
volume quota : success
```
查看配额
```
[root@glusterfs-node2 ~]# gluster v quota replica2 list
```
查看某一个目录的配额
`
gluster volume quota <VOLNAME> list <PATH>
`
```
[root@glusterfs-node2 brick]# gluster v quota replica2 list /test1
Path                   Hard-limit  Soft-limit      Used  Available  Soft-limit exceeded? Hard-limit exceeded?
-------------------------------------------------------------------------------------------------------------------------------
/test1                    1.0GB     80%(819.2MB)   0Bytes   1.0GB              No                   No
```
客户端挂载
```
[root@glusterfs-client ~]#`mount.glusterfs glusterfs-node1:/replica2/test1 /opt/
[root@glusterfs-client ~]# df -h
glusterfs-node1:replica2/test1  1.0G     0  1.0G   0% /opt
```
移除配额
```
gluster volume quota <VOLNAME> remove <PATH>
```
当超出配额的hard-limit时，clinet端不能写

当超出soft-limit时，list查看Soft-limit exceeded?为yes
```
[root@glusterfs-node1 brick1]# gluster volume quota replica2 list 
                  Path                   Hard-limit  Soft-limit      Used  Available  Soft-limit exceeded? Hard-limit exceeded?
-------------------------------------------------------------------------------------------------------------------------------
/glu                                       1.0GB     80%(819.2MB)  830.0MB 194.0MB             Yes                   No
```
当超出shard-limit时，list查看Soft-limit exceeded?和Hard-limit exceeded?都为yes
```
[root@glusterfs-node1 brick1]# gluster volume quota replica2 list 
                  Path                   Hard-limit  Soft-limit      Used  Available  Soft-limit exceeded? Hard-limit exceeded?
-------------------------------------------------------------------------------------------------------------------------------
/glu                                       1.0GB     80%(819.2MB)    1.0GB  0Bytes             Yes                  Yes
```
当超出配额，客户端再继续写入，会提示已超出配额
```
[root@ceph-node1 glu]# echo "123"> Kylin
-bash: Kylin: Disk quota exceeded
```

#### 9、配置卷及优化
```
gluster volume set <VOLNAME> <KEY> <VALUE>
gluster volume get <VOLNAME> <KEY>/all
gluster volume set help       #获取所有可用参数
```
常见配置：
1.  设置 cache 大小, 默认32MB
gluster volume set senyintvolume performance.cache-size 128MB

2. 设置 io 线程, 太大会导致进程崩溃
gluster volume set senyintvolume performance.io-thread-count 16

3. 设置 日志输出级别，默认为INFO
gluster volume set senyintvolume cluster.daemon-log-level INFO

4. 设置 写缓冲区的大小, 默认1M
gluster volume set senyintvolume performance.write-behind-window-size 1024MB

5. 设置 cluster.quorum-type, 默认none
   none|auto|fixed,none:关闭，auto：半数以上才允许写入，fixed：大于等于cluster.quorum-count设置的数量才允许写入
   cluster.server-quorum-type，默认off。百分比值，可以调节，一般设置50%以上，当集群不满足server-quorum时整个集群不可用

6. 设置 cluster.favorite-child-policy, 默认为none
   mtime 以最新的mtime自动修复脑裂

7. 设置nfs.disable，默认为 on
   off|on  on:关闭nfs挂载，off:开启nfs挂载

8. 设置network.ping-timeout，默认为42秒
   glusterfs client挂载时挂载任一一个server端的ip都可以，即使该server down机，只要集群正常就不影响client的使用。
   当挂载对应server节点down机时，会自动切换到其他的server，切换时会有一个默认检测超时时间，为network.ping-timeout，默认为42s，时间过长，可以修改为8s。
   sudo gluster volume set $volname network.ping-timeout 8

9. cluster.self-heal-daemon，集群自我修复daemon，on/off，默认为off
10. cluster.entry-self-heal,cluster.metadata-self-heal,cluster.data-self-heal，对entry，metadata，data是否启用自我修复
11. cluster.heal-timeout，自动愈合的检测间隔，默认600s
12. cluster.stripe-block-size，条带卷中每个条带的大小
13. gluster volume geo-replication <MASTER> <SLAVE> start | status | stop 跨地域复制，异地容灾
14. gluster volume profile <VOLNAME> start | info | stop IO信息查看
   Top命令允许你查看Brick的性能，例如：read,write, file open calls, file read calls, file write calls, directory opencalls, and directory real calls。所有的查看都可以设置 top数，默认100。
15. gluster volume top <VOLNAME> open[brick <BRICK>] [list-cnt <COUNT>] 其中，open可以替换为read, write, opendir, readdir等。
16. gluster volume top <VOLNAME> read-perf [bs <BLOCK-SIZE> count <COUNT>] [brick <BRICK>] [list-cnt <COUNT>] 查看每个 Brick 的读性能，其中，read-perf可以替换为write-perf等
    
#### 10、gluster相关日志
>相关日志，在`/var/log/glusterfs/`目录下，可根据需要查看；

>如/log/glusterfs/brick/下是各brick创建的日志；

>如ar/log/glusterfs/cmd_history.log是命令执行记录日志；

>如/var/log/glusterfs/glusterd.log是glusterd守护进程日志。

#### 11、查看文件在节点上的位置
在client端执行,对于包含分布式卷的节点查找文件有用
getfattr -n trusted.glusterfs.pathinfo 共享文件夹
```
[root@glusterfs-node2 ~]# df -h
glusterfs-node2:/replica2  5.0G  3.1G  2.0G  62% /mnt
[root@glusterfs-node2 mnt]# getfattr -n trusted.glusterfs.pathinfo /mnt/123 
getfattr: Removing leading '/' from absolute path names
# file: mnt/123
trusted.glusterfs.pathinfo="(<REPLICATE:replica2-replicate-0> <POSIX(/glusterfs/replica/brick1):glusterfs-node1:/glusterfs/replica/brick1/123> <POSIX(/glusterfs/replica/brick1):glusterfs-node2:glusterfs/replica/brick1/123>)"
```
可以看到文件保存在节点glusterfs-node1的/glusterfs/replica/brick1/123和glusterfs-node2的glusterfs/replica/brick1/123下

#### 12、清除glusterfs的锁
gluster volume clear-locks VOLNAME path kind {blocked | granted | all}{inode range | entry basename | posix range}
```
# gluster volume clear-locks VOLNAME path kind granted entry basename   #清除入口锁
gluster volume clear-locks test-volume / kind granted entry file1       
Volume clear-locks successful
test-volume-locks: entry blocked locks=0 granted locks=1    
# gluster volume clear-locks VOLNAME path kind granted inode range      #清除 inode 锁
gluster  volume clear-locks test-volume /file1 kind granted inode 0,0-0
# gluster volume clear-locks VOLNAME path kind granted posix range      #清除授予的 POSIX 锁
gluster volume clear-locks test-volume /file1 kind granted posix 0,8-1
# gluster volume clear-locks VOLNAME path kind blocked posix range      #清除被阻止的 POSIX 锁
gluster volume clear-locks test-volume /file1 kind blocked posix 0,0-1
# gluster volume clear-locks VOLNAME path kind all posix range          #清除 test-volume 上的所有 POSIX 锁
gluster volume clear-locks test-volume /file1 kind all posix 0,0-1
```
gluster volume clear-locks <volume> ksvd/in7/DELTA.IMG kind granted posix

查看锁是否开启：gluster volume get <volume> lookup-optimize

关闭锁：gluster volume set <volume> lookup-optimize off 

### FAQ
#### 1、客户端挂载报错Mount failed. Please check the log file for more details.
查看日志`/var/log/glusterfs/replica-.log`
日志输出如下：
DNS resolution failed on host glusterfs-node1

**客户端必须配置hosts文件才可以挂载**
```
[root@localhost ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.189.131 glusterfs-node1
192.168.189.132 glusterfs-node2
```
配置`hosts`后再次挂载正常
```
[root@localhost ~]# mount.glusterfs 192.168.189.132:/fuzhi /media/
[root@localhost ~]# df -h
glusterfs-node1:/fuzhi   5.0G   84M  5.0G   2% /media
```
#### 2、添加或者创建volume时失败volume create: replica2: failed: /gluster/brick1 is already part of a volume
要添加的brick有glusterfs的信息，需要清除掉再创建或添加
删除对应brick下的` .glusterfs`目录
```
setfattr -x trusted.glusterfs.volume-id brick
setfattr -x trusted.gfid brick 
```
#### 3、扩展或者创建volume时volume add-brick: failed: The brick glusterfs-node1:/test-replica1 is a mount point. Please create a sub-directory under the mount point and use that as the brick directory. Or use 'force' at the end of the command if you want to override this behavior.
```
[root@glusterfs-node1 test-replica1]# gluster v add-brick replica2 glusterfs-node1:/test-replica1/ glusterfs-node1:/test-replica2
volume add-brick: failed: The brick glusterfs-node1:/test-replica1 is a mount point. Please create a sub-directory under the mount point and use that as the brick directory. Or use 'force' at the end of the command if you want to override this behavior.
```
直接使用disk挂载点创建会报错
在挂载点下创建目录再创建volume
```
[root@glusterfs-node1 test-replica1]# mkdir /test-replica{1..2}/brick
[root@glusterfs-node1 test-replica1]# gluster v add-brick replica2 glusterfs-node1:/test-replica1/brick glusterfs-node1:/test-replica2/brick
volume add-brick: success
```
#### 4、缩容、扩容或者删除volume时volume remove-brick start: failed: Removing bricks from replicate configuration is not allowed without reducing replica count explicitly
```
[root@glusterfs-node1 brick]# gluster v remove-brick test-replica4 glusterfs-node4:/test-replica4/brick/ start
Running remove-brick with cluster.force-migration enabled can result in data corruption. It is safer to disable this option so that files that receive writes during migration are not migrated.
Files that are not migrated can then be manually copied after the remove-brick commit operation.
Do you want to continue with your current cluster.force-migration settings? (y/n) y
volume remove-brick start: failed: Removing bricks from replicate configuration is not allowed without reducing replica count explicitly.
```
当remove brick时如果volume为replica volume，remove的brick必须为volume的倍数，否则报错
#### 5、State: Peer in Cluster (Disconnected)对端节点未连接
```
[root@gluster2 peers]# gluster peer status
Number of Peers: 1

Hostname: gluster1
Uuid: 04c65736-3be9-4e4f-95dc-2d164aa76767
State: Peer in Cluster (Disconnected)
[root@gluster1 peers]# gluster peer status
Number of Peers: 1

Hostname: gluster2
Uuid: a17a13f8-515c-4d0d-9a3e-abdabad59e0d
State: Peer in Cluster (Disconnected)
```
查看配置文件的uuid和hostname
```
cd 到/var/lib/gluterd/peers/下，查看对端节点uuid和hostname
[root@gluster1 peers]# cd /var/lib/glusterd/peers/
[root@gluster1 peers]# ls
a17a13f8-515c-4d0d-9a3e-abdabad59e0d
[root@gluster1 peers]# cat a17a13f8-515c-4d0d-9a3e-abdabad59e0d 
uuid=a17a13f8-515c-4d0d-9a3e-abdabad59e0d
state=3
hostname1=gluster2
[root@gluster2 peers]# cat 04c65736-3be9-4e4f-95dc-2d164aa76767 
uuid=04c65736-3be9-4e4f-95dc-2d164aa76767
state=3
hostname1=gluster1
在两台服务器上分别查看uuid，uuid正确
[root@gluster2 peers]# gluster pool  list
UUID					Hostname 	State
04c65736-3be9-4e4f-95dc-2d164aa76767	gluster1 	Disconnected 
a17a13f8-515c-4d0d-9a3e-abdabad59e0d	localhost	Connected 
[root@gluster1 peers]# gluster pool list
UUID					Hostname 	State
a17a13f8-515c-4d0d-9a3e-abdabad59e0d	gluster2 	Disconnected 
04c65736-3be9-4e4f-95dc-2d164aa76767	localhost	Connected 
 ```
 查看/etc/hosts文件
 ```
[root@gluster1 peers]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.189.200 gluster1
192.168.189.201 gluster2
```
查看主机ip，这两台主机修改过ip地址，hosts无法解析到正确ip，导致gluster对端未连接，修改hosts文件或者ip地址
```
[root@gluster1 peers]# ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.189.102  netmask 255.255.255.0  broadcast 192.168.189.255
再次查看节点状态，恢复正常
[root@gluster1 peers]# gluster peer status
Number of Peers: 1

Hostname: gluster2
Uuid: a17a13f8-515c-4d0d-9a3e-abdabad59e0d
State: Peer in Cluster (Connected)
[root@gluster2 peers]# gluster peer status
Number of Peers: 1

Hostname: gluster1
Uuid: 04c65736-3be9-4e4f-95dc-2d164aa76767
State: Peer in Cluster (Connected)
```

修改完ip地址后，查看volume状态
```
[root@gluster1 peers]# gluster v status
Status of volume: fuzhi
Gluster process                             TCP Port  RDMA Port  Online  Pid

Brick gluster1:/data/brick1                 49153     0          Y       3544 
Brick gluster2:/data1/brick1                N/A       N/A        N       N/A  
Brick gluster2:/data/brick1                 N/A       N/A        N       N/A  
Brick gluster2:/data2/brick1                N/A       N/A        N       N/A  
Self-heal Daemon on localhost               N/A       N/A        N       N/A  
Self-heal Daemon on gluster2                N/A       N/A        Y       3928 
 
Task Status of Volume fuzhi
------------------------------------------------------------------------------
There are no active volume tasks
gluster2节点上的状态都未N
[root@gluster1 peers]# gluster volume heal fuzhi info
Brick gluster1:/data/brick1
Status: Connected
Number of entries: 0

Brick gluster2:/data1/brick1
Status: Transport endpoint is not connected	#状态：传输终结点未连接 
Number of entries: -

Brick gluster2:/data/brick1
Status: Transport endpoint is not connected
Number of entries: -

Brick gluster2:/data2/brick1
Status: Transport endpoint is not connected
Number of entries: -
```
重启gluster2节点的glusterd服务后恢复正常
#### 6、双复制一个节点的磁盘损坏
节点：gluster1、gluster2
gluster1节点的brick硬盘损坏
移除gluster1的brick,将副本数变为1，即可恢复业务，也可以修改cluster.quorum-type为fixed且cluster.quorum-count为1也可以立即恢复业务。
###### 1）初始状态
```
[root@gluster2 ~]# gluster volume status
Status of volume: fuzhi
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick gluster1:/data/brick1                 49153     0          Y       3047 
Brick gluster2:/data/brick1                 49155     0          Y       1617 
Self-heal Daemon on localhost               N/A       N/A        Y       1634 
Self-heal Daemon on gluster1                N/A       N/A        Y       3064 
 
Task Status of Volume fuzhi
------------------------------------------------------------------------------
There are no active volume tasks
```
###### 2）移除brick
```
[root@gluster2 ~]# gluster volume remove-brick fuzhi replica 1 gluster2:/data/brick1/ force
```
###### 3）查看当前状态
```
[root@gluster2 ~]# gluster volume status
Status of volume: fuzhi
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick gluster1:/data/brick1                 49153     0          Y       3047 
 
Task Status of Volume fuzhi
------------------------------------------------------------------------------
There are no active volume tasks
```
###### 4）添加新的brick
```
[root@gluster2 ~]# gluster volume add-brick fuzhi replica 2 gluster2:/data1/brick1/
Replica 2 volumes are prone to split-brain. Use Arbiter or Replica 3 to avoid this. See: http://docs.gluster.org/en/latest/Administrator%20Guide/Split%20brain%20and%20ways%20to%20deal%20with%20it/.
Do you still want to continue?
 (y/n) y
volume add-brick: failed: /data1/brick1 is already part of a volume
```
brick添加失败，该brick已是卷的一部分，对于硬盘原有做过gluster，该硬盘的brick1下仍存在原有的brick信息，将原有信息删除
###### 5）删除原有brick相关信息
```
[root@gluster2 ~]# rm -rf /data1/brick1/
```
###### 6）重新添加brick
```
[root@gluster2 data1]# gluster volume add-brick fuzhi replica 2 gluster2:/data1/brick1/
Replica 2 volumes are prone to split-brain. Use Arbiter or Replica 3 to avoid this. See: http://docs.gluster.org/en/latest/Administrator%20Guide/Split%20brain%20and%20ways%20to%20deal%20with%20it/.
Do you still want to continue?
 (y/n) y
volume add-brick: success
```
###### 7）查看当前状态
```
[root@gluster2 data1]# gluster volume status
Status of volume: fuzhi
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick gluster1:/data/brick1                 49153     0          Y       3047 
Brick gluster2:/data1/brick1                49155     0          Y       2063 
Self-heal Daemon on localhost               N/A       N/A        Y       2080 
Self-heal Daemon on gluster1                N/A       N/A        Y       3493 
 
Task Status of Volume fuzhi
------------------------------------------------------------------------------
There are no active volume tasks
```
#### 7、双复制场景单一节点故障分析
>当client和server端已建立连接后且cluster.quorum-type是auto时

* 任一节点或者双节点glusterd服务stop(glusterd被kill掉)不影响client端
* 任一节点的glusterfsd进程被kill掉，有一半几率会导致client不能读写，与挂载时写的哪个serverIP无关系，当节点恢复后，client会自动恢复正常，无需其他操作

此时第一个brick必须要active，也相当于没有高可用能力
第一个brick异常，client端不能正常读写
第二个brick异常，client可以正常读写

> 当cluster.quorum-type是fixed且cluster.quorum-count为1

* 此时任一brick异常或者任一节点down机都不会影响client端读写，此时有高可用能力，但是易脑裂，可以在双复制一节点坏盘或者down机时临时处理，让client端可正常读写
* 如果cluster.quorum-type是fixed且cluster.quorum-count为2，此时任一brick异常或者任一节点down机都会影响client端不能正常读写，此时没有高可用能力
> 当cluster.quorum-type是none
* 此时任一brick异常或者任一节点down机都不会影响client端读写，此时有高可用能力


客户端不能读写时报错
```
[root@ceph-node1 ~]# cd /replica 
-bash: cd: /replica: Transport endpoint is not connected
```
#### 8、glusterfs是否支持多副本？
创建4副本
```
[root@glusterfs-node1 brick]# gluster v create test-replica4 replica 4 glusterfs-node1:/test-replica1/brick/ glusterfs-node1:/test-replica2/brick/ glusterfs-node1:/test-replica3/brick/ glusterfs-node1:/test-replica4/brick/ force
volume create: test-replica4: success: please start the volume to access data

[root@glusterfs-node1 brick]# gluster v status test-replica4  
Volume test-replica4 is not started
```
启动test-replica4 volume
```
[root@glusterfs-node1 brick]# gluster v start test-replica4
volume start: test-replica4: success
```
查看volume状态
```
[root@glusterfs-node1 brick]# gluster v status test-replica4 
Status of volume: test-replica4
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick glusterfs-node1:/test-replica1/brick  49155     0          Y       5202 
Brick glusterfs-node1:/test-replica2/brick  49156     0          Y       5222 
Brick glusterfs-node1:/test-replica3/brick  49157     0          Y       5242 
Brick glusterfs-node1:/test-replica4/brick  49158     0          Y       5262 
Self-heal Daemon on localhost               N/A       N/A        Y       5283 
Self-heal Daemon on glusterfs-node2         N/A       N/A        Y       4466 
 
Task Status of Volume test-replica4
------------------------------------------------------------------------------
There are no active volume tasks
[root@glusterfs-node1 brick]# gluster v info test-replica4 
 
Volume Name: test-replica4
Type: Replicate
Volume ID: dd9cc607-8ef9-4c6b-ba63-492fadadd084
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 4 = 4
Transport-type: tcp
Bricks:
Brick1: glusterfs-node1:/test-replica1/brick
Brick2: glusterfs-node1:/test-replica2/brick
Brick3: glusterfs-node1:/test-replica3/brick
Brick4: glusterfs-node1:/test-replica4/brick
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```
可以看到gluster支持多副本(3副本以上)
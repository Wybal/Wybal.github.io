---
title: glusterfs脑裂
date: 2025-04-29 22:27:00 +0800
categories: [分布式存储,glusterfs]
tags: [分布式存储]
pin: true 
toc: true
---

### glusterfs脑裂


>脑裂：

裂脑是指文件的两个或多个复制副本变得不同的情况。当文件处于裂脑状态时，副本brick中文件的数据或元数据不一致，并且没有足够的信息来权威地选择原始副本并修复坏副本，尽管所有brick都已启动并联机。对于目录，还有一个条目拆分大脑，其中的文件可以在副本的brick上具有不同的 gfid/文件类型。发生裂脑主要是因为两个原因：

* 由于网络断开连接，客户端暂时失去与brick的连接。
  
2副本，服务器 1 上的brick 1 和服务器 2 上的brick 2。
 由于网络拆分，客户端 1 失去与brick 2 的连接，客户端 2 失去与brick 1 的连接。
从客户端 1 写入到brick 1，从客户端 2 写入到brick 2

* gluster进程异常或返回错误：
  
服务器 1 关闭，服务器 2 启动：写入发生在服务器 2 上。
服务器 1 启动，服务器 2 关闭（未发生修复/服务器 2 上的数据未在服务器 1 上同步）：写入发生在服务器 1 上。
服务器 2 启动：服务器 1 和服务器 2 都有彼此独立的数据。

**脑裂分为3种**

1. 数据脑裂：文件中的数据在副本组的brick中不同
2. 元数据脑裂：brick中元数据不同
3. entry脑裂：副本brick上的文件GFID不同，或者副本上的文件类型不同，文件类型不同无法修复，GFID可以修复，GFID脑裂对外表现为目录脑裂

glustershd进程负责self-heal，不论有多少birck和volume都只要一个shd进程，当修复时只要上次修复完成才会进行下一次修复
shd进程有两种self-heal crawls,一种是index heal,另外一种是full heal.每个文件在crawling时候，会执行metadata、data、entry的修复。metadata修复文件的属性、权限、mode。data的修复会修复文件内容;entry修复会修复entry里面的目录

index heal触发时，glustershd进程读取每个brick下的.glusterfs/indices/xattrop这个目录下的目录，加锁触发修复。.glusterfs/indices/xattrop目录中包含了以"xattrop-"开头的目录。这个目录下的其他木库是以gfid开头的，这些是需要修复的数据
```
[root@glusterfs-node2 .glusterfs]# gluster v heal replica2 info
Brick glusterfs-node1:/glusterfs/replica/brick1
/nh 
/ - Is in split-brain
/nh/year 
/nh/date 
Status: Connected
Number of entries: 4

Brick glusterfs-node2:/glusterfs/replica/brick1
/nh 
/ - Is in split-brain
/nh/year 
/nh/date 
Status: Connected
Number of entries: 4
此时查看.glusterfs/indices/xattrop目录下的文件，除了xattrop-还有4个文件，find  -samefile \<filename>查看可以发现这4个文件对应的GFID文件就是脑裂的4个文件
[root@glusterfs-node1 xattrop]# ls
00000000-0000-0000-0000-000000000001  08357fb2-bd73-4355-905a-0e53453bcf43  140f76f4-152c-45d7-b1b8-1d09f209bf52  5c6dff7e-27f2-488f-8235-3de6304c6672  xattrop-b96c2cf7-ac1e-45a8-829c-0940bad93bd8
```

index heal的触发条件
1. 每隔600s会触发一次，可以通过gluster volume set \<volumename> cluster.heal-timeout \<value>来设置,默认600s
2. 当客户端执行gluster volume heal \<volumename>会触发
3. 当副本卷所在节点或者glusterfsd宕机后，又再次恢复过来会触发
4. client 访问脑裂文件

常见脑裂修复命令

gluster v heal \<volume\>                  触发器为需要修复的文件索引自修复

gluster v heal \<volume\> info             查看待修复的文件

gluster v heal \<volume\> info split-brain 查看需要处于脑裂状态的文件，需要手动干预进行修复

gluster v heal \<volume\> full             触发所有文件的自我修复。时间较长

gluster v heal \<volume\> statistics       文件修复统计信息

gluster v heal \<volume\> statistics heal-count 需要修复的文件数量

#### 0、glusterfs的写入流程

1. 下发Write操作
2. 文件加锁Lock
3. Pre-op设置文件扩展属性,changelog+1
4. 文件写入
5. Post-op如果写入成功,changelog-1, 如果失败就保持此属性
6. 解锁UnLock
  
扩展属性是由一个24位的十六进制数字构成，被分为三个8位十六进制段

1-8位是数据内容(data)changelog
9-16位是元数据(metadata)changelog
17-24位是目录项(entry)changelog
>文件写入changelog变化示意图

![alt text](/assets/images/image-4.png)

Pre-OP前

![pre-op](/assets/images/image-5.png)

数据写入中

![数据写入中](/assets/images/image-6.png)

写入成功

![alt text](/assets/images/image-7.png)

数据写入Transaction完成后根据扩展属性changelog，文件存在以下几种状态：
- WISE – 自身changelog全零，其他副本changelog非零
- INNOCENT – 自身和其他changelog全为零
- FOOL – 自身changelog非零
- IGRANT – changelog丢失
- 副本都为WISE – 脑裂

如果文件写入后，不同副本changelog都保持WISE状态(副本互相指责对方出现错误)，这种情况即发生了脑裂(Brain Split)

#### 1、gluster cli修复data/metadata split-brain

>data/metadata的split-brain，\<FILE\>可以以指定gfid的方式标识,

\<FILE>\都是相对路径，相对于brick的相对路径，brick目录就是\<FILE>\的/

\<FILE>\可以为目录，当指定目录为源时，后面不能跟/，比如/date,不能写为/date/

```
eg：
gluster volume heal test split-brain source-brick test-host:/test/b1 gfid:c3c94de2-232d-4083-b534-5da17fc476ac
gluster volume heal test split-brain bigger-file /dir/file1
```
###### 选择较大的文件做为源文件

gluster volume heal \<VOLNAME\> split-brain bigger-file \<FILE\>
```shell
eg:gluster v heal replica2 split-brain bigger-file /stbn
No bigger file for file /stbn 如果文件size都一致，会报错
```

###### 选择最新的mtime的文件做为源文件

gluster volume heal \<VOLNAME\> split-brain latest-mtime \<FILE\>
```shell
eg:gluster volume heal replica2 split-brain latest-mtime /stbn
```

###### 选择副本中的一个brick的文件做为源文件

gluster volume heal \<VOLNAME\> split-brain source-brick <HOSTNAME:brick-directory-absolute-path> \<FILE\>
```shell
eg:gluster volume heal replica2 split-brain source-brick glusterfs-node2:/glusterfs/replica/brick1/ /stbn
```

###### 选择副本中的一个brick作为所有文件的源文件

gluster volume heal \<VOLNAME\> split-brain source-brick <HOSTNAME:brick-directory-absolute-path>
```shell
eg:gluster volume heal replica2 split-brain source-brick glusterfs-node2:/glusterfs/replica/brick1/
Healing gfid:00000000-0000-0000-0000-000000000001 failed:Operation not permitted.如果有GFID split-brain会报错不允许的操作
```

#### 2、gluster cli修复entry/GFID split-brain

使用getfattr -d -e hex -m. \<path-to-file>查看各brick下的文件权限trusted.gfid是否一致，不一致就是GFID脑裂

###### 选择较大的文件作为源

gluster volume heal \<VOLNAME\> split-brain bigger-file \<FILE\>

###### 选择最新的mtime的文件作为源

gluster volume heal \<VOLNAME\> split-brain latest-mtime \<FILE\>

###### 选择副本中的一个brick的文件做为源文件

gluster volume heal \<VOLNAME\> split-brain source-brick <HOSTNAME:brick-directory-absolute-path> \<FILE\>

> 注意：

-	data/metadata的split-brain，\<FILE\>可以为gfid。当entry的split-brain，不能指定gfid的方式标识，因为此时gfid各副本不同。
- \<FILE>\都是相对路径，相对于brick的相对路径，brick目录就是\<FILE>\的/。\<FILE>\可以为目录，当指定目录为源时，后面不能跟/，比如/date,不能写为/date/
-	无法一次性解决所有 GFID 裂脑，即无法选择副本中的一个brick作为所有文件的源文件来解决entry脑裂
-	使用带有“分布式复制”卷中的“brick块”选项的 CLI 解析目录 GFID 裂脑需要在处于此状态的所有子卷上显式完成。由于目录会在所有brick上创建，因此使用一个特定的brick作为目录 GFID 裂脑的源可以修复该特定子卷的目录。源brick的选择方式应使修复后所有子卷的所有brick都具有相同的 GFID。
-	如前所述，无法使用CLI解决文件系统类型不匹配的问题

#### 3、快速修复

###### 1、获取裂脑中文件的路径：

它可以通过
a） 命令获得。gluster volume heal info split-brain
b） 确定从客户端执行的文件操作不断失败并出现输入/输出错误的文件。

###### 2、关闭从装入点打开此文件的应用程序。 对于虚拟机，需要关闭它们的电源。

###### 3、确定正确的副本

这是通过观察文件的 afr 更改日志扩展属性来完成的 使用 getfattr 命令的brick;然后确定裂脑的类型 （数据裂脑、元数据裂脑、条目裂脑或裂脑由于 GFID-不匹配）;最后确定哪个brick包含“好副本” 的文件。
也可能一个砖可能包含正确的数据，而 其他可能包含正确的元数据。getfattr -d -m . -e hex \<file-path-on-brick\>

    0x 000003d7 00000001 00000000
            |      |       |
            |      |        \_ changelog of directory entries
            |       \_ changelog of metadata
            \ _ changelog of data

全为0认为是对的，不为0认为有问题，前8位为data，中间8位位metadata，后8位为directory entries

###### 4、重置包含 使用 setfattr 命令的文件数据/元数据的“错误副本”。

setfattr -n \<attribute-name\> -v \<attribute-value\> \<file-path-on-brick\>

###### 5、通过从客户端执行查找来触发对文件的自我修复：

ls -l \<file-path-on-gluster-mount\>

**example:**

###### 1）查看xattr权限

```shell
[root@pranithk-laptop vol]# getfattr -d -m . -e hex /gfs/brick-?/a
getfattr: Removing leading '/' from absolute path names
\#file: gfs/brick-a/a
trusted.afr.vol-client-0=0x000000000000000000000000     -->/gfs/brick-a/a 上的更新日志认为着某些数据和元数据操作成功了本身
trusted.afr.vol-client-1=0x000003d70000000100000000     -->/gfs/brick-a/a 上的更新日志认为着某些数据和元数据操作在/gfs/brick-a/b 上失败
trusted.gfid=0x80acdbd886524f6fbefa21fc356fed57
\#file: gfs/brick-b/a
trusted.afr.vol-client-0=0x000003b00000000100000000     -->/gfs/brick-b/a 上的更新日志认为着某些数据和元数据操作成功了本身
trusted.afr.vol-client-1=0x000000000000000000000000     -->/gfs/brick-b/a 上的更新日志认为着某些数据和元数据操作在/gfs/brick-a/a 上失败
trusted.gfid=0x80acdbd886524f6fbefa21fc356fed57
```
如果互相指责，就需要人为判断指定一个副本为正确的，并修改其他副本认为正确副本的trusted.afr.vol-client-x<即data/metadata/entries的changelog>为0
如果是3副本，根据xattr判断认为changlelog是对的最多的brick为正确的，修改其他节点认为正确副本的trusted.afr.vol-client-x<即data/metadata/entries的changelog>为0

###### 2）确认正确副本

此处认为gfs/brick-b/a的metadta为正确的，gfs/brick-a/a的data为正确的

###### 3）修改changelog

gfs/brick-a/a:

    修改trusted.afr.vol-client-1的0x000003d70000000100000000为0x000003d70000000000000000 
    /gfs/brick-a/a认为/gfs/brick-b/a的metadata是正确的，将中间8位修改为0

设置gfs/brick-a/a的xattr权限如下:

```shell
setfattr -n trusted.afr.vol-client-1 -v 0x000003d70000000000000000 /gfs/brick-a/a
```

gfs/brick-b/a:

    修改trusted.afr.vol-client-0的0x000003b00000000100000000为0x000000000000000100000000 
    /gfs/brick-b/a认为/gfs/brick-a/a的data是正确的，将前8位修改为0

设置gfs/brick-b/a的xattr权限如下:

```shell
setfattr -n trusted.afr.vol-client-0 -v 0x000000000000000100000000 /gfs/brick-b/a
```

完成上述操作后，查看xattr权限如下所示：

```shell
[root@pranithk-laptop vol]# getfattr -d -m . -e hex /gfs/brick-?/a
getfattr: Removing leading '/' from absolute path names
\#file: gfs/brick-a/a
trusted.afr.vol-client-0=0x000000000000000000000000
trusted.afr.vol-client-1=0x000003d70000000000000000
trusted.gfid=0x80acdbd886524f6fbefa21fc356fed57

\#file: gfs/brick-b/a
trusted.afr.vol-client-0=0x000000000000000100000000
trusted.afr.vol-client-1=0x000000000000000000000000
trusted.gfid=0x80acdbd886524f6fbefa21fc356fed57
```

###### 4）触发自我修复：

client执行以触发愈合。
`ls -l <file-path-on-gluster-mount>`或server端执行`gluster volume heal VOLNAME`

#### 4、设置自动修复脑裂[ctime|mtime|size|majority]

基于CLI和client的方法需要人工干预，有一个卷设置，当设置为各种可用策略之一时，无需用户干预即可自动恢复脑裂，默认被禁用。
设置以后直接生效，并开始
`cluster.favorite-child-policy`
查看help

```
[root@glusterfs-node2 brick]# gluster v set help  | grep -A3 cluster.favorite-child-policy
Option: cluster.favorite-child-policy
Default Value: none
Description: This option can be used to automatically resolve split-brains using various policies without user intervention.
 "size" picks the file with the biggest size as the source. "ctime" and "mtime" pick the file with the latest ctime and mtime respectively as the source. "majority" picks a file with identical mtime and size in more than half the number of bricks in the replica.
设置glusterfs以最新mtime自动修复脑裂
[root@glusterfs-node2 ~]# gluster v set replica2 cluster.favorite-child-policy mtime
volume set: success
[root@dockernode1 /]# gluster v info
Options Reconfigured:		
cluster.favorite-child-policy: mtime
...
```

#### 5、脑裂处理示例

**环境**

| 主机名           |     Ip地址      |     盘符 |
| ---------------- | :-------------: | -------: |
| glusterfs-node1  | 192.168.189.131 | /dev/sdb |
| glusterfs-node2  | 192.168.189.132 | /dev/sdb |
| glusterfs-client | 192.168.189.150 |

#### example1，使用latest-mtime进行脑裂修复

##### 1、手动制造脑裂

###### 1）修改quorum-type和quorum-count

将cluster.quorum-type改为fixed，cluster.quorum-count改为1
[root@glusterfs-node2 ~]# gluster v get replica2 cluster.quorum-type
Option                              Value                                                                     
cluster.quorum-type                     fixed
[root@glusterfs-node2 ~]# gluster v get replica2 cluster.quorum-count
Option          Value                                                                     
cluster.quorum-count                    1

###### 2）停止node1上的所有gluster相关进程

[root@glusterfs-node1 ~]# ps -ef | awk '/gluster/{print $2}' | xargs kill

###### 3）在client端写入文件

[root@gluster-client replica]# mkdir -p stbn/op
[root@gluster-client op]# echo "1900" >>year
[root@gluster-client op]# date >> DateFile
[root@gluster-client replica]# chmod 777 stbn/

###### 4）停止node2上的所有gluster相关进程

[root@glusterfs-node2 ~]# ps -ef | awk '/gluster/{print $2}' | xargs kill

###### 5）启动node1的glusterd服务，并在client端写入文件

[root@glusterfs-node1 ~]# systemctl start glusterd
[root@gluster-client replica]# mkdir -p stbn/op
[root@gluster-client op]# echo "2000" >>year
[root@gluster-client op]# date >> DateFile
[root@glusterfs-node2 ~]# systemctl start glusterd

###### 6）再启动node2的glusterd服务，此时就会脑裂

```shell
[root@glusterfs-node1 ~]# gluster v heal replica2 info
Brick glusterfs-node1:/glusterfs/replica/brick1
/stbn 
/ - Is in split-brain
/stbn/op 
/stbn/op/year 
/stbn/op/date 
Status: Connected
Number of entries: 5

Brick glusterfs-node2:/glusterfs/replica/brick1
/stbn 
/ - Is in split-brain
/stbn/op 
/stbn/op/year 
/stbn/op/date 
Status: Connected
Number of entries: 5

[root@glusterfs-node1 ~]# gluster v heal replica2 info split-brain
Brick glusterfs-node1:/glusterfs/replica/brick1
/
Status: Connected
Number of entries in split-brain: 1

Brick glusterfs-node2:/glusterfs/replica/brick1
/
Status: Connected
Number of entries in split-brain: 1
```

##### 2、开始修复脑裂

/ - Is in split-brain处于脑裂状态，一般目录脑裂为GFID脑裂的对外表现，查看各个brick的GFID是否一致

###### 1）node1上brick的xattr权限

```shell
[root@glusterfs-node1 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1
getfattr: Removing leading '/' from absolute path names
...
trusted.afr.replica2-client-1=0x000000000000000000000001
trusted.gfid=0x00000000000000000000000000000001             --->brick的/目录GFID相同
trusted.glusterfs.mdata=0x0100000000000000000000000065241b230000000003d07ba40000000065241b230000000003d07ba400000000000000000000000000000000
...

[root@glusterfs-node1 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn
...
trusted.afr.replica2-client-1=0x000000000000000100000002
trusted.gfid=0xfc9cb1976b11485f9b7104bfdd4c3ed7             --->brick的/stbn目录GFID不相同
trusted.glusterfs.mdata=0x0100000000000000000000000065241b230000000004085d520000000065241b230000000004085d520000000065241b230000000003d07ba4
...

[root@glusterfs-node1 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn/op
...
trusted.afr.replica2-client-1=0x000000000000000100000003
trusted.gfid=0x721ff242da0c49d787cc3c41b1f9c67e             --->brick的/stbn/op目录GFID不相同
trusted.glusterfs.mdata=0x0100000000000000000000000065241b2d000000003983f19f0000000065241b2d000000003983f19f0000000065241b230000000004085d52
...

[root@glusterfs-node1 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn/op/year
...
trusted.afr.replica2-client-1=0x000000020000000100000000
trusted.gfid=0x59581e2aeb0c42c98776f0dd906832da             --->brick的/stbn/op/year目录GFID不相同
trusted.gfid2path.ddb678ce11c8979b=0x37323166663234322d646130632d343964372d383763632d33633431623166396
33637652f79656172trusted.glusterfs.mdata=0x0100000000000000000000000065241b280000000013aeb43e0000000065241b280000000013
...

[root@glusterfs-node1 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn/op/date
...
trusted.afr.replica2-client-1=0x000000020000000100000000
trusted.gfid=0x3698bee4f8304ab09f6b5d6754759e6b             --->brick的/stbn/op/date目录GFID不相同
trusted.gfid2path.c17327bebd856a92=0x37323166663234322d646130632d343964372d383763632d33633431623166396
33637652f64617465trusted.glusterfs.mdata=0x0100000000000000000000000065241b2d0000000039afa1970000000065241b2d0000000039
...
```

###### 2）node2上brick的xattr权限

```shell
[root@glusterfs-node2 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1
...
trusted.afr.replica2-client-0=0x000000000000000000000001
trusted.gfid=0x00000000000000000000000000000001
trusted.glusterfs.mdata=0x0100000000000000000000000065241af60000000036efb7b70000000065241af60000000036efb7b700000000000000000000000000000000
...

[root@glusterfs-node2 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn
...
trusted.afr.replica2-client-0=0x000000000000000200000002
trusted.gfid=0xe2624aa3831c44c8a4e737eb53189795
trusted.glusterfs.mdata=0x0100000000000000000000000065241b1000000000233260110000000065241af6000000003729a2af0000000065241af60000000036efb7b7
...

[root@glusterfs-node2 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn/op
...
trusted.afr.replica2-client-0=0x000000000000000100000003
trusted.gfid=0x74b1aded1d514ff7acbef5556de1835c
trusted.glusterfs.mdata=0x0100000000000000000000000065241b0a0000000016c5ab720000000065241b0a0000000016c5ab720000000065241af6000000003729a2
...

[root@glusterfs-node2 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn/op/year
...
trusted.afr.replica2-client-0=0x000000020000000100000000
trusted.gfid=0x9419682c6ccb49c1a7d4b026f2d3633c
trusted.gfid2path.155e95a8e816ed12=0x37346231616465642d316435312d346666372d616362652d66353535366465313
83335632f79656172
trusted.glusterfs.mdata=0x0100000000000000000000000065241b05000000001bb5e47f0000000065241b05000000001bb5e47f0000000065241b05000000001b933aaa
...

[root@glusterfs-node2 ~]# getfattr -d -m. -e hex /glusterfs/replica/brick1/stbn/op/date
...
trusted.afr.replica2-client-0=0x000000020000000100000000
trusted.gfid=0xb75a8e99161849c79f166f5de3b29c37
trusted.gfid2path.93dee3f3fcd904a5=0x37346231616465642d316435312d346666372d616362652d6635353536646531383335632f64617465
trusted.glusterfs.mdata=0x0100000000000000000000000065241b0a000000001723633c0000000065241b0a000000001723633c0000000065241b0a0000000016c5ab72
...
```

###### 3）修复split-brain

当多级目录都发生split-brain时，需要首先修复父目录，如果父目录脑裂时子目录无法修复，修复时的/不是真正的/目录，brick的绝对路径为相对/目录
```
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /stbn/op/
Lookup failed on /stbn:Input/output error
Volume heal failed.
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /stbn/op
Lookup failed on /stbn:Input/output error
Volume heal failed.
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /stbn/op/year
Lookup failed on /stbn/op:Input/output error
Volume heal failed.
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /stbn/op/date
Lookup failed on /stbn/op:Input/output error
Volume heal failed.
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /stbn/
Lookup failed on /stbn/:Invalid argument.
Volume heal failed.
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /
Lookup failed on /:Invalid argument.
Volume heal failed.
------------------------------------------------------------------
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /stbn
GFID split-brain resolved for file /stbn
```

###### 4）修复完成查看文件和权限

修复以后执行gluster v heal \<volume\>手动触发self-heal
```
[root@glusterfs-node2 ~]# gluster v heal replica2 info 
Brick glusterfs-node1:/glusterfs/replica/brick1
Status: Connected
Number of entries: 0

Brick glusterfs-node2:/glusterfs/replica/brick1
Status: Connected
Number of entries: 0
```
查看文件内容和目录权限都是以最新mtime为准
```shell
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/stbn/op/year 
2000
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/stbn/op/date 
Mon Oct  9 23:24:29 CST 2023
[root@glusterfs-node1 ~]# ll -ad /glusterfs/replica/brick1/stbn
drwxr-xr-x 3 root root 16 Oct  9 21:49 /glusterfs/replica/brick1/stbn
```

#### example2，使用setfattr修复split-brain

##### 1、脑裂情况

```shell
[root@glusterfs-node1 ~]# gluster v heal replica2 info
Brick glusterfs-node1:/glusterfs/replica/brick1
/o 
/ - Is in split-brain
/o/p 
/l 
/l/p 
/l/p/date 
/o/p/date 
Status: Connected
Number of entries: 7

Brick glusterfs-node2:/glusterfs/replica/brick1
/l 
/ - Is in split-brain
/l/p 
/o 
/o/p 
/o/p/date 
/l/p/date 
Status: Connected
Number of entries: 7
```
node1
```shell
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/o/p/date 
Sun Oct  8 16:53:58 CST 2023
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/l/p/date 
Sun Oct  8 16:53:55 CST 2023
```
node2
```shell
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/o/p/date 
Sun Oct  8 16:53:17 CST 2023
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/l/p/date 
Sun Oct  8 16:53:21 CST 2023
```

##### 2、开始修复脑裂

###### 1）修复GFID脑裂，以node1为源，修改node2的trusted.afr.\<volume\>-client-x全为0

```
[root@glusterfs-node2 ~]# setfattr -n trusted.afr.replica2-client-0 -v 0x000000000000000000000000 /glusterfs/replica/brick1/
```

###### 2）修复以后执行gluster v heal \<volume\>手动触发self-heal

```shell
[root@glusterfs-node2 ~]# gluster v heal replica2 info 
Brick glusterfs-node1:/glusterfs/replica/brick1
Status: Connected
Number of entries: 0

Brick glusterfs-node2:/glusterfs/replica/brick1
Status: Connected
Number of entries: 0
```
node2的文件内容被修改为node1的文件内容
```shell
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/o/p/date 
Sun Oct  8 16:53:58 CST 2023
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/l/p/date 
Sun Oct  8 16:53:55 CST 2023

```

#### example3, 每个brick下都有最新的mtime文件，使用latest-mtime修复

>这种情况下会统一按照指定目录的latest-mtime去修复，不会检查目录下的文件的mtime是否为最新，即使目录的mtime是最新但下面的文件的mtime不是最新的，也会统一按照这个目录为源对其他brick进行内容替换，所以使用latest-mtime进行修复时，直接指定目录，可能会导致修复后的文件不是最新内容，导致丢失数据
cluster.favorite-child-policy为mtime也会有这样的问题

##### 1、手动制造脑裂

略
查看脑裂后的文件和目录的mtime和内容，可以看到opt/目录的latest-mtime为node2，opt/year的latest-mtime为node1，opt/date的latest-mtime为node2
```shell
[root@glusterfs-node1 ~]# stat /glusterfs/replica/brick1/opt/
  File: ‘/glusterfs/replica/brick1/opt/’
  Size: 30        	Blocks: 8          IO Block: 4096   directory
Device: 810h/2064d	Inode: 12583224    Links: 2
Access: (0755/drwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-10-09 22:35:45.381886704 +0800
Modify: 2023-10-09 22:35:05.754329017 +0800
Change: 2023-10-09 22:35:05.759745577 +0800
 Birth: -
[root@glusterfs-node1 ~]# stat /glusterfs/replica/brick1/opt/year 
  File: ‘/glusterfs/replica/brick1/opt/year’
  Size: 5         	Blocks: 16         IO Block: 4096   regular file
Device: 810h/2064d	Inode: 12583226    Links: 2
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-10-09 22:34:57.452908171 +0800
Modify: 2023-10-09 22:34:57.457241419 +0800
Change: 2023-10-09 22:34:57.458324731 +0800
 Birth: -
[root@glusterfs-node1 ~]# stat /glusterfs/replica/brick1/opt/date 
  File: ‘/glusterfs/replica/brick1/opt/date’
  Size: 29        	Blocks: 16         IO Block: 4096   regular file
Device: 810h/2064d	Inode: 12583229    Links: 2
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-10-09 22:35:05.754329017 +0800
Modify: 2023-10-09 22:35:05.759745577 +0800
Change: 2023-10-09 22:35:05.759745577 +0800
 Birth: -
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/opt/year 
2023
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/opt/date 
Tue Oct 10 00:16:01 CST 2023

```
```shell
[root@glusterfs-node2 ~]# stat /glusterfs/replica/brick1/opt/
  File: ‘/glusterfs/replica/brick1/opt/’
  Size: 30        	Blocks: 8          IO Block: 4096   directory
Device: 810h/2064d	Inode: 12582976    Links: 2
Access: (0755/drwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-10-09 22:35:02.263576466 +0800
Modify: 2023-10-09 22:35:07.557723813 +0800
Change: 2023-10-09 22:35:07.558807125 +0800
 Birth: -
[root@glusterfs-node2 ~]# stat /glusterfs/replica/brick1/opt/year 
  File: ‘/glusterfs/replica/brick1/opt/year’
  Size: 5         	Blocks: 16         IO Block: 4096   regular file
Device: 810h/2064d	Inode: 12582977    Links: 2
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-10-09 22:34:52.311186108 +0800
Modify: 2023-10-09 22:33:48.400055064 +0800
Change: 2023-10-09 22:33:48.400055064 +0800
 Birth: -
[root@glusterfs-node2 ~]# stat /glusterfs/replica/brick1/opt/date 
  File: ‘/glusterfs/replica/brick1/opt/date’
  Size: 29        	Blocks: 16         IO Block: 4096   regular file
Device: 810h/2064d	Inode: 12582981    Links: 2
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-10-09 22:35:07.440726081 +0800
Modify: 2023-10-09 22:35:07.445059330 +0800
Change: 2023-10-09 22:35:07.448309267 +0800
 Birth: -
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/opt/year 
1900
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/opt/date
Tue Oct 10 00:16:27 CST 2024 
```

##### 2、修复脑裂

>以latest-mtime的opt/目录进行修复，可以发现，
opt/目录和date文件是以latest-mtime进行修复的，
但是year文件不是以latest-mtime进行修复的，而是以latest-mtime的opt/下的year文件进行修复的
所以指定latest-mtime指定目录时，
```shell
[root@glusterfs-node1 ~]# gluster v heal replica2 split-brain latest-mtime /opt
GFID split-brain resolved for file /opt
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/opt/year 
1900
[root@glusterfs-node1 ~]# cat /glusterfs/replica/brick1/opt/date 
Tue Oct 10 00:16:27 CST 2024
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/nh/date 
Mon Oct  9 23:52:17 CST 2024
[root@glusterfs-node2 ~]# cat /glusterfs/replica/brick1/nh/year 
1900
```

#### example4, 从挂载点（client）修复脑裂

提供了一组 getfattr 和 setfattr 命令来检测文件的数据和元数据裂脑状态，并从挂载点解析裂脑（如果有）。
```
# gluster v heal test info split-brain
Brick test-host:/test/b0/
/file100
/dir
Number of entries in split-brain: 2

Brick test-host:/test/b1/
/file100
/dir
Number of entries in split-brain: 2

Brick test-host:/test/b2/
/file99
<gfid:5399a8d1-aee9-4653-bb7f-606df02b3696>
Number of entries in split-brain: 2

Brick test-host:/test/b3/
<gfid:05c4b283-af58-48ed-999e-4d706c7b97d5>
<gfid:5399a8d1-aee9-4653-bb7f-606df02b3696>
Number of entries in split-brain: 2
```
使用如下命令了解处于哪种脑裂状态
```
getfattr -n replica.split-brain-status <path-to-file>
```
eg:
处于元数据脑裂
```
# getfattr -n replica.split-brain-status file100
file: file100
replica.split-brain-status="data-split-brain:no    metadata-split-brain:yes    Choices:test-client-0,test-client-1"
```
处于数据脑裂
```
# getfattr -n replica.split-brain-status file99
file: file99
replica.split-brain-status="data-split-brain:yes    metadata-split-brain:yes    Choices:test-client-2,test-client-3"
```
元数据和数据都处于脑裂中
```
# getfattr -n replica.split-brain-status file99
file: file99
replica.split-brain-status="data-split-brain:yes    metadata-split-brain:yes    Choices:test-client-2,test-client-3"
```
dir不在数据或者元数据脑裂下，
```
# getfattr -n replica.split-brain-status dir
file: dir
replica.split-brain-status="The file is not under data or metadata split-brain"
```

##### 1、从client解决数据和元数据脑裂

尝试在挂载点上对脑裂的文件进行操作（例如 cat、getfattr 等），会出现输入/输出错误。为了使用户能够分析此类文件，提供了一个 setfattr 命令。
```
# setfattr -n replica.split-brain-choice -v "choiceX" <path-to-file>
```
使用此命令，可以选择一个特定的块来访问裂脑中的文件。
eg：
1、“file1”位于数据脑裂中。尝试从文件中读取会产生输入/输出错误。
```
# cat file1
cat: file1: Input/output error
```
为file1提供的服务端选择有test-cilent-1和test-client-2
将test-client-2设置为file1的脑裂选择源，将从test-client-2的brick中读取文件
```
# setfattr -n replica.split-brain-choice -v test-client-2 file1
```
然后再cat文件
```
cat file1
xyz123
```
要撤销已设置的脑裂选择，可以使用“none”作为setfattr的扩展属性的值
```
# setfattr -n replica.split-brain-choice -v none file1
```
再cat文件，就会和以前一样，输出Input/output error
```
# cat file
cat: file1: Input/output error
```
如果想解决脑裂问题，应该设置源brick
```
# setfattr -n replica.split-brain-heal-finalize -v <heal-choice> <path-to-file>
eg：
# setfattr -n replica.split-brain-heal-finalize -v test-client-2 file1
```
上述过程可以解决所有文件的数据/元数据脑裂
>注意：
1、如果禁用了“fopen-keep-cache”保险丝装载选项，则每次在选择新副本之前都需要使 inode 失效。拆分大脑选择检查文件。这可以通过使用来完成：
`# sefattr -n inode-invalidate -v 0 <path-to-file>`
2、上面从client修复脑裂的过程不适合nfs挂载，因为nfs不提供xattrs支持


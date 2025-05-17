---
title: journalctl
date: 2024-03-05 22:34:00 +0800
categories: [linux,系统运维,journalctl]
tags: [linux,journalctl]
[toc]
---

## journalctl详解


### 1、什么是journalctl详解
**journald是systemd的守护进程，它收集来自系统、内核和服务等不同来源的日志，并以二进制格式存储，以便于操作。
systemd从系统、内核、各种服务或守护进程等多个来源收集日志，并通过journald提供一个集中管理的解决方案。
这是一个高度精简的过程，可以根据需求查看日志，而syslogd的日志则是通过各种命令，如find、grep、cut等手动分析。**
### 2、日志持久化
journalctl日志默认存储在`/run/log/journal/`中，默认重启被删除
按如下步骤持久化以后，会自动创建`/var/log/journal/`目录，将二进制日志存储在该目录下
以root用户编辑`/etc/systemd/journald.conf `文件
```
sudo sed -i '/Storage/ c\Storage=persistent' /etc/systemd/journald.conf
```
重启 systemd-journald，如下所示。
```
$ sudo systemctl restart systemd-journald
```
更改文件权限，如下图所示。
```
sudo chown -R root:systemd-journal /var/log/journal
```
### 3、常用flag
  * -f : 只显示最新的日志和实时日志信息。

  * -e : 跳到日志的结尾，显示最新事件。

  * -r : 按时间倒序打印日志信息。

  * -k : 只显示内核的信息。

  * -u : 只显示指定系统单元的消息。

  * -b : 显示特定的启动信息，如果不包括特定的启动会话，则显示当前的启动信息。

  * -list-boots : 以表格形式显示启动会话，包括它们的 ID，以及与启动相关的第一条和最后一条消息的时间戳。

  * -utc : 以协调世界时（UTC）表示时间。

  * -p, -priority= : 按消息的优先级过滤输出。

  * -S, -since= : 根据启动时间过滤日志。

  * -U, -until= : 根据结束时间过滤日志。

  * -disk-usage：显示当前所有日志文件的磁盘使用情况。
### 4、读取日志
##### 1）显示日志的全部内容
```
$ sudo journalctl
```
##### 2）显示日志的详细内容
```
$ sudo journalctl -o verbose
```
##### 3）倒序显示日志的全部内容，先显示最近的日志
```
$ sudo journalctl -r
```
##### 4）只显示最近的`n`行日志，默认为10行
```
$ sudo journalctl -n 20
```
##### 5）实时显示日志，类似`tail -f`
```
$ sudo journalctl -f
```
### 5、过滤日志
##### 1）只显示内核日志
```
$ sudo journalctl -k
```
或者
```
$ sudo journalctl _TRANSPORT=kernel
```
##### 2）过滤查看指定的启动会话日志，0为最新
```
$ sudo journalctl --list-boots
-1 da83e3310bec400784d945306ecf234b Wed 2023-06-28 08:41:49 CST—Wed 2023-06-28 10:44:56 CST
 0 e2c85c77f9ae4b55ac4bf280a3f84059 Wed 2023-06-28 10:45:04 CST—Wed 2023-06-28 10:45:52 CST
 ```

###### a) 过滤查看指定的启动会话日志
   
 ```
$ sudo journalctl -b -1
 ```
  或
 ```
sudo journalctl -b e2c85c77f9ae4b55ac4bf280a3f84059
 ```
###### b)  过滤查看当前启动的日志
 ```
$ sudo journalctl -b
 ```
##### 3）过滤指定时间的日志
日志日志可以根据时间间隔进行过滤。时间过滤器可以使用多个参数，如下所示。要使用时间过滤器，请使用`-S或-since`和`-U或-until`命令行开关。

要过滤从昨天开始的日志，运行以下命令。
 ```
 $ sudo journalctl -S yesterday
 ```
要只过滤今天的日志，运行下面的命令（我们可以使用'today'或'00:00'，两者相同）。
 ```
$ sudo journalctl -S today
 ```
或
 ```
$ sudo journalctl -S 00:00
 ```
如果只过滤昨天的日志，请执行以下命令。
 ```
$ sudo journalctl --since yesterday --until 00:00
 ```
要过滤3月12日以来的日志，请执行以下命令。
 ```
$ sudo journalctl -S 2021-03-12
 ```
要用日期和时间过滤日志，请运行以下命令。

注意，"日期和时间 "使用以下格式，"年-月-日 "和 "小时:分钟:秒"。
 ```
$ sudo journalctl --since "2021-03-11 20:10:00" --until "2021-03-15"
 ```
要过滤最近一小时内的消息，请使用以下命令。
 ```
$ sudo journalctl -S -1h
 ```
##### 4）按日志级别过滤
`PRIORITY=6和-p 6`选择对应的过滤日志级别
可以通过-F选项来查看某个字段的可选值：
 ```
[root@node1 ~]# journalctl -F PRIORITY
4
5
3
6
2
7
 ```
可以指定的优先级如下：
1. emerg
2. alert
3. err
4. warning
5. notice
6. info
7. debug
   
要过滤内核的`err`日志
 ```
$ sudo journalctl -p 3 -k
 ```
或者
 ```
$ sudo journalctl -p err -k
 ```
##### 5）基于字段的过滤
日志日志可以通过特定字段进行过滤。需要匹配的字段的语法是`FIELD_NAME=MATCHED_VALUE`，如`_SYSTEMD_UNIT=network.service`。同时，您也可以在一个查询中指定多个匹配项，以更方便地过滤输出信息。
常见的匹配字段：`MESSAGE`、`MESSAGE_ID`、`_PID`、`_UID`、`_HOSTNAME`、`_SYSTEMD_UNIT`
###### a) 按服务过滤
要显示指定服务产生的消息，使用下面的命令。同样，你也可以过滤任何服务信息。要查看可用的服务日志，输入 "journalctl -u"，然后按TAB键两次。
 ```
$ sudo journalctl -u httpd
 ```
 或
 ```
$ sudo journalctl -u httpd.service
 ```
或
 ```
$ sudo journalctl _SYSTEMD_UNIT=httpd.service
 ```
###### b) 根据UID、GID和PID过滤日志

要显示特定进程ID产生的消息，运行以下命令。
 ```
$ sudo journalctl _PID=1039
 ```
要显示特定用户ID产生的消息，运行以下命令。
 ```
$ sudo journalctl _UID=1021
 ```
要显示特定组ID产生的消息，运行以下命令。
 ```
$ sudo journalctl _GID=1050
 ```
###### c) 按文件路径过滤

可以根据运行进程的文件路径进行过滤，如下图所示。
 ```
$ sudo journalctl /usr/bin/gnome-shell
 ```
###### d) 按设备路径过滤

要过滤与特定设备相关的消息，请运行以下命令：。
 ```
$ sudo journalctl /dev/sda
 ```
  
##### 6)过滤器可以同时应用于多个字段
###### a)只显示`httpd`服务且`pid=1500`的日志。
 ```
$ sudo journalctl -u httpd  _PID=1500
 ```
###### b)显示`Apache`服务进程的所有日志和`MySQL`服务的所有日志。
 ```
$ journalctl -u nginx.service -u php-fpm.service
 ```
 
###### c)实际的使用中更常见的用例是同时应用 match 和时间条件，比如要过滤出`2018年3月27日0点到1点`这个时间段中 `cron` 服务的日志记录：
 ```
$ sudo journalctl -u cron --since "2018-03-27" --until "2018-03-27 01:00"
 ```
### 5、检查所有日志文件的磁盘使用情况
当你为日志启用持久化存储时，它最多使用`/var/log/journal`所在文件系统的`10%`。

要查看日志文件使用了多少存储空间，请运行以下命令。
 ```
$ sudo journalctl --disk-usage
Archived and active journals take up 424.0M on disk.
```
### 6、控制输出
把结果重定向到标准输出

默认情况下，`journalctl` 会在 `pager` 内显示输出结果。如果大家希望利用文本操作工具对数据进行处理，则需要使用标准输出。在这种情况下，我们需要使用 `--no-pager` 选项。
```
$ sudo journalctl --no-pager
```
这样就可以把结果重定向到我们需要的地方(一般是磁盘文件或者是文本工具)。

格式化输出的结果

如果大家需要对日志记录进行处理，可能需要使用更易使用的格式以简化数据解析工作。幸运的是，`journalctl` 能够以多种格式进行显示，只须添加 -o 选项即可。-o 选项支持的类型如下：

`short`

这是默认的格式，即经典的 syslog 输出格式。
`
short-iso
`
与 short 类似，强调 ISO 8601 时间戳。
`
short-precise
`
与 short 类似，提供微秒级精度。
`
short-monotonic
`
与 short 类似，强调普通时间戳。
`
verbose
`
显示全部字段，包括通常被内部隐藏的字段。
`
export
`
适合传输或备份的二进制格式。
`
json
`
标准 json 格式，每行一条记录。
`
json-pretty
`
适合阅读的 json 格式。
`
json-sse
`
经过包装可以兼容 server-sent 事件的 json 格式。
`
cat
`
只显示信息字段本身。

比如我们要以 json 格式输出 `cron.service `的最后一条日志：
```
$ sudo journalctl -u cron.service -n 1 --no-pager -o json
```

---
title: logrotate
date: 2025-05-07 22:34:00 +0800
categories: [linux,系统运维,logrotate]
tags: [linux,logrotate]
---

logrotate是linux自带的管理日志
主要有两个配置文件/etc/logrotate.conf、/etc/logrotate.d/下的文件

```
[root@localhost xinetd.d]# cat /etc/logrotate.conf 
# see "man logrotate" for details
# rotate log files weekly每周转储log文件一次
weekly

# keep 4 weeks worth of backlogs转储4次
rotate 4

# create new (empty) log files after rotating old ones转储老文件后，创建一个新的文件
Create

#use date as a suffix of the rotated file rotate的文件以日期格式为后缀，比如：#access_log-20200422，如果不加这个选项，rotate的格式为：access_log.1，#access_log.2#等等。

dateext

# uncomment this if you want your log files compressed如果想压缩rotate后的文件，把下面compress前面的#号去掉就可以了。  

#compress

# RPM packages drop log rotation information into this directory •  RPM包的日志rotation配置信息，建议放到/etc/logrotate.d这个文件夹下，实现自定义控制log文件rotate   

include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly			#每月执行一次转储
    create 0664 root utmp		#创建空文件，权限是664，属主属组
	minsize 1M		#日志大小超过1M才转储，否则跳过
    rotate 1		#rotate时，只保留一份rotate文件
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
```

```
[root@master logrotate.d]# cat /etc/logrotate.d/syslog 
/var/log/cron
/var/log/maillog
/var/log/messages
/var/log/secure
/var/log/spooler
{
    missingok
    sharedscripts
    postrotate
	/bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
```
**system-specific logs may be also be configured here.**
```
compress	                通过gzip 压缩转储以后的日志
nocompress	                不需要压缩时，用这个参数
missingok	                如果日志不存在，不报错继续滚动下一个日志
rotate <count>	            指定日志文件删除之前转储的个数，0 指没有备份，5 指保留5个备份
daily	                    指定转储周期为每天
weekly	                    指定转储周期为每周
monthly	                    指定转储周期为每月
dateext	                    使用当期日期作为命名格式
size(或minsize) log-size	当日志文件到达指定的大小时才转储，Size 可以指定 bytes (缺省)以及KB (sizek)或者MB (sizem).log-size能指定bytes(缺省)及KB (sizek)或MB(sizem).
copytruncate	            用于还在打开中的日志文件，把当前日志备份并截断；是先拷贝再清空的方式，拷贝和清空之间有一个时间差，可能会丢失部分日志数据。
nocopytruncate	            备份日志文件但是不截断
create mode owner group 	转储文件，使用指定的文件模式创建新的日志文件。轮转时指定创建新文件的属性，如create 0777 nobody nobody
nocreate	                不建立新的日志文件
delaycompress	            必须和 compress 一起使用，转储的日志文件到下一次转储时才压缩
nodelaycompress	            覆盖 delaycompress 选项，转储同时压缩
ifempty	                    即使是空文件也转储，这个是 logrotate 的缺省选项。
notifempty	                如果是空文件的话，不转储
errors address 	            专储时的错误信息发送到指定的Email 地址
mail address 	            把转储的日志文件发送到指定的E-mail 地址
nomail	                    转储时不发送日志文件
olddir directory	        转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统
noolddir	                转储后的日志文件和当前日志文件放在同一个目录下
prerotate/endscript	        在logrotate转储之前需要执行的指令，例如修改文件的属性等动作；这两个关键字必须单独成行;
postrotate/endscript	    在logrotate转储之后需要执行的指令，例如重新启动 (kill -HUP) 某个服务！必须独立成行;
tabootext [+] list 让logrotate 	不转储指定扩展名的文件，缺省的扩展名是：.rpm-orig, .rpmsave, v, 和 ~ 
sharedscripts	            运行postrotate脚本，作用是在所有日志都轮转后统一执行一次脚本。如果没有配置这个，那么每个日志轮转后都会执行一次脚本
dateformat .%s 	            配合dateext使用，紧跟在下一行出现，定义文件切割后的文件名，必须配合dateext使用，只支持 %Y %m %d %s 这四个参数
```
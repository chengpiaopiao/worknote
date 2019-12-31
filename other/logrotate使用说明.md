# logrotate使用说明

## 简介

   对于日常管理linux来说，日志文件显得非常的重要，它可以看出问题出现的点与相关错误信息，同样还可以根据信息来分析问题所出现的原因所在，是管理系统与服务必不可少的工具之一。
  logrotate是个十分有用的工具，它可以自动对日志进行截断（或轮转）、压缩以及删除旧的日志文件。例如，你可以设置logrotate，让/var/log/XXX日志文件每10天轮循，并删除超过1个月的日志。配置完后，logrotate的运作完全自动化，其实与系统的定时任务调用自定义脚本作用相同，它的运行也是定时任务来调用它的配置文件，从而实现上述效果的



## 安装

一般系统会默认安装logrotate，如果没有安装可通过以下命令进行安装（centos）
``` shell
yum install logrotate crontabs
```

安装完成后，需要重启rsyslog服务

``` shell
systemctl restart rsyslog.service
```



## 使用

### 配置文件

所有需要logrotate功能的配置文件都存放在以下目录

```bash
/etc/logrotate.d
```

下面的配置是对tomcat日志文件进行压缩，并清空原日志文件。

```shell
[root@centos logrotate.d]# cat tomcat_log
/usr/local/tomcat-6.0.39/logs/catalina.*.log
{
daily
maxage 30
rotate 10
notifempty
compress
copytruncate
missingok
size +10M
}
```

### 常用参数说明

- daily(weekly, monthly)    指定转储周期为每天（每周、每月）
- compress    通过gzip压缩转储以后的日志
- nocompress    不需要压缩时，用这个参数
- maxage 30    只存储最近30天的切割出来的日志文件，超过30天则删除
- rotate 10    指定日志文件删除之前切割的次数，此处保留10个备份
- notifempty    表示日志为空则不处理
- copytruncate    用于还在打开中的日志文件，把当前日志备份并截断
- nocopytruncate 备份日志文件但是不截断
- missingok     如果日志文件丢失，不报错继续执行下一个
- nomissingok    找不到日志时，报错
- size +10M    表示日志超过10M大小才分割，size默认单位是Byte，可使用K、M和G来指定KB、MB和GB

- create mode owner group 转储文件，使用指定的文件模式创建新的日志文件。
- nocreate 不建立新的日志文件。
- prerotate/endscript 在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行。
- postrotate/endscript 在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行。



## 运行原理

### 配置文件logrotate.conf

1. 查看软件包信息说明：

``` shell
[root@localhost ~]# rpm -ql logrotate
/etc/cron.daily/logrotate
/etc/logrotate.conf
/etc/logrotate.d
/usr/sbin/logrotate
/usr/share/doc/logrotate-3.7.8
/usr/share/doc/logrotate-3.7.8/CHANGES
/usr/share/doc/logrotate-3.7.8/COPYING
/usr/share/man/man5/logrotate.conf.5.gz
/usr/share/man/man8/logrotate.8.gz
/var/lib/logrotate.status
```

2. 从logrotate的软件包信息看，它默认的主配置文件是/etc/logrotate.conf，另外还定义了一个目录logrotate.d，用户可以在该目录下面增加需要日志自动分割处理的程序的配置。

``` shell
[root@localhost ~]# cat /etc/logrotate.conf 
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
#compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly
    create 0664 root utmp
	minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}

# system-specific logs may be also be configured here.
```

logrotate通过inclue /etc/logrotate.d 将logrotate.d目录里面的所有文件都会被自动的读入到/etc/logrotate.conf中。



### 运行机制

logrotate并不能实时监控日志文件的大小，需要借助于定时轮询机制crond来进行。


在Linux中，周期执行的任务一般由cron这个守护进程来处理。cron读取一个或多个配置文件，这些配置文件中包含了命令行及其调用时间。


cron的配置文件称为“crontab”，是“cron table”的简写。cron的定时任务文件放在/etc/cron.hourly/、/etc/cron.daily/、/etc/cron.weekly/、 /etc/cron.monthly/等目录下面，

分别按小时、天、周、月等轮询周期，读取对应的配置脚本文件，执行定时任务。

```shell
[root@localhost ~]# cat /etc/anacrontab 
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

#period in days   delay in minutes   job-identifier   command
1	5	cron.daily		nice run-parts /etc/cron.daily
7	25	cron.weekly		nice run-parts /etc/cron.weekly
@monthly 45	cron.monthly		nice run-parts /etc/cron.monthly
```

run-parts是运行一个目录中的所有脚本或程序



logrotate这个任务默认放在cron的每日定时任务cron.daily下面，查看其默认的执行脚本

``` shell
[root@localhost logrotate.d]# cd /etc/cron.daily/
[root@66 cron.daily]# cat logrotate 
#!/bin/sh

/usr/sbin/logrotate /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```



## 配置实例

### nginx日志轮转配置

通过yum方式安装的nginx，系统默认会自动通过logrotate这个日志管理软件，按天进行分割，配置如下：

``` shell
[root@66 cron.daily]# cat /etc/logrotate.d/nginx 
/var/log/nginx/*log {
    create 0644 nginx nginx
    daily
    rotate 10
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

### tomcat日志轮转配置

以下配置为自己编写，可供参考，生产环境使用需根据实际情况修改配置

``` shell
[root@centos logrotate.d]# cat tomcat_log
/usr/local/tomcat-6.0.39/logs/catalina.*.log
{
daily
maxage 30
rotate 10
notifempty
compress
copytruncate
missingok
size +10M
}
```

### MySQL日志轮转配置

MySQL本省提供了一个rotate的参考配置文件，在/usr/share/mysql目录下，文件名为mysql-log-rotate

``` shell
[youcantguess@localhost mysql]$ cat mysql-log-rotate 
# The log file name and location can be set in
# /etc/my.cnf by setting the "log-error" option
# in either [mysqld] or [mysqld_safe] section as
# follows:
#
# [mysqld]
# log-error=/var/lib/mysql/mysqld.log
#
# In case the root user has a password, then you
# have to create a /root/.my.cnf configuration file
# with the following content:
#
# [mysqladmin]
# password = <secret> 
# user= root
#
# where "<secret>" is the password. 
#
# ATTENTION: The /root/.my.cnf file should be readable
# _ONLY_ by root !

/var/lib/mysql/mysqld.log {
        # create 600 mysql mysql
        notifempty
        daily
        rotate 5
        missingok
        compress
    postrotate
	# just if mysqld is really running
	if test -x /usr/bin/mysqladmin && \
	   /usr/bin/mysqladmin ping &>/dev/null
	then
	   /usr/bin/mysqladmin flush-logs
	fi
    endscript
}

```





## 参考文档

1. https://www.centos.bz/2017/12/tomcat%E6%97%A5%E5%BF%97%E5%88%87%E5%89%B2%E5%B7%A5%E5%85%B7-logrotate/
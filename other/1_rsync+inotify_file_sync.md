
# rsync+inotify 实现文件同步

## 1. inotity 介绍
inotify是一种强大的、细粒度的、异步的文件系统事件控制机制。linux内核从2.6.13起，加入了inotify支持，通过inotify可以监控文件系统中添加、删除、修改、移动等各种事件，利用这个内核接口，第三方软件就可以监控文件系统下文件的各种变化情况，而inotify-tools正是实施监控的软件。

## 2. rsync+inotify 同步逻辑图


## 3. 环境部署

主机名 | 主机IP地址 | 系统版本 | 系统内核版本
=============================================
master | 192.168.0.110 | CentOS release 7.3 | 3.10.0-693.5.2.el7.x86_64
slave  | 192.168.0.111 | CentOS release 7.3  | 3.10.0-693.5.2.el7.x86_64


## 4. inotify-slave 部署

### 4.1 检查是否安装rsync
```
[root@slave ~]# rpm -aq rsync
rsync-3.0.9-18.el7.x86_64
```

### 4.2 新建rsync用户及模块目录并更改其用户组

添加rsync用户
```
[root@slave ~]# useradd rsync -s /sbin/nologin -M
[root@slave ~]# grep rsync /etc/passwd
rsync:x:1001:1001::/home/rsync:/sbin/nologin
```

创建rsync daemon工作模式的模块目录
```
[root@slave ~]# mkdir /backup
[root@slave ~]# ll -d /backup/
drwxr-xr-x. 2 root root 6 Dec  5 01:01 /backup/
```

更改模块目录的用户组
```
[root@slave ~]# chown rsync.rsync /backup/
[root@slave ~]# ll -d /backup/
drwxr-xr-x. 2 rsync rsync 6 Dec  5 01:01 /backup/
```

### 4.3 修改rsync的配置文件 /etc/rsyncd.conf
```
[root@inotify-slave /]# cat /etc/rsyncd.conf
#工作中指定用户(需要指定用户)
uid = rsync
gid = rsync
#相当于黑洞.出错定位
use chroot = no
#有多少个客户端同时传文件
max connections = 200
#超时时间
timeout = 300
#进程号文件
pid file = /var/run/rsyncd.pid
#日志文件
lock file = /var/run/rsync.lock
#日志文件
log file = /var/log/rsyncd.log

#模块开始
#这个模块对应的是推送目录
#模块名称随便起
[backup]
#需要同步的目录
path = /backup/
#表示出现错误忽略错误
ignore errors
#表示网络权限可写(本地控制真正可写)
read only = false
#这里设置IP或让不让同步
list = false
#指定允许的网段
hosts allow = 192.168.0.0/24
#拒绝链接的地址，一下表示没有拒绝的链接。
hosts deny = 0.0.0.0/32
#不要动的东西(默认情况)
#虚拟用户
auth users = rsync_backup
#虚拟用户的密码文件
secrets file = /etc/rsync.password
```

### 4.4 配置虚拟用户的密码文件
```
[root@slave ~]# echo "rsync_backup:123456" >/etc/rsync.passwd
[root@slave ~]# cat /etc/rsync.passwd 
rsync_backup:123456  #注：rsync_backup为虚拟用户，leesir为这个虚拟用户的密码

[root@slave ~]# chmod 600 /etc/rsync.passwd  #为密码文件提权，增加安全性
[root@slave ~]# ll /etc/rsync.passwd 
-rw-------. 1 root root 20 Dec  5 01:11 /etc/rsync.passwd
```

### 4.5 启动 rsync 服务
```
[root@slave ~]# rsync --daemon
[root@slave ~]# ps -ef | grep rsync
root       2244      1  0 00:12 ?        00:00:00 rsync --daemon
root       2246   2064  0 00:12 pts/0    00:00:00 grep --color=auto rsync

[root@slave ~]# netstat -lnutp | grep rsync
tcp        0      0 0.0.0.0:873             0.0.0.0:*               LISTEN      2244/rsync          
tcp6       0      0 :::873                  :::*                    LISTEN      2244/rsync
```

### 4.6 通过 master 测试推送

#### 4.6.1 master 配置密码文件，测试推送

创建密码文件
```
[root@master ~]# echo "123456" >/etc/rsync.passwd
[root@master ~]# cat /etc/rsync.passwd 
123456

[root@master ~]# chmod 600 /etc/rsync.passwd 
[root@master ~]# ll /etc/rsync.passwd 
-rw-------. 1 root root 7 Dec  5 01:16 /etc/rsync.passwd
```

创建一个测试文件 test.txt 
```
[root@master ~]# echo "hello world" > test.txt
[root@master ~]# cat /test.txt 
hello world
```

测试推送
```
[root@master ~]# rsync -avz /test.txt rsync_backup@192.168.0.111::backup --password-file=/etc/rsync.passwd
sending incremental file list
test.txt

sent 81 bytes  received 27 bytes  216.00 bytes/sec
total size is 12  speedup is 0.11
```

#### 4.6.2 slave 检查
```
[root@slave ~]# ll /backup/
total 4
-rw-r--r--. 1 rsync rsync 12 Dec  4 23:25 test.txt
[root@slave ~]# cat /backup/test.txt 
hello world
```

## 5. master 部署


### 5.1 查看系统是否支持 inotify
```
[root@master backup]# ll /proc/sys/fs/inotify/
total 0
-rw-r--r--. 1 root root 0 Dec  5 00:16 max_queued_events
-rw-r--r--. 1 root root 0 Dec  5 00:16 max_user_instances
-rw-r--r--. 1 root root 0 Dec  5 00:16 max_user_watches
```
显示这三个文件则证明支持。


> /proc/sys/fs/inotify/max_queued_evnets      
表示调用inotify_init时分配给inotify   instance中可排队的 event 的数目的最大值， 超出这个值的事件被丢弃， 但会触发IN_Q_OVERFLOW事件。

> /proc/sys/fs/inotify/max_user_instances 
表示每一个 real user ID可创建的inotify instatnces的数量上限。

> /proc/sys/fs/inotify/max_user_watches 
表示每个inotify instatnces可监控的最大目录数量。 如果监控的文件数目巨大，需要根据情况，适当增加此值的大小

### 5.2 安装 inotify
```
root@inotify-master tools]# wget http://github.com/downloads/rvoicilas/inotify-tools/inotify-tools-3.14.tar.gz  #下载inotify源码包
..................................
root@inotify-master tools]# ll inotify-tools-3.14.tar.gz
-rw-r--r-- 1 root root 358772 3月  14 2010 inotify-tools-3.14.tar.gz
[root@inotify-master tools]# tar zxf inotify-tools-3.14.tar.gz
[root@inotify-master tools]# cd inotify-tools-3.14
[root@inotify-master inotify-tools-3.14]# ./configure --prefix=/usr/local/inotify-3.14 #配置inotify,并指定安装路径为/usr/local/inotify-3.14
................................
[root@inotify-master inotify-tools-3.14]# make && make install
```

### 5.3 inotifywait 命令参数详解
```
[root@master inotify-3.14]# ./bin/inotifywait --help
-r|--recursive   Watch directories recursively. #递归查询目录
-q|--quiet      Print less (only print events). #打印监控事件的信息
-m|--monitor   Keep listening for events forever.  Without this option, inotifywait will exit after one  event is received.        #始终保持事件监听状态
--excludei <pattern>  Like --exclude but case insensitive.    #排除文件或目录时，不区分大小写。
--timefmt <fmt> strftime-compatible format string for use with %T in --format string. #指定时间输出的格式
--format <fmt>  Print using a specified printf-like format string; read the man page for more details.
#打印使用指定的输出类似格式字符串
-e|--event <event1> [ -e|--event <event2> ... ] Listen for specific event(s).  If omitted, all events are  listened for.   #通过此参数可以指定需要监控的事件，如下所示:
Events：
access           file or directory contents were read       #文件或目录被读取。
modify           file or directory contents were written    #文件或目录内容被修改。
attrib            file or directory attributes changed      #文件或目录属性被改变。
close            file or directory closed, regardless of read/write mode    #文件或目录封闭，无论读/写模式。
open            file or directory opened                    #文件或目录被打开。
moved_to        file or directory moved to watched directory    #文件或目录被移动至另外一个目录。
move            file or directory moved to or from watched directory    #文件或目录被移动另一个目录或从另一个目录移动至当前目录。
create           file or directory created within watched directory     #文件或目录被创建在当前目录
delete           file or directory deleted within watched directory     #文件或目录被删除
unmount         file system containing file or directory unmounted  #文件系统被卸载
```

### 5.4 编写监控脚本并加载到后台执行
```
[root@master scripts]# cat inotify.sh
#!/bin/bash
#para
host01=192.168.0.111  #inotify-slave的ip地址
src=/backup/        #本地监控的目录
dst=backup         #inotify-slave的rsync服务的模块名
user=rsync_backup      #inotify-slave的rsync服务的虚拟用户
rsync_passfile=/etc/rsync.password   #本地调用rsync服务的密码文件
inotify_home=/usr/local/inotify-3.14    #inotify的安装目录
#judge
if [ ! -e "$src" ] \
|| [ ! -e "${rsync_passfile}" ] \
|| [ ! -e "${inotify_home}/bin/inotifywait" ] \
|| [ ! -e "/usr/bin/rsync" ];
then
echo "Check File and Folder"
exit 9
fi
${inotify_home}/bin/inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T %w%f' -e close_write,delete,create,attrib $src \
| while read file
do
#  rsync -avzP --delete --timeout=100 --password-file=${rsync_passfile} $src $user@$host01::$dst >/dev/null 2>&1
cd $src && rsync -aruz -R --delete ./  --timeout=100 $user@$host01::$dst --password-file=${rsync_passfile} >/dev/null 2>&1
done
exit 0
[root@master scripts]# sh inotify.sh &  #将脚本加入后台执行
[1] 13438
[root@master scripts]#
```

### 5.5 实时同步测试

#### master操作：
```
[root@inotify-master scripts]# cd /backup/
[root@inotify-master backup]# ll
总用量 0
[root@inotify-master backup]# for a in `seq 200`;do touch $a;done  #创建200个文件
[root@master backup]# ll --time-style=full-iso
total 0
-rw-r--r--. 1 root root 0 2017-12-05 02:24:19.109995849 -0800 1
-rw-r--r--. 1 root root 0 2017-12-05 02:24:19.118995869 -0800 10
-rw-r--r--. 1 root root 0 2017-12-05 02:24:19.193996041 -0800 100
-rw-r--r--. 1 root root 0 2017-12-05 02:24:19.194996043 -0800 101
-rw-r--r--. 1 root root 0 2017-12-05 02:24:19.195996045 -0800 102
..........................
```

#### slave检查
```
[root@slave backup]# ll --time-style=full-iso
total 0
-rw-r--r--. 1 rsync rsync 0 2017-12-05 02:24:19.199760476 -0800 1
-rw-r--r--. 1 rsync rsync 0 2017-12-05 02:24:19.199760476 -0800 10
-rw-r--r--. 1 rsync rsync 0 2017-12-05 02:24:19.200760477 -0800 100
-rw-r--r--. 1 rsync rsync 0 2017-12-05 02:24:19.384760661 -0800 101
-rw-r--r--. 1 rsync rsync 0 2017-12-05 02:24:19.384760661 -0800 102
..........................
```
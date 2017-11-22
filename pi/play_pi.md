## 开始

### 启用 root 登录账户
	sudo passwd root 


## 更新软件源

### 备份源列表文件
	sudo cp /etc/apt/sources.list /etc/apt/sources.list.orig

### 修改源列表文件
	sudo nano /etc/apt/sources.list 或者sudo vi /etc/apt/sources.list

``` 
#大连东软信息学院(北方用户)
deb http://mirrors.neusoft.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi
deb-src http://mirrors.neusoft.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi

#中国科学技术大学
deb http://mirrors.ustc.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi
deb-src http://mirrors.ustc.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi

#清华大学
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi

#重庆大学(中西部用户)
deb http://mirrors.cqu.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi
deb-src http://mirrors.cqu.edu.cn/raspbian/raspbian/ wheezy main contrib non-free rpi

#前面几个都是教育网的

#搜狐
deb http://mirrors.sohu.com/raspbian/raspbian/ wheezy main contrib non-free rpi
deb-src http://mirrors.sohu.com/raspbian/raspbian/ wheezy main contrib non-free rpi

#阿里云
deb http://mirrors.aliyun.com/raspbian/raspbian/ wheezy main contrib non-free rpi
deb-src http://mirrors.aliyun.com/raspbian/raspbian/ wheezy main contrib non-free rpi

``` 

## 系统更新及vim安装

### 更新软件包
	sudo apt-get update && sudo apt-get upgrade

### 安装vim
安装vim

	sudo apt-get install vim
	
编辑 .vimrc  分别输入 vim ~/.vimrc, sudo vim /root/.vimrc 输入以下内容：
```
set nu
syntax on
set tabstop=4
```

### 更新固件
这里固件指的是GPU firmware and kernel , 一般不用更新，有更新强迫症的输入

	sudo rpi-update

等待固件更新完成，然后重启



## 安装miniconda

参考下面的链接
https://stackoverflow.com/questions/39371772/how-to-install-anaconda-on-raspberry-pi-3-model-b

### 下载 miniconda 

	wget http://repo.continuum.io/miniconda/Miniconda3-latest-Linux-armv7l.sh

### 校验 md5
	sudo md5sum Miniconda3-latest-Linux-armv7l.sh

### 安装
	sudo /bin/bash Miniconda3-latest-Linux-armv7l.sh
默认是安装到 /root/miniconda3 可以修改为 /home/pi/miniconda3

### 修改 .bashrc
	sudo nano /home/pi/.bashrc
在 .bashrc 文件的最后面添加
`export PATH="/home/pi/miniconda3/bin:$PATH"`
保存并重启


## 文件共享

### samba

安装 samba

	sudo apt-get install samba samba-common-bin -y

先备份，然后编辑/etc/samba/smb.conf文件

	sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
	sudo vim /etc/samba/smb.conf

需要修改添加的内容如下，
```
[global]
security = user
encrypt passwords = true
guest account = nobody
map to guest = bad user

#======================= Share Definitions =======================
[share]
comment = Guest access shares
path = /home/pi
browseable = yes
writable = yes
#read only = yes
guest ok = yes
public = yes
```

最后重启samba服务。然后同一局域网的其他设备就可以访问RPi的共享目录

	sudo service smbd restart


### FTP

安装 vsftp

	sudo apt-get install vsftpd -y

先备份，然后编辑配置文件

	sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.orig
	sudo vim /etc/vsftpd.conf

vsftp的配置文件，它允许你设置所有类型的限制和策略，目前没有深入研究，修改如下

```
# 不允许匿名访问
anonymous_enable=NO
# 设定可以进行写操作
write_enable=YES
# 设定本地用户可以访问
local_enable=YES
```

重启vsftpd服务

	sudo service vsftpd restart

通过ftp连接树莓派系统，以用户名pi登录，密码是pi用户的密码。ftp的根目录是/home/pi，即pi用户的HOME目录，可上传或下载文件了。



## 监控关键服务

### 安装配置monit

首先安装monit,

	sudo apt-get install monit

编辑配置文件，

	sudo vim /etc/monit/monitrc

下面只列出我认为可能需要修改的地方,
```
# 检查周期，默认为120秒，可以根据需要自行调节
set daemon  120

# 开启内嵌的web服务，可以访问 http://RPi-IP:2812 (若pi的IP为192.168.0.243，则用浏览器访问 192.168.0.243:2812)查看监控状态，去掉最前面的“#”可以开启
set httpd port 2812 and
    # use address localhost  # only accept connection from localhost
    allow localhost        # allow localhost to connect to the server and
    allow 192.168.0.0/24   # 允许IP段访问
    allow admin:monit      # require user 'admin' with password 'monit'


# 包含文件夹
include /etc/monit/conf.d/*
```

### 新建相关配置文件

#### samba

新建配置文件samba.conf，

	sudo vim /etc/monit/conf.d/samba.conf

输入，

```
check process samba with pidfile /var/run/samba/smbd.pid
        start program = "/etc/init.d/samba start"
        stop program = "/etc/init.d/samba stop"
        if failed port 139 then restart
        if 5 restarts within 5 cycles then timeout
```
#### ftp

新建配置文vsftpd.conf,

	sudo vim /etc/monit/conf.d/vsftpd.conf

输入，

```
check process vsftpd
	matching vsftpd
    start program = "/etc/init.d/vsftpd start"
    stop program = "/etc/init.d/vsftpd stop"
    if failed port 21 use type tcp then restart
```


### 重启服务

	sudo service monit reload && sudo service monit restart

通过查看log文件，验证monit服务已经启动

	sudo tail -f /var/log/monit.log

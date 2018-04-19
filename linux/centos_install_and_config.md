

## 安装CentOS
略

## centos 换源
http://blog.csdn.net/chavo0/article/details/51939362

## 配置防火墙
CentOS7.0 默认使用的是firewall作为防火墙，这里改为iptables

1. 关闭firewall

systemctl stop firewalld.serivce
systemctl disable firewalld.service

2. 安装iptables防火墙

yum install iptables-services

3. 配置防火墙
见下

4. 保存并设置开机启动

systemctl restart iptables.service
systemctl enable iptables.service


由于 MySQL 主从机之间需要进行网络通信，通信端口号为 3306，因此，需要对其进行防火墙配置，有两种配置方式：

* 永久关闭防火墙，以 root 身份执行如下命令，可以永久关闭防火墙：

chkconfig iptables off

此种关闭方式，在主机重启后，防火墙仍然处于关闭状态，非常危险，测试环境中可以使用，生产环境中一定不要这样使用，本文在虚拟机测试环境中就采用此种关闭方式。

* 仅仅关闭端口号 3306 的防火墙，以 root 身份编辑 /etc/sysconfig/iptables 文件，向其中添加如下内容：

:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT

## 安装MySQL

可以通过 yum、rpm 或者源码编译的方式安装 MySQL，建议使用源码编译方式安装到一个指定的目录内（比如 /opt/MySQL），如果需要在另一台主机上安装 MySQL，拷贝该目录即可。

注意：在 Master 和 Slave 主机上都需要安装 MySQL 客户端和 MySQL 服务器。

### 通过 yum 安装
查看有没有安装过 MySQL：

yum list installed mysql*
rpm -qa | grep mysql*

查看有没有安装包：

yum list mysql*

安装mysql客户端：

yum install mysql
安装mysql 服务器端：

yum install mysql-server
yum install mysql-devel


### 通过 rpm 安装

在 CentOS 的 DVD 镜像包中已经提供了 MySQL 的 rpm 包，因此，本文采用 rpm 方式安装 MySQL。

需要安装的 rpm 包有如下三个：

mysql-5.1.71-1.el6.x86_64.rpm
mysql-devel-5.1.71-1.el6.x86_64.rpm 
mysql-server-5.1.71-1.el6.x86_64.rpm
安装时，可以通过如下命令来检测上述三个 rpm 包的依赖包：

rpm -ivh --test mysql-5.1.71-1.el6.x86_64.rpm mysql-devel-5.1.71-1.el6.x86_64.rpm mysql-server-5.1.71-1.el6.x86_64.rpm
本文检测出的依赖包如下：

[root@localhost CentOS-6.5-x86_6]# rpm -ivh --test mysql-5.1.71-1.el6.x86_64.rpm mysql-devel-5.1.71-1.el6.x86_64.rpm mysql-server-5.1.71-1.el6.x86_64.rpm
warning: mysql-5.1.71-1.el6.x86_64.rpm: Header V3 RSA/SHA1 Signature, key ID c105b9de: NOKEY
error: Failed dependencies:
    openssl-devel is needed by mysql-devel-5.1.71-1.el6.x86_64
    perl(DBI) is needed by mysql-server-5.1.71-1.el6.x86_64
    perl-DBD-MySQL is needed by mysql-server-5.1.71-1.el6.x86_64
    perl-DBI is needed by mysql-server-5.1.71-1.el6.x86_64
[root@localhost CentOS-6.5-x86_6]#
因此，通过如下命令即可安装 MySQL 客户端和 MySQL 服务器：

rpm -ivh mysql-5.1.71-1.el6.x86_64.rpm  mysql-devel-5.1.71-1.el6.x86_64.rpm mysql-server-5.1.71-1.el6.x86_64.rpm openssl-devel-1.0.1e-15.el6.x86_64.rpm perl-DBI-1.609-4.el6.x86_64.rpm perl-DBD-MySQL-4.013-3.el6.x86_64.rpm 


## 启动 MySQL 服务

安装完成，启动 MySQL 服务：

service mysqld start
通过以下命令可以查看 MySQL Server 的运行状态：

service mysqld status













































CentOS7下解决yum install mysql-server没有可用包的问
http://www.centoscn.com/CentosBug/osbug/2016/0111/6643.html

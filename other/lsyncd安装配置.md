
安装依赖库文件和编译工具：yum install lua lua-devel asciidoc cmake

安装 rsync

yum install -y rsync

安装 lsyncd

1. 从 googlecode lsyncd(http://code.google.com/p/lsyncd/downloads/list) 上下载的lsyncd-2.1.5.tar.gz

2. 编译安装

./configure --prefix=/usr/local/lsyncd-2.1.5/

make && make install

3. 安装出错解决

configure: error: Need a Lua toolchain with matching versions ('lua' library and 'lua' and 'luac' programs)

3.1. download lua-devel

https://centos.pkgs.org/6/centos-x86_64/lua-devel-5.1.4-4.1.el6.x86_64.rpm.html


3.2. rpm -ivh /usr/local/tools/lua-devel-5.1.4-4.1.el6.x86_64.rpm


同步配置

```
# cd /usr/local/lsyncd-2.1.5
# mkdir etc var
# vi etc/lsyncd.conf
settings {
    logfile ="/usr/local/lsyncd-2.1.5/var/lsyncd.log",
    statusFile ="/usr/local/lsyncd-2.1.5/var/lsyncd.status",
    inotifyMode = "CloseWrite or Modify",
    maxProcesses = 10,
    statusInterval = 10,
    nodaemon = true,
    maxDelays = 20
}
sync {
    default.rsync,
    source    = "/usr/local/apache-tomcat-7.0.84/webapps/adservice/adFiles/",
    target    = "root@10.123.211.230:/usr/local/apache-tomcat-7.0.84/webapps/adservice/adFiles/",
    -- 上面target，注意如果是普通用户，必须拥有写权限
    maxDelays = 5,
    delay = 30,
    -- init = true,
    rsync     = {
        binary = "/usr/bin/rsync",
        archive = true,
        compress = true,
        bwlimit   = 2000
        -- rsh = "/usr/bin/ssh -p 22 -o StrictHostKeyChecking=no"
        -- 如果要指定其它端口，请用上面的rsh
    }
}

```

启动lsyncd

使用命令加载配置文件，启动守护进程，自动同步目录操作。

./lsyncd -log Exec /usr/local/lsyncd-2.1.5/etc/lsyncd.conf > /dev/null &



在远端被同步的服务器上开启ssh无密码登录，请注意用户身份：

user$ ssh-keygen -t rsa
一路回车...
user$ cd ~/.ssh
user$ cat id_rsa.pub >> authorized_keys

把id_rsa私钥拷贝到执行lsyncd的机器上

user$ chmod 600 ~/.ssh/id_rsa
测试能否无密码登录
user$ ssh root@10.123.211.229




## 安装依赖

> yum -y install gcc gcc-c++ make libtool zlib zlib-devel openssl openssl-devel pcre pcre-devel


## 配置和编译安装

配置
> ./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre

编译安装
> make && make install


## 常用编译选项说明

> --prefix=PATH ： 指定 nginx 的安装目录。默认 /usr/local/nginx，我的是 /usr/local/webserver/nginx

> --conf-path=PATH ： 设置nginx.conf配置文件的路径。nginx允许使用不同的配置文件启动，通过命令行中的-c选项。默认为conf/nginx.conf

> --user=name ： 设置nginx工作进程的用户。安装完成后，可以随时在nginx.conf配置文件更改user指令。默认的用户名是nobody。--group=name类似

> --with-pcre ： 设置PCRE库的源码路径，如果已通过yum方式安装，使用--with-pcre自动找到库文件。使用--with-pcre=PATH时，需要从PCRE网站下载pcre库的源码（8.39）并解压，指定 pcre 的源码路径 ，比如：--with-pcre=/root/pcre-8.39/。perl正则表达式使用在location指令和 ngx_http_rewrite_module模块中。

> --with-zlib=PATH ： 指定 zlib（版本1.1.3 - 1.2.5）的源码解压目录。在默认就启用的网络传输压缩模块ngx_http_gzip_module时需要使用zlib 。

> --with-http_ssl_module ： 使用https协议模块。默认情况下，该模块没有被构建。前提是openssl与openssl-devel已安装

> --with-http_stub_status_module ： 用来监控 Nginx 的当前状态

> --with-http_realip_module ： 通过这个模块允许我们改变客户端请求头中客户端IP地址值(例如X-Real-IP 或 X-Forwarded-For)，意义在于能够使得后台服务器记录原始客户端的IP地址

> --add-module=PATH ： 添加第三方外部模块，如nginx-sticky-module-ng或缓存模块。每次添加新的模块都要重新编译（Tengine可以在新加入module时无需重新编译）

## 启动 停止 nginx

    /usr/local/webserver/nginx/sbin/nginx  #启动 nginx 服务

    /usr/local/webserver/nginx/sbin/nginx -s stop  #停止 nginx 服务
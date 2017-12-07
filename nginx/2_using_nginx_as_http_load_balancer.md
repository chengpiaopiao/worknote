# Nginx 负载均衡配置

## 1. 配置前的准备

* Server1  192.168.0.110
* Server2  192.168.0.111

## 2. 负载均衡配置

这里Server1 即充当后台服务器又充当负载均衡服务器

Server1 的配置如下：

```
http {

    # ... 省略其它配置

    upstream 192.168.0.110 {
        server 192.168.0.110:8080;
        server 192.168.0.111:8080;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://192.168.0.110;
            proxy_set_header    Host    $host;
            proxy_set_header    X-Real-IP   $remote_addr;
            proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    # ... 省略其它配置
}

```

* `proxy_pass` http://192.168.0.110：表示将所有请求转发到上面upstream服务器组中配置的某一台服务器上。

* `upstream模块` 配置反向代理服务器组，Nginx会根据配置，将请求分发给组里的某一台服务器。

* `upstream模块下的server指令` 配置处理请求的服务器IP地址或域名，通过上面的配置，Nginx默认将请求依次分配给110，111来处理，可以通过修改下面这些参数来改变默认的分配策略：

> weight
```
upstream tomcats {
    server 192.168.0.110:8080 weight=2;  # 2/6次
    server 192.168.0.111:8080 weight=3;  # 3/6次
    server 192.168.0.112:8080 weight=1;  # 1/6次
}
```
上面配置表示6次请求中，110分配2次，111分配3次，112分配1次


> max_fails  fail_timeout
```
upstream tomcats {
    server 192.168.0.110:8080 weight=2 max_fails=3 fail_timeout=15;
    server 192.168.0.111:8080 weight=3;
    server 192.168.0.112:8080 weight=1;
}
```
上面配置表示192.168.0.100这台机器，如果有3次请求失败，nginx在15秒内，不会将新的请求分配给它。

> max_conns
```
upstream tomcats {
    server 192.168.0.100:8080 max_conns=1000;
}
```
表示最多给110这台Server分配1000个请求，如果这台Server正在处理1000个请求，nginx将不会分配新的请求给到它。假如有一个请求处理完了，还剩下999个请求在处理，这时nginx也会将新的请求分配给它。


* upstream模块server指令的其它参数和详细配置说明，请参考 [官方文档](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)




## 参考文档
* Nginx 反向代理和负载均衡配置  http://www.yduba.com/biancheng-3152689737.html
* Nginx负载均衡配置 http://blog.csdn.net/xyang81/article/details/51702900
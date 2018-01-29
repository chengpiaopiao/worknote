# logrotate 配置

## 1. 新建配置文件
```bash
	vi /etc/logrotate.d/rtomcatlog
```

添加以下内容，其功能是对日志文件进行压缩，并清空原日志文件。
```
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
该配置会对/usr/local/tomcat-6.0.39/logs/catalina.*.log进行切割。

- maxage 30：只存储最近365天的切割出来的日志文件，超过30天则删除。
- rotate 10：指定日志文件删除之前切割的次数，此处保留10个备份。
- notifempty：表示日志为空则不处理。
- compress：通过gzip压缩转储以后的日志。
- copytruncate：用于还在打开中的日志文件，把当前日志备份并截断。
- missingok：如果日志文件丢失，不报错继续执行下一个。
- size +10M：表示日志超过10M大小才分割，size默认单位是KB，可使用K、M和G来指定KB、MB和GB。

更多配置项说明
- compress 通过gzip压缩转储以后的日志。
- missingok 找不到日志时，跳过。
- nomissingok 找不到日志时，报错。
- nocompress 不需要压缩时，用这个参数。
- copytruncate 用于还在打开中的日志文件，把当前日志备份并截断。
- nocopytruncate 备份日志文件但是不截断。
- create mode owner group 转储文件，使用指定的文件模式创建新的日志文件。
- nocreate 不建立新的日志文件。
- prerotate/endscript 在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行。
- postrotate/endscript 在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行。
- daily 指定转储周期为每天。
- weekly 指定转储周期为每周。
- monthly 指定转储周期为每月。
- rotate count 指定日志文件删除之前转储的次数，0指没有备份，5指保留5个备份。
- size 当日志文件到达指定的大小时才转储，size可以指定bytes（缺省）以及KB、MB或者GB。

## 2. 配置crontab
编辑crontab
``` bash
	crontab -e
```

添加以下内容，其功能是每天凌晨1点执行 logrotate
```
# dom=day of month
# dow=day of week
# m h  dom mon dow   command
0 1 * * * /usr/sbin/logrotate -f /etc/logrotate.d/rtomcatlog
```


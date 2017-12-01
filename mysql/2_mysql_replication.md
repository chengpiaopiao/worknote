# mysql 双机热备配置

## 1. 创建同步资源

MySQL 的 Binary Log 机制可以同步整个 MySQL 库，也可以只同步一个或者多个数据库，无论哪种同步方式，都需在 Slave 上创建一个同步用户，允许该用户可以远程访问 Master 数据库，并可读取 Master 数据库的 Binary Log 日志。

理论上，可以在 Master 和 Slave 的 MySQL Server 中创建不同的用户用于同步，只要其中的初始数据一致即可，但在大多数实际生产环境中，为了降低运维成本，在 Master 和 Slave 上所创建的用于同步的用户名和数据库名通常都保持一致，本文在 Master 和 Slave 创建的同步用户名为 repl，密码是 123456。


**主从服务器分别作以下操作**

1. 修改 MySQL 服务器 root 用户的密码：

    mysqladmin -u root password 'root'

2. 以 root 用户身份登录 MySQL Server：

    mysql -uroot -proot

3. 创建同步用户 repl，密码是 123456：

    create user 'repl' identified by  '123456';

repl 用户建好以后，需要为之赋予一些权限，以便该用户可以执行登录、建表、insert等操作，在实际生产环节中，可以根据实际需要赋予特定的角色，本文使用如下语句为之赋予最大权限：

Master 端
```
grant all privileges on *.*  to 'repl'@'*' with grant option;
grant all privileges on *.*  to 'repl'@'%' with grant option;
grant all privileges on *.*  to 'repl'@'localhost' with grant option;
grant all privileges on *.*  to 'repl'@'Master' with grant option;
flush privileges;
```

Slave 端
```
grant all privileges on *.*  to 'repl'@'*' with grant option;
grant all privileges on *.*  to 'repl'@'%' with grant option;
grant all privileges on *.*  to 'repl'@'localhost' with grant option;
grant all privileges on *.*  to 'repl'@'Slave' with grant option;
flush privileges;
```
4. 测试以 repl 用户登录 MySQL Server：

    mysql -urepl -p123456

如果登录的过程中出现如下错误：

> ERROR 1045 (28000): Access denied for user 'repl'@'localhost' (using password: YES)
使用 mysqladmin 更正 repl 密码即可：

    mysqladmin -u repl password '123456'


## 2. 修改主服务器 master
``` 
[root@localhost ~]# vi /etc/my.cnf
   [mysqld]
   log-bin=mysql-bin  //[必须]启用二进制日志
   server-id=110      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```
   
## 3. 修改从服务器 slave
```
[root@localhost ~]# vi /etc/my.cnf
   [mysqld]
   log-bin=mysql-bin  //[必须]启用二进制日志
   server-id=111      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```

## 4. 重启两台服务器的 mysql

	[root@localhost ~]# service mysqld restart


## 5. 在主服务器上建立帐户并授权slave

	[root@localhost ~]# mysql -uroot -proot

	mysql> grant replication slave on *.* to 'repl'@'Slave' identified by '123456';


## 6. 登录主服务器的mysql，查询master的状态

```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000010 |     1415 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

## 7. 配置从服务器Slave
```
mysql> stop slave;

mysql> CHANGE MASTER TO MASTER_HOST='192.168.0.110',
    -> MASTER_USER='repl',MASTER_PASSWORD='123456',
    -> MASTER_PORT=3306,MASTER_CONNECT_RETRY=60,
    -> MASTER_LOG_FILE='mysql-bin.000010',MASTER_LOG_POS=1415;

mysql> start slave;
```

> 在执行 CHANGE MASTER TO 语句之前，一定要先执行 stop slave 指令。

> MASTER_HOST 表示需要进行同步的 MySQL 库所在的主机名，可以写 IP 地址，也可以填写 HOSTNAME。

> MASTER_LOG_FILE 和 MASTER_LOG_POS 一定要和步骤6在 Master 端执行 show master status 命令的结果保持一致，否则无法同步。

> 最后需要执行 SLAVE START 指令，开始主从同步。


## 8. 检查从服务器复制功能状态
```
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: Master //主服务器 hostname 或 IP
                  Master_User: repl   //授权账户名，尽量避免用root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000010
          Read_Master_Log_Pos: 1415
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 890
        Relay_Master_Log_File: mysql-bin.000010
             Slave_IO_Running: Yes  //此状态必须YES
            Slave_SQL_Running: Yes  //此状态必须YES
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1415  //同步读取二进制日志的位置
              Relay_Log_Space: 1064
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 110
                  Master_UUID: a943e673-d665-11e7-8e3d-000c29fe95e6
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)

ERROR: 
No query specified
```

> 注： Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态

至此，主从服务器配置完成。


## 9. 主从服务器测试

主服务器 mysql 中建立数据库，并插入数据;

在从服务器 mysql 中查看是否也自动增加了和主服务器中数据库和数据表。


## 10. 异常监控
待续







## Refrences
MySQL 双机热备配置步骤 https://www.zybuluo.com/Spongcer/note/69119
mysql主从复制（超简单）http://blog.51cto.com/369369/790921
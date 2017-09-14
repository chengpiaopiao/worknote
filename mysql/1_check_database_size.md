# mysql中查看表的大小

## 1. 查询数据库中各数据表占用大小

### 1.1 进去指定schema 数据库（存放了其他的数据库的信息）
``` sql
mysql> use information_schema 
Database changed
```

### 1.2 查询数据库 adserver 中各表的行数及占用空间大小
``` sql
mysql> select table_name, table_rows, data_length+index_length from tables where table_schema = 'adserver';
+---------------+------------+--------------------------+
| table_name    | table_rows | data_length+index_length |
+---------------+------------+--------------------------+
| adresult      |          0 |                    16384 |
| advertisement |          5 |                    32768 |
| link          |         13 |                    16384 |
| resources     |          5 |                    32768 |
| role          |          3 |                    32768 |
| role_link     |         31 |                    49152 |
| stbgroup      |     101159 |                 18399232 |
| stbs          |     101385 |                 11567104 |
| targets       |          3 |                    32768 |
| testmachine   |          0 |                    16384 |
| user          |          3 |                    16384 |
| user_role     |          3 |                    49152 |
+---------------+------------+--------------------------+
```

## 2. 查询具体数据表的大小（人性化显示结果）

### 2.1 进去指定schema 数据库（存放了其他的数据库的信息）
``` 
mysql> use information_schema 
Database changed
```

### 2.2 查询具体数据库的大小，如数据库adserver
```
mysql> select concat(round(sum(DATA_LENGTH/1024/1024), 2), 'MB')
    -> as data from TABLES;
+---------+
| data    |
+---------+
| 14.84MB |
+---------+
1 row in set (0.22 sec)
```

### 2.3 查询具体数据库的大小，如数据库 adserver
```
mysql> select concat(round(sum(DATA_LENGTH/1024/1024), 2), 'MB')
    -> as data from TABLES where table_schema='adserver';
+---------+
| data    |
+---------+
| 13.19MB |
+---------+
1 row in set (0.01 sec)
```

### 2.4 查看指定数据库的表的大小，比如说数据库 adserver 中的 stbgroup 表
```
mysql> select concat(round(sum(DATA_LENGTH/1024/1024), 2), 'MB')
    -> as data from TABLES where table_schema='adserver'
    -> and table_name='stbgroup';
+--------+
| data   |
+--------+
| 6.52MB |
+--------+
1 row in set (0.00 sec)
```
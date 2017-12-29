---
title: MySQL通识-高可用
date: 2017-12-14 09:17:52
tags: 数据库
categories: MySQL
---


## 二进制日志

准备两台MySQL服务器，版本越接近越好


###　查看二进制日志状态

```bash
show variables like 'log_bin';
show variables like '%log_bin%';
show variables like 'binlog_format';
```

###　开启二进制日志
修改my.cnf，在[mysqld]下面增加log_bin=mysql_bin_log，重启MySQL后，你就会发现log_bin变为了ON。如果在my.cnf里面只设置log_bin，但是不指定file_name，然后重启数据库。你会发现二进制日志文件名称为${hostname}-bin 这样的格式。
```bash
[mysqld]
log_bin=/mysql/bin_log/mysql_binlog
```

###　查看二进制日志文件

```bash
show binary logs;
show master logs;
show master status;
```

### 切换二进制日志

```bash
flush logs;
```
每次MySQL重启也会生成新的二进制日志文件。

### 删除二进制日志

`purge binary logs to xxx; `表示删除某个日志之前的所有二进制日志文件。这个命令会修改index中相关数据：
```bash
purge binary logs to 'DB-Server-bin.000002';
```

清除某个时间点以前的二进制日志文件：
```bash
purge binary logs before '2017-03-10 10:10:00';
```

清除7天前的二进制日志文件：
```bash
purge master logs before date_sub( now( ), interval 7 day);
```

清除所有的二进制日志文件（当前不存在主从复制关系）:
```bash
reset master;
```

也可以设置expire_logs_days参数，设置自动清理，其默认值为0,表示不启用过期自动删除功能，如果启用了自动清理功能，表示超出此天数的二进制日志文件将被自动删除，自动删除工作通常发生在MySQL启动时或FLUSH日志时。
```bash
show variables like 'expire_logs_days';
```

### 查看二进制日志内容

#### show binlog events

使用show binlog events方式可以获取当前以及指定binlog的日志，不适宜提取大量日志。`SHOW BINLOG EVENTS[IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count]`。生产环境不要执行这个命令，可能会导致卡死，应该使用limit限制查询大小：
```sql
show binlog events;
show binlog events in 'DB-Server-bin.000012';
show binlog events in 'DB-Server-bin.000012' from 336;
```

#### mysqlbinlog
当bin-log的模式设置为row时不仅日志长得快 并且查看执行的sql时 也稍微麻烦一点：1.干扰语句多；2生成sql的编码需要解码。直接mysqlbinlog出来的文件执行sql部分的sql显示为base64编码格式，故生成sql记录的时候不能用常规的办法去生成，需要加上相应的参数才能显示出sql语句`--base64-output=decode-rows -v`。
使用时可能会报错`mysqlbinlog: unknown variable 'default-character-set=utf8mb4'`，原因是mysqlbinlog这个工具无法识别binlog中的配置中的default-character-set=utf8这个指令，解决的办法有两个:
1. 将/etc/my.cnf中配置的default-character-set = utf8mb4修改为character-set-server = utf8mb4 但是这种修改方法需要重启数据库, 在线上业务库中使用这种方法查看 binlog 日志并不划算
2. mysqlbinlog --no-defaults mysql-bin.000256 完美解决;

```bash
# 提取指定的binlog日志  
mysqlbinlog /opt/data/APP01bin.000001  
mysqlbinlog /opt/data/APP01bin.000001|grep insert  

# 提取指定position位置的binlog日志  
mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001  
  
# 提取指定position位置的binlog日志并输出到压缩文件  
mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001 |gzip >extra_01.sql.gz  
  
# 提取指定position位置的binlog日志导入数据库  
mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001 | mysql -uroot -p  
  
# 提取指定开始时间的binlog并输出到日志文件  
mysqlbinlog --start-datetime="2014-12-15 20:15:23" /opt/data/APP01bin.000002 --result-file=extra02.sql  
  
# 提取指定位置的多个binlog日志文件  
mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001 /opt/data/APP01bin.000002|more  
  
# 提取指定数据库binlog并转换字符集到UTF8  
mysqlbinlog --database=test --set-charset=utf8 /opt/data/APP01bin.000001 /opt/data/APP01bin.000002 >test.sql  
  
# 远程提取日志，指定结束时间   
mysqlbinlog -urobin -p -h192.168.1.116 -P3306 --stop-datetime="2014-12-15 20:30:23" --read-from-remote-server mysql-bin.000033 |more  
  
# 远程提取使用row格式的binlog日志并输出到本地文件  
mysqlbinlog -urobin -p -P3606 -h192.168.1.177 --read-from-remote-server -vv inst3606bin.000005 >row.sql  
```




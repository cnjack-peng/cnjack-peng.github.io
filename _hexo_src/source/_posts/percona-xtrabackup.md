---
title: Percona XtraBackup
date: 2017-12-25 13:02:52
tags: 数据库
categories: MySQL
---

Percona XtraBackup是一个相对完美的免费开源数据备份工具，支持在线无锁表同步复制和可并行高效率的安全备份恢复。

## 备份原理
Percona XtraBackup 基于InnoDB引擎的故障恢复功能。它拷贝InnoDB的数据文件，但是数据文件内存的数据并不是一致的；但是通过 XtraBackup 执行故障恢复确保了数据的一致性。Percona XtraBackup 保证数据一致性 是通过InnoDB的redo log（也叫事务日志 transaction log）。redo log记录了InnoDB 数据的每次改变。当InnoDB（MySQL）启动，它检查数据文件和事务日志（redo / transaction log），然后执行如下两步:
1. 应用已经提交的事务
2. 回滚已经修改但是没有提交的事务

Percona XtraBackup 备份过程如下：
1. 备份InnoDB引擎：
当 Percona XtraBackup 开始工作，它记录所有 LSN （ log sequence number ），然后拷贝所有数据文件。
拷贝数据文件的同时， Percona XtraBackup 的一个后台进程监测 事务日志（redo / transaction log），然后从事务日志中拷贝改变的部分。
由于事务日志（log/transaction log）写日志采取round-robin方式，短时间事务日志就能够被重用，所以Percona XtraBackup 必须一致保持监测事务日志并拷贝改变部分。
Percona XtraBackup 从执行开始，就必须记录每一条事务日志，不然无法保证一致性和完整性。
2. 备份其他存储引擎（如MyISAM）：
Percona XtraBackup 备份 InnoDB/XtraDB 数据文件和事务日志完毕后。使用 FLUSH TABLES WITH READ LOCK 锁，阻止对MyISAM的DML操作,（从MySQL 5.6开始），然后备份拷贝MyISAM等non-InnoDB的相关文件，如.frm, .MRG, .MYD, .MYI,.TRG, .TRN, .ARM, .ARZ, .CSM, .CSV, .par, 和 .opt 文件。

`注意，备份MyISAM时，执行FLUSH TABLES WITH READ LOCK ，并不会影响InnoDB。`.


## 安装与使用

官方网站：`https://www.percona.com/downloads/XtraBackup/LATEST/`，下载适合你的平台软件，然后安装，需要注意的是如果下载的是RPM包或者源码，需要手动解决依赖的相关问题(报错:`Transaction Check Error:`)，推荐使用Percona仓库安装：
```bash
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
yum list | grep percona
yum install percona-xtrabackup-24
# 用于解压备份文件
yum install qpress
```

全量备份：
```bash
# 备份并压缩
xtrabackup --backup --compress --target-dir=/backup_store/db_back/
# 解压，必须解压才能--prepare
xtrabackup --backup --decompress --target-dir=/backup_store/db_back/
# prepare之前，数据文件是不一致的，因为它们在不同时间点被备份因此，--prepare会使所有数据文件的步调达成一致
xtrabackup --prepare --target-dir=/backup_store/db_back/
```

增量备份：
```bash

```



## 基于XtraBackup搭建主从

```bash
#原有主数据库版本
mysql -V
mysql  Ver 14.14 Distrib 5.5.31, for Linux (x86_64) using readline 5.1
#迁移从数据库版本
mysql -V
mysql  Ver 14.14 Distrib 5.6.25, for linux-glibc2.5 (x86_64) using  EditLine wrapper
#检查数据库引擎
show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
#主从数据库同步注意点
[mysqld]
#主从之间的id不能相同
server-id
#启用二进制日志
log-bin
#一般在从库开启（可选）
read_only
#推荐使用InnoDB并做好相关配置
#检查主从数据库状态
mysql -S /tmp/mysql.sock -e "show global variables like 'server_id';"
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
mysql -S /tmp/mysql.sock -e "show global variables like 'log_bin';"
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

### 备份和恢复
通常一般都直接使用innobackupex，因为它能同时备份InnoDB和MyISAM引擎的表。重点关注Slave_IO_Running和Slave_SQL_Runningd的状态是否为YES。
```bash
#备份
innobackupex --socket=/usr/local/var/mysql2/mysql2.sock --user=root --password --defaults-file=/etc/mysqld_multi.cnf --parallel=4 --database=passport /tmp/backup
#保持事务一致性
innobackupex --socket=/usr/local/var/mysql2/mysql2.sock --user=root --password --defaults-file=/etc/mysqld_multi.cnf --database=passport --apply-log /tmp/backup/2015-08-05_16-08-14
#传输
scp -r /tmp/backup/2015-08-05_16-08-14 10.10.16.24:/tmp/backup/ 
#恢复
innobackupex --socket=/tmp/mysql.sock --user=root --password --defaults-file=/app/local/mysql/my.cnf --copy-back /tmp/backup/2015-08-05_16-08-14/
#还原权限
chown -R mysql:mysql /app/data/mysql/data
service mysqld start
/app/local/mysql/scripts/mysql_install_db --basedir=/app/local/mysql --datadir=/app/data/mysql/data --no-defaults --skip-name-resolve --user=mysql
#主库授权同步帐号
SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
GRANT REPLICATION SLAVE ON *.* TO 'slave_passport'@'10.10.16.24' IDENTIFIED BY 'slave_passport';
FLUSH PRIVILEGES;
#从库开启同步
cat /tmp/backup/2015-08-05_16-08-14/xtrabackup_binlog_info 
mysql-bin.002599    804497686
CHANGE MASTER TO
MASTER_HOST='10.10.16.51',
MASTER_USER='slave_passport',
MASTER_PASSWORD='slave_passport',
MASTER_PORT=3307,
MASTER_LOG_FILE='mysql-bin.002599',
MASTER_LOG_POS=804497686;
#开启主从同步
start slave;
#查看从库状态
show slave status\ G
#从库的检查参数
Slave_IO_Running=Yes
Slave_SQL_Running=Yes
#主库的检查参数
show master status \G
+------------------+-----------+--------------+------------------+
| File             | Position  | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+-----------+--------------+------------------+
| mysql-bin.002600 | 454769337 |              |                  |
+------------------+-----------+--------------+------------------+
1 row in set (0.00 sec)
show processlist;
Master has sent all binlog to slave; waiting for binlog to be updated
```

### MySQL主从切换
切换前断开主库访问连接观察进程状态，无写操作后再停止从库IO_THREAD进行切换。
```bash
#查看主库状态
show processlist;
Master has sent all binlog to slave; waiting for binlog to be updated
show master status \G
#从库停止 IO_THREAD 线程
stop slave IO_THREAD;
show processlist;
Slave has read all relay log; waiting for the slave I/O thread to update it
show slave status \G
#从库切换为主库
stop slave;
reset master;
reset slave all;
show master status \G
#激活帐户
SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
GRANT REPLICATION SLAVE ON *.* TO 'slave_passport'@'10.10.16.51' IDENTIFIED BY 'slave_passport';
FLUSH PRIVILEGES;
#切换原有主库为从库
reset master;
reset slave all;
CHANGE MASTER TO
MASTER_HOST='10.10.16.24',
MASTER_USER='slave_passport',
MASTER_PASSWORD='slave_passport',
MASTER_PORT=3306,
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=804497686;
#检查主库
SHOW PROCESSLIST;
show master status \G
#启动从库
SHOW PROCESSLIST;
start slave;
show slave status \G
```

### 常见问题
Slave_SQL_Running:No
```bash
#一般是事务回滚造成的
stop slave;
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
start slave;
```
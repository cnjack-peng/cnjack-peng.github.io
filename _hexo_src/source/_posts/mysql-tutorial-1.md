---
title: MySQL通识-基础应用
date: 2017-12-14 09:17:52
tags: 数据库
categories: MySQL
---

## 基础运维命令

### 用户管理

#### 查询MySQL所有用户：
```sql
SELECT User, Host, Password FROM mysql.user;
SELECT DISTINCT User FROM mysql.user;
```

#### 添加MySQL用户：
```sql
create user keaimo identified by 'keaimopw';
grant all privileges on *.* to keaimo@'%' identified by 'keaimopw';

show grants for 'keaimo';
```

#### 忘记root密码：

**方法一**
1. 更改配置文件`/etc/my.cnf`,在`[mysqld]`的段上加上一句`skip-grant-tables`保存并退出；
2. 重启MySQL；
3. 登录并修改密码：
```bash
mysql> USE mysql ; 
mysql> UPDATE user SET Password=password( 'new-password' ) WHERE User='root'; 
mysql> flush privileges ; 
mysql> quit
```
4. 将MySQL配置改回来；
5. 重启MySQL；

**方法二**
1. KILL掉系统里的MySQL进程； 
```bash
killall -TERM mysqld
```

2. 用以下命令启动MySQL，以不检查权限的方式启动(safe_mysqld是mysqld_safe的符号链接)； 
```bash
safe_mysqld --skip-grant-tables &
```

3. 然后用空密码方式使用root用户登录 MySQL； 
```bash
mysql -u root
```

4. 修改root用户的密码； 
```bash
mysql> update mysql.user set password=PASSWORD('新密码') where User='root';
mysql> flush privileges;
mysql> quit
```


**方法三**
如果系统中没有`safe_mysqld`，可以使用如下方式恢复：
1. 停止mysqld(您可能有其它的方法,总之停止mysqld的运行就可以了)； 
    ```bash
    /etc/init.d/mysql stop
    ```

2. 用以下命令启动MySQL，以不检查权限的方式启动； 
    ```bash
    mysqld --skip-grant-tables &
    ```

3. 然后用空密码方式使用root用户登录 MySQL； 
    ```bash
    mysql -u root
    ```

4. 修改root用户的密码； 
    ```bash
    mysql> update mysql.user set password=PASSWORD('newpassword') where User='root'; 
    mysql> flush privileges; 
    mysql> quit
    ```

5. 重新启动MySQL
    ```bash
    /etc/init.d/mysql restart
    ```

### 环境信息

#### MySQL版本信息

1. Shell命令：
```bash
mysql -V
```

2. MySQL中：
```bash
mysql> status;
```

3. help中查找:
```bash
mysql --help | grep Distrib
```

4. 使用mysql函数：
```bash
mysql> select version();
```

### 实时SQL监控

进入Mysql，启用Log功能(general_log=ON) 
```bash
SHOW VARIABLES LIKE "general_log%"; 
SET GLOBAL general_log = 'ON';
```
设置Log文件地址(所有Sql语句都会在general_log_file里) 
```bash
SET GLOBAL general_log_file = 'c:\mysql.log';
```

### 查询结果导出到文件

```bash
select count(1) from table   into outfile '/data/test.xls';
```

```bash
mysql> pager cat > /tmp/test.txt ;
PAGER set to 'cat > /tmp/test.txt'
# 之后的所有查询结果都自动写入/tmp/test.txt'，并前后覆盖
mysql> select * from table ;
30 rows in set (0.59 sec)
# 在框口不再显示查询结果
```

### 直接执行SQL文件

**第一种方式**
在未连接数据库的情况下，输入 `mysql -h localhost -u root -p 123456  < d:\book.sql` 回车即可；

**第二种方式**
在已连接数据库的情况下，此时命令提示符为`mysql>`，输入 `source d:\book.sql`  或者 `\. d:\book.sql` 回车即可。
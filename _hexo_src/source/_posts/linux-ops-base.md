---
title: Linux运维基础
date: 2017-12-08 15:02:27
tags:
categories: Linux运维
---

## 常用命令

命令|用法
----|----
ls | ls -lha (以mb为单位显示)

## 磁盘管理
查看文件(夹)所在分区(挂载点)
```bash
df /path
cat /etc/mtab
mount
fdisk -l
```

查找大文件/文件夹：
```bash
find . -type f -size +800M
find . -type f -size +800M  -print0 | xargs -0 ls -l
find . -type f -size +800M  -print0 | xargs -0 du -h
find . -type f -size +800M  -print0 | xargs -0 du -h | sort -nr
```

## 运行状态
TOP查看系统运行情况：
- 输入大写P，结果按CPU占用降序排序；
- 输入大写M，结果按内存占用降序排序；


## NFS

```bash

```


## 网络安全

当VPS暴露在外网中，就会有人不断暴力破解你的SSH登录。于是就有必要使用SSH密钥来登录。并关闭密码登录。用以下命令可以查看别人暴力破解你SSH密码登录的大概情况：
```bash
grep "Failed password for invalid user" /var/log/secure | awk '{print $13}' | sort | uniq -c | sort -nr | more
```
这时可以使用SSH秘钥登录来防止暴力破解。
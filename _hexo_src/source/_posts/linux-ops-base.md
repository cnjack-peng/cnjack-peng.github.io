---
title: Linux运维基础
date: 2017-12-08 15:02:27
tags:
categories: Linux运维
---


查看文件(夹)所在分区(挂载点)
```bash
df /path
cat /etc/mtab
mount
fdisk -l
```

TOP查看系统运行情况：
- 输入大写P，结果按CPU占用降序排序；
- 输入大写M，结果按内存占用降序排序；
---
title: find命令-print0和xargs中-0的奥妙
date: 2017-12-08 10:30:56
tags:
categories: Linux命令
---

## -print0

默认情况下, find 每输出一个文件名, 后面都会接着输出一个换行符 ('\n'), 因此我们看到的 find 的输出都是一行一行的:

```bash
[bash-4.1.5] ; ls -l
total 0
-rw-r--r-- 1 root root 0 2010-08-02 18:09 file1.log
-rw-r--r-- 1 root root 0 2010-08-02 18:09 file2.log

[bash-4.1.5] ; find -name '*.log'
./file2.log
./file1.log
```

比如我想把所有的 .log 文件删掉, 可以这样配合 xargs 一起用:
```bash
[bash-4.1.5] ; find -name '*.log'
./file2.log
./file1.log
[bash-4.1.5] ; find -name '*.log' | xargs rm
[bash-4.1.5] ; find -name '*.log'
```

 find+xargs 真的很强大. 然而:

```bash
[bash-4.1.5] ; ls -l
total 0
-rw-r--r-- 1 root root 0 2010-08-02 18:12 file 1.log
-rw-r--r-- 1 root root 0 2010-08-02 18:12 file 2.log
[bash-4.1.5] ; find -name '*.log'
./file 1.log
./file 2.log
[bash-4.1.5] ; find -name '*.log' | xargs rm
rm: cannot remove `./file': No such file or directory
rm: cannot remove `1.log': No such file or directory
rm: cannot remove `./file': No such file or directory
rm: cannot remove `2.log': No such file or directory
```

 原因其实很简单, xargs 默认是以空白字符 (空格, TAB, 换行符) 来分割记录的, 因此文件名 ./file 1.log 被解释成了两个记录 ./file 和 1.log, 不幸的是 rm 找不到这两个文件.
 
为了解决此类问题, 聪明的人想出了一个办法, 让 find 在打印出一个文件名之后接着输出一个 NULL 字符 ('\0') 而不是换行符, 然后再告诉 xargs 也用 NULL 字符来作为记录的分隔符. 这就是 find 的 -print0 和 xargs 的 -0 的来历吧.

```bash
[bash-4.1.5] ; ls -l
total 0
-rw-r--r-- 1 root root 0 2010-08-02 18:12 file 1.log
-rw-r--r-- 1 root root 0 2010-08-02 18:12 file 2.log
[bash-4.1.5] ; find -name '*.log' -print0 | hd
           0  1  2  3   4  5  6  7   8  9  A  B   C  D  E  F  |0123456789ABCDEF|
--------+--+--+--+--+---+--+--+--+---+--+--+--+---+--+--+--+--+----------------|
00000000: 2e 2f 66 69  6c 65 20 31  2e 6c 6f 67  00 2e 2f 66  |./file 1.log../f|
00000010: 69 6c 65 20  32 2e 6c 6f  67 00                     |ile 2.log.      |
[bash-4.1.5] ; find -name '*.log' -print0 | xargs -0 rm
[bash-4.1.5] ; find -name '*.log'
```

你可能要问了, 为什么要选 '\0' 而不是其他字符做分隔符呢? 这个也容易理解: 一般的编程语言中都用 '\0' 来作为字符串的结束标志, 文件的路径名中不可能包含 '\0' 字符.


## find|xargs实例

查找当前目录大文件并按大小排序：
```bash
find . -type f -size +800M
find . -type f -size +800M  -print0 | xargs -0 ls -l
find . -type f -size +800M  -print0 | xargs -0 du -h
find . -type f -size +800M  -print0 | xargs -0 du -h | sort -nr
```

查找Linux下大目录：
```base
du -h --max-depth=1
du -h --max-depth=2 | sort -n
du -hm --max-depth=2 | sort -n
du -hm --max-depth=2 | sort -nr | head -12
```

删除以html结尾的10天前的文件，包括带空格的文件：
```base
find /usr/local/backups -name "*.html" -mtime +10 -print0 |xargs -0 rm -rfv
find /usr/local/backups -mtime +10 -name "*.html" -exec rm -rf {} \;
```

当前目录下文件从大到小排序（包括隐藏文件），文件名不为"."：
```base
find . -maxdepth 1 ! -name "." -print0 | xargs -0 du -b | sort -nr | head -10 | nl
nl：可以为输出列加上编号,与cat -n相似，但空行不编号
```

以下功能同上，但不包括隐藏文件：
```bash
for file in *; do du -b "$file"; done|sort -nr|head -10|nl
```

xargs结合sed替换：
```bash
find . -name "*.txt" -print0 | xargs -0 sed -i 's/aaa/bbb/g'
```

xargs结合grep：
```bash
find . -name '*.txt' -type f -print0 |xargs -0 grep -n 'aaa'    #“-n”输出行号
```


用rm删除太多的文件时候，可能得到一个错误信息：`/bin/rm Argument list too long`. 用xargs去避免这个问题(xargs -0将\0作为定界符)：
```bash
find . -type f -name "*.log" -print0 | xargs -0 rm -f
```

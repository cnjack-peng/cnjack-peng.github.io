---
title: HEXO Tutorial
year: 2017
month: 11
day: 20
categories: 小技巧
---
HEXO是一个静态BLOG APPS。

## 安装配置

### 安装

```bash
cnpm install hexo-cli -g
```

<!--more-->

### 启动
```bash
hexo server
```

### 配置
语言配置
```bash
language: zh-Hans
```

### 部署
```bash
hexo clean
hexo generate
```



## NEXT主题


### 打开侧边栏
```bash
sidebar:
  display: always
```


## 高级功能


### 搜索

```bash
cnpm install hexo-generator-search --save
cnpm install hexo-generator-searchdb --save
```
在站点配置中加入：
```bash
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
在NEXT主题配置文件中开启：
```bash
local_search:
  enable: true
```

### 关于我页面
```bash
hexo new page about
```
在NEXT主题中开启关于我页面即可

## 写作

### 阅读全文
在适当的位置加入如下标签即可：
```bash
<!--more-->
```
### 创建文章
```base
hexo new [layout] <title>
hexo new MyPage
```


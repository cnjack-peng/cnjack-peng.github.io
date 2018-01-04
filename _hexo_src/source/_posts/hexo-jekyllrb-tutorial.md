---
title: HEXO&Jekyllrb Tutorial
date: 2017/11/20 17:30:00
categories: 小技巧
---
HEXO是一个静态BLOG APPS。

## Hexo安装配置

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

### NEXT主题
```bash
$ cd your-hexo-site
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
克隆/下载 完成后，打开 站点配置文件，找到theme字段，并将其值更改为next。
> 这里注意区分两个配置文件：
> 站点配置文件：是你的 hexo 博客目录下面的 _config.yml 文件。
> 主题配置文件：是 themes/next 目录下的 _config.yml 文件。

### 访问统计

打开 Hexo 目录下的 \themes\next\ _config.yml 文件
{% asset_img next-visit-count.png %}

### 打开侧边栏
```bash
sidebar:
  display: always
```


### 高级功能


#### 搜索

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

#### 关于我页面
```bash
hexo new page about
```
在NEXT主题中开启关于我页面即可

#### 写作

#### 阅读全文
在适当的位置加入如下标签即可：
```bash
<!--more-->
```
#### 创建文章
```base
hexo new [layout] <title>
hexo new MyPage
```

### 插入图片

修改`_config.yml`配置文件`post_asset_folder`项为`true`。创建文章使用命令如下：
```bash
hexo new 'article title'
```
使用完命令之后，在source/_post文件夹里面就会出现一个“article title.md”的文件和一个“article title”的文件夹。



## Jekyllrb安装配置

Jekyllrb依赖Ruby，需要安装Ruby环境

### 安装

### Minimal Mistakes
Minimal Mistakes（https://mademistakes.com/work/minimal-mistakes-jekyll-theme/）是一个Jekyllrb主题


## 评论服务

Disqus第三方评论服务。

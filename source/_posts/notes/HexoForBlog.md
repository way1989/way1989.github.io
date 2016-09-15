---
date: 2016-09-12 22:56:51
title: Hexo换了电脑处理方法
description: 为了可以在多个电脑上面都处理hexo博客，所以我把source文件和网站的文件分别放在hexo和master分支上面
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - Hexo
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

## 本文只讲述hexo换了电脑处理方法，如果想了解使用hexo搭建个人博客，请访问：[使用hexo搭建个人博客](http://www.jianshu.com/p/73ca570e4d61)

##  为了可以在多个电脑上面都处理hexo博客，所以我把source文件和网站的文件分别放在hexo和master分支上面，首先将hexo分支克隆到本地：
``` bash
$ git clone -b hexo git@github.com:way1989/way1989.github.io.git
$ cd way1989.github.io
$ npm install
```

## 此时，你的博客就OK了，运行 hexo s --debug看一看，是不是一切正常？然后测试是否能在新的电脑上发表文章。
``` bash
$ hexo s --debug 然后打开`localhost:4000`就可以看见刚才发表的测试文章了
```

## 当在本地确认博客效果后，就可以将md文件生成静态网页上传至GithubPages，在终端定位到way1989.github.io目录下，执行下面的命令即可:
``` bash
$ hexo clean #清除网页缓存
$ hexo g #生成静态网页
￥ hexo d #开始部署
//当然也可以使用一次性命令：
￥ hexo clean && hexo g && hexo d
```

## 添加百度/谷歌/本地 自定义站点内容搜索
### 安装`hexo-generator-search`，在站点的根目录下执行以下命令：
``` bash
$ npm install hexo-generator-search --save
```

### 编辑``站点配置文件``，新增以下内容到任意位置:
``` bash
search:
  path: search.xml
  field: post
```

### hexo部署失败 ERROR Deployer not found: git
``` bash
$ npm install hexo-deployer-git --save
```

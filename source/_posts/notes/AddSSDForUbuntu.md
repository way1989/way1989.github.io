---
date: 2016-09-02 11:16:24
title: ubuntu新增SSD指南
description: ubuntu新增SSD自动挂载
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - SSD
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

##  打开Applications-->System Tools-->Preferences-->Disk Utility，先格式化240G的那个硬盘，选择ext4格式，然后创建分区：

##  查看硬盘信息，终端执行：(/dev/sdb1可能不同，具体参考Disk Utility中显示)：
``` bash
$ sudo blkid /dev/sdb1
```

###  显示类似以下信息:

``` bash
/dev/sdb1: LABEL="SSD" UUID="861ada17-a2a2-4781-8ec7-5b93794adf9b" TYPE="ext4"
```

## 修改自动挂载点，在最后增加一行(需要先创建好SSD目录sudo mkdir /SSD)：
``` bash
$ sudo vim /etc/fstab
```

### 显示类似以下信息:
``` bash
UUID=861ada17-a2a2-4781-8ec7-5b93794adf9b /SSD           ext4    defaults        1       2
```

## 执行以下命令即可，以后开机都会自动挂载到/SSD目录了(目录名字可以随意取)
``` bash
$ sudo mount -a
```

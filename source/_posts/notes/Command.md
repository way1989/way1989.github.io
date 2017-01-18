---
date: 2017-01-14 17:10:24
title: 各种常用脚本
description: 各种常用脚本备忘
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - command
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

## `alogcat`可以过滤打印出对应包名的log：

``` bash
# 作用：能够通过进程名显示log
# 用法：alogcat com.android.calendar or alogcat calendar
# 当监控的进程异常退出时，需要重新运行此命令
#function alogcat() {
    OUT=$(adb shell ps | grep -i $1 | awk '{print $2}')
    OUT=$(echo $OUT | sed 's/[[:blank:]]\+/\|/g')
    # 当进程异常退出，log是通过 AndroidRuntime 输出的
    adb logcat -v time | grep -E "$OUT|AndroidRuntime"
#}
```

## `tingpng`批量压缩图片资源的脚本：

```bash
#!/bin/bash
pngquant -f --skip-if-larger --ext .png --quality 50-80 `find . -name "*.png" -type f ! -name "*.9.png"`
```
## 未完待续...
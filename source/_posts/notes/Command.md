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

##  在Mac、Linux 终端显示 Git 当前所在分支

编辑.bashrc文件,将下面的代码加入到文件的最后处,保存退出，执行加载命令`source ./.bashrc`:

```bash
## Parses out the branch name from .git/HEAD:
find_git_branch () {
    local dir=. head
    until [ "$dir" -ef / ]; do
        if [ -f "$dir/.git/HEAD" ]; then
            head=$(< "$dir/.git/HEAD")
            if [[ $head = ref:\ refs/heads/* ]]; then
                git_branch=" → ${head#*/*/}"
            elif [[ $head != '' ]]; then
                git_branch=" → (detached)"
            else
                git_branch=" → (unknow)"
            fi
            return
        fi
        dir="../$dir"
    done
    git_branch=''
}

PROMPT_COMMAND="find_git_branch; $PROMPT_COMMAND"

# Here is bash color codes you can use
  black=$'\[\e[1;30m\]'
    red=$'\[\e[1;31m\]'
  green=$'\[\e[1;32m\]'
 yellow=$'\[\e[1;33m\]'
   blue=$'\[\e[1;34m\]'
magenta=$'\[\e[1;35m\]'
   cyan=$'\[\e[1;36m\]'
  white=$'\[\e[1;37m\]'
 normal=$'\[\e[m\]'

PS1="$white[$white\u$white@$white\h$white:$white\w$yellow\$git_branch$white]\$ $normal"
```

## 未完待续...

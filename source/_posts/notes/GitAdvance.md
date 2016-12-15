---
date: 2016-12-15 11:22:15
title: Git进阶用法
description: git rebase高阶用法
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - GIT
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

##  Git重要概念：

对于任何一个文件，在 Git 内都只有三种状态：已修改（modified）、已暂存（staged）、已提交（committed）：

* 已修改表示修改了某个文件，但还没有提交保存.
* 已暂存表示把已修改的文件放在下次提交时要保存的清单中.
* 已提交表示该文件已经被安全地保存在本地数据库中了.

 <img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/1.png"/>

## 两种分支合并的方法,rebase 与 merge：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/2.png"/>

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/3.png"/>

## Rebase的原理:

* 先把本地的未push的提交生成一个个patch
* 然后移除本地未上传的提交
* 把服务器最新的提交全拉下来
* 按修改顺序，重新应用本地的提交生成的patch

由此可见，rebase过程可能产生冲突，而且可能需要解决多次

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/4.png"/>


## 删除一个未上传的提交

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/5.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/6.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/7.png"/>

注意点:
* Rebase时log显示是按修改的顺序显示的，和git log 显示刚好是相反的
* Rebase完以后编辑器里面最少需要保留一行有效内容，否则不会进行rebase
* 适合某个commit,在走读以后发现是不需要修改的

## 修改非最后一个提交

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/8.png"/>

注意点:
* 这个方案适合走读发现的小问题修改，适合少量的代码修改。
* 大量的代码修改，建议把要修改的内容变成一个commit,然后合并到原来的commit

## 合并本地未上传的提交

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/9.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/10.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/11.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/12.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/13.png"/>

注意点:
* 只能合并到未上传的commit
* 如果多个提交合并到一个提交也是同理的，把要合并的commit按顺序移动到待合并的commit后面，要合并的commit前面的pick都修改成f,然后保存退出
* Rebase过程可能出现冲突，此时需要修改冲突然后执行”git add . “ 和”git rebase --continue”

## 已经commit未push的提交，不小心reset掉了，怎么找回来

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/14.png"/>

操作步骤及注意事项:
* 根据后面的commit_msg找到对应的commit
* 执行命令git cherry-pick commit_id
* 各种rebase操作错误的时候，也可以利用git reflog，找到rebase之前的commit,然后执行git reset –hard commit_id (这个会丢弃本地未commit的修改内容)

## 把自己的修改内容合并到别人的commit里面去了

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/15.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/16.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/17.png"/>

操作步骤及注意事项
* 根据后面的commit_msg找到对应的comit
* git reset –soft commit_id

## 从gerrit获取别人提交的commit

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/18.png"/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/gitAdvance/19.png"/>

操作步骤及注意事项
* 从gerrit查看到要下载的commit的commit_id
* 执行命令git ls-remote | grep 66e1d6ec3597，获取要下载commit在gerrit的路径
* 执行命令 git pull --rebase origin refs/changes/06/3306/1, 将要下载的commit rebase到本地
* 要下载的commit 所依赖的其它commit都会同时下载下来，走读时可只下载最后的commit

## 常见错误

* git pull –rebase，出现冲突时，解决后不知道执行git rebase –continue
* git cherry-pick commit_id 出现冲突时,解决后不知道执行git cherry-pick –continue
* 出现冲突时，直接切到其它分支




---
title: git log命令展示过滤技巧
date: 2016-11-08 11:21:21
tags: git
categories: git 
---

git log命令强大，掌握几条使用技巧可以让你的工作事半功倍，这篇文章总结了git log基础的相关使用技巧，满足了绝大数git log使用场景．

![Better git-log](http://www.linuxdiyf.com/linux/uploads/allimg/160812/2-160Q215063C28.jpg)

<!--more-->

# git log命令展示过滤技巧


###1. git log -n
>展示前n条数据

###2.git log --stat
>展示简要的每次提交行数的变化，及其他基本信息。

###3.git log -p
>展示每次提交详细的代码变化

###4.git log --pretty=oneline
>用一行展示每次提交的*commit id*   和 *提交注释信息*

###5. git log  --graph
>展示分支信息



###6.git log --pretty=format:" "

``` 
 git log --pretty=format:"%h %s" 
 #个人log配置个性化输出命令
 git log --pretty=format:"%H %cd *%an*:%s(%ar)" --graph 
```
- %H  提交对象（commit）的完整哈希字串
- %h  提交对象的简短哈希字串
- %T  树对象（tree）的完整哈希字串
- %t  树对象的简短哈希字串
- %P  父对象（parent）的完整哈希字串
- %p  父对象的简短哈希字串
- %an 作者（author）的名字
- %ae 作者的电子邮件地址
- %ad 作者修订日期（可以用 -date= 选项定制格式）
- %ar 作者修订日期，按多久以前的方式显示
- %cn 提交者(committer)的名字
- %ce 提交者的电子邮件地址
- %cd 提交日期
- %cr 提交日期，按多久以前的方式显示
- %s  提交说明

###7.git log --since --author --grep
展示指定log信息,时间参数需要用UTC格式时间。

- -n 仅显示最近的 n 条提交
- --since, --after 仅显示指定时间之后的提交。
- --until, --before 仅显示指定时间之前的提交。
- --author 仅显示指定作者相关的提交。
- --committer 仅显示指定提交者相关的提交。
- git log hash.. 可以输出指定hash之后的提交

###8.git log 参数参考

```
git log 命令支持的选项

-p 按补丁格式显示每个更新之间的差异。

--stat 显示每次更新的文件修改统计信息。

--shortstat 只显示 --stat 中最后的行数修改添加移除统计。

--name-only 仅在提交信息后显示已修改的文件清单。

--name-status 显示新增、修改、删除的文件清单。

--abbrev-commit 仅显示 SHA-1 的前几个字符，而非所有的 40 个字符。

--relative-date 使用较短的相对时间显示（比如，“2 weeks ago”）。

--graph 显示 ASCII 图形表示的分支合并历史。

--pretty 使用其他格式显示历史提交信息。可用的选项包括 oneline，short，full，fuller 和 format（后跟指定格式）。
```
###9.软件辅助
推荐两款软件都特别好用： gitk 和 gitg
这两款软件mac和linux 都有相关版本

###10.自我学习
```
$ git log --help
```




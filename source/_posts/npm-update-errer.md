---
title: Node 升级工具n 大坑 
date: 2016-04-08 00:46:19
categories: node 
tags: node
---

接触node是应为hexo博客框架的使用， 的确很方便很便捷，然后正是由于太简单，大部分工具都是一键搞定，对于程序员来说确不是工作挣钱的最佳语言，除非学的非常精湛，自己架构框架供别人使用。
所以node可以当成程序员个人语言爱好，业余时间了解学习。只是个人理解，不喜勿喷。
![npm instll n](http://www.uml.org.cn/itnews/images/201312030901.jpg)

<!-- more -->

本文环境基于Mac OS X EI Capitan V10.11.4,应该是mac环境的通病。

## 好奇心害死猫

心血来潮想更新一下node.js ,
在命令行里输入(网上的方法)：

```
sudo npm install -g n
```

接着又输入 sudo n stable

然后命令行里开始显示百分比，从1% 慢慢变到100%，我以为更新完了，结果。。。
输入 node -v  显示：

```
    dyld: Symbol not found:
    Referenced from: /usr/local/bin/node
    Expected in: /usr/lib/libstdc++.6.dylib
    Trace/BPT trap: 5
```

然后就知道麻烦来了，总之，npm后都是这样子，网上百度各种办法，重装gcc ， 卸载node重装， 添加环境变量等等。。。。

反正各种方法都行不同， 真不知道 *n* 这个工具到底是否能够在mac上用，反正好多人遇到类似办法都没有解决。

## 抛弃n工具

既然n不能够在我的mac上起到升级作用，且还搞坏了node系统，且网上没有搜到有效的相关解决方案，那边只好卸载完全卸载node后重装了。

由于用了*brew*安装的node ，用  ```brew uninstall node``` 卸载node后发现还是没有解决问题。

那么一定是这个命令没有完全卸载node，那么只好自己手动卸载了。

###  cd 到根目录 

```
    find . -name "node"
    find . -name "npm"
```
删除所有搜索与node相关的结果

### 重新 brew instll node

安装结束肯能会提示err：

```
    Error: The `brew link` step did not complete successfully
    The formula built, but is not symlinked into /usr/local
    Could not symlink lib/dtrace/node.d
    Target /usr/local/lib/dtrace/node.d
    already exists. You may want to remove it:
    rm '/usr/local/lib/dtrace/node.d'

    To force the link and overwrite all conflicting files:
    brew link --overwrite node
```

不用慌张，安照错误提示操作
```
rm '/usr/local/lib/dtrace/node.d'｀ 
brew link --overwrite node
```

重新运行命令发现node －v 安装成功了。npm －v后确认，重装成果。

node又恢复正常了！


---
title: Hexo-免费博客搭建使用讲解
date: 2016-03-18 15:40:40
categories: hexo 
tags: hexo
---

初识hexo就给人以眼前一亮的感觉, 查看资料到自己搭建个人博客, 简直是给人"带你装B,带你F"的快感,简单的博客生成操作, 多样化美观的主题选择, 功能强大的插件定制,关键是这些都是免费开源的,作为一个程序员,没有什么比遇到这种好使的软件更加给人已激动了.


![casper](http://img.blog.csdn.net/20160318142830765)

<!--more-->

![美到爆,有木有](http://img.blog.csdn.net/20160318142902109) ![想拥有吗?继续学习把...](http://img.blog.csdn.net/20160318143205032)

## 配置环境
安装Node（必须）作用：用来生成静态页面的, win\mac\linux都有相关版本自行到官网下载。
安装Git（必须）作用：作为一个21时间程序员,这个肯定大家都会用, 测试过程发现最好配置ssh, 体验会更好。

## 开发及配置

### 1. 安装hexo

			$ npm install -g hexo
	新版本需要安装git插件  $ npm install hexo-deployer-git --save
	
### 2. 初始化项目
新建一个你放hexo的新项目目录, cd到里面执行:

			$ hexo init
			$ npm install  #安装相关依赖

### 3. Demo生成及预览

		$ hexo generate #生成静态页面
		$ hexo server #启动本地预览服务
然后用浏览器访问http://localhost:4000/，此时，你应该看到了一个漂亮的博客了

### 4. 主题选择及下载
  hexo3.0使用的默认主题是landscape, 我们可以自行下载主题到theme目录下

		$ npm install <plugin-name> --save
		$ git clone <repository> themes/<theme-name>
安装失败情况可参考切换国内镜像源:
[nmp国内镜像](http://npm.taobao.org/)

无论是插件还是主题在安装后都需要在根目录下_config.yml中修改plugins和theme的值以启用他们。
```
fancybox - 是否启用Fancybox图片灯箱效果
duoshuo - 多说评论 shortname
disqus - Disqus评论 shortname
google_search - 默认使用Google搜索引擎
baidu_search - 若想使用百度搜索，将其设定为true
swiftype - Swiftype 站内搜索key
tinysou - 微搜索 key
self_search - 基于jQuery的本地搜索引擎，需要安装hexo-generator-search插件使用。
google_analytics - Google Analytics 跟踪ID
baidu_analytics - 百度统计 跟踪ID
shareto - 是否使用分享按鈕
busuanzi - 是否使用不蒜子页面访问计数
menu - 自定义页面及菜单，依照已有格式填写。填写后请在source目录下建立相应名称的文件夹，并包含index.md文件，以正确显示页面。导航菜单中集成了FontAwesome图标字体，可以在这里选择新的图标，并按照相关说明使用。
widgets - 选择和排列希望使用的侧边栏小工具。
links - 友情链接，请依照格式填写。
Static files - 静态文件存储路径，方便设置CDN缓存。
Theme version - 主题版本，便于静态文件更新后刷新CDN缓存。
```
* 可以在这里参考各种	[美到爆的主题](https://www.zhihu.com/question/24422335)

### 5.   发布到github上
配置根目录 _config.yml
```
	deploy:type: git   
    repository: https://your_github_url.git    
    branch: master
```

相关属性设置注释:
```
# Hexo Configuration
## Docs: http://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site	这下面的几项配置都很简单，你看我的博客就知道分别是什么意思
title: 常兴E站	#博客名
subtitle: Goals determine what you are going to be	#副标题
description: Goals determine what you are going to be #用于搜索，没有直观表现
author: changxing	#作者
language: zh-CN	#语言
timezone: 	#时区，此处不填写，hexo会以你目前电脑的时区为默认值

# URL	暂不配置，使用默认值
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory		暂不配置，使用默认值
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing	文章布局等，使用默认值
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  tab_replace:

# Category & Tag	暂不配置，使用默认值
default_category: uncategorized
category_map:
tag_map:

# Date / Time format	时间格式，使用默认值
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination	
## Set per_page to 0 to disable pagination
per_page: 10	#每页显示的文章数，0表示不分页
pagination_dir: page

# Extensions	插件配置，暂时不配置
## Plugins: http://hexo.io/plugins/
## Themes: http://hexo.io/themes/
plugins:
- hexo-generator-feed
theme: light	#使用的主题

feed:	#之后配置rss会用，此处先不配置这个
  type: atom
  path: atom.xml
  limit: 20  

# Deployment	用于部署到github，之前已经配置过
## Docs: http://hexo.io/docs/deployment.html

deploy: 
  type: git
  repository: https://your.git
  branch: master
```

执行命令上传到云端github上
> hexo deploy


----------
### 介绍几个hexo常用的命令,#后面为注释。
```
$ hexo g #完整命令为hexo generate,用于生成静态文件
$ hexo s #完整命令为hexo server,用于启动服务器，主要用来本地预览
$ hexo d #完整命令为hexo deploy,用于将本地文件发布到github上
$ hexo n #完整命令为hexo new,用于新建一篇文章
```

## 发表一篇文章

### 1. ```$ hexo new "my new post"```
### 2. 编辑 my-new-post.md

```
title: my new post #可以改成中文的，如“新文章”
date: 2015-04-08 22:56:29 #发表日期，一般不改动
categories: blog #文章文类
tags: [博客，文章] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
#这里是正文，用markdown写，你可以选择写一段显示在首页的简介后，加上<!--more-->，在<!--more-->之前的内容会显示在首页，之后的内容会被隐藏，当游客点击Read more才能看到。
```

### 3.  $ hexo g 生成静态文件
### 4.  $ hexo d 同步到github


----------
## 后续
[个人博客地址](http://cankingapp.github.io/)
[新浪微博](http://weibo.com/canking666)

**欢迎沟通学习**


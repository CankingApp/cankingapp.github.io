---
title: 视频转gif图片格式－好用的软件
date: 2016-03-30 19:07:51
categories: tool 
tags: ffmpeg
---

FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。采用LGPL或GPL许可证。它提供了录制、转换以及流化音视频的完整解决方案。它包含了非常先进的音频/视频编解码库libavcodec，为了保证高可移植性和编解码质量，libavcodec里很多codec都是从头开发的。

![支持格式](http://cankingapp.github.io/2016/03/30/ffmpeg/2016-03-30 19-10-29.png)

## 引文

博客中一直引用图片，感觉没有其他人博文中动态图更加有效果，一直以来因为懒,且markdown不支持视频，就一直延续着截图插入，今天发现一款便捷小巧的转化软件，命令行操作，非常方便，分享出来，也作为自己的博客记录．

## 安装命令

该软件是linux环境下一款视频处理软件，命令行操作，非常方便，安装方法如下：

`sudo apt-get install ffmpeg`

## 基本用法

软件功能十分强大，不仅能把视频转为图片，还能把图片换为适配，当然加水印什么的更不在话下．本文中只针对视频转化为gif格式图片讲解．

### 查看ffmpeg支持格式

`ffmpeg –formats`

![支持格式](2016-03-30 19-10-29.png)

从图片中可以看到gif的支持



### 转化指定时间端视频

转化本目录下的文件cangjinkong.wmv　6s后10s视频　为　*demo.gif*


`ffmpeg -ss 6 -t 10 -i cangjinkong.wmv -f gif ./demo.gif`

转化后效果如下： 


![支持格式](http://img.blog.csdn.net/20160330195113710)  ![支持格式](a.gif)


### 转化参数设置
你的源文件可能是1080P的高清视频，帧率可能还比较高。为了便于网络分享，GIF文件最好小一点。于是，我们需要使用-s参数来进行图像的缩放，使用-r参数来限制目标文件的帧率,帧率降到了1 fps（从源视频里每隔一秒抽取一帧图像输出到目标文件）。命令行如下：

`ffmpeg -ss  6 -t 10 -i cangjinkong.wmv -s 320x240 -f gif -r 1 ./demo.gif`

如果丢针效果不好怎么办? 我们可以分两部先均匀取出图片，然后转化为gif

* 首先，执行ffmpeg -ss 25 -t 10 -i cangjinkong.wmv -r 1 -s 320x240 -f image2 .\foo-%03d.jpeg，从源视频中每秒钟抽取一帧图像，保存为一系列JPEG文件。
* 然后，再执行ffmpeg -f image2 -framerate 5 -i .\foo-%03d.jpeg .\c.gif，将这一系列JPEG图像合成为帧率5 fps的GIF文件。

截取视频内任意时间点（比如第16.1秒处）的一帧图像保存为JPEG文件：

`ffmpeg -ss 16.1 -i cangjinkong.wmv -s 320x240 -vframes 1 -f image2 ./d.jpeg`


[百度百科介绍](http://baike.baidu.com/link?url=yeAKY2Zfi310bwaYfsjKzZMfrcnGYCnuFdRvC9QdWzIMXtxErZOUv_kMIQGA32fh6ufuYPD5G5Jh-GqVitBEX_)



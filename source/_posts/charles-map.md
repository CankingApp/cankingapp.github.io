---
title: 知识总结之 Charles抓包工具使用总结
date: 2017-07-03 20:05:24
tags: Charles
categories: Tool
---

Charles是一款HTTP/HTTPS协议流量包分析工具，适用于PC及各种移动设备。在使用过程中，有很多强大而方便的功能，总结下来，方便日后使用。

![车厘子？](http://www.canking.win/2017/07/03/charles-map/charles-logo.png)

<!--more-->

# Charles抓包工具使用总结

[官方功能定义:](https://www.charlesproxy.com/)

>Charles is an HTTP proxy / HTTP monitor / Reverse Proxy that enables a developer to view all of the HTTP and SSL / HTTPS traffic between their machine and the Internet. This includes requests, responses and the HTTP headers (which contain the cookies and caching information).

本文例子及场景主要来自移动设备上的数据包分析。

## 一、动态修改请求返回数据

为了不修改服务器端返回逻辑，不用修改移动软件中的数据请求逻辑，或者不用查看和分析业务代码，只要我们在Charles中找到目标请求接口，然后设置一些参数，这一切很方便的可以实现。

### 1.Map Local
本地映射，可以让请求数据直接取自己本地磁盘数据，本地数据自己可以动态修改，此方法灵活性强。

找到目标接口，然后右键，找到**Map local*.
![右键](locala.jpeg)

![右键](localmap.jpeg)


### 2.Map Remote

远程映射，可以动态将请求数据映射到别的接口上，同样的步骤，找到**Map remote**

![右键](locala.jpeg)

![右键](localmap.jpeg)

### 3.BreakPoints
上面两种方法是修改映射的方法，*BreakPoints*断点方法可以像代码断点调试一样，连接指定请求，并且修改请求返回数据。

同样也是找到目标请求接口->右键->BreakPoints

![右键](break.png)

点击**Edit Request**,即可修改目标参数。

修改完后，点击**Execute**，继续执行，返回到指定应用。


## 二、抓取https包

默认情况下，设置好代理地址，我们抓取到的包只能看到http的明文包，而对于https却是一堆乱码，下面以android手机为例，分析https的抓包方法。

> HTTPS(Hyper Text Transfer Protocol Secure)，是一种基于SSL/TLS的HTTP.HTTPS协议是在HTTP协议的基础上，添加了SSL/TLS握手以及数据加密传输，也属于应用层协议。

### 1. 安装SSL证书到目标手机上

*Help -> SSL Proxying -> Install Charles Root Certificate on a Mobile Device*


![install](chls1.png)

*弹窗提示，得到地址 chls.pro/ssl* 

![url](chls2.png)

*用手机浏览器打开上面地址，然后安装证书*

![phone](chls3.png)

### 2.Charles设置Proxy

![proxy](chls4.png)

找到SSL proxy 勾选*Enable SSL Proxying*

![enable ssl proxying](chls5.png)

添加要抓取的目标host地址及端口，如果抓取全部的可以用 \* 代替。端口号默认都是 *443*

### 3.重新获取数据

![succeed](chls6.png)

这里抓取的是*手机百度*信息流的数据，可以看到数据已经抓取到，可以看到汉字还做了编码处理。我们用chrome插件看下json如下：

[抓包手机百度视频信息流](https://mbd.baidu.com/searchbox?action=feed&cmd=184&service=bdbox&uid=0u27agiYvagpiv83j8vP8gaN-ig8uSu_livlfga4vi8Wuv8p_P2oi_uS2fgua2tDA&from=1019105l&ua=_a-qi4uq-igBNE6lI5me6NN0-I_Uh2Nl_uDvieLqA&ut=r93jO6kEBYyLaXiDouD58gN-WREu0qpGB&osname=baiduboxapp&osbranch=a0&pkgname=com.baidu.searchbox&network=1_0&cfrom=1000813a&ctv=2&cen=uid_ua_ut&typeid=0&sid=386_798-396_817-1000128_344-400_831-1001558_4349-1001430_3932-153_261-480_998-1001454_3997-488_1011-1001535_4266-373_775-1001460_4014-1001457_4004&imgtype=webp&refresh=1)

![baidu search](chls7.png)

ios系统设置步骤雷同，只要在按照手机证书步骤正确安装即可。





---
...
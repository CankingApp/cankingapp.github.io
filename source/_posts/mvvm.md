---
title: Lifecycle+Retrofit+Room完美结合 领略架构之美
date: 2017-12-09 15:37:25
categories: android 
tags: android
---

安卓开发技术发展到现在已经非常成熟，有很多的技术专项如插件，热修，加固，瘦身，性能优化，自动化测试等已经在业界有了完善的或者开源的解决方案。
作为一枚多年的安卓研发，有必要学习或了解下这些优秀的解决方案，领略那些行业*开创者*的思想魅力，然后转化为自己的技术技能，争取应用到日常的开发中去，提供自己研发水平。

<!--more-->

# Lifecycle+Retrofit+Room完美结合 云端漫步飞一般的感觉

安卓项目的开发结构，有原来最初的mvc，到后来有人提出的mvp，然后到mvvm的发展，无非就是依着*六大设计原则*的不断解耦，不断演变，使得项目的开发高度组件化，满足日常复杂多变的项目需求。

* 依赖倒置原则－Dependency Inversion Principle (DIP) 
* 里氏替换原则－Liskov Substitution Principle (LSP) 
* 接口分隔原则－Interface Segregation Principle (ISP) 
* 单一职责原则－Single Responsibility Principle (SRP) 
* 开闭原则－The Open-Closed Principle (OCP)

目前针对MVVM框架结构，[安卓官方](https://developer.android.google.cn/topic/libraries/architecture/adding-components.html)也给出了稳定的架构版本1.0。
本文也是围绕者官方思想总结的学习感受，最后将这些控件二次封装，更加便于理解使用。其也会涉及到其他优秀的的库，如Gson,Glide,BaseRecyclerViewAdapterHelper等


## 一.起因
去年还在手机卫士团队做垃圾清理模块时候，赶上模块化二次代码重构技术需求，项目分为多个进程，其中后台进程负责从DB中获取数据（DB可云端更新升级），
然后结合云端配置信息做相关逻辑操作，将数据回传到UI进程，UI进程中在后台线程中二次封装，最后post到主UI线程Update到界面展示给用户。

当时就和团队的同学沟通交流，面对这种数据多层次复杂的处理逻辑，设想可以做一种机制，将UI层的更新绑定到一个数据源，数据源数据的更新可以自动触发UI更新，实现与UI的解耦。
数据源控制着数据的来源，每种来源有着独立的逻辑分层，共享底层一些公共lib库。

后来想想，其实差不多就是MVVM思想，直到谷歌官方宣布[Android 架构组件 1.0 稳定版](https://mp.weixin.qq.com/s/9rC_5GhdAA_EMEbWKJT5vQ)的发布，才下决心学习下官方这一套思想，感受优秀的架构。


## 二.各组件库原理及基本用法

这里主要探究下主要组件库的基本用法和原理，以理解其优秀思想为主。

### 谷歌官方Android Architecture Components

>A collection of libraries that help you design robust, testable, and maintainable apps. Start with classes for managing your UI component lifecycle and handling data persistence.



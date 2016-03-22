---
title: Python入门-函数的使用到Python的发布安装
date: 2016-03-18 20:16:22
categories: python
tags: python
---

本文主要适合有一定编程经验，至少掌握一门编程语言的人查看。

文中例子大多都是简单到认识英文单词就能看懂的水平，主要讲的是Python的整体用法和结构，不会设计高深层次，对Python入门有一定帮助。 
<!--more-->

[Python和Java对比][2]，会看到Python设计思想在于简洁、实用、强大，每一个程序员都值得学习和掌握。

##Python函数的定义及实用
Python中的函数是一个命名的代码块，和Java一样，可以带0个或多个参数。主要形式如
``` python
def $函数名（$参数）：
    ...
    函数体
    ...
```
可以看出Python通过缩进语句代替了java中的{}，将代码归组到一起。
如Python中的基本语句：
``` python
for item in list:
    ...
    do something
    ...

while true:
     ...
     do something
     ...

if true:
     ...
else:
     ...
```

写一个通过参数的类型来打印不同的结果例子：
``` python
###如果是一个列表类型，则循环打印，否则打印当前
def print_test(is_list):
     if isinstance(is_list, list):
           for t in is_list:
                 print(t)
                 print_test("not list")
     else:
           print(the_list)

```
Python中的列表可以理解卫java中的列表，元组看成java中的数组（用小括号扩住），貌似比数据更强大和简洁一点，我们可以理解为“打了鸡血”的数据，可以随便伸缩，相关方法有：
>len(list)
>list.insert（1，‘’）
>list.remove('')
>list.append('')

上述实例中，用到递归调用，更具传入参数类型类递归调用自己。可以看到，方法名字前就加了def修饰，参数也是直接随便写。**Python设计哲学把任何事物都看成了对象或集合，类型并不关心内部到底是什么类型，变量标识符根本不需要类型，java中则声明变量时必须要表明类型。可以把Python看成高层集合，对于列表来说，里面可以存储不同类型的数值，只要你给出一个名字，其他的由Python搞定**

例子中isinstance 函数为Python内置函数，和java中的 instanceof 类似。

函数的调用，保存method.py, F5运行后，直接在shell和idle中键入：
``` python
### 句未加‘；’ 和写多行句子
import method.py
print_test(["item1","item2","item3"])
```

##Python程序的发布和安装

模块化Python代码，像java一样，可以构建复杂而强大的系统。把Python代码模块化为类库，方便管理，业方便后续的代码重用和架构。
>import sys； sys.path 产看python在计算机上存储位置。

把上例函数封装为一个模块，然后发布安装为例：

- **为刚写的方法文件建立一个文件夹：method**
      把method.py 放到里面
- **新建立一个文件 “setup.py”**
    文件中为发布的元数据，编辑如下：
    
``` python
# 元数据
from distutils.core import setup

setup(
       name       = 'CankingApp',
       version    = '1.0',
       py_modules = ['method'],
       author     = 'CankingApp',
       author_email = 'king@gmail.com',
       url        = 'www.baidu.com',
       descripthin= 'test',      
 )
```

- **构建发布文件**
  打开终端键入命令：
  > $python3 setup.py sdist
  >running sdist
  >running check
  >warning: check: missing required meta-data: url
  >warning: sdist: manifest template 'MANIFEST.in' does not exist (using default file list)
  >warning: sdist: standard file not found: should have one of README, README.txt
  >writing manifest file 'MANIFEST'
  >creating CankingApp-1.0
  >making hard links in CankingApp-1.0...
  >hard linking hello.py -> CankingApp-1.0
  >hard linking setup.py -> CankingApp-1.0
  >Creating tar archive
  >removing 'CankingApp-1.0' (and everything under it)

  
- **按装到Python本地副本中**
终端中命令：
>$ sudo setup.py install
>/usr/lib/python3.4/distutils/dist.py:260: UserWarning: Unknown distribution option: 'descripthin'
  warnings.warn(msg)
>running install
>running build
>running build_py
>creating build
>creating build/lib
>copying method.py -> build/lib
>running install_lib
>copying build/lib/method.py -> /usr/local/lib/python3.4/dist-packages
>byte-compiling /usr/local/lib/python3.4/dist-packages/method.py to method.cpython-34.pyc
>running install_egg_info

操作完后会看到文件夹中多了**build**和**dist**文件夹及**MANIFEST**文件。

- **构建成功，测试代码**

直接在idle中测试：
``` python
import method
method.print_test(["item1","item2","item3"])
```
测试函数调用需要加上method，是python中命名空间规定。
Python中的所有代码都与一个命名空间关联，主程序中的代码与"_main_"命名空间关联。我们单独的代码模块自然自动创建一个与代码块同名的命名空间。所以需要带上method.print_test。
>from method import print_test
>print_test()
>//也可以这样用，但是如果此命名空间有同名时会冲突失效，个人认为还是第一种比较好。


**成功打印出item则标识安装成功。**

文中实例源代码已上传[GitHub][1],  有兴趣的同学欢迎一起交流学习。

[csdn](http://blog.csdn.net/cankingapp/article/details/46381781)
---------

[1]: https://github.com/CankingApp/Python
[2]: http://developer.51cto.com/art/201003/187962.htm



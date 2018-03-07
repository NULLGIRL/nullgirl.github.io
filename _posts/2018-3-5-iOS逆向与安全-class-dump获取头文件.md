---
layout: post
title: iOS逆向与安全 - class-dump获取头文件
date: 2018-3-5
categories: blog
tags: [iOS逆向与安全]
description: 今天就来获取头文件吧。
---

class-dump 一款从已砸壳的可执行文件中列举出**类名**与**公开方法**的工具

****
##开始安装
[class-dump下载地址](http://stevenygard.com/projects/class-dump/)

下载完毕后
去这里看怎么进行安装 [class-dump最新安装方法](http://bbs.iosre.com/t/10-11-usr-bin-class-dump/1936)

安装完成后，打开终端，输入class-dump
![成功安装](http://img.blog.csdn.net/20180305174526683?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
安装成功后，如上图显示。


##获取文件
终端输入
```
class-dump -H 砸壳后的可执行文件路劲 -o 存储文件的路径
```

-H : 获取头文件
-o : 输出到

![文件名集合](http://img.blog.csdn.net/20180305175011453?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

class-dump 也会把.m文件中的头文件也dump出来 所以看到的头文件特别多

选择其中一个文件看看：
![文件之一](http://img.blog.csdn.net/20180305175215380?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


##问题
**如何避免头文件class-dump被其他程序员赤裸裸地观看？**
即如何做代码混淆。

请看大神念茜的文章：

http://blog.csdn.net/yiyaaixuexi/article/details/29201699


感谢。







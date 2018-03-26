---
layout: post
title: iOS逆向与安全 - Reveal查看页面层次结构
date: 2018-3-12
categories: blog
tags: [iOS逆向与安全]
description: 来看看App界面的结构层次吧。
---

# 简介
`Reveal` 是一个界面调试工具，可以在iOS开发时动态地查看和修改应用程序的界面。不但可以在运行时看到iOS程序的界面层级关系，也可以实时地修改程序界面，不需要重新运行程序即可看到效果。

****

# 下载

去[Reveal官网](http://revealapp.com/)下载Reveal试用版（土豪随意）。

****

# 使用介绍
## 1. 自己应用内使用（不逆向对Reveal的使用）
### -手动导入
1.  打开Reveal，在菜单栏 --> Help --> Show Reveal Library in Finder --> iOS Library ，将RevealServer.framework拷贝出来。
![取出Reveal动态库](http://img.blog.csdn.net/20180312160050374?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

2.  打开自己工程，将RevealServer.framework拖入工程内，在配置项，General --> Embedded Binaries --> 引入RevealServer.framework

3.  --> Build Settings --> 输入other linker flags --> 添加 -ObjC 和 -l"z"

4.   --> Build Phase  --> 添加 libz.tbd 库。

5. 运行工程后，打开Reveal， 选择运行的程序，可以开始进行页面的修改。
![选择程序](http://img.blog.csdn.net/20180312160845420?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

6. **实时修改前**

![实时修改前](http://img.blog.csdn.net/2018031215544390?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**实时修改后**

![实时修改后](http://img.blog.csdn.net/20180312155457277?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
在使用时，我们将Reveal连接上模拟器或真机上正在运行的iOS程序，然后就可以查看和调试iOS程序的界面。

### -通过 CocoaPods 导入

1.  Podfile 文件添加 pod 'Reveal-SDK', :configurations => ['Debug']
2.  终端执行 pod update 或 pod install
3.  运行 APP, 就可以在 Reveal 中查看你的 APP 界面布局了, 并且只有在 debug 模式下才能查看, 如果是 release 模式则不能
（通过这种方式集成的 Reveal, 可以查看模拟器和真机上 APP 界面布局。）


## 2.逆向分析中App对Reveal的使用

在Cydia中下载Reveal2Loader，安装后，--> 设置 --> Reveal --> Enable Applications --> 选择你要打开的应用
![设置中的Reveal](http://img.blog.csdn.net/20180312172141353?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![选择TIM](http://img.blog.csdn.net/20180312172159808?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

打开终端, ssh连接越狱设备
```
ssh root@192.168.xx.xx
```
输入密码，连接设备后，手机上可以打开TIM，电脑上打开Reveal软件，
此时如下图， 发现越狱设备中的RevealServer.framework版本过低，
![版本过低1](http://img.blog.csdn.net/20180312172424477?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![提示版本过低](http://img.blog.csdn.net/20180312172451875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

`版本过低解决方案`
1.  打开Reveal，在菜单栏 –> Help –> Show Reveal Library in Finder –> iOS Library ，将RevealServer.framework拷贝出来。
2.  打开IFunBox， 找到路径， --> Library --> Frameworks --> 删除原有的RevealServer.framework  --> 将 Step1 的RevealServer.framework拷贝进去
3.  连接越狱设备后，输入killall SpringBoard重启手机

```
killall SpringBoard
```

![重启越狱机](http://img.blog.csdn.net/20180312172902828?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

4.  手机再次打开TIM，Mac电脑再次打开Reveal，会发现

![程序1](http://img.blog.csdn.net/20180312173004691?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

选中一个进入，修改界面View的背景颜色：
![这里写图片描述](http://img.blog.csdn.net/20180312173032550?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

此时手机上TIM也跟着改变颜色：
`改变其中View背景颜色前`
![改变前](http://img.blog.csdn.net/20180312173125750?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

`改变其中View背景颜色后`
![改变后](http://img.blog.csdn.net/20180312173145420?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYm9yaW5nX2NhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


****
Reveal 还是很方便让我们知道了app的结构层次。今天才知道TIM登录页是这样构成的哈哈。

感谢。



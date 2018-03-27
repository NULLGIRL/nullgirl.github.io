---
layout: post
title: iOS逆向与安全 - Cycript脚本使用
date: 2018-3-20
categories: blog
tags: [iOS逆向与安全]
description: 使用Cycript脚本语言注入程序，修改界面布局方法等。
---

# 一、Cycript介绍及安装
### 简介

Cycript是ECMAScript some-6，Objective-C ++和Java的混合体。它以Cycript-to-JavaScript编译器的形式实现，并为其虚拟机使用（未修改的）JavaScriptCore。它集中于通过采用其语法和语义的方面而不是将另一种语言作为二等公民来提供其他语言的“流利的FFI”。

Cycript的主要用户是目前在iOS上进行逆向工程工作的人员。Cycript具有一个高度交互式控制台，该控制台具有实时语法高亮显示和语法辅助选项卡完成功能，甚至可以使用Cydia Substrate将其注入正在运行的进程（类似于调试程序）。这使它成为“spelunking”的理想工具。

然而，Cycript是专门为编程环境设计的，并且对此用例保留很少（如果有的话）“包袱”。来自node.js的许多模块可以加载到Cycript中，同时它也可以直接访问为Objective-C和Java编写的库。因此它非常适合作为脚本语言。


### 安装
越狱机打开Cydia，搜索Cycript安装


# 二、Cycript脚本命令的简单使用

##  1. 选择器和消息
Objective-C不是“调用方法”，而是基于 `“发送消息”` 的思想。这些消息的语法涉及一组方括号内的参数的关键字中缀表示法。消息的名称称为“选择器”，Objective-C编译器将这些转换为对objc_msgSend的调用。Cycript也是如此。

```
/** 方式1  Cycript自动地从字符串中构建一个选择器*/
cy# ?debug
debug == true
/** 第2个参数，实际上是一个C字符串*/
cy# [@"hello" stringByReplacingOccurrencesOfString:@"ell" withString:@"ipp"]
cy= objc_msgSend(Instance.box("hello"),"stringByReplacingOccurrencesOfString:withString:",Instance.box("ell"),Instance.box("ipp"))
@"hippo"


/** 方式2  通过@selector获得一个实际的Selector对象*/
cy# ?debug
debug == true
cy# @selector(this:is:a:message:)
cy= sel_registerName("this:is:a:message:")
@selector(this:is:a:message:)


/** 调用： 使用.call来向对象发送一个任意的消息，这通常比Cycript的objc_msgSend更好*/
cy# var capitalize = @selector(capitalizedString)
cy# capitalize.call(@"hello")
@"Hello"
```

##  2.  获取对象

```
/** 方式1  使用 #对象地址*/
cy# s = #0x10e803490

/** 方式2  使用 #对象地址*/
cy# var p = new Instance(0x8614390)
```


## 3. choose在内存中找出属于这个类的对象

```
cy# choose(SBIconModel)
[#"<SBIconModel: 0x1590c8430>"]

cy# var views = choose(SBIconView)
[#"<SBIconView: 0x159460fa0; frame = (27 92; 60 74); opaque = NO; gestureRecognizers = <NSArray: 0x159518ae0>; layer = <CALayer: 0x159461220>>",#"<SBIconView: 0x159468e50; frame = (114 356; 60 74); opaque = NO; gestureRecognizers = <NSArray: 0x15946d2f0>; layer = <CALayer: 0x1592c9a70>>",...
cy# for (var view of views) if ([[[view icon] application] displayName] == "Photos") photos = view; photos;
#"<SBIconView: 0x15fc75e90; frame = (201 4; 60 74); opaque = NO; gestureRecognizers = <NSArray: 0x15fbfacc0>; layer = <CALayer: 0x15fc76110>>"
```

## 4. 打印当前界面视图层次

```
UIApp.keyWindow.recursiveDescription().toString()
```

## 5. 根据类获取方法

```
/** 获取所有方法*/
function printMethods(className, isa) {
var count = new new Type("I");
var classObj = (isa != undefined) ? objc_getClass(className)->isa :
objc_getClass(className);
var methods = class_copyMethodList(classObj, count);
var methodsArray = [];
for(var i = 0; i < *count; i++) {
var method = methods[i];
methodsArray.push({selector:method_getName(method),
implementation:method_getImplementation(method)});
}
free(methods);
return methodsArray;
}
```

## 6. 获取当前控制器

```
function currentVC() {
var app = [UIApplication sharedApplication]
var keyWindow = app.keyWindow
var rootController = keyWindow.rootViewController
var visibleController = rootController.visibleViewController
if (!visibleController){
return rootController
}
return visibleController.childViewControllers[0]
}
var vc = currentVC()
```

## 7. 创建和调用block

```
cy# block = ^ int (int value) { return value + 5; }
^int(int){}
cy# block(10)
15
```

[更多使用方法可以去官网查看： http://www.cycript.org/manual/](http://www.cycript.org/manual/)



# 三、实战

##  1.  将TIM背景隐藏

`先来看一下效果图`

![效果图](http://nullgirl.com/img/Posts/20180320/20180320162938119.png)

`操作步骤`

```
/** 确认进程名或者PID */
ZYs-iPhone5s:~ root# ps -e | grep TIM
PID TTY           TIME CMD
7277 ??         0:25.32 /var/mobile/Containers/Bundle/Application/A25EBB56-899F-41F8-BCCE-4FB645F5253E/TIM.app/TIM
7432 ttys000    0:00.02 grep TIM
ZYs-iPhone5s:~ root# cycript -p 7277

/** 打开方式 1*/
cycript -p TIM
/** 打开方式2  勾住进程id */
cycript -p 7277

/** 打印当前视图层次 */
cy# UIApp.keyWindow.recursiveDescription().toString()

/** 隐藏某个视图*/
cy# #0x1743f8a00.hidden = YES

/** 修改某个视图的背景颜色*/
#0x140035040.backgroundColor = [UIColor redColor]

/** 退出cycript */
Control + D
```

![操作图](http://nullgirl.com/img/Posts/20180320/20180320114710688.png)


## 2.在手机屏保页面弹出弹窗口和截屏

`弹窗效果图`

![弹窗效果图](http://nullgirl.com/img/Posts/20180320/20180320162054729.png)

`cycript 运行代码图`

![cycript 运行代码图](http://nullgirl.com/img/Posts/20180320/20180320162200954.png)

注意要输错方法。

```
Last login: Tue Mar 20 14:03:30 on ttys000

/** ssh连接越狱机*/
momozhuangdeMacBook-Pro:~ zhuangyuan$ ssh root@192.168.31.53
root@192.168.31.53's password:

/** 注入屏幕进程*/
ZYs-iPhone5s:~ root# cycript -p SpringBoard
/** 初始化弹窗 （写错了一个单词）*/
cy# var alert = [[UIAlertView alloc] initWithTitle:@"Title" message:@"hello ZY" delegate:nil cancelBtnTitle:@"cancel" otherBttonTitles:nil]
/** 由于写错而报错*/
throw new Error("unrecognized selector initWithTitle:message:delegate:cancelBtnTitle:otherButtonTitles: sent to object 0x1471d2a80") /*
objc_msgSend@[native code] */
/** 初始化弹窗 （正确写法）*/
cy# var alert = [[UIAlertView alloc] initWithTitle:@"Title" message:@"hello ZY" delegate:nil cancelButtonTitle:@"cancel" othrButtonTitles:nil]
#"<UIAlertView: 0x1471b12c0; frame = (0 0; 0 0); layer = <CALayer: 0x174e2c380>>"
/** 屏幕展示弹窗*/
cy# [alert show]
/** 获取截屏单例*/
cy# var shot = [SBScreenShotter sharedInstance]
#"<SBScreenShotter: 0x17082a9c0>"
/** 截屏保存在相册*/
cy# [shot saveScreenshot:YES]
cy#
```


# 四、其他-链接分享

- [cycript官网：http://www.cycript.org/](http://www.cycript.org/)
- [cycript手册：http://www.cycript.org/manual/](http://www.cycript.org/manual/)
- [cycript使用技巧：http://iphonedevwiki.net/index.php/Cycript_Tricks](http://iphonedevwiki.net/index.php/Cycript_Tricks)



感谢观看！


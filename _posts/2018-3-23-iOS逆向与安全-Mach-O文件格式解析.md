---
layout: post
title: iOS逆向与安全 - Mach-O文件格式解析
date: 2018-3-23
categories: blog
tags: [iOS逆向与安全]
description: 使用Cycript脚本语言注入程序，修改界面布局方法等。
---

### 前言
> 1. [Mach-O文件格式源码](https://opensource.apple.com/source/xnu/xnu-1456.1.26/EXTERNAL_HEADERS/mach-o/loader.h)
> 2. [Mach-O苹果官方手册](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/MachOTopics/0-Introduction/introduction.html#//apple_ref/doc/uid/TP40001827-SW1)

想要程序跑起来，那么这个可执行文件的格式就需要被当前的操作系统所理解，比如：
- Linux 操作系统下可执行文件格式是 ELF
- Windows 的可执行文件格式是 PE32/PE32+
- Android 的可执行文件格式是 ELF
-  **OSX和iOS** 的可执行文件格式是 **`Mach-O`**

----------
# 准备工具
## 方式1:   终端查看
来到Mach-O文件所在位置，输入相关命令得到Mach-O文件信息。为了更直观点，推荐方式2查看Mach-O文件。

## 方式2： 工具查看
首先要下载一个可以查看Mach-O文件格式的工具，本来想用[MachOView](https://github.com/gdbinit/MachOView)，无奈下载完后打开Mach-O文件后，会闪退，就去逆向论坛找了个大神些的替代MachOView的工具[MachOExplorer](https://github.com/everettjf/MachOExplorer/releases/tag/v0.4.0)。我这里下载使用的是MachOExplorer，下载完后，打开，点击菜单栏的 file -> Mach-O文件 （此处我用模拟器的运行，打开了Debug-iphonesimulator文件夹，找到ipa，邮件显示包内容后可以获得可执行文件）。

-------

# Mach-O文件是什么
在OSX和iOS系统下，平时接触到的可执行文件、库文件、dsym文件、动态库、动态链接器（dyld）都是这种格式。Mach-O的组成结构包括：`Header`(头部)、`Load commands`（加载命令）、`Data`（Data包含多个`Segment`（段），Segment中包含多个`Section`（节））

![Mach-O文件格式](https://img-blog.csdn.net/20180323152821659?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

简单介绍dsym文件，后续开篇介绍。
>  - **dSYM 文件是什么：** Xcode编译项目后，会有一个项目同名的 dSYM 文件，dSYM 是保存 16 进制函数地址映射信息的中转文件。
>  -  **dSYM 文件的作用：** release 模式打包或上线后，崩溃错误不直观，这时就需要分析 crash report 文件，iOS 设备中会有日志文件保存我们每个应用出错的函数内存地址，通过 Xcode 的 Organizer 可以将 iOS 设备中的 DeviceLog 导出成 crash 文件，这个时候我们就可以通过出错的函数地址去查询 dSYM 文件中程序对应的函数名和文件名。大前提是我们需要有软件版本对应的 dSYM 文件，这也是为什么我们很有必要保存每个发布版本的 Archives 文件了。

## Header (头部)
### header的数据结构
Mach-O的头部信息，可以使我们快速得到一些信息，比如
32位结构还是64位结构，比如文件类型架构类型等等。
让我们先来看看header的数据结构定义。

```
/** 32位架构对于header的定义*/
struct mach_header {
uint32_t    magic;      /* mach magic number identifier */
cpu_type_t  cputype;    /* cpu specifier */
cpu_subtype_t   cpusubtype; /* machine specifier */
uint32_t    filetype;   /* type of file */
uint32_t    ncmds;      /* number of load commands */
uint32_t    sizeofcmds; /* the size of all the load commands */
uint32_t    flags;      /* flags */
};

/** 64位架构对于header的定义*/
struct mach_header_64 {
uint32_t    magic;      /* mach magic number identifier */
cpu_type_t  cputype;    /* cpu specifier */
cpu_subtype_t   cpusubtype; /* machine specifier */
uint32_t    filetype;   /* type of file */
uint32_t    ncmds;      /* number of load commands */
uint32_t    sizeofcmds; /* the size of all the load commands */
uint32_t    flags;      /* flags */
uint32_t    reserved;   /* reserved */
};
```
32位和64位架构的头文件对比，多了一个reserved（保留字段）。
现在来解释说明一下这些字段都有什么意义：
- magic ： 魔数，用于快速确认该文件是64位还是32位的。若值为0xfeedfacf是64位的，值为0xfeedface则是32位的。
- cputype ：CPU架构类型。比如arm，例如x86_64
- cpusubtype : 对应的架构具体类型。比如arm64、armv7等。
- filetype ： 文件类型。比如可执行文件（MH_EXECUTE）、库文件、Dsym文件等。
- ncmds ： 加载命令条数。
- sizeofcmds ： 所有加载命令大小。
- flags ： dyld加载时需要的标志位。
- reserved ：保留字段。
```
/* filetype 类型 */
#define  MH_OBJECT   0x1     /* relocatable object file */
#define  MH_EXECUTE  0x2     /* demand paged executable file */
#define  MH_FVMLIB   0x3     /* fixed VM shared library file */
#define  MH_CORE     0x4     /* core file */
#define  MH_PRELOAD  0x5     /* preloaded executable file */
#define  MH_DYLIB    0x6     /* dynamically bound shared library */
#define  MH_DYLINKER 0x7     /* dynamic link editor */
#define  MH_BUNDLE   0x8     /* dynamically bound bundle file */
#define  MH_DYLIB_STUB   0x9     /* shared library stub for static */
#define  MH_DSYM     0xa     /* companion file with only debug */
#define  MH_KEXT_BUNDLE  0xb     /* x86_64 kexts */

/* flags ： dyld加载时需要的标志位。*/
#define       MH_NOUNDEFS 0x1     // 目前没有未定义的符号，不存在链接依赖
#define    MH_DYLDLINK  0x4     // 该文件是dyld的输入文件，无法被再次静态链接
#define    MH_PIE 0x200000     // 加载程序在随机的地址空间，只在 MH_EXECUTE中使用
#define    MH_TWOLEVEL  0x80    // 二级名称空间
```

**dyld**
动态链接器，苹果的开源项目，[下载](https://link.jianshu.com/?t=http://opensource.apple.com/tarballs/dyld/dyld-360.18.tar.gz) ，当内核执行到`LC_DYLINK`（后文讲述）的时候，链接器会启动，查找进程所依赖的动态库，并加载到内存中。

**flags ： MH_PIE - 随机地址空间（ASLR）**
进程每一次启动，地址空间都会随机化。如果采用传统的方式，程序每启动一次，启动的虚拟内存镜像一致的话，黑客很容易就重写内存来破解程序。所以，ASLR可以有效避免黑客的攻击。

打开Xcode，来到Main函数，打断点，运行程序开启lldb调试。当到达断点位置时，在控制台输入：
```
/** 加载模块地址*/
image list -o -f
```
可以发现每次运行程序，地址都在变化。

**flags ： MH_TWOLEVEL - 二级名称空间**
这是dyld的一个独有特性，说是符号空间中还包括所在库的信息，这样子就可以让两个不同的库导出相同的符号，与其对应的是平摊名称空间.


### 演示查看header结构
**`方式1： 通过命令行来查看Mach-O文件的header结构`**

```
/* file + 可执行文件路径 ： 查看文件类型 */
zhuangyuan$ file GV_VOUCHER_CN
-> Mach-O 64-bit executable x86_64

/* lipo -info + 可执行文件路径 ： 查看文件架构 */
zhuangyuan$ lipo -info GV_VOUCHER_CN
-> Non-fat file: GV_VOUCHER_CN is architecture: x86_64

/* otool -h + 可执行文件路径 ： 查看Mach-O文件的header信息 */
zhuangyuan$ otool -h GV_VOUCHER_CN
->
Mach header
magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
0xfeedfacf 16777223          3  0x00           2    51       6184 0x00218085

/* otool -hv + 可执行文件路径 ： 查看Mach-O文件的header信息的翻译*/
zhuangyuan$ otool -hv GV_VOUCHER_CN
->
Mach header
magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64  X86_64        ALL  0x00     EXECUTE    51       6184   NOUNDEFS DYLDLINK TWOLEVEL WEAK_DEFINES BINDS_TO_WEAK PIE
```
如下图：
![Mach-O头部文件说明](https://img-blog.csdn.net/2018032316184636?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


**`方式2： 利用MachOExplorer工具查看Mach-O文件的header结构`**
![MachOExplorer工具查看header信息](https://img-blog.csdn.net/20180323162054558?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

比终端看的更直观看，终端的好处就是装逼还挺成功的，哈哈。


-----------


## Load commands 结构 (加载指令模块)

Load commands 紧跟在header之后，说明了操作系统应当如何加载文件中的数据，对系统内核加载器和动态链接器起了指导性作用。这些加载指令告诉`加载器`如何处理二进制数据，有些命令是由内核处理的，有些是由动态链接器处理的。在源码中有明显的注释说明这些是动态链接器处理的。
### Load commands
```
struct load_command {
uint32_t cmd;       /* type of load command */
uint32_t cmdsize;   /* total size of command in bytes */
};
```
- cmd： 加载指令类型
- cmdsize ：加载指令大小
![Load commands](https://img-blog.csdn.net/20180323173438176?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### Segments
**数据结构**
```
/*segment_command_64 数据结构，其他段的数据结构类似，感兴趣可以去阅读源码*/
struct segment_command_64 { /* for 64-bit architectures */
uint32_t    cmd;        /* LC_SEGMENT_64 */
uint32_t    cmdsize;    /* includes sizeof section_64 structs */
char        segname[16];    /* segment name */
uint64_t    vmaddr;     /* memory address of this segment */
uint64_t    vmsize;     /* memory size of this segment */
uint64_t    fileoff;    /* file offset of this segment */
uint64_t    filesize;   /* amount to map from the file */
vm_prot_t   maxprot;    /* maximum VM protection */
vm_prot_t   initprot;   /* initial VM protection */
uint32_t    nsects;     /* number of sections in segment */
uint32_t    flags;      /* flags */
};
```
**结构字段说明**
`LC_SEGMENT_64`和`LC_SEGMENT`是加载的主要命令，负责指导内核来设置进程的内存空间。

1. cmd : 即 Load commands 的类型，比如`LC_SEGMENT_64`代表将文件中64位的段映射到进程的地址空间。还有LC_SEGMENT、LC_DYLD_INFO_ONLY等等类型。
2. cmdsize ： 代表当前 'load command' 的大小
3. VM Address ：段的虚拟内存地址
4. VM Size ： 段的虚拟内存大小
5. file offset： 段在Mach-O文件中的偏移量
6. file size ： 段在文件中的大小
7. nsects ： 标识了Segment中有多少section
8. segname[16] ： 段的名称。例如__PAGEZERO、__TEXT等等。

将段对应的文件内容加载到内存中的流程：
**从file offset处 加载 file size 大小到 虚拟内存VM Address处**。如果当前段是LC_SEGMENT_64(__PAGEZERO) ，则这个段的file offset、file size 、VM Address为0，因为这个段不具备访问权限，用来处理空指针的。

**具体段说明**
- LC_SEGMENT_64(__PAGEZERO) : 零页,捕获空指针。
- LC_SEGMENT_64(__TEXT) :  代码段
- LC_SEGMENT_64(__DATA) :  可读/可写数据存放段。
- LC_SEGMENT_64(__LINKEDIT) :  链接的部分，支持dyld，包含了一些符号表等数据。
- LC_DYLD_INFO_ONLY、LC_SYMTAB、LC_DYSYMTAB：符号表和动态符号表
- LC_LOAD_DYLINKER ： 使用Mach-O文件的时候链接器，可以看到name为 /usr/lib/dyld 的链接器来加载Mach-O文件。
- LC_UUID ： 对每个Mach-O文件都是唯一值，这个值可以在我们分析崩溃文件的时候，通过它对应上一个符号文件。
- LC_VERSION_MIN_IPHONEOS ： 最低ios运行的系统版本 ，打开xcode ，输入iOS Development Target 可以查看版本
- LC_MAIN ： 可执行文件的入口地址
- LC_RPATH ： 环境变量路径
- LC_FUNCTION_STARTS ： 函数开始的地方
- LC_CODE_SIGNATURE ： 记录可执行文件的代码是否签名

### Section
大写的表示Segment，小写的表示section。
例如__TEXT 和 __text.

**数据结构**
```
struct section { /* for 32-bit architectures */
char        sectname[16];   /* name of this section */
char        segname[16];    /* segment this section goes in */
uint32_t    addr;       /* memory address of this section */
uint32_t    size;       /* size in bytes of this section */
uint32_t    offset;     /* file offset of this section */
uint32_t    align;      /* section alignment (power of 2) */
uint32_t    reloff;     /* file offset of relocation entries */
uint32_t    nreloc;     /* number of relocation entries */
uint32_t    flags;      /* flags (section type and attributes)*/
uint32_t    reserved1;  /* reserved (for offset or index) */
uint32_t    reserved2;  /* reserved (for count or sizeof) */
};
```

**结构字段说明**
- sectname：比如_text、stubs
- segname ：该section所属的segment，比如__TEXT
- addr ： 该section在内存的起始位置
- size： 该section的大小
- offset： 该section的文件偏移
- align ：字节大小对齐
- reloff ：重定位入口的文件偏移
- nreloc： 需要重定位的入口数量
- flags：包含section的type和attributes

**主要的节说明**
- __text：主程序代码
- __stub_helper ：用于动态链接器的存根
- __symbolstub1 ： 用于动态链接器的存根
- __objc_methname ：OC的方法名
- __objc_classname ： OC的类名
- __cstring ： 字符串


### 演示查看load commands结构
**`方式1： 终端`**

**查看Mach-O文件的所有数据**
终端输入：
```
otool -lv + 文件
```
![终端查看Mach-O文件数据](https://img-blog.csdn.net/2018032318183249?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**查看__text 节的全部内容(二进制)**
```
otool -s __TEXT __text + 文件
```
![查看__text 节的内容](https://img-blog.csdn.net/20180323182833742?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**查看__text 节的全部内容（汇编**
![查看__text 节的内容（汇编）](https://img-blog.csdn.net/2018032318310184?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**查看__text 节的内容前10条数据（汇编）**
![查看__text 节的内容前10条数据](https://img-blog.csdn.net/2018032318313055?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



**`方式2： 利用MachOExplorer工具`**
![MachOExplorer查看Mach-O文件数据](https://img-blog.csdn.net/20180323181955864?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



------------
## 演示部分
### Mach-O文件开始加载的地方
1.  查看__TEXT部分的虚拟地址0x100000000
2.  打开xcode 来到main函数打断点 ，输入 image list -o -f  查看模块加载的地址，可以看到加载可执行文件的基地址 0x000fabcd
3.  在控制台输入 p/x 0x000fabcd + 0x100000000  得到了一个新的地址$0
4.  选择xcode菜单的Debug  —> debug workflow —>  view memory  之后再Address里面输入$0地址，回车
5. 可以看到Mach-O文件格式的一些数据 ,所以0x100000000就是Mach-O文件加载开始的地方.

![查看__TEXT的虚拟地址](https://img-blog.csdn.net/20180323180958474?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![查看基地址](https://img-blog.csdn.net/20180323181124813?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![查看新地址内容](https://img-blog.csdn.net/20180323181145659?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvcmluZ19jYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


本文参考了的文章：
[趣探 Mach-O：文件格式分析](https://www.jianshu.com/p/54d842db3f69)

感谢。

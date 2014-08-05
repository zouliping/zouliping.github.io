---
layout: post
title: NDK在Mac OS X下的安装配置
category: NDK
tags: [NDK]
---

###JNI

JNI是Java Native Interface，用来沟通Java代码和C/C++代码的桥梁。通过JNI，Java代码可以调用C/C++代码，且C/C++代码也可以回调Java代码。

###使用JNI的场景

为什么使用JNI？一般是以下几个原因：

* 要求高性能
* 使用现有的C/C++库

###NDK

NDK是Native Development Kit，Google开发的一套开发和编译工具集，用于Android的JNI开发。

###安装

1. 安装前必须确认SDK已经安装。
2. 从[官网](https://developer.android.com/tools/sdk/ndk/index.html#Installing)上，选择合适的最新版本的NDK安装包下载，解压并放在合适的位置上。
3. 设置环境变量，在.bash_profile中添加`export PATH=$PATH:/Users/xxx/Documents/android-ndk-r10`。
4. 执行 `source ~/.bash_profile`	使其立即生效，通过执行`ndk-build`查看是否生效。

###配置ADT

1. 打开Preference->Android->NDK，选择NDK所在的位置。
2. 可安装Help->Install New Software，安装C/C++ Development Tools。

###确认

1. 导入ndk文件夹中的HelloJni项目。
2. 右键该项目，选择Android Tools->Add Native Support，接受默认的名字，点击完成。
3. 从命令行进入该项目文件夹，执行`ndk-build`。
4. 编译运行。

小问题：在项目使用Android 2.3时，出现`java.nio.BufferOverflowException`异常，改到Android 4.4，则可以运行。

如果成功安装和配置了NDK，运行结果如下：

![安装成功]({{ site.url }}/assets/images/ndk1.jpg)







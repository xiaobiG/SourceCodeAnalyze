# JNI概述

JNI（Java Native Interface，Java本地接口），用于打通Java层与Native(C/C++)层。Java语言是跨平台的语言，而这跨平台的背后都是依靠Java虚拟机，虚拟机采用C/C++编写，适配各个系统，通过JNI为上层Java提供各种服务，保证跨平台性。

在Android中JNI是Java上层与Native的纽带。

## JNI注册

JNI注册的两种时机：

- Android系统启动过程中Zygote注册，可通过查询AndroidRuntime.cpp中的gRegJNI，看看是否存在对应的register方法；
- 调用System.loadLibrary()方式注册。
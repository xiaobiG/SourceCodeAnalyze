# JNI概述

JNI（Java Native Interface，Java本地接口），用于打通Java层与Native(C/C++)层。Java语言是跨平台的语言，而这跨平台的背后都是依靠Java虚拟机，虚拟机采用C/C++编写，适配各个系统，通过JNI为上层Java提供各种服务，保证跨平台性。

在Android中JNI是Java上层与Native的纽带。

> https://developer.android.com/training/articles/perf-jni

## JNI注册

JNI注册的两种时机：

- Android系统启动过程中Zygote注册，可通过查询AndroidRuntime.cpp中的gRegJNI，看看是否存在对应的register方法；
- 调用System.loadLibrary()方式注册。

## JNI的命名规则

```undefined
JNIExport jstring JNICALL Java_com_example_hellojni_MainActivity_stringFromJNI( JNIEnv* env,jobject thiz ) 
```

**`jstring`** 是**返回值类型**
 **`Java_com_example_hellojni`** 是**包名**
 **`MainActivity`** 是**类名**
 **`stringFromJNI`** 是**方法名**

其中**`JNIExport`**和**`JNICALL`**是不固定保留的关键字不要修改

## JNI开发流程的步骤

```
第1步：在Java中先声明一个native方法

第2步：编译Java源文件javac得到.class文件

第3步：通过javah -jni命令导出JNI的.h头文件

第4步：使用Java需要交互的本地代码，实现在Java中声明的Native方法（如果Java需要与C++交互，那么就用C++实现Java的Native方法。）

第5步：将本地代码编译成动态库(Windows系统下是.dll文件，如果是Linux系统下是.so文件，如果是Mac系统下是.jnilib)

第6步：通过Java命令执行Java程序，最终实现Java调用本地代码。
```

JNI并非Android专属，不过在Android开发中较为常见。

# NDK

Android 开发语言是Java，不过我们也知道，Android是基于Linux的，其核心库很多都是C/C++的，比如Webkit等。那么NDK的作用，就是Google为了提供给开发者一个在Java中调用C/C++代码的一个工作。

NDK本身其实就是一个交叉工作链，包含了Android上的一些库文件，然后，NDK为了方便使用，提供了一些脚本，使得更容易的编译C/C++代码。总之，在Android的SDK之外，有一个工具就是NDK，用于进行C/C++的开发。一般情况，是用NDK工具把C/C++编译为.co文件，然后在Java中调用。

> Android NDK 是一套允许您使用原生代码语言(例如C和C++) 实现部分应用的工具集。在开发某些类型应用时，这有助于您重复使用以这些语言编写的代码库。

用处：

1、在平台之间移植其应用

2、重复使用现在库，或者提供其自己的库重复使用

3、在某些情况下提性能，特别是像游戏这种计算密集型应用

4、使用第三方库，现在许多第三方库都是由C/C++库编写的，比如Ffmpeg这样库。

5、不依赖于Dalvik Java虚拟机的设计

6、代码的保护。由于APK的Java层代码很容易被反编译，而C/C++库反编译难度大。

# ABI

因为C语言的不跨平台，使用NDK编译在Linux下能执行的函数库——so文件。其本质就是一堆C、C++的头文件和实现文件打包成一个库。

目前Android系统支持以下七种不用的CPU架构，每一种对应着各自的应用程序二进制接口ABI：(Application Binary Interface)定义了二进制文件(尤其是.so文件)如何运行在相应的系统平台上，从使用的指令集，内存对齐到可用的系统函数库。对应关系如下：

```
ARMv5——armeabi
ARMv7 ——armeabi-v7a
ARMv8——arm64- v8a
x86——x86
MIPS ——mips
MIPS64——mips64
x86_64——x86_64
```

# Android JNI示例

> https://www.jianshu.com/p/b4431ac22ec2

# NDK

## 基于Make(Android.mk)

`Android.mk` 文件位于项目 `jni/` 目录的子目录中，用于向构建系统描述源文件和共享库。它实际上是一个微小的 GNU makefile 片段，构建系统会将其解析一次或多次。`Android.mk` 文件用于定义 [`Application.mk`](https://developer.android.com/ndk/guides/application_mk?hl=zh-cn)、构建系统和环境变量所未定义的项目级设置。它还可替换特定模块的项目级设置。

`Android.mk` 的语法支持将源文件分组为“模块”。模块是静态库、共享库或独立的可执行文件。您可在每个 `Android.mk` 文件中定义一个或多个模块，也可在多个模块中使用同一个源文件。构建系统只将共享库放入您的应用软件包。此外，静态库可生成共享库。

除了封装库之外，构建系统还可为您处理各种其他事项。例如，您无需在 `Android.mk` 文件中列出头文件或生成的文件之间的显式依赖关系。NDK 构建系统会自动计算这些关系。因此，您应该能够享受到未来 NDK 版本中支持的新工具链/平台功能带来的益处，而无需处理 `Android.mk` 文件。

> [Google对Android.mk的说明](https://developer.android.com/ndk/guides/android_mk?hl=zh-cn)

## 基于CMake

> [CMake手册](https://www.zybuluo.com/khan-lau/note/254724)

AndroidStudio创建C++工程自带简单例子。




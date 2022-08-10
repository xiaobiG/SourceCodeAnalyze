## 如何查找native方法

当大家在看framework层代码时，经常会看到native方法，这是往往需要查看所对应的C++方法在哪个文件，对应哪个方法？下面从一个实例出发带大家如何查看java层方法所对应的native方法位置。

#### 2.2.1 实例(一)

当分析Android消息机制源码，遇到`MessageQueue.java`中有多个native方法，比如：

```Java
package android.os;
public final class MessageQueue {
    private native void nativePollOnce(long ptr, int timeoutMillis);
}
```

**步骤1：** `MessageQueue.java`的全限定名为android.os.MessageQueue.java，完整的方法名为android.os.MessageQueue.nativePollOnce()，与之相对应的native层方法名是将点号替换为下划线，即android_os_MessageQueue_nativePollOnce()。

**步骤2：** 有了native方法，那么接下来需要知道该native方法所在那个文件。前面已经介绍过Android系统启动时就已经注册了大量的JNI方法，见AndroidRuntime.cpp的`gRegJNI`数组。这些注册方法命令方式：

```
register_[包名]_[类名]
```

那么MessageQueue.java所定义的jni注册方法名应该是`register_android_os_MessageQueue`，的确存在于gRegJNI数组，说明这次JNI注册过程是在开机过程完成。 该方法在`AndroidRuntime.cpp`申明为extern方法：

```
extern int register_android_os_MessageQueue(JNIEnv* env);
```

这些extern方法绝大多数位于`/framework/base/core/jni/`目录，大多数情况下native文件命名方式：

```
[包名]_[类名].cpp
[包名]_[类名].h
```

**Tips：** /android/os路径下的MessageQueue.java ==> android_os_MessageQueue.cpp

打开`android_os_MessageQueue.cpp`文件，搜索android_os_MessageQueue_nativePollOnce方法，这便找到了目标方法：

```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```

到这里完成了一次从Java层方法搜索到所对应的C++方法的过程。

#### 2.2.2 实例(二)

对于native文件命名方式，有时并非`[包名]_[类名].cpp`，比如/android/os路径下的Binder.java 所对应的native文件：android_util_Binder.cpp

```Java
package android.os;
public class Binder implements IBinder {
    public static final native int getCallingPid();
}
```

根据实例(一)方式，找到getCallingPid ==> android_os_Binder_getCallingPid()，并且在AndroidRuntime.cpp中的gRegJNI数组中找到`register_android_os_Binder`。

按实例(一)方式则native文名应该为android_os_Binder.cpp，可是在`/framework/base/core/jni/`目录下找不到该文件，这是例外的情况。其实真正的文件名为`android_util_Binder.cpp`，这就是例外，这一点有些费劲，不明白为何google要如此打破规律的命名。

```
static jint android_os_Binder_getCallingPid(JNIEnv* env, jobject clazz)
{
    return IPCThreadState::self()->getCallingPid();
}
```

有人可能好奇，既然如何遇到打破常规的文件命令，怎么办？这个并不难，首先，可以尝试在`/framework/base/core/jni/`中搜索，对于binder.java，可以直接搜索binder关键字，其他也类似。如果这里也找不到，可以通过grep全局搜索`android_os_Binder_getCallingPid`这个方法在哪个文件。

**jni存在的常见目录：**

- `/framework/base/core/jni/`
- `/framework/base/services/core/jni/`
- `/framework/base/media/jni/`

#### 2.2.3 实例(三)

前面两种都是在Android系统启动之初，便已经注册过JNI所对应的方法。 那么如果程序自己定义的jni方法，该如何查看jni方法所在位置呢？下面以MediaPlayer.java为例，其包名为android.media：

```
public class MediaPlayer{
    static {
        System.loadLibrary("media_jni");
        native_init();
    }

    private static native final void native_init();
    ...
}
```

通过static静态代码块中System.loadLibrary方法来加载动态库，库名为`media_jni`, Android平台则会自动扩展成所对应的`libmedia_jni.so`库。 接着通过关键字`native`加在native_init方法之前，便可以在java层直接使用native层方法。

接下来便要查看`libmedia_jni.so`库定义所在文件，一般都是通过`Android.mk`文件定义LOCAL_MODULE:= libmedia_jni，可以采用[grep](http://gityuan.com/2015/09/13/grep-and-find/)或者[mgrep](http://gityuan.com/2016/03/19/android-build/#section-3)来搜索包含libmedia_jni字段的Android.mk所在路径。

搜索可知，libmedia_jni.so位于/frameworks/base/media/jni/Android.mk。用前面实例(一)中的知识来查看相应的文件和方法名分别为：

```
android_media_MediaPlayer.cpp
android_media_MediaPlayer_native_init()
```

再然后，你会发现果然在该Android.mk所在目录`/frameworks/base/media/jni/`中找到android_media_MediaPlayer.cpp文件，并在文件中存在相应的方法：

```
  static void
android_media_MediaPlayer_native_init(JNIEnv *env)
{
    jclass clazz;
    clazz = env->FindClass("android/media/MediaPlayer");
    fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
    ...
}
```

**Tips：**MediaPlayer.java中的native_init方法所对应的native方法位于`/frameworks/base/media/jni/`目录下的`android_media_MediaPlayer.cpp`文件中的`android_media_MediaPlayer_native_init`方法。


> 参考：http://gityuan.com/2016/05/28/android-jni/
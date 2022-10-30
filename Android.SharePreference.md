SharedPreference在创建的时候会把整个文件全部加载进内存，如果你的sp文件比较大，那么会带来几个严重问题：

1. 第一次从sp中获取值的时候，有可能阻塞主线程，使界面卡顿、掉帧。
2. 解析sp的时候会产生大量的临时对象，导致频繁GC，引起界面卡顿。
3. 这些key和value会永远存在于内存之中，占用大量内存。

# 子线程IO就一定不会阻塞主线程吗？

下面是默认的sp实现SharedPreferenceImpl这个类的getString函数：

```java
public String getString(String key, @Nullable String defValue) {
    synchronized (this) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

继续看看这个awaitLoadedLocked：

```java
private void awaitLoadedLocked() {
    while (!mLoaded) {
        try {
            wait();
        } catch (InterruptedException unused) {
        }
    }
}
```

如果你直接调用getString，主线程会等待加载sp的那么线程加载完毕！这不就把主线程卡住了么？

有一个叫诀窍可以节省一下等待的时间：既然getString之类的操作会等待sp加载完成，而加载是在另外一个线程执行的，我们可以让sp先去加载，做一堆事情，然后再getString！如下：

```java
// 先让sp去另外一个线程加载
SharedPreferences sp = getSharedPreferences("test", MODE_PRIVATE);
// 做一堆别的事情
setContentView(testSpJson);
// ...

// OK,这时候估计已经加载完了吧,就算没完,我们在原本应该等待的时间也做了一些事!
String testValue = sp.getString("testKey", null);
```

# 所有的SharePreference会被缓存在内存中

被加载进来的这些大对象，会永远存在于内存之中，不会被释放。我们看看ContextImpl这个类，在getSharedPreference的时候会把所有的sp放到一个静态变量里面缓存起来：

```java
private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    final String packageName = getPackageName();
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    return packagePrefs;
}
```

# Activity finish时等待写入完成

```java
SharedPreferences sp = getSharedPreferences("test", MODE_PRIVATE);
sp.edit().putString("test1", "sss").apply();
sp.edit().putString("test2", "sss").apply();
sp.edit().putString("test3", "sss").apply();
sp.edit().putString("test4", "sss").apply();
```

apply不是在别的线程些磁盘的吗，怎么可能卡界面？仔细看一下源码。

```java
public void apply() {
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };

    QueuedWork.add(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            public void run() {
                awaitCommit.run();
                QueuedWork.remove(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
    notifyListeners(mcr);
}
```

注意两点，第一，把一个带有await的runnable添加进了QueueWork类的一个队列；第二，把这个写入任务通过enqueueDiskWrite丢给了一个**只有单个线程**的线程池执行。

到这里一切都OK，在子线程里面写入不会卡UI。但是，你去ActivityThread类的handleStopActivity里看一看：

```java
private void handleStopActivity(IBinder token, boolean show, int configChanges, int seq) {

    // 省略无关。。
    // Make sure any pending writes are now committed.
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }

    // 省略无关。。
}
```

waitToFinish?? 又要等？源码如下：

```java
public static void waitToFinish() {
    Runnable toFinish;
    while ((toFinish = sPendingWorkFinishers.poll()) != null) {
        toFinish.run();
    }
}
```

还记得这个toFinish的Runnable是啥吗？就是上面那个awaitCommit它里面就一句话，等待写入线程！！如果在Activity Stop的时候，已经写入完毕了，那么万事大吉，不会有任何等待，这个函数会立马返回。但是，如果你使用了太多次的apply，那么意味着写入队列会有很多写入任务，而那里就只有一个线程在写。当App规模很大的时候，这种情况简直就太常见了！

## 不能用来跨进程

还有童鞋发现sp有一个貌似可以提供「跨进程」功能的FLAG——MODE_MULTI_PROCESS,我们看看这个FLAG的文档：

> @deprecated MODE_MULTI_PROCESS does not work reliably in
> some versions of Android, and furthermore does not provide any mechanism for reconciling concurrent modifications across processes. Applications should not attempt to use it. Instead, they should use an explicit cross-process data management approach such as {@link android.content.ContentProvider ContentProvider}.

文档也说了，这玩意在某些Android版本上不可靠，并且未来也不会提供任何支持，要是用跨进程数据传输需要使用类似ContentProvider的东西。而且，SharedPreference的文档也特别说明：

> Note: This class does not support use across multiple processes.

那么我们姑且看一看，设置了这个Flag到底干了啥；在SharedPreferenceImpl里面，没有发现任何对这个Flag的使用；然后我们去ContextImpl类里面找找getSharedPreference的时候做了什么：

```java
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    checkMode(mode);
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}
```

这个flag保证了啥？保证了**在API 11以前**的系统上，如果sp已经被读取进内存，再次获取这个sp的时候，如果有这个flag，会重新读一遍文件，仅此而已！
当我们讨论协程和线程的关系时，很容易**陷入中文的误区**，两者都有一个「程」字，就觉得有关系，其实就英文而言，Coroutines 和 Threads 就是两个概念。

从 Android 开发者的角度去理解它们的关系：

- 我们所有的代码都是跑在线程中的，而线程是跑在进程中的。
- 协程没有直接和操作系统关联，但它不是空中楼阁，它也是跑在线程中的，可以是单线程，也可以是多线程。
- 单线程中的协程总的执行时间并不会比不用协程少。
- Android 系统上，如果在主线程进行网络请求，会抛出 `NetworkOnMainThreadException`，对于在主线程上的协程也不例外，这种场景使用协程还是要切线程的。

不过，我们学习 Kotlin 中的协程，一开始确实可以从线程控制的角度来切入。因为在 Kotlin 中，协程的一个典型的使用场景就是线程控制。就像 Java 中的 `Executor` 和 Android 中的 `AsyncTask`，Kotlin 中的协程也有对 Thread API 的封装，让我们可以在写代码时，不用关注多线程就能够很方便地写出并发操作。



# 基本使用

```java
// 方法一，使用 runBlocking 顶层函数
runBlocking {
    getImage(imageId)
}

// 方法二，使用 GlobalScope 单例对象
//            👇 可以直接调用 launch 开启协程
GlobalScope.launch {
    getImage(imageId)
}

// 方法三，自行通过 CoroutineContext 创建一个 CoroutineScope 对象
//                                    👇 需要一个类型为 CoroutineContext 的参数
val coroutineScope = CoroutineScope(context)
coroutineScope.launch {
    getImage(imageId)
}
```

- 方法一通常适用于单元测试的场景，而业务开发中不会用到这种方法，因为它是线程阻塞的。
- 方法二和使用 `runBlocking` 的区别在于不会阻塞线程。但在 Android 开发中同样不推荐这种用法，因为它的生命周期会和 app 一致，且不能取消（什么是协程的取消后面的文章会讲）。
- 方法三是比较推荐的使用方法，我们可以通过 `context` 参数去管理和控制协程的生命周期（这里的 `context` 和 Android 里的不是一个东西，是一个更通用的概念，会有一个 Android 平台的封装来配合使用）。

协程最常用的功能是并发，而并发的典型场景就是多线程。可以使用 `Dispatchers.IO` 参数把任务切到 IO 线程执行：

```java
coroutineScope.launch(Dispatchers.IO) {
    ...
}
```

也可以使用 `Dispatchers.Main` 参数切换到主线程：

```java
coroutineScope.launch(Dispatchers.Main) {
    ...
}
```

所以在「协程是什么」一节中讲到的异步请求的例子完整写出来是这样的：

```java
coroutineScope.launch(Dispatchers.Main) {   // 在主线程开启协程
    val user = api.getUser() // IO 线程执行网络请求
    nameTv.text = user.name  // 主线程更新 UI
}
```

如果遇到的场景是多个网络请求需要等待所有请求结束之后再对 UI 进行更新。比如以下两个请求：

```java
api.getAvatar(user, callback)
api.getCompanyLogo(user, callback)
```

如果使用回调式的写法，那么代码可能写起来既困难又别扭。于是我们可能会选择妥协，通过先后请求代替同时请求：

```java
api.getAvatar(user) { avatar -&gt;
    api.getCompanyLogo(user) { logo -&gt;
        show(merge(avatar, logo))
    }
}
```

在实际开发中如果这样写，本来能够并行处理的请求被强制通过串行的方式去实现，可能会导致等待时间长了一倍，也就是性能差了一倍。

而如果使用协程，可以直接把两个并行请求写成上下两行，最后再把结果进行合并即可：

```kotlin
coroutineScope.launch(Dispatchers.Main) {
    //            👇  async 函数之后再讲
    val avatar = async { api.getAvatar(user) }    // 获取用户头像
    val logo = async { api.getCompanyLogo(user) } // 获取用户所在公司的 logo
    val merged = suspendingMerge(avatar, logo)    // 合并结果
    //                  👆
    show(merged) // 更新 UI
}
```

可以看到，即便是比较复杂的并行网络请求，也能够通过协程写出结构清晰的代码。需要注意的是 `suspendingMerge` 并不是协程 API 中提供的方法，而是我们自定义的一个可「挂起」的结果合并方法。
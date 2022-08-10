# Handler机制

- 线程间通信，延时消息。

- 两种用法：

​	在未来的某一刻去执行一个message或者是runnable.

​	将一个操作转到另一个线程中执行

- 一个线程Thread对应一个Looper，一个消息队列，一个线程中可以有多个Handler。

### 常用片段

```java
Message msg = new Message();
Bundle bundle = new Bundle();
bundle.putString("hh",mEt.getText().toString());
msg.what = 1;
msg.setData(bundle);
mHandler.sendMessage(msg);
```

```
mHandler.obtainMessage(i).sendToTarget();
```

```
private class MyThread extends Thread{
 
        public Handler mHandler;
        @Override
        public void run() {
            //创建线程的looper和messagequeue
            Looper.prepare();
            //子线程的handler进行处理消息
            mHandler = new Handler(){
                @Override
                public void handleMessage(Message msg) {
                       switch (msg.what){
                        case 1:
                            Log.i("fang",msg.getData().getString("hh"));
                            break;
                        default:
                            break;
                    }
                }
            };
            //开启消息循环
            Looper.loop();
        }
    }
```



### 子线程发消息给主线程处理

主线程定义消息处理

```java
private Handler mHandler = new Handler(){
		public void handleMessage(android.os.Message msg) {		
			switch (msg.what) {
			    case 0:
		         	mReceiveMsg.setText(format.format(System.currentTimeMillis())+":"
			    +"获取到来自子线程的消息为："+msg.what);
				    break;
				default:
					break;
			}
		};
	};
```

子线程发送消息

`mHandler.obtainMessage(i).sendToTarget();`

```java
class SubThread implements Runnable {
 
		@Override
		public void run() {
			for (int i = 0; i < 10; i++) {
				mHandler.obtainMessage(i).sendToTarget();
				try {
					TimeUnit.MILLISECONDS.sleep(1000);
				} catch (InterruptedException e) {
					mHandler.obtainMessage(CHILD_THREAD_EXCEPTION).sendToTarget();
				};
			}	
			mHandler.obtainMessage(FINISH_CHILD_THREAD).sendToTarget();
		}
}
```

或创建message对象用来传递非字符串类型数据 

```java
Message msg = new Message();
Bundle bundle = new Bundle();
bundle.putString("hh",mEt.getText().toString());
msg.what = 1;
msg.setData(bundle);
mHandler.sendMessage(msg);
```

### 主线程发消息给子线程处理

主线程通过子线程handler发送消息：

```java
@Override
    public void onClick(View v) {
        switch (v.getId()){
            case R.id.test:
                Message msg = new Message();
                Bundle bundle = new Bundle();
                bundle.putString("hh",mEt.getText().toString());
                msg.what = 1;
                msg.setData(bundle);
                mThread.mHandler.sendMessage(msg);
                break;
            default:
                break;
        }
    }
```

子线程处理消息

```java
private class MyThread extends Thread{
 
        public Handler mHandler;
        @Override
        public void run() {
            //创建线程的looper和messagequeue  这里创建了当前线程的Looper
            Looper.prepare();
            //子线程的handler进行处理消息
            mHandler = new Handler(){
                @Override
                public void handleMessage(Message msg) {
                       switch (msg.what){
                        case 1:
                            Log.i("fang",msg.getData().getString("hh"));
                            break;
                        default:
                            break;
                    }
                }
            };
            //开启消息循环
            Looper.loop();
        }
    }
```

### Looper

一个Thread一个Looper一个MessageQueue。

Looper是事件循环的核心。

Handler 的无参构造函数：

```java
public Handler() {
    if (FIND_POTENTIAL_LEAKS) {
      // 当非静态类使用时提出警告
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = null;
}
```

Handler对象生成时便持有了当前线程的Looper和消息队列的引用。每个Thead对应的Looper通过Looper.prepare()产生：

```java
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

Looper 维护一个 MessageQueue 消息队列 , 用于存储从 Handler 中发送来的消息。

Looper的循环方法loop：

```java
 public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        if (me.mInLoop) {
            Slog.w(TAG, "Loop again would have the queued messages be executed"
                    + " before this one completed.");
        }

        me.mInLoop = true;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        me.mSlowDeliveryDetected = false;

        for (;;) {
            if (!loopOnce(me, ident, thresholdOverride)) {
                return;
            }
        }
    }
```

loopOnce中调用MessageQueue的next方法拿出下个待处理消息，消息在处理后被回收：

```java
 private static boolean loopOnce(final Looper me,
            final long ident, final int thresholdOverride) {
        Message msg = me.mQueue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return false;
        }
// ...
           msg.recycleUnchecked();
   // ...
```



### Looper 循环是怎样实现的

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    //通过Looper实例获取消息队列
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    for (;;) { //消息循环
        //从消息队列中取出一条消息，如果没有消息则会阻塞。
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        //将消息派发给target属性对应的handler，调用其dispatchMessage进行处理。
        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            //log
        }
        msg.recycle();
    }
}
```



### Looper主要工作：

- 自身实例的创建，创建消息队列，保证一个线程中最多有一个Looper实例。
- 消息循环，从消息队列中取出消息，进行派发。

Looper用于为线程运行消息循环的类，默认线程没有与它们相关联的消息循环；如果要想在子线程中进行消息循环，则需要在线程中调用**prepare()**，创建Looper对象。然后通过**loop()**方法来循环读取消息进行派发，直到循环结束。

程序中使用Looper的地方：

1. 主线程（UI线程） 
   UI线程中Looper已经都创建好了，不用我们去创建和循环。
2. 普通线程 
   普通线程中使用Looper需要我们自己去prepare()、loop()。 
   看一下普通线程中创建使用Looper的方式,代码如下:

```java
class LooperThread extends Thread {
    public Handler mHandler;
    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            public void handleMessage(Message msg) {
                 // process incoming messages here
            }
        };
        Looper.loop();
   }
}
```

这段代码是Looper源码注释中给的典型列子，主要步骤：

1. Looper 准备，（Looper实例创建）；
2. 创建发送消息、处理消息的Handler对象；
3. Looper开始运行。

Looper实例创建的方法： 

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

Looper构造方法是私有的，只能通过prepare()进行创建Looper对象。prepare()会调用私有方法prepare(boolean quitAllowed)。  第6行 sThreadLocal为ThreadLocal类型变量，用来存储线程中的Looper对象。  prepare方法中首先判断sThreadLocal是否存储对象，如果存储了则抛出异常，这是因为在同一个线程中Loop.prepare()方法不能调用两次，也就是同一个线程中最多有一个Looper实例（当然也可以没有，如果子线程不需要创建Handler时）。  该异常应该许多朋友都遇见过，如在UI线程中调用Looper.prepare()，系统会替UI线程创建Looper实例，所以不需要再次调用prepare()。 

Looper的构造器： 

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mRun = true;
    mThread = Thread.currentThread();
}
```

在构造器中，创建了一个MessageQueue消息队列；然后获取当前的线程，使Looper实例与线程绑定。  由prepare方法可知一个线程只会有一个Looper实例，所以一个Looper实例也只有一个MessageQueue实例。 

Looper的消息循环： 

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    //通过Looper实例获取消息队列
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    for (;;) { //消息循环
        //从消息队列中取出一条消息，如果没有消息则会阻塞。
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        //将消息派发给target属性对应的handler，调用其dispatchMessage进行处理。
        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            //log
        }
        msg.recycle();
    }
}
```

**总结：**

1. UI线程会自动创建Looper实例、并且调用loop()方法，不需要我们再调用prepare()和loop().
2. Looper与创建它的线程绑定，确保一个线程最多有一个Looper实例，同时一个Looper实例只有一个MessageQueue实例。
3. loop()函数循环从MessageQueue中获取消息，并将消息交给消息的target的dispatchMessage去处理。如果MessageQueue中没有消息则获取消息可能会阻塞。
4. 通过调用Looper的quit或quitsafely终止消息循环。

### Handler主要职责：

1. 发送消息给MessageQueue（消息队列）；
2. 处理Looper派送过来的消息； 
   我们使用Handler一般都要初始化一个Handler实例。看下Handler的构造函数：

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

第8行 Looper.myLooper();获取当前线程保存的Looper实例，如果当前线程没有Looper实例则会抛出异常。这也就是说**在线程中应该先创建Looper实例（通过Looper.prepare()），然后才可以创建Handler实例。**  第13行 获取Looper实例所保存的MessageQueue。之后使用Handler sendMesage、post都会将消息发送到该消息队列中。保证handler实例与该线程中唯一的Looper对象、及该Looper对象中的MessageQueue对象联系到一块。

####  sendMessage

先看sendMessage的流程： 

```java
public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

sendMessage最终调用到enqueueMessage函数，接着看下enqueueMessage。 

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
     msg.target = this;
     if (mAsynchronous) {
         msg.setAsynchronous(true);
     }
     return queue.enqueueMessage(msg, uptimeMillis);
 }
```

在enqueueMessage中，首先设置msg的target属性，值为this。之前在Looper的loop方法中，从消息队列中取出的msg，然后调用msg.target.dispatchMessage(msg);其实也就是调用当前handler的dispatchMessage函数。 

#### post

```java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

post方法中调用getPostMessage方法，创建一个Message对象，设置此Message对象的callback属性为创建Runnable对象。  然后调用sendMessageDelayed，最终和sendMessage一样，都是调用到sendMessageAtTime。调用enqueueMessage方法，将此msg添加到MessageQueue中。  

#### dispatchMessage

这里主要说下handler是如何处理消息的。在Looper.loop方法中通过获取到的msg，然后调用msg.target.dispatchMessage(msg);也就是调用handler的dispatchMessage方法，看下Handler中dispatchMessage源码 

```java
public void dispatchMessage(Message msg) {
   if (msg.callback != null) {
       handleCallback(msg);
   } else {
       if (mCallback != null) {
           if (mCallback.handleMessage(msg)) {
               return;
           }
       }
       handleMessage(msg);
   }
}
```

在dispatchMessage方法中首先判断msg的callback属性，如果不为空则调用handleCallback函数，  handleCallback函数如下： 

```java
private static void handleCallback(Message message) {
    message.callback.run();
}
```

handleCallback函数中messag.callback也就是我们传的Runnable对象，也就是调用Runnable对象的run方法。  如果msg.callback属性为空，判断Handler属性mCallback是否为空， 不为空则让mCallback处理该msg。  mCallback为空则调用Handler的handleMessage，这就是我们创建Handler对象时一般都实现其handleMessage方法的原因。 

### MessageQueue 

#### MessageQueue结构上是Message组成的链表

MessageQueue 中典型的链表查找操作：

```java
    boolean hasMessages(Handler h, Runnable r, Object object) {
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.callback == r && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
```



源码路径：frameworks/base/core/java/android/os/MessageQueue.java 
MessageQueue 消息队列：

- enqueueMessage将消息加入队列
- next从队列取出消息
- removeMessage移除消息

### 消息的Post和Send

Handler post系列方法，将runnable封装成一个Message，然后再调用对应的send系列函数把最终它压入到MessageQueue中。

```java
final boolean post(Runnable r)
final boolean postAtTime(Runnable r, long uptimeMillis)
final boolean postAtTime(Runnable r, Object token, long uptimeMillis)
final boolean postDelayed(Runnable r, long delayMillis)
final boolean postAtFrontOfQueue(Runnable r)
```

Handler send系列方法，参数直接是Message，经过一些赋值后，直接压入到MessageQueue中。

```java
final boolean sendEmptyMessage(int what)
final boolean sendEmptyMessageDelayed(int what, long delayMillis)
final boolean sendEmptyMessageAtTime(int what, long uptimeMillis)
final boolean sendMessageDelayed(Message msg, long delayMillis)
boolean sendMessageAtTime(Message msg, long uptimeMillis)
final boolean sendMessageAtFrontOfQueue(Message msg) 
```

消息的处理`dispatchMessage`：

```java
// Handler.java
public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            //post系列方法走这里
            handleCallback(msg);
        } else {
            if (mCallback != null) {
              	//Handler构造函数中定义了Callback的这里处理
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
          	//sendMessage系列方法走这里
            handleMessage(msg);
        }
    }
```

### 同步屏障机制

同步屏障消息就是在消息队列中插入一个屏障，在屏障之后的所有普通消息都会被挡着，不能被处理。不过异步消息却例外，屏障不会挡住异步消息，因此可以认为，屏障消息就是为了确保异步消息的优先级，设置了屏障后，只能处理其后的异步消息，同步消息会被挡住，除非撤销屏障。

设置同步屏障和创建异步Handler的方法都是标志为hide，说明谷歌不想要我们去使用它。在系统源码中使用比较多，比如在View更新时，draw、requestLayout、invalidate等很多地方都调用了。

一般来说，MessageQueue里面的Message是按照时间从前往后有序排列的。`enqueueMessage`方法中Message对比when大小后插入：

```java
              for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
```

每一个Message在被插入到MessageQueue中的时候，会强制其`target`属性不能为null：

```java
// MessageQueue.java

boolean enqueueMessage(Message msg, long when) {
  // Hanlder不允许为空
  if (msg.target == null) {
      throw new IllegalArgumentException("Message must have a target.");
  }
  ...
}
```

但同时又提供了一个方法来插入一个特殊的消息，强行让`target==null`：

```java
private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        // 把当前需要执行的Message全部执行
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 插入同步屏障
        if (prev != null) { 
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

同步屏障就是是通过MessageQueue的postSyncBarrier方法开启的，target为null的消息被插入消息队列顶部，当这个消息（屏障）存在时，只有`isAsynchronous`为true的消息被返回：

```java
Message next() {
    ···
    if (msg != null && msg.target == null) {
        // 同步屏障，找到下一个异步消息
        do {
            prevMsg = msg;
            msg = msg.next;
        } while (msg != null && !msg.isAsynchronous());
    }
    ···
}
```

**同步屏障不会自动移除，使用完成之后需要手动进行移除。**

### Message的池化复用

Message通过链表结构实现简单的池化复用，sPool即表头：

```java
    private static Message sPool;
    private static int sPoolSize = 0;
```

`Message.obtain`清除标记后返回复用的Message，

```java
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

`recycle`方法回收Message到池中，同时重置Message，最终调用方法：

```java
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```






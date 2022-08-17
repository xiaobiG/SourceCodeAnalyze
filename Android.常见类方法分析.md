# AsyncTask

代码例子：

```java
class ReverseTask extends AsyncTask<String,Integer,String> {
    @Override
    protected void onPreExecute() {
        progressDialog.show();
    }

    @Override
    protected String doInBackground(String... params) {
        int progress = 0;
        try {
            for(int i = 0;i <= 10;i ++){
                progress += 10;
                Thread.sleep(200);
                publishProgress(progress);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return Reverse(params[0]);
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        progressDialog.setMessage(values[0]+"% Finished");
    }

    @Override
    protected void onPostExecute(String s) {
        textView.setText(s);
        progressDialog.dismiss();
    }
}
Reverse()代码:

String Reverse(String string){
    StringBuilder builder = new StringBuilder(string);
    return new String(builder.reverse());
}
最后执行这个任务：

ReverseTask reverseTask = new ReverseTask();
reverseTask.execute(text);
```

## 源码分析

```java
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(), sThreadFactory);
        threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

内部使用了线程池，定义了一个ThreadPoolExecutor。

这个线程池是可以自定义的：

```java
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

耗时工作在非主线程中执行，通过`Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);`设置了线程的优先级：

```java
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
```

执行结束后走到`postResult`方法：

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

通过Handler将消息传到主线程。其实这个主线程也是打引号的，因为构造函数可以指定一个Handler，而post之后执行的动作也是在传入的Handler对应的线程处理的，只不过不传的时候默认为主线程Handler:

```java
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
```



## **AsyncTask与手动开Handler的区别**

其实Android多线程处理很常用的一个方法就是自己实现一个handler来异步处理消息。
优点： 这样的做法非常直观明白，而且代码弹性大，程序员有很大的掌控权。
缺点： 但是当线程和消息多起来了，不仅管理不方便，效率也可能产生问题，因为大量的线程在不断地出生和死亡，没有任何复用。

AsyncTask的优缺点也很明显

- 优点: 使用线程池管理多线程，资源利用和效率高，管理方便。通过第二章的介绍，我们知道其实AsyncTask的execute是可以传进一个Executor对象作为参数的。也就是说我们甚至可以自己实现自己的线程池来配套AsyncTask处理多线程问题。
- 缺点：不难发现AsyncTask的default Exectuor是一个Serial_Executor并且这个线程池是设为Static的串行管理线程池。也就是说，如果你使用默认的asynctask, 无论你开了多少个AsyncTask对象，所有这些对象其实是共用一个线程池的，而且这个线程池的策略只是很简单地按序一个个处理。
  这样会出现什么问题？假设你有一个大文件要下载，网络不怎么好，需要很长时间来完成。如果这个下载任务是用asynctask来实现的，那么所有其他的用asynctask的任务将会被这个下载任务阻塞住，直到它完成才能运行。因此，如果使用默认线程池，asynctask是不适合处理长时间后台任务的。

# runonUITread

一个简单的封装方法：

```java
    public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            mHandler.post(action);
        } else {
            action.run();
        }
    }
```

Activity有一个不公开分mHandler。

# IntentService


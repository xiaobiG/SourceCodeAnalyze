# 基本使用

保存每个线程独有的值。对同一个对象的get/set，每个线程都能有自己独立的值。

**需要说明的是，ThreadLocal对象一般都定义为static，以便于引用。**

```java
static ThreadLocal<Integer> local = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        Thread child = new Thread() {
            @Override
            public void run() {
                System.out.println("child thread initial: " + local.get());
                local.set(200);
                System.out.println("child thread final: " + local.get());
            }
        };
        local.set(100);
        child.start();
        child.join();
        System.out.println("main thread final: " + local.get());
    }
```
main线程对local变量的设置对child线程不起作用，child线程对local变量的改变也不会影响main线程，它们访问的虽然是同一个变量local，但每个线程都有自己的独立的值，这就是线程本地变量的含义。

- initialValue 方法用于提供初始值，它是一个受保护方法，可以通过匿名内部类的方式提供，当调用get方法时，如果之前没有设置过，会调用该方法获取初始值，默认实现是返回null。

- remove删掉当前线程对应的值，如果删掉后，再次调用get，会再调用initialValue获取初始值。

```java
public class ThreadLocalInit {
    static ThreadLocal<Integer> local = new ThreadLocal<Integer>(){

        @Override
        protected Integer initialValue() {
            return 100;
        }
    };

    public static void main(String[] args) {
        System.out.println(local.get());
        local.set(200);
        local.remove();
        System.out.println(local.get());
    }
}
```

# 使用场景

## DateFormat/SimpleDateFormat

DateFormat/SimpleDateFormat，提到它们是非线程安全的，实现安全的一种方式是使用锁，另一种方式是每次都创建一个新的对象，更好的方式就是使用ThreadLocal，每个线程使用自己的DateFormat，就不存在安全问题了，在线程的整个使用过程中，只需要创建一次，又避免了频繁创建的开销，示例代码如下：

```java
public class ThreadLocalDateFormat {
    static ThreadLocal<DateFormat> sdf = new ThreadLocal<DateFormat>() {

        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };

    public static String date2String(Date date) {
        return sdf.get().format(date);
    }

    public static Date string2Date(String str) throws ParseException {
        return sdf.get().parse(str);
    }
}
```

## ThreadLocalRandom

即使对象是线程安全的，使用ThreadLocal也可以减少竞争，比如，我们在34节介绍过Random类，Random是线程安全的，但如果并发访问竞争激烈的话，性能会下降，所以Java并发包提供了类ThreadLocalRandom，它是Random的子类，利用了ThreadLocal，它没有public的构造方法，通过静态方法current获取对象，比如：

```java
public static void main(String[] args) {
    ThreadLocalRandom rnd = ThreadLocalRandom.current();
    System.out.println(rnd.nextInt());
}
```
current方法的实现为：
```java
public static ThreadLocalRandom current() {
    return localRandom.get();
}
```

localRandom就是一个ThreadLocal变量：
```java
private static final ThreadLocal<ThreadLocalRandom> localRandom =
    new ThreadLocal<ThreadLocalRandom>() {
        protected ThreadLocalRandom initialValue() {
            return new ThreadLocalRandom();
        }
};
```

## 上下文信息

ThreadLocal的典型用途是提供上下文信息，比如在一个Web服务器中，一个线程执行用户的请求，在执行过程中，很多代码都会访问一些共同的信息，比如请求信息、用户身份信息、数据库连接、当前事务等，它们是线程执行过程中的全局信息，如果作为参数在不同代码间传递，代码会很啰嗦，这时，使用ThreadLocal就很方便，所以它被用于各种框架如Spring中，我们看个简单的示例：

```java
public class RequestContext {
    public static class Request { //...
    };

    private static ThreadLocal<String> localUserId = new ThreadLocal<>();
    private static ThreadLocal<Request> localRequest = new ThreadLocal<>();

    public static String getCurrentUserId() {
        return localUserId.get();
    }

    public static void setCurrentUserId(String userId) {
        localUserId.set(userId);
    }

    public static Request getCurrentRequest() {
        return localRequest.get();
    }

    public static void setCurrentRequest(Request request) {
        localRequest.set(request);
    }
}
```

# 基本实现原理

每个线程都有一个Map，类型为ThreadLocalMap，调用set实际上是在线程自己的Map里设置了一个条目，键为当前的ThreadLocal对象，值为value。

ThreadLocalMap是一个内部类，它是专门用于ThreadLocal的，与一般的Map不同，它的键类型为WeakReference<ThreadLocal>。

# 线程池与ThreadLocal

线程池中的线程在执行完一个任务，执行下一个任务时，其中的ThreadLocal对象并不会被清空，修改后的值带到了下一个异步任务。那怎么办呢？有几种思路：

- 第一次使用ThreadLocal对象时，总是先调用set设置初始值，或者如果ThreaLocal重写了initialValue方法，先调用remove

- 使用完ThreadLocal对象后，总是调用其remove方法

- 使用自定义的线程池



title: Java多线程之Callable接口及线程池
date: 2016-02-01 22:16:59
tags: [Java]
categories: Programming Notes

---

#### Java实现多线程的三种方式
- 继承`Thread`类
```java
public class Test extends Thread {
    public static void main(String[] args) {
        Thread t = new Test();
        t.start();
    }

    @Override
    public void run() {
        System.out.println("Override run() ...");
    }
}
```
- 实现`Runnable`接口，并覆写`run`方法
```java
public class Test implements Runnable {
    public static void main(String[] args) {
        Thread t = new Thread(new Test());
        t.start();
    }

    @Override
    public void run() {
        System.out.println("Override run() ...");
    }
}
```
- 实现`Callable`接口，并覆写`call`方法
```java
public class Test implements Callable {
    public static void main(String[] args) {
        FutureTask futureTask = new FutureTask(new Test());
        Thread thread = new Thread(futureTask);
        thread.start();
        try {
            Object o = futureTask.get();
            System.out.println(o);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Object call() throws Exception {
        return "Override call() ...";
    }
}
```

前两种方式应该很熟悉了，不再赘述，本文主要介绍第三种方式。
#### Executor框架
`Executor`框架是一个根据一组执行策略调用，调度，执行和控制的异步任务的框架。无限制的创建线程会引起应用程序内存溢出。所以创建一个线程池是个更好的的解决方案，因为可以限制线程的数量并且可以回收再利用这些线程。`Executor`框架包括：线程池，`Executor`，`Executors`，`ExecutorService`，`CompletionService`，`Future`，`Callable`等。
##### 什么是Executor
`Executor`仅仅是一个接口，只有一个方法`execute(Runnable command)`，是在`JDK1.5`中引入的，主要是用来运行提交的可运行的任务。一般我们并不直接使用该接口，而是使用其不同的子接口，主要是`ExecutorService`，而通常情况，与`ExecutorService`一起使用的是`Executors`类，该类由著名的并发编程大师`Doug Lea`实现。`Executor`框架可以用来控制线程的启动、执行和关闭，可以简化并发编程的操作。
##### 什么是Callable和Future
`Callable`接口使用泛型去定义它的返回类型。`Executors`类提供了一些有用的方法去在线程池中执行`Callable`内的任务。由于`Callable`任务是并行的，我们必须等待它返回的结果。`java.util.concurrent.Future`对象为我们解决了这个问题。在线程池提交`Callable`任务后返回了一个`Future`对象，使用它我们可以知道`Callable`任务的状态和得到`Callable`返回的执行结果。`Future`提供了`get()`方法让我们可以等待`Callable`结束并获取它的执行结果。
##### 什么是FutureTask
`FutureTask`是`Future`的一个基础实现，我们可以将它同`Executors`一起使用处理异步任务。通常我们不需要使用`FutureTask`类，单当我们打算重写`Future`接口的一些方法并保持原来基础的实现时，它就变得非常有用。我们可以仅仅继承于它并重写我们需要的方法。

##### 什么是Executors
`Executors`提供了一系列工厂方法用于创建线程池，返回的线程池都实现了`ExecutorService`接口：
- `newCachedThreadPool`可缓存线程池，对于每个线程，如果有空闲线程可用，立即让它执行，如果没有，则创建一个新线程
- `newFixedThreadPool`具有固定大小的线程池，如果任务数大于空闲的线程数，则把它们放进队列中等待
- `newSingleThreadPool`大小为1的线程池，任务一个接着一个完成
- `newScheduledThreadPool`调度型线程池，可控制线程最大并发数，支持定时及周期性任务执行，用来代替`Timer`

##### 什么是ExecutorService
在`ExecutorService`中提供了重载的`submit()`方法，该方法既可以接收`Runnable`实例又能接收`Callable`实例。对于实现`Callable`接口的类，需要覆写`call()`方法，并且只能通过`ExecutorService`的`submit()`方法来启动`call()`方法。那么既然存在`Runnable`接口，为什么还要添加`Callable`接口呢？这是因为`Runnable`不会返回结果，并且无法抛出经过检查的异常而`Callable`会返回结果，而且当获取返回结果时可能会抛出异常。`Callable`中的`call()`方法类似`Runnable`的`run()`方法，区别同样前者有返回值，而后者没有。
#### 使用示例
##### 获取线程的返回值
通过`FutureTask`包装一个`Callable`的实例，再通过`Thread`包装`FutureTask`的实例，然后调用`Thread`的`start()`方法，且看示例如下
```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadTest implements Callable {
    private AtomicInteger atomicInteger = new AtomicInteger(0);

    @Override
    public Object call() throws Exception {
        for (int i = 0; i <= 100; i++) {
            //Do something what U want
            atomicInteger.set(atomicInteger.get() + 1);
        }
        return atomicInteger.get();
    }

    @org.junit.Test
    public void test() throws ExecutionException, InterruptedException {
        FutureTask future = new FutureTask(new ThreadTest());
        Thread thread = new Thread(future);
        thread.start();
        if (future.isDone()) {
            future.cancel(true);
        }
        System.out.println(future.get());
    }
}
```
##### 通过ExecutorService执行线程
```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadTest implements Callable {
    private AtomicInteger atomicInteger = new AtomicInteger(0);

    @Override
    public Object call() throws Exception {
        for (int i = 0; i <= 100; i++) {
            //Do something what U want
            atomicInteger.set(atomicInteger.get() + 1);
        }
        return atomicInteger.get();
    }

    @org.junit.Test
    public void test() throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        Future submit = executorService.submit(new ThreadTest());
        System.out.println(submit.get());
    }
}
```
##### 批量提交任务
```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadTest<T> implements Callable {
    private AtomicInteger atomicInteger = new AtomicInteger(0);

    @Override
    public Object call() throws Exception {
        synchronized (this) {
            for (int i = 0; i <= 100; i++) {
                //Do something what U want
                atomicInteger.set(atomicInteger.get() + 1);
            }
            TimeUnit.SECONDS.sleep(1);
        }
        return atomicInteger.get();
    }


    @org.junit.Test
    public void test() throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        List<Callable<T>> tasks = new ArrayList<>();
        ThreadTest threadTest = new ThreadTest();
        for (int i = 0; i < 5; i++) {
            tasks.add(threadTest);
        }
        //该方法返回某个完成的任务
        Object o = executorService.invokeAny(tasks);
        System.out.println(o);
        System.out.println("One completed!");
        long start = System.currentTimeMillis();
        threadTest = new ThreadTest();
        tasks.clear();
        for (int i = 0; i < 5; i++) {
            tasks.add(threadTest);
        }
        List<Future<T>> futures = executorService.i
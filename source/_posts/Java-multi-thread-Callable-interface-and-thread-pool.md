title: Java多线程之Callable接口及线程池
date: 2016-02-01 22:16:59
tags: [Java]
categories: Programming Notes

---

####Java实现多线程的三种方式
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
####Executor框架
`Executor`框架是一个根据一组执行策略调用，调度，执行和控制的异步任务的框架。无限制的创建线程会引起应用程序内存溢出。所以创建一个线程池是个更好的的解决方案，因为可以限制线程的数量并且可以回收再利用这些线程。`Executor`框架包括：线程池，`Executor`，`Executors`，`ExecutorService`，`CompletionService`，`Future`，`Callable`等。
#####什么是Executor
`Executor`仅仅是一个接口，只有一个方法`execute(Runnable command)`，是在`JDK1.5`中引入的，主要是用来运行提交的可运行的任务。一般我们并不直接使用该接口，而是使用其不同的子接口，主要是`ExecutorService`，而通常情况，与`ExecutorService`一起使用的是`Executors`类，该类由著名的并发编程大师`Doug Lea`实现。`Executor`框架可以用来控制线程的启动、执行和关闭，可以简化并发编程的操作。
#####什么是Callable和Future
`Callable`接口使用泛型去定义它的返回类型。`Executors`类提供了一些有用的方法去在线程池中执行`Callable`内的任务。由于`Callable`任务是并行的，我们必须等待它返回的结果。`java.util.concurrent.Future`对象为我们解决了这个问题。在线程池提交`Callable`任务后返回了一个`Future`对象，使用它我们可以知道`Callable`任务的状态和得到`Callable`返回的执行结果。`Future`提供了`get()`方法让我们可以等待`Callable`结束并获取它的执行结果。
#####什么是FutureTask
`FutureTask`是`Future`的一个基础实现，我们可以将它同`Executors`一起使用处理异步任务。通常我们不需要使用`FutureTask`类，单当我们打算重写`Future`接口的一些方法并保持原来基础的实现时，它就变得非常有用。我们可以仅仅继承于它并重写我们需要的方法。

#####什么是Executors
`Executors`提供了一系列工厂方法用于创建线程池，返回的线程池都实现了`ExecutorService`接口：
- `newCachedThreadPool`可缓存线程池，对于每个线程，如果有空闲线程可用，立即让它执行，如果没有，则创建一个新线程
- `newFixedThreadPool`具有固定大小的线程池，如果任务数大于空闲的线程数，则把它们放进队列中等待
- `newSingleThreadPool`大小为1的线程池，任务一个接着一个完成
- `newScheduledThreadPool`调度型线程池，可控制线程最大并发数，支持定时及周期性任务执行，用来代替`Timer`

#####什么是ExecutorService
在`ExecutorService`中提供了重载的`submit()`方法，该方法既可以接收`Runnable`实例又能接收`Callable`实例。对于实现`Callable`接口的类，需要覆写`call()`方法，并且只能通过`ExecutorService`的`submit()`方法来启动`call()`方法。那么既然存在`Runnable`接口，为什么还要添加`Callable`接口呢？这是因为`Runnable`不会返回结果，并且无法抛出经过检查的异常而`Callable`会返回结果，而且当获取返回结果时可能会抛出异常。`Callable`中的`call()`方法类似`Runnable`的`run()`方法，区别同样前者有返回值，而后者没有。
####使用示例
#####获取线程的返回值
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
#####通过ExecutorService执行线程
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
#####批量提交任务
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
        List<Future<T>> futures = executorService.invokeAll(tasks);
        long end = System.currentTimeMillis();
        System.out.println(end - start + " ms之后，返回运行结果！");
        for (int i = 0; i < 5; i++) {
            System.out.println(futures.get(i).get());
        }
    }
}
```
运行结果
>184
One completed!
5001 ms之后，返回运行结果！
101
505
404
404
202

`invokeAny`方法提交所有任务到一个`Callable`对象的集合中，并且返回某个已经完成了的任务的结果，返回的任务是不确定的。`invokeAll`方法则返回所有任务的结果，但是其一个弊端是，如果第一个任务花费了很长时间，则不得不等待，待所有任务都完成之后，才返回。在某些情况下，可能只需要一个任务出了结果就可以中止所有任务，这样就得不偿失。将结果按照可获得的顺序保存起来可能更好，这时可以使用`ExecutorCompletionService`，其中的`take()`方法会移除下一个已经完成的结果（Future），如果没有可用结果则阻塞。

#####通过CompletionService提交多组任务并获取返回值
```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadTest implements Callable {
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
    public void test() throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        CompletionService<Integer> completionService = new ExecutorCompletionService(executorService);
        ThreadTest threadTest = new ThreadTest();
        long start = System.currentTimeMillis();
        for (int i = 0; i < 5; ++i) {
            completionService.submit(threadTest);//提交五组任务
        }
        long end = System.currentTimeMillis();
        System.out.println(end - start + " ms之后，返回运行结果！");
        for (int i = 0; i < 5; ++i) {
            Integer res = completionService.take().get();//通过take
            System.out.println(res);
        }
    }
}
```
输出结果：
>1 ms之后，返回运行结果！
101
202
303
404
505

由运行结果可知，`ExecutorCompletionService`并不会阻塞，在提交任务之后，继续向下运行，哪个任务完成即返回，并不受任务提交顺序的影响。

#####通过自维护列表管理多组任务并获取返回值
```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
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
        List<Future> futures = new ArrayList<>();
        ThreadTest threadTest = new ThreadTest();
        for (int i = 0; i < 5; ++i) {
            Future submit = executorService.submit(threadTest);
            futures.add(submit);
        }
        Iterator<Future> iterator = futures.iterator();
        while (iterator.hasNext()) {
            Future next = iterator.next();
            System.out.println(next.get());
        }
    }

}
```
采用自维护`Future`集合方法，`submit`的`task`不一定是按照加入自己维护的列表顺序完成的。从`list`中遍历的每个`Future`对象并不一定处于完成状态，这时调用`get()`方法就会被阻塞住，如果系统是设计成每个线程完成后就能根据其结果继续做后面的事，这样对于处于`list`后面的但是先完成的线程就会增加了额外的等待时间。

而`CompletionService`的实现是维护一个保存`Future`对象的`BlockingQueue`。只有当这个`Future`对象状态是结束的时候，才会加入到这个`Queue`中，`take()`方法其实就是`Producer-Consumer`中的`Consumer`。它会从`Queue`中取出`Future`对象，如果`Queue`是空的，就会阻塞在那里，直到有完成的`Future`对象加入到`Queue`中。

所以，先完成的必定先被取出。这样就减少了不必要的等待时间。
#####ScheduledExecutor任务调度
`ScheduledExecutor`提供了基于开始时间与重复间隔的任务调度，可以实现简单的任务调度需求。每一个被调度的任务都会由线程池中一个线程去执行，因此任务是并发执行的，相互之间不会受到干扰。需要注意的是，只有当任务的执行时间到来时，`ScheduedExecutor`才会真正启动一个线程，其余时间`ScheduledExecutor`都是在轮询任务的状态。

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledExecutorTest implements Runnable {
    private String jobName = "";

    public ScheduledExecutorTest(String jobName) {
        super();
        this.jobName = jobName;
    }

    @Override
    public void run() {
        System.out.println("execute " + jobName);
    }

    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newScheduledThreadPool(10);

        long initialDelay1 = 1;
        long period1 = 1;
        // 从现在开始1秒钟之后，每隔1秒钟执行一次job1
        service.scheduleAtFixedRate(
                new ScheduledExecutorTest("job" + period1), initialDelay1,
                period1, TimeUnit.SECONDS);
        long initialDelay2 = 2;
        long period2 = 2;
        // 从现在开始2秒钟之后，每隔2秒钟执行一次job2
        service.scheduleWithFixedDelay(
                new ScheduledExecutorTest("job" + period2), initialDelay2,
                period2, TimeUnit.SECONDS);
    }
}
```
运行结果
>execute job1
execute job1
execute job2
execute job1
execute job1
execute job2
execute job1
execute job1
execute job2

####取消向线程池提交的某个任务
在`ExecutorService`中提供了`submit()`方法，用于向线程池提交任务，该方法返回一个包含结果集的`Future`实例。而`Future`提供了`cancel(boolean mayInterruptIfRunning)`方法用于取消提交的运行任务，如果向该函数传递`true`，那么不管该任务是否运行结束，立即停止，如果向该函数传递`false`，那么等待该任务运行完成再结束之。同样`Future`还提供了`isDone()`用于测试该任务是否结束，`isCancelled()`用于测试该任务是否在运行结束前已取消。

####关闭线程池
在`ExecutorService`中提供了`shutdown()`和`List<Runnable> shutdownNow()`方法用来关闭线程池，前一个方法将启动一次顺序关闭，有任务在执行的话，则等待该任务运行结束，同时不再接受新任务运行。后一个方法将取消所有未开始的任务并且试图中断正在执行的任务，返回从未开始执行的任务列表，不保证能够停止正在执行的任务，但是会尽力尝试。例如，通过`Thread.interrupt()`来取消正在执行的任务，但是对于任何无法响应中断的任务，都可能永远无法终止。
title: "Java之volatile boolean，AutomaticBoolean分析"
date: 2015-04-10 17:42:13
tags: [Java]
categories: Programming Notes

---
### volatile概述
volatile关键字是一个类型修饰符，被设计用来修饰被不同线程访问和修改的变量，在JVM1.2之前，Java的内存模型实现总是从主存读取变量，是不需要进行特别的注意的。而随着JVM的成熟和优化，现在在多线程环境下volatile关键字的使用变得非常重要。

在当前的Java内存模型下，线程可以把变量保存在本地内存（比如机器的寄存器）中，而不是直接在主存中进行读写。这就可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中的变量值的拷贝，造成数据的不一致。

要解决这个问题，需要把变量声明为volatile（不稳定的），以后用到该变量都会到主存中进行存取，一般多任务环境下各任务间共享的标志都应该加volatile修饰。volatile修饰的成员变量在每次被线程访问时，都强迫从共享内存中重读该成员变量的值，而且当成员变量发生变化时，强迫线程将变化值回写到共享内存。

Java语言中的volatile变量可以被看作是一种“程度较轻的synchronized”；与synchronized块相比，volatile变量所需的编码较少，并且运行时开销也较少，但是它所能实现的功能也仅是synchronized的一部分。Volatile变量具有synchronized的可见性特性，但是不具备原子特性。volatile仅仅用来保证该变量对所有线程的可见性，但不保证原子性。
### volatile是线程不安全的
```java
public class Test implements Runnable {
	public static volatile boolean flag = true;
	public static void main(String[] args) {
		for (int j = 0; j < 20; j++) {
			Thread t = new Thread(new Test());
			t.start();
		}
	}
	public void run() {
		if (flag) {
			System.out.println("我成功了！");
			flag = false;
		}
	}
}
```
输出结果：
我成功了！
我成功了！
我成功了！
我成功了！

注意：不要将volatile用在getAndOperate场合，仅仅set或者get的场景是适合volatile的。

由以上测试用例可以看出，volatile boolean是无法保证线程安全的，在一个线程占用flag变量的时候，
其他线程也可以占用flag变量，所以多个线程均可以打印出消息。

### Atomic概述
java.util.concurrent.atomic是在JDK1.5之后引入的包，在JDK中的说明如下
>A small toolkit of classes that support lock-free thread-safe programming on single variables. In essence, the classes in this package extend the notion of volatile values, fields, and array elements to those that also provide an atomic conditional update operation of the form:

```java   
   boolean compareAndSet(expectedValue, updateValue);
```

>This method (which varies in argument types across different classes) atomically sets a variable to the updateValue if it currently holds the expectedValue, reporting true on success. The classes in this package also contain methods to get and unconditionally set values, as well as a weaker conditional atomic update operation weakCompareAndSet described below.

该包提供了一组原子类，在多线程环境下，当有多个线程同时执行这些类的实例包含的方法时，具有排他性，即当某个线程进入方法，执行其中的指令时，不会被其他线程打断，而别的线程就像自旋锁一样，一直等到该方法执行完成，才由JVM从等待队列中选择另一个线程进入。

### Atomic是线程安全的
```java
import java.util.concurrent.atomic.AtomicBoolean;
public class Test implements Runnable {
	public static volatile AtomicBoolean flag = new AtomicBoolean(true);
	public static void main(String[] args) {
		for (int j = 0; j < 20; j++) {
			Thread t = new Thread(new Test());
			t.start();
		}
	}
	public void run() {
		if (flag.compareAndSet(true, false)) {
			System.out.println("我成功了！");
		}
	}
}
```
输出结果：
我成功了！

Atomic类不仅仅提供了对数据操作的线程安全保证，而且提供了一系列的语义清晰的方法，如incrementAndGet() 、getAndIncrement() 等，使用方便，Atomic的内部实现使用的是更加高效的CAS（compareand swap）+volatile，从而避免了synchronized的高开销，执行效率大幅提升，出于性能考虑强烈建议使用Atomic，而不是synchronized关键字。

---
参考资料
【1】 http://www.blogjava.net/aoxj/archive/2012/06/16/380926.html
【2】 http://blog.csdn.net/xieyuooo/article/details/8594713


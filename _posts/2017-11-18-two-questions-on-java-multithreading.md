---
layout: post
title: java多线程中的两个疑问
date: 2017-11-18
categories:
- 后端技术
tags: [Java, 线程]
status: publish
type: post
published: true
---

## 写在前面

前一篇关于线程相关的[文章]({{ site.url }}/后端技术/2017/11/15/Java中线程相关常用方法/)中，讲到 notify() 方法和 wait() 方法时，借用了某篇博客中的一个例子，这个例子是两个线程，轮流打印1和2，当时写了这个例子之后，考虑了如果有更多线程加入进来会怎样，即：多线程时，使用 notify() 方法和 wait() 方法会发生什么？

---

## 测试代码

这段代码的目的是让四个线程随机打印 i 的值

```java
public class NotifyAndWaitTest2 implements Runnable {
	public int i = 0;
	public Object lock;
	public SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss.SSS");

	public NotifyAndWaitTest2(Object o) {
		this.lock = o;
	}

	@Override
	public void run() {
		synchronized (lock) {
			System.out.println(Thread.currentThread().getName() + " enter the SYNCHRONIZED block --- "+ sdf.format(new Date()));
			try {
				while (i < 9) {
					Thread.sleep(500);
					lock.notify();
					lock.wait();
					System.out.println(Thread.currentThread().getName() + " say:" + i++ + " --- " + sdf.format(new Date()));
				}
				lock.notify();
				return;
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String[] args) {
		Object lock = new Object();
		NotifyAndWaitTest2 test = new NotifyAndWaitTest2(lock);
		Thread t1 = new Thread(test,"Thread A");
		Thread t2 = new Thread(test,"Thread B");
		Thread t3 = new Thread(test,"Thread C");
		Thread t4 = new Thread(test,"Thread D");
		t1.start();
		t2.start();
		t3.start();
		t4.start();
	}
}
```

---

## 测试结果

这是某一次运行后的结果：

```
Thread A enter the SYNCHRONIZED block --- 15:24:28.358
Thread C enter the SYNCHRONIZED block --- 15:24:28.859
Thread D enter the SYNCHRONIZED block --- 15:24:29.359
Thread B enter the SYNCHRONIZED block --- 15:24:29.860
Thread D say:0 --- 15:24:30.361
Thread B say:1 --- 15:24:30.861
Thread D say:2 --- 15:24:31.362
Thread B say:3 --- 15:24:31.863
Thread D say:4 --- 15:24:32.363
Thread B say:5 --- 15:24:32.863
Thread D say:6 --- 15:24:33.364
Thread B say:7 --- 15:24:33.865
Thread D say:8 --- 15:24:34.366
Thread B say:9 --- 15:24:34.366
Thread C say:10 --- 15:24:34.366
Thread A say:11 --- 15:24:34.366
```

---

## 发现问题

原本以为调用 notify() 方法时，会是任意一个线程被唤醒，然后打印 i，但当我执行几次后却发现了两个奇怪的现象：

* 为啥一开始都是四个线程依次进入同步代码块？

* 为啥 notify() 方法唤醒的线程是"固定"的？

以上面的某次测试结果来说这两个现象：

为什么线程A、C进入了同步代码块之后，不能触发线程A打印 i，而是另外两个线程先"挤"进同步代码块？无论我怎么设置线程优先级，无论我起多少个线程(最多我尝试了100个线程)，都是线程们先依次进入同步代码块

为什么使用 notify() 方法后，只能唤醒上一个(最后一个)调用了 wait() 方法的线程，就像线程B进入同步代码块之后唤醒了线程D，而线程D又唤醒了线程B，两个线程轮流唤醒对方，而没有唤醒线程A和线程C

---

## 请教大佬

思考了多次之后也没什么头绪，[官方文档](https://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#notify()) 明明也是这么说的：

> Wakes up a single thread that is waiting on this object's monitor. If any threads are waiting on this object, one of them is chosen to be awakened. The choice is arbitrary and occurs at the discretion of the implementation. A thread waits on an object's monitor by calling one of the wait methods.
>
> The awakened thread will not be able to proceed until the current thread relinquishes the lock on this object. The awakened thread will compete in the usual manner with any other threads that might be actively competing to synchronize on this object; for example, the awakened thread enjoys no reliable privilege or disadvantage in being the next thread to lock this object.

然后就去 Stack Overflow 提了[这个问题](https://stackoverflow.com/questions/47321030/two-questions-on-java-multiple-threads-use-notify-method)，CSDN 上顺便也[提了一个](http://ask.csdn.net/questions/669879)

等了几天，目前只有两个人回答，他们其实也是说理论上是随机的，但实际哪个线程先抢到对象锁，拿到资源还受很多其他因素影响，比如Java 版本啊、操作系统啊、线程调度啊、资源分配啊等等等等，CSDN 上的哥们说不用太纠结，就当成是随机的来用就行了，听他这么一说，我也觉得可能是有点钻牛角尖了

不过......

---

## 继续钻一下牛角尖

多折腾了一些场景

#### 0. 代码不变，改变系统环境

本机的环境是 WIN10 + jdk1.8，所以：

在VMware中部署了一个单核 CPU 的 WIN7，安装了 jdk1.8，再测试：

* 依旧是线程先进入同步代码块

* 依旧是最后两个进入同步代码块的线程交替唤醒对方

然后在 Ubuntu16 + jdk1.8 下测试：

* 依旧是线程先进入同步代码块

* 依旧是最后两个进入同步代码块的线程交替唤醒对方


#### 1. 改用 notifyAll()

把上面代码中的两处 notify() 方法改为 notifyAll()，再测试的时候就发现：

* 依旧是线程先进入同步代码块

* 打印 i 的线程变**随机**了

#### 2. 减少 sleep() 时间

最早的测试代码中，sleep(500) 是随便写的，于是逐渐减小 sleep(100) -> sleep(10) -> sleep(1) 直到不调用 sleep() 方法，测试后发现：

* 依旧是线程先进入同步代码块

* 依旧是最后两个进入同步代码块的线程交替唤醒对方

#### 3. 增多线程数

使用 for 循环不断加大开启的线程数，20 -> 30 -> 50 -> 100，测试后发现：

* 依旧是线程先进入同步代码块

* 依旧是最后两个进入同步代码块的线程交替唤醒对方

#### 4. 减少 sleep() 时间并增多线程数

当 sleep() 的时间非常短，比如 sleep(1) 甚至不调用 sleep() 时，并且线程数超过30的时候，测试发现：

* **线程进入同步代码块 和 线程打印 i 随机发生**

* **不完全是两个线程之间互相唤醒**

这是其中某次测试的结果（部分）：

```
Thread 1 enter the SYNCHRONIZED block --- 17:32:52.630
Thread 9 enter the SYNCHRONIZED block --- 17:32:52.632
Thread 1 say:0 --- 17:32:52.632
Thread 9 say:1 --- 17:32:52.632
Thread 1 say:2 --- 17:32:52.632
Thread 9 say:3 --- 17:32:52.632
Thread 27 enter the SYNCHRONIZED block --- 17:32:52.632
Thread 1 say:4 --- 17:32:52.632
Thread 27 say:5 --- 17:32:52.632
Thread 16 enter the SYNCHRONIZED block --- 17:32:52.633
Thread 1 say:6 --- 17:32:52.633
Thread 16 say:7 --- 17:32:52.633
Thread 1 say:8 --- 17:32:52.633
Thread 20 enter the SYNCHRONIZED block --- 17:32:52.633
Thread 16 say:9 --- 17:32:52.633
Thread 39 enter the SYNCHRONIZED block --- 17:32:52.633
Thread 30 enter the SYNCHRONIZED block --- 17:32:52.633
Thread 35 enter the SYNCHRONIZED block --- 17:32:52.633
...
```

---

## 个人分析

从上面的各种测试来看，我觉得最大的问题应该出在调用线程的 sleep() 方法上，休眠时间越大，线程即使有几百个，也很难出现理论中的随机的情况

有一个不太成熟的猜想：

正是因为线程开始执行时，(run方法中)一上来就是要休眠，而休眠时会让出 CPU 资源，所以操作系统(或者是 CPU ？)认为这些线程比那些已经休眠过、调用了 wait() 方法、等待被唤醒、分配资源继续执行的线程**更有利**，就选择在随机的前提下，**优先甚至强制**执行这些方法

因为对操作系统、CPU、JVM 等过于底层的知识不懂，所以只是一个草率的猜想，一边等待 Stack Overflow 和 CSDN 上有大牛解答，一边自己也加快学习吧

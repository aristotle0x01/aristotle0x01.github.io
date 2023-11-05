---
layout: post
title:  "java锁,协作与线程-1-Synchronized, Monitor和Lock及AQS"
date:   2021-06-02
categories: java synchronization lock cooperation 同步 协作 AQS condition monitor
---


## 1.概述

本文意在探讨java两大类同步机制**`synchronized`**与锁(如**`ReentrantLock`**)的异同，及其背后各自的实现原理。首先给出一些结论，后续验证和分析：

- **`synchronized`**基于**`monitor`**概念设计，在jvm内部实现
- **`ReentrantLock`**等锁基于**`AQS(AbstractQueuedSynchronizer)`**，在jdk中实现
- 偏向/轻量/重量锁等概念是jdk1.6针对**`synchronized`**的性能优化，与其它锁机制无关，对其它锁而言，对象头不会变化
- **`synchronized`**通过Object的wait/notify方法提供隐性条件协作

<br/>



## 2.基础概念

### 2.1 java对象头

任意java对象都包含对象头，以保存相关描述信息

| 对象头                                                       | Mark word                                                    |
| :----------------------------------------------------------- | ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/280229841-719fc7c0-c06d-47fe-95e9-a9267f1ae964.png" alt="header" style="zoom:50%; float: left;" /> | <img src="https://user-images.githubusercontent.com/2216435/280229977-ef3fa1da-8f5a-4756-98e2-ad998b4f1932.png" alt="mark word" style="zoom:40%; float: left;" /> |

<br/>



### 2.2 monitor

monitor是由C.A.R. Hoare等人提出的一种线程同步机制，提供：

- 互斥功能
- 线程间协作

ref: [^1] [^2]

<br/>



## 3.synchronized实现

### 3.1 偏向/轻量/重量锁

上述三种锁是jdk1.6为了提升synchronized性能，针对不同场景进行的三种内部优化，这三种实现会修改**markword:tag**字段。通过jol包打印header取值情况：

```
		Thread.sleep(5000);
    Object o = new Object();
    System.out.println("未进入同步块，MarkWord 为：");
    System.out.println(ClassLayout.parseInstance(o).toPrintable());
    synchronized (o){
    		System.out.println(("进入同步块，MarkWord 为："));
    		System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
```

| <img src="https://user-images.githubusercontent.com/2216435/280236517-1b7b9547-66bf-4623-bc7e-e16cb9a1f087.png" alt="object header output when synchronized" style="zoom:35%; float: left;" /> | object header同步前后变化 |
| ------------------------------------------------------------ | ------------------------- |

至于上述三种内部锁适用于何种场景及其流转，不是本文重点，详见[难搞的偏向锁终于被 Java 移除了](https://segmentfault.com/a/1190000041194920)

<br/>



### 3.2 字节码

上述代码编译后字节码：

| **`synchronized`**在编译后, 在jvm内部由指令**`monitorenter/monitorexit`**实现 | <img src="https://user-images.githubusercontent.com/2216435/280238904-5da1ca66-2e8c-4522-ba12-4e7556a189b2.png" alt="bytecode" style="zoom:60%; float: left;" /> |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| the java virtual machine specification 8中的定义             | <img src="https://user-images.githubusercontent.com/2216435/280265501-d5c43ee8-625d-4926-8d8d-c23579b48aba.png" alt="monitorenter" style="zoom:30%; float: left;" /> |

根据描述，当锁对象关联的monitor的entry count：

- 为**0** 时，取得锁，成为持有者
- 不为**0**当已经持有时，则增1，重入持有
- 不为**0**且被其它线程持有，则阻塞

<br/>



### 3.3 所谓每个Object关联一个monitor意味着什么？

根据[Monitors and Exceptions : How to Implement Java efficiently: Andreas Krall and Mark Probst](https://people.cs.vt.edu/~ryder/oosem99/talks/isaila-krall.pdf)

 sun的实现是使用锁对象的object identifier在hashmap中关联monitor对象(首次加锁时生成)。当然，实际上各个版本的jvm到底如何实现，还需要具体研究jvm源码，本文暂时止步于此[^4][^5]。有个monitor参考实现[^3]

<br/>



### 3.4 线程间如何协作？

通过Object对象中的**`wait()/notify()`**方法，二者包含且仅包含了一个隐性Condition变量。

<br/>



### 3.5 synchronized锁模型示意

| <img src="https://user-images.githubusercontent.com/2216435/280504547-9f3ed02a-c788-419b-99af-9b540503179c.png" alt="synchronized locking model" style="zoom:60%; float: left;" /> | synchronized实现逻辑模型 |
| ------------------------------------------------------------ | ------------------------ |

<br/>





## 4.Lock实现

### 4.1 对象头**markword:tag**会变吗？

```
ReentrantLock rl = new ReentrantLock();
System.out.println("new ReentrantLock: ");
System.out.println(ClassLayout.parseInstance(rl).toPrintable());
rl.lock();
System.out.println("    after locking: ");
System.out.println(ClassLayout.parseInstance(rl).toPrintable());
try {
   System.out.println("    critical running：");
} finally {
   rl.unlock();
   System.out.println("    release locking: ");
   System.out.println(ClassLayout.parseInstance(rl).toPrintable());
}
```

| <img src="https://user-images.githubusercontent.com/2216435/280270571-3619fec8-c0ff-425a-8935-c593128767b7.png" alt="lock object header" style="zoom:35%; float: left;" /> | 加锁前后object header无变化 |
| ------------------------------------------------------------ | --------------------------- |

<br/>



### 4.2 AbstractQueuedSynchronizer

**AQS**是java中几乎所有锁的基类，内置了CAS，等待队列，park/unpark，Condition等核心功能。

<br/>



### 4.3 ReentrantLock中的Condition实现

锁通过生成新的Condition，原则上可以有无限多个等待队列。等待队列只能通过await()方法进入，通过signal()方法退出。

| <img src="https://user-images.githubusercontent.com/2216435/120757977-da3f6300-c543-11eb-9a25-1f80e7a48e92.png" alt="thread state" style="zoom:40%; float: left;" /> | case1: 释放锁等待           |
| ------------------------------------------------------------ | --------------------------- |
| <img src="https://user-images.githubusercontent.com/2216435/120756871-63559a80-c542-11eb-9ef8-148097cfbaae.png" alt="thread state" style="zoom:40%; float: left;" /> | case2a: 通知其它线程/释放锁 |
| <img src="https://user-images.githubusercontent.com/2216435/120756901-6ea8c600-c542-11eb-8d73-841034873e33.png" alt="thread state" style="zoom:40%; float: left;" /> | case2b: 其它线程抢锁成功    |

从代码角度分析，首先看进入等待队列

```
        /**
         * ConditionObject implements Condition
         * 
         * Implements interruptible condition wait.
         */
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            // 进入Condition等待队列
            Node node = addConditionWaiter();
            // 释放锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            // 查看是否有被signal()通知，没有则休眠
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 退出前，再次拿锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

再来看看signal()实现

```
        /**
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
                	do {
                	  // 将等待队列里第一个线程出队
                		if ( (firstWaiter = first.nextWaiter) == null)
                    		lastWaiter = null;
                		first.nextWaiter = null;
            				} while (!transferForSignal(first) &&
                     				(first = firstWaiter) != null);
        }
        
        // 再看看transferForSignal
        transferForSignal(first)
        // 将退出等待队列的第一个线程放入抢锁队列
        enq(node)
```

<br/>



## 5.线程唤醒notify的隐含语义

| 线程状态                                                     | notify语义[^6]                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/120476594-54a1a300-c3dd-11eb-8884-98b8991cdf60.png" alt="thread state" style="zoom:40%; float: left;" /> | <img src="https://user-images.githubusercontent.com/2216435/120476436-2c19a900-c3dd-11eb-9309-4170048a34a4.png" alt="thread state" style="zoom:30%; float: left;" /> |

java使用的是mesa语义，也就是说仅将激活线程放入ready 队列，并不一定会立即得到操作系统调度机会

<br/>



## References

[^1]: [What Is a Monitor in Computer Science?](https://www.baeldung.com/cs/monitor)
[^2]: [Better monitors for Java：A Hoare-style monitor package for multithreaded programming in Java](https://www.infoworld.com/article/2077769/better-monitors-for-java.html)
[^3]: [source code: Better monitors for Java](https://www.engr.mun.ca/~theo/Misc/monitors/monitors.html)
[^4]: [Thread Synchronization: Chapter 20 of Inside the Java Virtual Machine by Bill Venners](https://www.artima.com/insidejvm/ed2/threadsynch.html)
[^5]:[In Java, what is the difference between a monitor and a lock](https://stackoverflow.com/questions/49610644/in-java-what-is-the-difference-between-a-monitor-and-a-lock)
[^6]: [Lecture 6: Semaphores and Monitors](https://cseweb.ucsd.edu/classes/fa05/cse120/lectures/120-l6.pdf)

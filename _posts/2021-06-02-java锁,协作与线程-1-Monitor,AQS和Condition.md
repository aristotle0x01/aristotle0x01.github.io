---
layout: post
title:  "java锁,协作与线程-1-Monitor,AQS和Condition"
date:   2021-06-02
categories: java synchronization lock cooperation 同步 协作 AQS condition monitor
---
## java对象之lock

在java里面，任意对象都可以作为lock来使用，是因为java对象在内存中的构造天然包含了lock字段

![](https://user-images.githubusercontent.com/2216435/120470693-6df32100-c3d6-11eb-8d7d-43755af0b729.png)

![](https://user-images.githubusercontent.com/2216435/120477633-95e68280-c3de-11eb-8c92-4d9e77626ed9.png)

[ObjectHeader32.txt](https://gist.github.com/arturmkrtchyan/43d6135e8a15798cc46c)

可知，通过cpu层面原子操作的支持，即可达成互斥

## Object类与字节码

```
/**
 * Class {@code Object} is the root of the class hierarchy.
 * Every class has {@code Object} as a superclass. All objects,
 * including arrays, implement the methods of this class.
 */
public class Object {
    /**
     * Wakes up a single thread that is waiting on this object's
     * monitor. If any threads are waiting on this object, one of them
     * is chosen to be awakened. The choice is arbitrary and occurs at
     * the discretion of the implementation. A thread waits on an object's
     * monitor by calling one of the {@code wait} methods.
     */
    public final native void notify();

    /**
     * Wakes up all threads that are waiting on this object's monitor. A
     * thread waits on an object's monitor by calling one of the
     * {@code wait} methods.
     */
    public final native void notifyAll();

    /**
     * Causes the current thread to wait until it is awakened, typically
     * by being <em>notified</em> or <em>interrupted</em>.
     */
    public final void wait() throws InterruptedException {
        wait(0L);
    }
    
    ...
}
```

**字节码**

![](https://user-images.githubusercontent.com/2216435/120472834-f5da2a80-c3d8-11eb-9612-b6affc79d193.png)

可见**synchronized**是通过**monitorenter / monitorexit**实现

## monitor和lock是什么关系？

lock: 狭义而言，lock仅实现互斥

monitor: 是一种语言层面的高级同步结构，支持互斥以及线程间的协同。

java中任意一个对象可以说就是或者包含了一个monitor对象。互斥是通过头部lock字段支持，协同通过**wait/notify**方法提供

![](https://user-images.githubusercontent.com/2216435/120474578-f378d000-c3da-11eb-92b2-6ccf90079ef7.png)

special room：互斥资源

entry set:  等待获得锁的线程

wait set:  仅通过调用**wait()**进入，等待满足某种条件的线程队列

**参考**

[In Java, what is the difference between a monitor and a lock](https://stackoverflow.com/questions/49610644/in-java-what-is-the-difference-between-a-monitor-and-a-lock)

### **Semaphores vs Monitors**

可以认为monitor是semaphore的封装优化版本

![](https://user-images.githubusercontent.com/2216435/120475758-59198c00-c3dc-11eb-89dc-671ad843a563.png)

### notify如何激活等待线程

**线程状态**

![](https://user-images.githubusercontent.com/2216435/120476594-54a1a300-c3dd-11eb-8884-98b8991cdf60.png)

**notify的两种语义**

- hoare monitor
- mesa monitor (java uses)

![](https://user-images.githubusercontent.com/2216435/120476436-2c19a900-c3dd-11eb-9309-4170048a34a4.png)

java使用的是mesa语义，也就是说仅将激活线程放入ready 队列，并不一定会立即得到操作系统调度机会

**参考**

[cseweb.ucsd.edu: Semaphores and Monitors](https://cseweb.ucsd.edu/classes/fa05/cse120/lectures/120-l6.pdf)

[Monitors – The Basic Idea of Java Synchronization](https://www.programcreek.com/2011/12/monitors-java-synchronization-mechanism/)

## 重要的类及其关系

![](https://user-images.githubusercontent.com/2216435/120740514-e9171d00-c525-11eb-9c8b-5a28d897b132.jpg)

## ReentrantLock的一些分析

重点在于ReentrantLock，Sync，AbstractQueuedSynchronizer，Condition之间的关系，可从上面类图和源码中分析



一个锁通过生成新的Condition，原则上可以有无限多个等待队列。等待队列只能通过await()方法进入，通过signal()方法退出

![](https://user-images.githubusercontent.com/2216435/120757977-da3f6300-c543-11eb-9a25-1f80e7a48e92.png)



持锁线程，抢锁线程，等待线程之间的流转

![](https://user-images.githubusercontent.com/2216435/120756871-63559a80-c542-11eb-9ef8-148097cfbaae.png)

争抢完毕

![](https://user-images.githubusercontent.com/2216435/120756901-6ea8c600-c542-11eb-8d73-841034873e33.png)



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



## LockSupport

```
unpark
park
```

锁即为同步线程所用，要更好的理解锁，少不了去深究java线程，暂且不表


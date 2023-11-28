---
layout: post
title:  "java锁与线程-2-java之thread模型"
date:   2021-06-09
catalog: true
tags:
    - java
    - thread
    - hotspot
    - jvm
---
## 1.green threads

所谓绿色线程或者虚拟线程，是指由虚拟机或者程序库自行实现。java1.2(含)以前使用的即为绿色线程，1.3之后切换为native 线程，即底层操作系统线程

> In [computer programming](https://en.wikipedia.org/wiki/Computer_programming), **green threads** or **virtual threads** are [threads](https://en.wikipedia.org/wiki/Thread_(computing)) that are scheduled by a [runtime library](https://en.wikipedia.org/wiki/Runtime_library) or [virtual machine](https://en.wikipedia.org/wiki/Virtual_machine) (VM) instead of natively by the underlying [operating system](https://en.wikipedia.org/wiki/Operating_system) (OS). Green threads emulate multithreaded environments without relying on any native OS abilities, and they are managed in [user space](https://en.wikipedia.org/wiki/User_space) instead of [kernel](https://en.wikipedia.org/wiki/Kernel_(operating_system)) space, enabling them to work in environments that do not have native thread support
>
> Green threads refers to the name of the original thread [library](https://en.wikipedia.org/wiki/Library_(computing)) for the programming language [Java](https://en.wikipedia.org/wiki/Java_(programming_language)) (that was release in version [1.1](https://en.wikipedia.org/wiki/Java_version_history#JDK_1.1) and then Green threads were abandoned in version [1.3](https://en.wikipedia.org/wiki/Java_version_history#J2SE_1.3) to native threads). It was designed by *The Green Team* at [Sun Microsystems](https://en.wikipedia.org/wiki/Sun_Microsystems)

详见[Green threads](https://en.wikipedia.org/wiki/Green_threads)

<br/>

## 2.hotspot 线程模型[^1]

对于hotspot虚拟机，java线程(java.lang.Thread)跟底层操作系统线程是**1:1**映射的，java线程由操作系统进行调度。



对于linux操作系统，其线程实现采用[NPTL](https://man7.org/linux/man-pages/man7/nptl.7.html)

```
NPTL (Native POSIX Threads Library) is the GNU C library POSIX threads implementation that is used on modern Linux systems
```

类似进程轻量级fork()机制实现，完全由os scheduler调度分配。通过`ps -eLf`，可以查看java进程和线程的关系

![](https://user-images.githubusercontent.com/2216435/121337636-558f8300-c94f-11eb-8e49-47aaf56474ab.png)

<br/>



## 3.java,jvm和操作系统线程关系[^2]

![](https://user-images.githubusercontent.com/2216435/121341870-aa34fd00-c953-11eb-8642-06120633e755.png)

其中JavaThread，OSThread由hotspot用c++实现，并持有java.lang.thread对象

[OSThread](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/share/vm/runtime/osThread.hpp#L63)

```
class OSThread: public CHeapObj<mtThread> {
 private:
  OSThreadStartFunc _start_proc;  // Thread start routine
  void* _start_parm;              // Thread start routine parameter
  
 public:
  OSThread(OSThreadStartFunc start_proc, void* start_parm);
  ~OSThread();

  // Platform dependent stuff
#ifdef TARGET_OS_FAMILY_linux
# include "osThread_linux.hpp"
#endif
#ifdef TARGET_OS_FAMILY_windows
# include "osThread_windows.hpp"
#endif
...
```

[osThread_linux](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/os/linux/vm/osThread_linux.hpp)

```
#ifndef OS_LINUX_VM_OSTHREAD_LINUX_HPP
#define OS_LINUX_VM_OSTHREAD_LINUX_HPP
 public:
  typedef pid_t thread_id_t;

 private:
  int _thread_type;

 public:

  int thread_type() const {
    return _thread_type;
```

[thread.hpp](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/share/vm/runtime/thread.hpp)

```
class Thread: public ThreadShadow {
  friend class VMStructs;

protected:
  // OS data associated with the thread
  OSThread* _osthread;  // Platform-specific thread information
```

[JavaThread](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/share/vm/runtime/thread.hpp)

```
// Class hierarchy
// - Thread
//   - NamedThread
//     - VMThread
//     - ConcurrentGCThread
//     - WorkerThread
//       - GangWorker
//       - GCTaskThread
//   - JavaThread
//   - WatcherThread

class JavaThread: public Thread {
  friend class VMStructs;
 private:
  JavaThread*    _next;                          // The next thread in the Threads list
  oop            _threadObj;                     // The Java level thread object
 
 // JSR166 per-thread parker
 private:
   Parker*    _parker;
 public:
   Parker*     parker() { return _parker; }
 ......
```

<br/>

## 4.vm视角的线程状态

- **_thread_new**: a new thread in the process of being initialized
- **_thread_in_Java**: a thread that is executing Java code
- **_thread_in_vm**: a thread that is executing inside the VM
- **_thread_blocked**: the thread is blocked for some reason (acquiring a lock, waiting for a condition, sleeping, performing a blocking I/O operation and so forth)

为了debug方便，还支持额外状态

- **MONITOR_WAIT**: a thread is waiting to acquire a contended monitor lock
- **CONDVAR_WAIT**: a thread is waiting on an internal condition variable used by the VM (not associated with any Java level object)
- **OBJECT_WAIT**: a thread is performing an Object.wait() call

<br/>

### 4.1 虚拟机内部的不同线程

对于“hello world”这样简单的java程序，在jvm内部也会启动很多线程

- **VM thread**: This singleton instance of VMThread is responsible for executing VM operations, which are discussed below
- **Periodic task thread**: This singleton instance of WatcherThread simulates timer interrupts for executing periodic operations within the VM
- **GC threads**: These threads, of different types, support parallel and concurrent garbage collection
- **Compiler threads**: These threads perform runtime compilation of bytecode to native code
- **Signal dispatcher thread**: This thread waits for process directed signals and dispatches them to a Java level signal handling method

<br/>

### 4.2 VM Operations and Safepoints

虚拟机将所有线程维护在**Threads_list**里，由**Threads_lock**来同步

所有java代码在**JavaThread**线程内执行，虚拟机操作在**VMThread**执行

何谓**Safepoints**，垃圾回收中**stop the world**即为safepoint，此时虚拟机等待其它线程都被block，并取得**Threads_lock**，然后执行相关操作

其它的vm operation包括：偏向锁操作, 线程栈 dump, 线程中止等 

<br/>



## 5. java.lang.Thread

- start, run

- sleep, join, yield

- **stop**, suspend

  > Stopping a thread with Thread.stop causes it to unlock all of the monitors that it has locked. If any of the objects previously protected by these monitors were in an inconsistent state, the damaged objects become visible to other threads, potentially resulting in arbitrary behavior

  stop主要问题是有可能造成数据不一致情况，所以已经不建议使用

- **interrupt**[^3] [^5]

- ThreadLocal

上述API中，主要讨论一下interrupt实现[^6]，其它可管窥。

### 5.1 interrupt

```
public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
```

上面说**stop**方法已不建议使用，而对**interrupt0**的注释也仅是just set the flag，标记而已。这意味着，被中断线程其**run**方法必须自行检测中断状态，否则就会继续执行下去。别人对中断标记，是可以爱搭不理的。

```
	// can not be interrupted
	Thread abnormal = new Thread(() -> {
            System.out.println("abnormal thread start");
            int i = 0;
            while (true) { i++;}
        }, "abnormal_thread");
        
        // interruptible
        Thread normal = new Thread(() -> {
            System.out.println("normal thread start");
            int i = 0;
            while (!Thread.currentThread().isInterrupted()) { i++;}
             System.out.println("normal thread end");
        }, "normal_thread");

        normal.start();
        TimeUnit.MILLISECONDS.sleep(100);
        normal.interrupt();
```

#### 5.1.1 interrupt0 in jvm

[jdk8u/hotspot/src/os/linux/vm/os_linux.cpp](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/os/linux/vm/os_linux.cpp)

```
void os::interrupt(Thread* thread) {
  assert(Thread::current() == thread || Threads_lock->owned_by_self(),
    "possibility of dangling Thread pointer");

  OSThread* osthread = thread->osthread();

  if (!osthread->interrupted()) {
    // 设置中断标记
    osthread->set_interrupted(true);
    OrderAccess::fence();
    ParkEvent * const slp = thread->_SleepEvent ;
    // 1.唤醒休眠
    if (slp != NULL) slp->unpark() ;
  }

  // For JSR166. Unpark even if interrupt status already was set
  if (thread->is_Java_thread())
  	// 2.唤醒同步
    ((JavaThread*)thread)->parker()->unpark();

  ParkEvent * ev = thread->_ParkEvent ;
  // 3.唤醒同步
  if (ev != NULL) ev->unpark() ;

}
```

拿linux操作系统为例，interrupt主要做了两件事：

* 设置中断标记
* 唤醒休眠和同步等待

更详尽解释可参考[^4]，至于**unpark**干了什么，可参考本系列第三篇文章。

<br/>

## 6.参考

[^1]: [hotSpot Runtime Overview: Thread Management](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html)
[^2]: [Distinguishing between Java threads and OS threads?](https://stackoverflow.com/questions/1888160/distinguishing-between-java-threads-and-os-threads)
[^3]:  [What does java.lang.Thread.interrupt() do?](https://stackoverflow.com/questions/3590000/what-does-java-lang-thread-interrupt-do)
[^4]: [JNi - Analysis of System Level Thread Interruption Principles from JVM Source Codes](https://programmerall.com/article/24842128375/)
[^5]: [Managing the Java Thread Lifecycle: Java Thread Interrupts vs. Hardware/OS Interrupts](https://www.dre.vanderbilt.edu/~schmidt/cs891s/2020-PDFs/13.4.5-thread-lifecycle-pt5-java-interrupts-vs-hardware-os-interrupts.pdf)
[^6]: [Java’s Mysterious Interrupt](https://carlmastrangelo.com/blog/javas-mysterious-interrupt)


---
layout: post
title:  "java锁与线程-3-LockSupport之park/unpark实现溯源"
date:   2021-06-10
categories: java thread hotspot jvm locksupport park unpark futex
---
## 1.茴字写法

AQS是jdk中锁实现的基类，而其底层实现依赖于LockSuport的park()/unpark()方法。

好奇park英文原意，下面是韦氏大词典的部分解释：

| 3.b语意解释比较符合park在锁实现中起的作用，即在某个地方停留一段时间 |
| ------------------------------------------------------------ |
| <img alt="dictionary" src="https://user-images.githubusercontent.com/2216435/121449922-5404a000-c9cd-11eb-96b8-7d91b595177a.png" style="zoom:40%; float: left;"> |

<br/>

## 2.LockSupport

```
public class LockSupport {
  // Hotspot implementation via intrinsics API
  private static final Unsafe U = Unsafe.getUnsafe();
  private static final long PARKBLOCKER = U.objectFieldOffset(Thread.class, "parkBlocker");

  // Disables the current thread for thread scheduling purposes unless 
  // the permit is available.
  public static void park() {
        U.park(false, 0L);
  }
  
  // Makes available the permit for the given thread, if it was not already available. 
  // If the thread was blocked on park then it will unblock. Otherwise, its next call to
  // park is guaranteed not to block. This operation is not guaranteed to have any effect 
  // at all if the given thread has not been started.
  public static void unpark(Thread thread) {
        if (thread != null)
            U.unpark(thread);
  }
}
```

从上面代码来看，**park/unpark**很简单，主要实现在**Unsafe**类：

```
public final class Unsafe {
	public native void unpark(Object thread);
	
	public native void park(boolean isAbsolute, long time);
}
```

但是Unsafe中的定义是native，也就意味着，得去hotspot源码中看真正的底层实现

<br/>

## 3.溯源

### 3.1 hotspot中的Unsafe实现

[jdk8u/hotspot/src/share/vm/prims/unsafe.cpp](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/share/vm/prims/unsafe.cpp)

首先看**park**方法

```
// These are the methods for 1.8.0
// 方法名映射
static JNINativeMethod methods_18[] = {
    ...
    ...
    {CC "park",               CC "(ZJ)V",                  FN_PTR(Unsafe_Park)},
    {CC "unpark",             CC "(" OBJ ")V",               FN_PTR(Unsafe_Unpark)}
};

UNSAFE_ENTRY(void, Unsafe_Park(JNIEnv *env, jobject unsafe, jboolean isAbsolute, jlong time))
  UnsafeWrapper("Unsafe_Park");
  EventThreadPark event;
  ...
  JavaThreadParkedState jtps(thread, time != 0);
  // 重要方法
  thread->parker()->park(isAbsolute != 0, time);
	...
UNSAFE_END
```

再看**unpark**方法

```
UNSAFE_ENTRY(void, Unsafe_Unpark(JNIEnv *env, jobject unsafe, jobject jthread))
  UnsafeWrapper("Unsafe_Unpark");
  Parker* p = NULL;
  ...
  java_thread = JNIHandles::resolve_non_null(jthread);
  // jvm线程
  JavaThread* thr = java_lang_Thread::thread(java_thread);
  p = thr->parker();
  if (p != NULL) {
    // 关键点
    p->unpark();
  }
UNSAFE_END
```

可见，其关键实现在**JavaThread**中的parker对象

<br/>



### 3.2 JavaThread

Java.lang.Thread是jdk应用层面的线程，JavaThread为hotspot中的线程，一一对应，关系如下

|                         jvm中的线程                          |
| :----------------------------------------------------------: |
| <img src="https://user-images.githubusercontent.com/2216435/121341870-aa34fd00-c953-11eb-8642-06120633e755.png" alt="header" style="zoom:35%; float: left;" /> |

[jdk8u/hotspot/src/share/vm/runtime/thread.hpp->JavaThread](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/share/vm/runtime/thread.hpp)

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

class Thread: public ThreadShadow {
  
  protected:
  // OS data associated with the thread
  OSThread* _osthread;  // Platform-specific thread information
  
  ......
}

class JavaThread: public Thread {
	oop          _threadObj; // The Java level thread object
	
	// JSR166 per-thread parker
	private:
  	Parker*    _parker;  
  
  ......
}
```

还没到，我们还需要进一步看Parker实现

<br/>

### 3.3 Parker和PlatformParker

[jdk8u/hotspot/src/share/vm/runtime/park.hpp](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/share/vm/runtime/park.hpp)

```
class Parker : public os::PlatformParker {
private:
  // 重要变量，通过0/1表示授权，决定是否阻塞
  volatile int _counter ;
  JavaThread * AssociatedWith ; // Current association

public:
  Parker() : PlatformParker() {
    _counter       = 0 ;
    AssociatedWith = NULL ;
  }

public:
  void park(bool isAbsolute, jlong time);
  void unpark();
  ...
  ...
};
```

Parker继承自os::PlatformParker，也就是说，由各个平台具体实现

下面我们看linux下的实现：[jdk8u/hotspot/src/os/linux/vm/os_linux.hpp->PlatformParker](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/os/linux/vm/os_linux.hpp)

```
class PlatformParker : public CHeapObj<mtInternal> {
    int _cur_index;  // which cond is in use: -1, 0, 1
    // 锁
    pthread_mutex_t _mutex [1] ;
    // 条件变量
    pthread_cond_t  _cond  [2] ; // one for relative times and one for abs.
    ...
    ...
};
```

**park()/unpark()最终实现**：[jdk8u/hotspot/src/os/linux/vm/os_linux.cpp](https://github.com/openjdk/jdk8u/blob/master/hotspot/src/os/linux/vm/os_linux.cpp)

```
void Parker::park(bool isAbsolute, jlong time) {
  // 原_counter不为零，有权限，不需等待
  // _counter 被设置为0
  if (Atomic::xchg(0, &_counter) > 0) return;
  
	// 加锁
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }

	......
	
  if (time == 0) {
  	// 等待
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    // 计时等待
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
  }
  
  ......
  
	// 从pthread_cond_wait return表明接收到signal，_counter=1
	// 授权已经使用一次，作废
  _counter = 0 ;
  status = pthread_mutex_unlock(_mutex) ;
}
```

```
void Parker::unpark() {
  // 整体按照加锁->通知->解锁的顺序
  int s, status ;
  status = pthread_mutex_lock(_mutex);
  s = _counter;
  _counter = 1;
  if (s < 1) {
    // thread might be parked
    if (_cur_index != -1) {
      // thread is definitely parked
      if (WorkAroundNPTLTimedWaitHang) {
        status = pthread_cond_signal (&_cond[_cur_index]);
        status = pthread_mutex_unlock(_mutex);
      } else {
        int index = _cur_index;
        status = pthread_mutex_unlock(_mutex);
        status = pthread_cond_signal (&_cond[index]);
      }
    } else {
      pthread_mutex_unlock(_mutex);
    }
  } else {
    pthread_mutex_unlock(_mutex);
  }
}
```

到这里，我们应该可以说是较为透彻但仅是粗线条的搞清了锁和线程同步底层实现的脉络

<br/>

### 3.4 类的关系

上面介绍的几个底层实现类，其关系如下

|                 线程与关键属性及方法调用关系                 |
| :----------------------------------------------------------: |
| <img src="https://user-images.githubusercontent.com/2216435/121477019-e6229d80-c9f9-11eb-981e-2f4a65db31e9.png" alt="header" style="zoom:35%; float: left;" /> |

<br/>

## 4.pthread_cond_wait？

### 4.1 继续追踪

追踪至**pthread_cond_wait**，可以说明确了实现脉络，其内部究竟如何实现呢？

通过本节参考，使用strace，可以再进一步。示例代码节选自本节参考

```
#include <stdio.h>
...

int main() {
    pthread_cond_t cond;
    pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

    if (initializeCond(&cond)) {
        return -1;
    }
    ...
    
    struct timespec start, end, deadline;
    if (pthread_mutex_lock(&lock)) {
        return -1;
    }
    clock_gettime(CLOCK_TYPE, &start);
    addNanos(&start, &deadline, deltaNanos);

    int iteration = 0;
    while (!isDeadlineReached(&deadline)) {
        // pthread_cond_timedwait(&cond, &lock, &deadline);
    	  pthread_cond_wait(&cond, &lock);
        iteration++;;
    }
		...
    clock_gettime(CLOCK_TYPE, &end);
    return 0;
}
```

编译：`gcc test.c -o a.out -lpthread`

追踪：`strace -e futex ./a.out`

|                            futex                             |
| :----------------------------------------------------------: |
| <img src="https://user-images.githubusercontent.com/2216435/121489839-0e64c900-ca07-11eb-804f-cc4d276cc79b.png" alt="header" style="zoom:70%; float: left;" /> |

本节参考[^4]

由此可见，**pthread_cond_wait/pthread_cond_timedwait**由futex实现。那么futex又是什么东西呢？

<br/>

### 4.2 futex

> The `futex()` system call provides a method for waiting until a certain condition becomes true. It is typically used as a blocking construct in the context of shared-memory synchronization. When using futexes, the majority of the synchronization operations are performed in user space. A user-space program employs the `futex()` system call only when it is likely that the program has to block for a longer time until the condition becomes true. Other `futex()` operations can be used to wake any processes or threads waiting for a particular condition.

futex方法里重要的是**FUTEX_WAIT** 和 **FUTEX_WAKE**，基本对应了wait/notify



何谓fast，大多数情况下，锁是没有竞争的，通过用户态原子操作即可完成，轻量级；少数需要等待情况下，才进入内核态，是谓重量级操作[^5] [^6] [^7]

|                   fast user space locking                    |
| :----------------------------------------------------------: |
| <img src="https://user-images.githubusercontent.com/2216435/122011603-dab2e600-cdee-11eb-9bb9-1636a2eb247e.png" alt="header" style="zoom:45%; float: left;" /> |

<br/>

## 5.线程休眠与唤醒的实现

### 5.1 java.lang.Object

这块作为对比，放在后面

```
public class Object {
  // Causes the current thread to wait until it is awakened,
  // typically by being notified or interrupted
	public final void wait() throws InterruptedException {
        wait(0L);
  }
  
  public final native void wait(long timeoutMillis) throws InterruptedException;
  
  // Wakes up a single thread that is waiting on this object's monitor. 
  // If any threads are waiting on this object, one of them is chosen to be awakened. 
  // The choice is arbitrary and occurs at the discretion of the implementation. 
  // A thread waits on an object's monitor by calling one of the wait methods.
  // 
  // The awakened thread will not be able to proceed until the current thread 
  // relinquishes the lock on this object. The awakened thread will compete in the 
  // usual manner with any other threads that might be actively competing to synchronize 
  // on this object; for example, the awakened thread enjoys no reliable privilege or
  // disadvantage in being the next thread to lock this object.
  // 
  // This method should only be called by a thread that is the owner of this object's
  // monitor. A thread becomes the owner of the object's monitor in one of three ways:
  public final native void notify();
}
```

本节**参考**[^8] [^9]

<br/>

### 5.2 进程或者线程如何sleep和wait()？

锁等待状态需要主动放弃时间片时，底层需要调用常见方法**sleep/wait/yield**，起到释放的作用

我们可以看看 [**xv6** proc.c](https://github.com/selfpoised/xv6-public/blob/master/proc.c)操作系统是如何实现上面三个方法的，以管窥linux系统实现

```
// Give up the CPU for one scheduling round.
void yield(void)
{
  acquire(&ptable.lock);  //DOC: yieldlock
  myproc()->state = RUNNABLE;
  sched();
  release(&ptable.lock);
}

// Wait for a child process to exit and return its pid.
int wait(void)
{
  ...
  for(;;){
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state == ZOMBIE){
        // 回收子进程资源
        pid = p->pid;
        p->killed = 0;
        ...
        p->state = UNUSED;
        return pid;
      }
    }

    sleep(curproc, &ptable.lock);
  }
}

void sleep(void *chan, struct spinlock *lk)
{
  ...
  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;
  ...
}
```

可见这三个方法的核心是**sched()**

```
// Enter scheduler
void sched(void)
{
  int intena;
  struct proc *p = myproc();
  ...
  intena = mycpu()->intena;
  // 切换当前进程上下文，调度器上下文，进入调度器
  swtch(&p->context, mycpu()->scheduler);
  mycpu()->intena = intena;
}
```

swtch.S是一段汇编代码，保存当前线程寄存器，并将寄存器内容设置为调度器上下文，以此完成切换

```
# Context switch
#
#   void swtch(struct context **old, struct context *new);
# 
# Save the current registers on the stack, creating
# a struct context, and save its address in *old.
# Switch stacks to new and pop previously-saved registers.

.globl swtch
swtch:
  movl 4(%esp), %eax
  movl 8(%esp), %edx

  # Save old callee-saved registers
  pushl %ebp
  pushl %ebx
  pushl %esi
  pushl %edi

  # Switch stacks
  movl %esp, (%eax)
  movl %edx, %esp

  # Load new callee-saved registers
  popl %edi
  popl %esi
  popl %ebx
  popl %ebp
  ret
```

<br/>

## 6.参考

[^1]: [HotSpot Runtime Overview](https://openjdk.java.net/groups/hotspot/docs/RuntimeOverview.html)
[^2]: [啃透Java并发-LockSupport源码详解](https://juejin.cn/post/6844903938202796039)
[^3]: [LockSupport source code reading and analysis](https://www.programmersought.com/article/85695649618/)
[^4]:[LockSupport.parkNanos() Under the Hood and the Curious Case of Parking](https://hazelcast.com/blog/locksupport-parknanos-under-the-hood-and-the-curious-case-of-parking/)
[^5]: [Basics of Futexes](https://eli.thegreenplace.net/2018/basics-of-futexes/)
[^6]:[Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf)
[^7]:[Linux Locking Mechanisms](https://www.slideshare.net/kerneltlv/linux-locking-mechanisms)
[^8]: [Unsafe.park vs Object.wait](https://newbedev.com/unsafe-park-vs-object-wait)
[^9]: [park与unpark、notify与notifyAll](https://segmentfault.com/a/1190000037681656)

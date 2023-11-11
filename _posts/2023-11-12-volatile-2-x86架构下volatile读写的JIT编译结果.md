---
layout: post
title:  "volatile-2-x86架构下volatile读写的JIT编译结果"
date:   2023-11-11
categories: jitwatch volatile JIT instruction memory barrier
---



## 0.为什么要写本篇？

既然**jitwatch**可以查看**JIT**编译后的机器码，那么应该可以调戏volatile变量，对其进行读写，然后看看产生的**内存屏障(memory barrier)**到底是什么样的？能看到**loadload, loadstore, storeload, storestore**对应的指令吗？

<br/>

## 1.JIT编译后的屏障是什么样的？

环境：jdk1.8, Intel Core i5

| **volatile read**  | <img src="https://user-images.githubusercontent.com/2216435/282236207-a03d55ec-1cc3-4656-99ff-055d17e30d89.png" alt="read" style="zoom:50%; float: left;" /> |
| ------------------ | ------------------------------------------------------------ |
| **volatile write** | <img src="https://user-images.githubusercontent.com/2216435/282236758-23b5d644-65b6-4d6c-95b5-0bd935d95d4c.png" alt="write" style="zoom:50%; float: left;" /> |

可见在x86架构下，JIT编译后的代码，仅在写时(更精确是**storeload**)会生成**mem bar**。

<br/>

## 2.内存屏障分类

**volatile**语义保证有两个内涵：

* 编译器层面，防止指令重排
* cpu层面，从cache间数据可见性角度防止指令重排

为了细化场景，提升性能，分为四种屏障[^3]：

* loadload：**Load1; LoadLoad; Load2**, [^1]

  ```
  if (IsPublished)                   // Load and check shared flag
  {
      LOADLOAD_FENCE();              // Prevent reordering of loads
      return Value;                  // Load published value
  }
  ```

* storestore：**Store1; StoreStore; Store2**

  ```
  Value = x;                         // Publish some data
  STORESTORE_FENCE();
  IsPublished = 1;                   // Set shared flag to indicate availability of data
  ```

* loadstore：**Load1; LoadStore; Store2**

* storeload：**Store1; StoreLoad; Load2**，相当于完整内存屏障，也是代价最大的屏障[^2]

  | 唯一可避免：r1 = 0 and r2 = 0出现的屏障[^1]                  |
  | ------------------------------------------------------------ |
  | <img src="https://user-images.githubusercontent.com/2216435/282244955-8f7f176a-428c-445b-b8d5-49676a4d8b46.png" alt="comment out" style="zoom:100%; float: left;" /> |

<br/>

## 3. jvm中的实现

### 3.1 屏障定义[^4]

[openjdk: jdk/src/hotspot/share/runtime/orderAccess.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/orderAccess.hpp)

```
class OrderAccess : public AllStatic {
 public:
  static void     loadload();
  static void     storestore();
  static void     loadstore();
  static void     storeload();

  static void     acquire();
  static void     release();
  static void     fence();
  ...
};

```

<br/>

### 3.2 x86平台下linux实现

[openjdk: jdk/src/hotspot/os_cpu/linux_x86/orderAccess_linux_x86.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/os_cpu/linux_x86/orderAccess_linux_x86.hpp)

```
// Implementation of class OrderAccess.

// A compiler barrier, forcing the C++ compiler to invalidate all memory assumptions
static inline void compiler_barrier() {
  __asm__ volatile ("" : : : "memory");
}

inline void OrderAccess::loadload()   { compiler_barrier(); }
inline void OrderAccess::storestore() { compiler_barrier(); }
inline void OrderAccess::loadstore()  { compiler_barrier(); }
inline void OrderAccess::storeload()  { fence();            }

inline void OrderAccess::acquire()    { compiler_barrier(); }
inline void OrderAccess::release()    { compiler_barrier(); }

inline void OrderAccess::fence() {
   // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
  __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
  __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  compiler_barrier();
}
```

在x86平台上，可见**storeload**是唯一需要指令实现的屏障且等效于完整内存屏障**fence**；其它屏障在cpu层面就符合可见性要求，只需防止编译重排即可。

| 各平台cache可见性标准                                        |
| ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/282245622-80529551-b94f-4143-b771-919c80dbf9eb.png" alt="cache coherence of platforms" style="zoom:45%; float: left;" /> |

<br/>

## 4.`__asm__ volatile`含义[^5]

* `__asm__`表示在c语言中嵌入汇编代码
* `__asm__ volatile`表示不允许优化器删除，cache，乱序等
* `__asm__ volatile ("" : : : "memory")`禁止编译器指令重排
* `__asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory")` 表示`lock; addl $0,0(%%esp)`适用上述限制

<br/>

## 5.x86屏障指令

根据**IA32-3 7.2 memory odering**小节:

- **SFENCE**—指令将程序指令流中在其之前发生的所有存储（写入）操作进行序列化，但不影响加载（读取）操作
- **LFENCE**—指令将程序指令流中在其之前发生的所有加载（读取）操作进行序列化，但不影响存储（写入）操作
- **MFENCE**—指令将程序指令流中在其之前发生的所有存储（写入）和加载（读取）操作进行序列化
- **LOCK**—在多处理器环境中，LOCK指令防止读写重排，并独占共享内存且完成原子化操作。<u>**Locked instructions have a total order**</u> [^6]。一般而言lock性能较高，会替代**MFENCE**使用。

<br/>

## 6.references

[^1]: [Memory Barriers Are Like Source Control Operations](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
[^2]: [内存屏障及其在 JVM 内的应用（下）](https://segmentfault.com/a/1190000022508589)
[^3]: [jdk MemoryBarriers](https://github.com/openjdk/jdk/blob/6bab0f539fba8fb441697846347597b4a0ade428/src/jdk.internal.vm.ci/share/classes/jdk.vm.ci.code/src/jdk/vm/ci/code/MemoryBarriers.java)
[^4]:[全网最硬核 Java 新内存模型解析与实验 - 5. JVM 底层内存屏障源码分析](https://juejin.cn/user/501033033545053/posts)
[^5]: [What does __asm__ __volatile__ do in C?](https://stackoverflow.com/questions/26456510/what-does-asm-volatile-do-in-c)
[^6]: [Intel® 64 Architecture Memory Ordering White Paper ](https://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf)


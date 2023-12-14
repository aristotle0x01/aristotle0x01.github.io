---
layout: post
title:  "volatile-2-volatile在jvm内部的实现"
date:   2023-11-12
catalog: true
tags:
    - volatile
    - java
    - JIT
    - memory barrier
    - jitwatch
---



## 0.为什么要写本篇？

既然**jitwatch**可以查看**JIT**编译后的机器码，那么应该可以调戏volatile变量，对其进行读写，然后看看产生的**内存屏障(memory barrier)**到底是什么样的？能看到**loadload, loadstore, storeload, storestore**对应的指令吗？

## 1.JIT编译后的屏障是什么样的？

环境：jdk1.8, Intel Core i5

| **volatile read**  | <img src="https://user-images.githubusercontent.com/2216435/282236207-a03d55ec-1cc3-4656-99ff-055d17e30d89.png" alt="read" style="zoom:50%; float: left;" /> |
| ------------------ | ------------------------------------------------------------ |
| **volatile write** | <img src="https://user-images.githubusercontent.com/2216435/282236758-23b5d644-65b6-4d6c-95b5-0bd935d95d4c.png" alt="write" style="zoom:50%; float: left;" /> |

可见在x86架构下，JIT编译后的代码，仅在写时(更精确是**storeload**)会生成**mem bar**。

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

## 3. 内存屏障的实现

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
| <img src="https://user-images.githubusercontent.com/2216435/282245622-80529551-b94f-4143-b771-919c80dbf9eb.png" alt="cache coherence of platforms" style="zoom:40%; float: left;" /> |

### 3.3 `__asm__ volatile`含义[^5]

* `__asm__`表示在c语言中嵌入汇编代码
* `__asm__ volatile`表示不允许优化器删除，cache，乱序等
* `__asm__ volatile ("" : : : "memory")`禁止编译器指令重排
* `__asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory")` 表示`lock; addl $0,0(%%esp)`适用上述限制

### 3.4 x86屏障指令

根据**IA32-3 7.2 memory odering**小节:

- **SFENCE**—指令将程序指令流中在其之前发生的所有存储（写入）操作进行序列化，但不影响加载（读取）操作
- **LFENCE**—指令将程序指令流中在其之前发生的所有加载（读取）操作进行序列化，但不影响存储（写入）操作
- **MFENCE**—指令将程序指令流中在其之前发生的所有存储（写入）和加载（读取）操作进行序列化
- **LOCK**—在多处理器环境中，LOCK指令防止读写重排，并独占共享内存且完成原子化操作。<u>Locked instructions have a total order</u> [^6]。一般而言lock性能较高，会替代**MFENCE**使用。

## 4.volatile变量读写实现[^7]

### 4.1 字节码实现

`getfield & putfield`

| volatile变量读写的字节码编译                                 |
| ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/282274605-667a6613-78a2-4994-bead-16cf445f3328.png" alt="cache coherence of platforms" style="zoom:60%; float: left;" /> |

### 4.2 jvm实现(x86)

**1）定义**： *jdk8u/hotspot/src/cpu/x86/vm/templateTable_x86_32.cpp*

```
void TemplateTable::getfield(int byte_no) {
  getfield_or_static(byte_no, false);
}

void TemplateTable::putfield(int byte_no) {
  putfield_or_static(byte_no, false);
}
```

**2）具体实现**

```
void TemplateTable::getfield_or_static(int byte_no, bool is_static) {
  // 删除大量非直接相关代码
	...

  __ bind(Done);
  // Doug Lea believes this is not needed with current Sparcs (TSO) and Intel (PSO).
  // volatile_barrier( );
}

void TemplateTable::putfield_or_static(int byte_no, bool is_static) {
	// 删除大量非直接相关代码
	...
	
  // Check for volatile store
  __ testl(rdx, rdx);
  __ jcc(Assembler::zero, notVolatile);
  volatile_barrier(Assembler::Membar_mask_bits(Assembler::StoreLoad |
                                               Assembler::StoreStore));
  __ bind(notVolatile);
}

```

对于volatile的读，在x86上是不需要的，也跟**OrderAccess**中的实现一致，所以关键就看写是如何实现的。再看**`volatile_barrier`**和`Assembler::Membar_mask_bits`干了什么：

```
void TemplateTable::volatile_barrier(Assembler::Membar_mask_bits order_constraint ) {
  // Helper function to insert a is-volatile test and memory barrier
  if( !os::is_MP() ) return;    // Not needed on single CPU
  __ membar(order_constraint);
}
```

**3）最终**：*jdk8u/hotspot/src/cpu/x86/vm/assembler_x86.hpp*

```
enum Membar_mask_bits {
    StoreStore = 1 << 3,
    LoadStore  = 1 << 2,
    StoreLoad  = 1 << 1,
    LoadLoad   = 1 << 0
  };

  // Serializes memory and blows flags
  void membar(Membar_mask_bits order_constraint) {
    if (os::is_MP()) {
      // We only have to handle StoreLoad
      if (order_constraint & StoreLoad) {
        // All usable chips support "locked" instructions which suffice
        // as barriers, and are much faster than the alternative of
        // using cpuid instruction. We use here a locked add [esp],0.
        // This is conveniently otherwise a no-op except for blowing
        // flags.
        // Any change to this code may need to revisit other places in
        // the code where this idiom is used, in particular the
        // orderAccess code.
        lock();
        addl(Address(rsp, 0), 0);// Assert the lock# signal here
      }
    }
  }
```

可见只对**StoreLoad**加了屏障(基于**lock**实现)，其它几个屏障在cpu指令层面是不需要的，也与前面**OrderAccess**实现一致。

## 5.references

[^1]: [Memory Barriers Are Like Source Control Operations](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
[^2]: [内存屏障及其在 JVM 内的应用（下）](https://segmentfault.com/a/1190000022508589)
[^3]: [jdk MemoryBarriers](https://github.com/openjdk/jdk/blob/6bab0f539fba8fb441697846347597b4a0ade428/src/jdk.internal.vm.ci/share/classes/jdk.vm.ci.code/src/jdk/vm/ci/code/MemoryBarriers.java)
[^4]:[全网最硬核 Java 新内存模型解析与实验 - 5. JVM 底层内存屏障源码分析](https://juejin.cn/post/7080890011217821710)
[^5]: [What does __asm__ __volatile__ do in C?](https://stackoverflow.com/questions/26456510/what-does-asm-volatile-do-in-c)
[^6]: [Intel® 64 Architecture Memory Ordering White Paper ](https://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf)
[^7]: [How Volatile Modifier Works On JVM Level?](https://deepschneider.medium.com/how-volatile-works-on-jvm-level-7a250d38435d#:~:text=So%2C%20write%20operation%20to%20volatile,is%20synchronized%20with%20other%20core's)


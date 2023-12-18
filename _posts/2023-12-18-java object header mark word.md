---
layout: post
title:  "java object header markword诡异的实现(openjdk8u)"
date:   2023-12-18
catalog: true
tags:
    - jvm
    - java
    - hotspot
---

## 0.问题

java对象头如下所示，比较奇怪的是openjdk8u里面的实现：

```asciiarmor
|------------------------------------------------------------------------------|--------------------|
|                                  Mark Word (64 bits)                         |       State        |
|------------------------------------------------------------------------------|--------------------|
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |       Normal       |
|------------------------------------------------------------------------------|--------------------|
| thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 |       Biased       |
|------------------------------------------------------------------------------|--------------------|
|                       ptr_to_lock_record:62                         | lock:2 | Lightweight Locked |
|------------------------------------------------------------------------------|--------------------|
|                     ptr_to_heavyweight_monitor:62                   | lock:2 | Heavyweight Locked |
|------------------------------------------------------------------------------|--------------------|
|                                                                     | lock:2 |    Marked for GC   |
|------------------------------------------------------------------------------|--------------------|
```

```c++
// java.lang.Object在jvm内部基类
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
  ......
}

typedef class   markOopDesc*                markOop;

// mark word操作类
class markOopDesc: public oopDesc {
 private:
  // Conversion
  uintptr_t value() const { return (uintptr_t) this;}
 
  bool has_locker() const {
    return ((value() & lock_mask_in_place) == locked_value);
  }
  ......
}
```

可以看到**oopDesc**里**_mark**是个指针，为什么不是个简单的uint数值呢？**(uintptr_t) this**将自身直接转换为一个整型数值，以**has_locker**为例来看，其直接操作的竟然是指针本身而非指针所指内容？

根据相关文献，加上本人的臆断，使用指针的初衷是为了支持markword后续做更为灵活的扩展；如果这个被指向的对象size超过uint，那么对象头本身size并不会增加，也不会破坏对象自身的既有功能。当然，从后续jdk版本来看，并没有进行灵活扩展。

## 1.指针的非常规使用

下面我们简单验证一下把指针本身当作变量使用的场景，虽然一般情况我们都是利用指针来操作所指内容。

```c++
// t.c
int main() {
	char* p = "hello";
	unsigned long t = p;
	void** pv = &p;
	printf("1: 0x%08x 0x%08x 0x%08x %s\n", t, pv, *pv, p);
	
	int ii = 101;
	p = &ii;
	printf("2: 0x%08x 0x%08x %d\n", p, &p, *p);
  
  p = 0x01234567;
  printf("3: 0x%08x 0x%08x\n", p, &p);
}

// cc t.c
```

编译后输出：

```
// ./a.out
1: 0x00400660 0x7d3f60c8 0x00400660 hello
2: 0x7d3f60c4 0x7d3f60c8 101
3: 0x01234567 0x7d3f60c8
```

上面这段代码，p首先指向**"hello"**，其后又指向**ii**。p作为变量本身其取值(指向)变了，但是其在内存中的地址并未变化。我们甚至可以随意变更p的取值(指向)，**0x01234567**。

这意味着，我们可以不关心p的具体指向，而仅把p本身当作一个整型变量来使用。这也是**volatile markOop  _mark**如此使用的内涵所在：

> Remember, _mark is an arbitrary bit pattern describing the object. We treat it as if it were a pointer to a markOopDesc object, but it's not pointing to such an object. While using that (weird) pointer we call value() and extract it's bit pattern to be used for further bit pattern checking functions. AFAICT, this is also undefined behavior. At least is_being_inflated() const is effectively 'return (uintptr_t) this == NULL'. This UB was recently discussed here[^1]

上面是<u>JDK-8229258</u>[^1]提案里面的一段话，该提案最终将**volatile markOop  _mark**简化为整型变量。

## 2.hotspot实现

```c++
instanceOop InstanceKlass::allocate_instance(TRAPS) // hotspot/src/share/vm/oops/instanceKlass.cpp
	-> oop CollectedHeap::obj_allocate // hotspot/src/share/vm/gc_interface/collectedHeap.inline.hpp
  	-> CollectedHeap::post_allocation_setup_obj 
  		-> CollectedHeap::post_allocation_setup_common
  			-> CollectedHeap::post_allocation_setup_no_klass_install
  				-> obj->set_mark(klass->prototype_header())
  
markOop prototype_header() const      { return _prototype_header; }
```

实例化对象时，对象**__mark**默认指向同一个实体，即klass自身prototype_header。所有对象**__mark**字段都指向类对象字段，也就意味着**__mark**并不关心实际指向了什么，也验证了其自身作为变量使用。

## 3.最新实现

应该是从openjdk14[^2]后，markword字段做了简化和重构[^1]，直接使用了uint变量，理解起来就清晰多了。

```c++
class oopDesc {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  volatile markWord _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
  ......
}

class markWord {
 private:
  uintptr_t _value;
  uintptr_t value() const { return _value; }
  ......
}
```

## 4.references

[^1]: [Rework markOop and markOopDesc into a simpler mark word value carrier](https://bugs.openjdk.org/browse/JDK-8229258)
[^2]: [class oopDesc](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/oopsHierarchy.hpp)


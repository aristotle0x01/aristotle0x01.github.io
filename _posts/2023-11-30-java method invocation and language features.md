---
layout: post
title:  "java方法调用的实现"
date:   2023-11-30
catalog: true
tags:
    - jvm
    - java
    - method invocation
---



## 0.why？

1.

* invokevirtual
* invokestatic
* invokespecial
* invokeinterface
* Invokedynamic

是如何实现java类的多态和继承特性的。

2.什么是method table?

3.什么是stackframe?



## 1.既生瑜何生亮？

### 1.1 invokevirtual

**invokevirtual**比较好理解，它根据运行时对象的真实类型，来决定应该调用的方法。

```
class Dog {
    public void bark() {
        System.out.println("dog bark!");
    }
}

public class CockerSpaniel extends Dog {
    public static void main(String args[]) {
        Random r = new Random();
        Dog dog;
        if (r.nextInt() % 2 == 0){
            dog = new Dog();
        } else {
            dog = new CockerSpaniel();
        }
        dog.bark();
    }

    public void bark() {
        System.out.println("CockerSpaniel bark!");
    }
}
```

编译后字节码：

| main方法字节码                                               | dog.bark调用类型                                             |
| :----------------------------------------------------------- | ------------------------------------------------------------ |
| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/8f9720e9-0bc1-4391-9d0b-d1fa2f4b896b" alt="bytecode" style="zoom:100%; float: left;" /> | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/b994ad5e-c1e7-4d91-b552-1d72b12ee26e" alt="methodref" style="zoom:100%; float: left;" /> |

可见，尽管**dog.bark**方法字节码所在类是**Dog**，但是实际调用时，还是会根据对象实际类型来动态决定，而不是根据静态编译类型。这一点，肯定取决于jvm内部对**invokevirtual**的实现，后面再说。

### 1.2 为什么需要invokespecial？

* 对象初始化**init**方法

* **private** 方法
* **super.xxx()** 形式调用

```
class Baseclass {
    protected void f1() {
        System.out.println("base f1");
    }
}

public class Superclass extends Baseclass{
    private void interestingMethod() {
        System.out.println("Superclass's interesting method.");
    }

    void exampleMethod() {
        interestingMethod();
    }

    protected void f1() {
        System.out.println("super f1");
    }
}

class Subclass extends Superclass {
    private void interestingMethod() {
        System.out.println("Subclass's interesting method.");
    }

    protected void f1() {
        super.f1();
        System.out.println("sub f1");
    }

    public static void main(String args[]) {
        Subclass me = new Subclass();
        me.exampleMethod();
        me.f1();
    }
}
```

编译结果

| Subclass::f1              | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/d5689b4e-b693-49c5-8caf-ff4a25cfbf10" alt="f1" style="zoom:100%; float: left;" /> |
| ------------------------- | ------------------------------------------------------------ |
| Subclass::init            | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/1e6bf97f-c691-482f-affd-292e4b049e63" alt="init" style="zoom:100%; float: left;" /> |
| Superclass::exampleMethod | <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/ee06f3f3-ace4-4b31-8d77-6637faf0642a" alt="exampleMethod" style="zoom:100%; float: left;" /> |

上述三种场景下，如果仍然使用**invokevirtual**显然是错误的，此时必须根据编译时静态类型来决定调用哪个方法。因此必须使用**invokespecial**来区分。

* 如果注释掉Superclass中的**f1**方法，则Subclass中**f1**方法中调用**super.f1()**实际调用**Baseclass.f1**，即由近及远向上追溯
* 尝试修改一下**Subclass::main()**中**me.f1()**字节码，将其从**invokevirtual->invokespecial**(IntelliJ **jclasslib**支持修改字节码)

| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/6dd523da-b12e-4123-9a50-a0e9dae0fdb6" alt="invokevirtual" style="zoom:100%; float: left;" /> | Superclass's interesting method.<br/>super f1<br/>sub f1 |
| ------------------------------------------------------------ | -------------------------------------------------------- |
| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/6aa2af3e-f112-4b71-98b1-636fb35d13f9" alt="invokespecial" style="zoom:100%; float: left;" /> | Superclass's interesting method.<br/>super f1            |

### 1.2 为什么需要invokeinterface?

```
public class UnrelatedClass {
    public static void main(String args[]) {
        Subclass2 sc = new Subclass2(0); 
        sc.interfaceMethod(); // invokevirtual

        InYourFace iyf = sc;
        iyf.interfaceMethod();  // invokeinterface
    }
}

interface InYourFace {
    void interfaceMethod ();
}

class Subclass1 implements InYourFace {
    Subclass1(int i) {
        super(); 
    }

    public void interfaceMethod() {}
}

class Subclass2 extends Subclass1 {
    Subclass2(int i) {
        super(i);
    }
}
```

编译后字节码：

| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/fa467379-5f19-4f5c-a564-121577eea042" alt="invokeinterface" style="zoom:100%; float: left;" /> |
| ------------------------------------------------------------ |

>The invokeinterface opcode performs the same function as invokevirtual, it invokes instance methods and uses dynamic binding. The difference between these two instructions is that invokevirtual is used when the type of the reference is a class, whereas invokeinterface is used when the type of the reference is an interface.
>
>The Java Virtual Machine uses a different opcode to invoke a method on an interface reference because it can't make as many assumptions about the method table offset given an interface reference as it can given a class reference. (Method tables are described in Chapter 8, "The Linking Model.") Given a class reference, a method will always occupy the same position in the method table, independent of the actual class of the object. This is not true given an interface reference. The method could occupy different locations for different classes that implement the same interface.[^1]

* invokeinterface 和 invokevirtual大体相同，都实现多态。当且仅当使用interface类型调用方法时，使用invokeinterface
* invokevirtual在方法表(method table)中的索引是确定的，而invokeinterface是不确定的

那么问题来了，什么是method table?

## 2.什么是method table

首先看一段代码[^2]：

```
interface Friendly {
    void sayHello();
    void sayGoodbye();
}

class Dog {
    // How many times this dog wags its tail when saying hello.
    private int wagCount = ((int) (Math.random() * 5.0)) + 1;

    void sayHello() {
        System.out.print("Wag");
        for (int i = 0; i < wagCount; ++i) {
            System.out.print(", wag");
        }
        System.out.println(".");
    }

    public String toString() {
        return "Woof!";
    }
}

class CockerSpaniel extends Dog implements Friendly {
    // How many times this Cocker Spaniel woofs when saying hello.
    private int woofCount = ((int) (Math.random() * 4.0)) + 1;

    // How many times this Cocker Spaniel wimpers when saying goodbye.
    private int wimperCount = ((int) (Math.random() * 3.0)) + 1;

    public void sayHello() {
        // Wag that tail a few times.
        super.sayHello();

        System.out.print("Woof");
        for (int i = 0; i < woofCount; ++i) {
            System.out.print(", woof");
        }
        System.out.println("!");
    }

    public void sayGoodbye() {
        System.out.print("Wimper");
        for (int i = 0; i < wimperCount; ++i) {
            System.out.print(", wimper");
        }
        System.out.println(".");
    }
}

class Cat implements Friendly {
    public void eat() {
        System.out.println("Chomp, chomp, chomp.");
    }

    public void sayHello() {
        System.out.println("Rub, rub, rub.");
    }

    public void sayGoodbye() {
        System.out.println("Scamper.");
    }

    protected void finalize() {
        System.out.println("Meow!");
    }
}

class Example4 {
    public static void main(String[] args) {
        Dog dog = new CockerSpaniel();

        dog.sayHello();

        Friendly fr = (Friendly) dog;

        // Invoke sayGoodbye() on a CockerSpaniel object through a reference of type Friendly.
        fr.sayGoodbye();

        fr = new Cat();

        // Invoke sayGoodbye() on a Cat object through a reference of type Friendly.
        fr.sayGoodbye();
    }
}
```

对象在jvm中的结构：

| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/99a9499e-175c-4aa1-83fe-675c40bba193" alt="object layout" style="zoom:100%; float: left;" /> | 对象在堆上的结构(假定按定义顺序，且基类先于子类) |
| ------------------------------------------------------------ | ------------------------------------------------ |
| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/f1e1d435-a4c3-4fd9-b702-47e4ec62e57c" alt="dog method table" style="zoom:100%; float: left;" /> | Dog对象method table                              |
| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/d5a1ad53-6bd0-43a8-be9f-2f97aff7c77f" alt="CockerSpaniel layout" style="zoom:100%; float: left;" /> | **`CockerSpaniel`**对象method table              |
| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/7ca0353e-91a6-447b-8e91-159b03fbd239" alt="cat layout" style="zoom:100%; float: left;" /> | Cat对象method table                              |

* 具有类继承关系对象方法，其在method table内的索引保持不变
* 接口的方法实现，由于其不一定基于同一个基类，同一个方法索引可能不同，因此需要使用invokeinterface来区分

## 3.method table 存在于何处，长什么样子[^3]？

| <img src="https://github.com/aristotle0x01/aristotle0x01.github.io/assets/2216435/a09b2f3d-cea5-4a54-8a93-3f1d8580e7d0" alt="object layout" style="zoom:45%; float: left;" /> |
| ------------------------------------------------------------ |

上面**Data**作为一个java class，完成加载后，在jvm内部由一个instanceKlass对象表示，那么这个对象其在内存中的结构即如上图第二行所示。可以看到**VTable**紧接内存对象头之后，那么它到底是什么时候分配的，长度几何呢？我们来从 jdk8u源码一探究竟。

a. <u>hotspot/src/share/vm/oops/instanceKlass.hpp</u>

```c++
static InstanceKlass* allocate_instance_klass(
                                          ClassLoaderData* loader_data,
                                          int vtable_len,
                                          int itable_len,
                                          int static_field_size,
                                          int nonstatic_oop_map_size,
                                          ReferenceType rt,
                                          AccessFlags access_flags,
                                          Symbol* name,
                                          Klass* super_klass,
                                          bool is_anonymous,
                                          TRAPS);
											<- ClassFileParser::parseClassFile
```

该方法分配一个InstanceKlass实例，由**ClassFileParser::parseClassFile**在类加载中完成**.class**文件解析后调用，以生成java类定义在jvm中的数据结构表达。上面有个关键参数**vtable_len**，即为java类中需要多态调用关系方法(一般为public/protected 型实例方法)的数量(实际是存储方法指针数组长度)。同理**itable_len**表示接口方法数量，暂且不表。

b. **vtable_len**是怎么来的呢？

<u>hotspot/src/share/vm/oops/klassVtable.cpp</u>

由`klassVtable::compute_vtable_size_and_num_mirandas`在`parseClassFile`中完成，在分配`allocate_instance_klass`之前。内部逻辑不赘述，其长度为父类**vtable_len**和本类相关方法长度之和。

c. **allocate_instance_klass**实现

```c++
int size = InstanceKlass::size(vtable_len, itable_len, nonstatic_oop_map_size,
                                 access_flags.is_interface(), is_anonymous);
  // Allocation
	// normal class
      ik = new (loader_data, size, THREAD) InstanceKlass(
        vtable_len, itable_len, static_field_size, nonstatic_oop_map_size, rt,
        access_flags, is_anonymous);
  ......
  return ik;
}
```

首先可以看到**size**综合了各种长度，其次利用了c++中的**placement new**语义完成对象构造。这个**new**运算符重载在基类

<u>hotspot/src/share/vm/oops/klass.hpp</u>中。

d. **vtable**定义

```c++
// instanceKlass.hpp
static int vtable_start_offset()    { return header_size(); }
intptr_t* start_of_vtable() const        { return ((intptr_t*)this) + vtable_start_offset(); }
static int vtable_length_offset()   { return offset_of(InstanceKlass, _vtable_len) / HeapWordSize; }
```

从上面的定义可以明确，vtable启始于对象header后。至此，vtable定义完成了，预留空间包含了父类和当前类方法，与前面的数组图示相符。接下来看是如何指向方法并且实现多态调用的。

## 4.多态调用的实现

先倒序罗列**vtable**初始化整体过程，再对其中重点方法加以说明。

<u>hotspot/src/share/vm/oops/klassVtable.hpp</u>，说明klassVtable可以理解为一个helper，**vtable**实际仍然存在于**instanceKlass**实例中。

```c++
klassVtable::initialize_from_super(super)
klassVtable::update_inherited_vtable
	<- klassVtable::initialize_vtable
  	<- instanceKlass::link_class_impl
  		<- instanceKlass::eager_initialize_impl
  			<- instanceKlass::eager_initialize
  				<- SystemDictionary::parse_stream
  					or 
  				<- SystemDictionary::define_instance_class
```

**parse_stream**或者**define_instance_class**很显然是类加载过程的关键方法，上述调用路径并未穷尽所有，只是其中一种。

首先，**initialize_from_super**将父类**vtable**内容完整拷贝到当前类：

```c++
// Copy super class's vtable to the first part (prefix) of this class's vtable
int klassVtable::initialize_from_super(KlassHandle super) {
  // copy methods from superKlass
    assert(super->oop_is_instance(), "must be instance klass");
    InstanceKlass* sk = (InstanceKlass*)super();
    klassVtable* superVtable = sk->vtable();
    superVtable->copy_vtable_to(table());
    ......
    return superVtable->length();
}

void klassVtable::copy_vtable_to(vtableEntry* start) {
  Copy::disjoint_words((HeapWord*)table(), (HeapWord*)start, _length * vtableEntry::size());
}
```

其次，**update_inherited_vtable**根据方法名和签名决定是否覆盖父类方法，否则新增放入**vtable**：

```c++
// Update child's copy of super vtable for overrides
// OR return true if a new vtable entry is required.
// Only called for InstanceKlass's, i.e. not for arrays
// If that changed, could not use _klass as handle for klass
bool klassVtable::update_inherited_vtable(InstanceKlass* klass, methodHandle target_method,
                                          int super_vtable_len, int default_index,
                                          bool checkconstraints, TRAPS) {
  bool allocate_new = true;

  Symbol* name = target_method()->name();
  Symbol* signature = target_method()->signature();
  
  Symbol* target_classname = target_klass->name();
  for(int i = 0; i < super_vtable_len; i++) {
    Method* super_method;
    if (is_preinitialized_vtable()) {
      // If this is a shared class, the vtable is already in the final state (fully
      // initialized). Need to look at the super's vtable.
      klassVtable* superVtable = super->vtable();
      super_method = superVtable->method_at(i);
    } else {
      super_method = method_at(i);
    }
    // Check if method name matches
    if (super_method->name() == name && super_method->signature() == signature) {
       ......
       put_method_at(target_method(), i);
       if (!is_default) {
         target_method()->set_vtable_index(i);
       } 
        ......
      } 
    }
  }
  return allocate_new;
}
```

至此，也就完成了**vtable**的初始化。对象实例在调用某个方法时，根据头部类指针回溯到当前类**instanceKlass**，再从**vtable**索引找到方法实际指针，若为覆盖则调用基类方法，若已覆盖则调用当前类 方法 ，从而实现多态。

## 5.references

[^1]: [Inside the Java Virtual Machine: by Bill Venners](https://www.artima.com/insidejvm/ed2/index.html)
[^2]: [Chapter 8 of Inside the Java Virtual Machine The Linking Model](https://www.artima.com/insidejvm/ed2/linkmod12.html)

[^3]:  [Dissecting the Java Virtual Machine - Architecture - part 3](https://martin-toshev.com/index.php/software-engineering/architectures/80-dissecting-the-java-virtual-machine)


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



## 5.references

[^1]: [Inside the Java Virtual Machine: by Bill Venners](https://www.artima.com/insidejvm/ed2/index.html)
[^2]: [Chapter 8 of Inside the Java Virtual Machine The Linking Model](https://www.artima.com/insidejvm/ed2/linkmod12.html)


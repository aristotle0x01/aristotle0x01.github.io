---
layout: post
title:  "volatile-1-一道字节面试题的考验"
date:   2023-11-11
catalog: true
tags:
    - volatile
    - java
---



## 0.接受实践检验

实践是检验真理的唯一标准这句话在哲学上未必站的住脚，但经由实践或者实际问题检验自己对理论的理解，却是非常必要和重要的。近期看了不少JMM及volatile相关的东西，觉得大体明白了。无意间看到leetcode上网友发的一道字节面试题，方知远矣。

## 1.字节面试原题

为什么`int a = v.s` 会影响线程1的正常停止? 

```
public class Vol {
    boolean run = true;
    volatile int s = 1;
    
    public static void main(String[] args) throws InterruptedException {
        Vol v = new Vol();
        //thread 1
        new Thread(() ->{
            while (v.run) {
                int a = v.s; //如果注释这行，线程1无法中止
            }
        }).start();
        //thread 2
        new Thread(() ->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            v.run = false;
        }).start();
    }
}
```

原题链接[^1]

### 1.1 关于原问题链接下的答案

其中来自知乎的一个回答[^2]挺好，并且利用**jitwatch**从字节码和二进制码角度实证了注释前后jvm内部的不同实现。但我认为话术表达略有问题(作者肯定是明白的)，或者说逻辑略有问题。jvm编译结果也需要遵循一定的JMM原则，这个原则到底是什么，没有说清楚。经过一些搜索研究，我倾向于认为**piggyback**这个基于**volatile**的增强特性才是问题的根本。

JMM这个模型，还是有一定难度和复杂度。专家给出原则，但在具体的例子上并不都容易适用；给出例子，又无法穷尽所有情况。所以我暂时给出个人倾向的答案，以后有修订的可能。

## 2.piggyback

> **piggyback on**
>
> piggyback on somebody/something to use something that already exists as a support for your own work; to use a larger organization, etc. for your own advantage

大体就是利用别人或者别的事物来达成自己的目的，搭便车。

### 2.1 解释1[^3]

任何写`ready`变量之前的操作，对读取`ready`变量之后的操作都可见。因此，`number`变量利用了由`ready`变量强制执行的内存可见性。简单来说，尽管它不是一个`volatile`变量，但它表现出了`volatile`变量的行为

```
public class TaskRunner {
    private static int number; // not volatile
    private volatile static boolean ready;

    // same as before
}
```

### 2.2 解释2: Full volatile Visibility Guarantee[^4]

Java中的`volatile`关键字提供的内存可见性保证超出了`volatile`变量本身。其内存可见性保证如下：

* 如果线程A写入一个`volatile`变量，然后线程B随后读取同一个`volatile`变量，那么线程A在写入`volatile`变量之前可见的所有变量，在线程B读取完`volatile`变量后也将对线程B可见。
* 如果线程A读取一个`volatile`变量，那么线程A在读取`volatile`变量时可见的所有变量也将从主内存中重新读取

```
public class MyClass {
    private int years;
    private int months
    private volatile int days;

    public int totalDays() {
        int total = this.days;
        total += months * 30;
        total += years * 365;
        return total;
    }

    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
```

`totalDays()` 方法在读取 `days` 的值时，`months` 和 `years` 的值也会从主内存读取。因此通过上述读取顺序，可以确保看到 `days`、`months` 和 `years` 的最新值。

### 2.3 JSR 133 JMM模型的增强语义

上述增强理解可以参考[^5]**Brian Goetz**写的一篇解释文章。

## 3.volatile变量与while的相对位置

### 3.1 之前

将`int a = v.s`置与**while**之前

```
//thread 1
        new Thread(() ->{
        	  int a = v.s;
            while (v.run) {
            }
        }).start();
```

### 3.2 之后

将`int a = v.s`置与**while**之前

```
//thread 1
        new Thread(() ->{
            while (v.run) {
            }
            int a = v.s;
        }).start();
```

### 3.3 结果

thread1在上述两种情况都是不能正常退出的。这点跟while在jvm中如何编译有关，是作为一个代码块整体处理的(待以后深究)。

## 4.编译结果对比

默认环境**jdk1.8**，后续文章同。

### 4.1 volatile变量注释后结果

```
//thread 1
new Thread(() ->{
	while (v.run) {
		// int a = v.s; //注释后thread1无法正常退出
	}
}).start();
```

| 注释后结果                                                   |
| ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/282222695-77b083eb-75bd-4d53-aca6-b34f53b34a01.png" alt="comment out" style="zoom:50%; float: left;" /> |

### 4.2 未注释结果

```
//thread 1
new Thread(() ->{
	while (v.run) {
		int a = v.s; //注释后thread1无法正常退出
	}
}).start();
```

| 未注释结果                                                   |
| ------------------------------------------------------------ |
| <img src="https://user-images.githubusercontent.com/2216435/282225797-5c5a7d0c-5c28-4f79-8dd4-84a903ea4b80.png" alt="uncomment" style="zoom:60%; float: left;" /> |

## 5.jitwatch

**jitwatch**是一个图形化分析查看**jit**编译字节码和二进制代码的工具[^6]&[^7]，支持导入日志分析，或者sandbox直接编译java代码。

若要在业务代码输出日志，可添加如下jvm参数：

```
jdk8:
-XX:+UnlockDiagnosticVMOptions
-XX:+TraceClassLoading
-XX:+LogCompilation
-XX:+PrintAssembly
-XX:+DebugNonSafepoints
-XX:LogFile=jit.log

jdk9+
-XX:+UnlockDiagnosticVMOptions
-Xlog:class+load=info
-XX:+LogCompilation
-XX:+PrintAssembly
-XX:+DebugNonSafepoints
-XX:LogFile=mylogfile.log

-Xint // 解释执行
-Xcomp // 编译执行
```

下载代码后可执行`mvn clean compile exec:java`启动jitwatch。mac上可能需要执行：

```
cp .../jitwatch/hsdis-amd64.dylib .../jdk1.8.0_181.jdk/Contents/Home/jre/lib/server
cp .../jitwatch/hsdis-amd64.dylib .../jdk1.8.0_181.jdk/Contents/Home/jre/bin
cp .../jitwatch/hsdis-amd64.dylib .../jdk1.8.0_181.jdk/Contents/Home/bin
```

## 6.references

[^1]: [题目求助｜字节跳动一面问题｜读取volatile变量会影响其他no-volatile变量在工作内存的值吗？](https://leetcode.cn/circle/discuss/8X13Ub/?um_chnnl=huawei?um_from_appkey=5fcda41c42348b56d6f8e8d5)
[^2]: [为什么volatile注释变量，对其下面的变量也会有影响？](https://www.zhihu.com/question/348513270)
[^3]: [Guide to the Volatile Keyword in Java](https://www.baeldung.com/java-volatile)
[^4]: [Java Volatile Keyword ](https://jenkov.com/tutorials/java-concurrency/volatile.html#full-volatile-visibility-guarantee)
[^5]:[Java theory and practice: Fixing the Java Memory Model, Part 2](https://archive.ph/pHqcD#selection-333.0-340.0)
[^6]: [AdoptOpenJDK/jitwatch](https://github.com/AdoptOpenJDK/jitwatch/wiki/Instructions)
[^7]:[JITWatch很折腾？有这篇文章在可以放心](https://juejin.cn/post/7162926791928053796)


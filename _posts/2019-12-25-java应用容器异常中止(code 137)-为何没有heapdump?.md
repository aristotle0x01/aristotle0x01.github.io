---
layout: post
title:  "java应用容器异常中止(code 137)-为何没有heapdump?"
date:   2019-12-25 11:29:35 +0800
categories: java docker container
---
# java应用之容器内存溢出被杀死

![java-docker](https://user-images.githubusercontent.com/2216435/71412533-0c7b3e80-2689-11ea-8961-46f0dfa69c15.png)

所谓的容器内存溢出被杀死，即为oom(out of memory) killed exit code 137。

## 初次遭遇
相信玩java容器化部署的朋友肯定遇到过oom killed exit code 137，解决方案很多，糙快猛的就重启或者直接docker run --restart=always。去年第一次遇到的时候，哈哈，是秀一下自己还知道点jvm调优的时候了：

`docker run ... JAVA_OPTS="-server -Xmx1024m"`

顺道还长了点其它知识，

`docker run ... --memory 1300M JAVA_OPTS="-server -Xmx1024m"`

容器的内存要大于Xmx内存，此前有不知道的同学配置失误导致启动失败，宣导之。以为从此可以高枕无忧了。

## 服务又挂了
岂料，好景仅半年，近日频遭下游业务方投诉，你们的服务又又不能用了。去现场扫了一眼，oom killed exit code 137，太好了，又可以jvm调优了。于是再给配上heapdump


`docker run ... --memory 1300M JAVA_OPTS="-server -Xmx1024m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/"`


服务很配合的一两天就又挂了，再去现场，咦，我的heapdump呢？为什么/data/logs 下除了日志啥都没有呢？xxx.hprof呢？

此时，以为这是一个传统的jvm调优问题，通过heapdump揪出问题代码即可。好，开始解决为啥没有生成heapdump的问题，认真的google了半天，但是网上似乎没有朋友遇到类似的问题，配置大同小异，为啥人家能dump出来。算了试试百度，还是算了。

晚上没招了，决定换个思路，还是顺藤摸瓜，搜搜oom killed exit code 137试试吧，不想还真是搜出来点东西。

## 为什么会oom killed？
首先，交代点背景，线上容器的jdk版本是1.8.0_171。

原来，在jdk1.8的早期版本时候，docker还没出生呢，所以对于java应用来说，其实它并不知道自己是在物理机上还是在容器里面，身在福中不知福啊。这会导致一个问题，如果不加Xmx配置，那么jvm默认会最大获取物理机内存的四分之一；而如果配置了呢，一般因为java容器主要只跑单个java应用，那么容器的最大内存和xmx内存差距比较小，尽管前者较大，由于堆外内存/metaspace的存在，导致java应用仍然会在负载较大的时候超出容器内存，导致容器oom killed，但这个时候对于jvm的认知来说，仍然认为自己ok的，还能玩呢，自然不会heapdump，就这么安乐死了。不废话，怎么让jvm意识到自己在容器里呢？

解决方案：
>docker run ... --memory 1300M 
>
>JAVA_OPTS="-server -XX:+UnlockExperimentalVMOptions 

>-XX:+UseCGroupMemoryLimitForHeap  -XX:MaxRAMFraction=2 

>-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/"

可以看到，这次多了三个参数，少了Xmx(不能再用了，会覆盖该三个参数)
>-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap
>
>-XX:MaxRAMFraction=2

前两个参数告诉jvm，你身在何处。MaxRAMFraction控制最大堆内存占容器内存的比例，即容器内存/MaxRAMFraction，只能取整数。好在oracle把这些参数支持backport到jdk1.8_171上了，太好了，不用升级了。

不过，还不完美，如果MaxRAMFraction取1，jvm Xmx接近容器最大内存，很容易被oom killed。而如果取2或者更大，则xmx又太小，或者容器内存要给很大才能让xmx满足需求，但是这样会浪费物理内存，在鄙司要点物理资源可是很不容易的。好在jdk1.8_191以上版本加入了百分比参数，可以精确控制，太好了，还是得升级。

> docker run ... --memory 1300M 
> 
> JAVA_OPTS="-server -XX:+UnlockExperimentalVMOptions 
> 
> -XX:+UseCGroupMemoryLimitForHeap  -XX:MaxRAMPercentage=80.0 
> 
> -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/"

这次从MaxRAMFraction变成了MaxRAMPercentage，完美，不过记住必须用double型，用整型竟然会报错启动不了。
此时经测试，可以正常生成heapdump文件。

## jvm配置+Xmx为啥仍然会多用内存？
### 1.不同配置下jvm的消耗情况
1)容器内存100M,xmx不限(此时xmx最大会用到物理机1/4)：

`docker run -m 100MB -it --rm image:jdk1.8.0_171 java -XshowSettings:vm -version
`
![1577180033737](https://user-images.githubusercontent.com/2216435/71406614-b64fd080-2673-11ea-9737-77a4b4f5b434.jpg)

远远超过容器内存限制

2)容器内存100M, xmx=80M：

`docker run -m 100MB -it --rm image:jdk1.8.0_171 java -Xmx80m -XshowSettings:vm -version
`
![1577180604730](https://user-images.githubusercontent.com/2216435/71407183-59551a00-2675-11ea-913d-7f441dea1c3c.jpg)

3)容器内存100M, 增加容器感知：

>docker run -m 100MB -it --rm image:jdk1.8.0_171 java 
>
>-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap 
>
>-XshowSettings:vm -version

![1577180300638](https://user-images.githubusercontent.com/2216435/71407161-47737700-2675-11ea-9c24-472594fc5c51.jpg)

自行计算Xmx，且不会超过容器限制。

### 2.jvm的内存消耗分布
总内存 = Heap + Code Cache + Metaspace + Symbol tables +
               Other JVM structures + Thread stacks +
               Direct buffers + Mapped files +
               Native Libraries + Malloc overhead + ...
 
![1577187846650](https://user-images.githubusercontent.com/2216435/71411793-c83a6f00-2685-11ea-8c69-0ab5f6c52561.jpg)

可见，除了堆内存之外，还有其它堆外内存消耗。二者加总，可能超过容器限制。

### 3.另一种解决方案
`-XX:MaxRAM=1g -XX:MaxRAMFraction=2`

![1577188983966](https://user-images.githubusercontent.com/2216435/71412391-6e877400-2688-11ea-83ee-eacba3e255f6.jpg)

可以使用MaxRAM限制java应用整体内存消耗：堆内存+堆外，但是问题是MaxRAMFraction控制不够精确，还是会造成内存浪费，所以作为次优方案。

## 结语
综上，容器oom killed，跟一般传统的java.lang.OutOfMemoryError异常是两码事。
java.lang.OutOfMemoryError发生是因为堆内存不够，此时需要增加Xmx。而容器oom killed，是因为堆外内存+堆内存总体超出限制而导致，是容器行为，所以不会产生heapdump。

## 参考
[OOM Killer and Java applications in containers](https://medium.com/logistimo-engineering-blog/oom-killer-and-java-applications-c0dfd7f6b036)

[Analyzing java memory usage in a Docker container](http://trustmeiamadeveloper.com/2016/03/18/where-is-my-memory-java/)

[Docker and Java: Why My App Is OOMKilled](https://dzone.com/articles/why-my-java-application-is-oomkilled)

[Running a JVM in a Container Without Getting Killed](https://blog.csanchez.org/2017/05/31/running-a-jvm-in-a-container-without-getting-killed/)

[Docker support in Java 8 — finally!](https://blog.softwaremill.com/docker-support-in-new-java-8-finally-fd595df0ca54)

[Java using much more memory than heap size (or size correctly Docker memory limit)
](https://stackoverflow.com/questions/53451103/java-using-much-more-memory-than-heap-size-or-size-correctly-docker-memory-limi)

[OpenJDK and Containers
](https://developers.redhat.com/blog/2017/04/04/openjdk-and-containers/#more-433899)

[JVM Memory Settings in a Container Environment
](https://medium.com/adorsys/jvm-memory-settings-in-a-container-environment-64b0840e1d9e)





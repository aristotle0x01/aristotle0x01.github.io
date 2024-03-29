---
layout: post
title:  "zookeeper-1-基本概念"
date:   2021-05-12
categories: zookeeper
catalog: true
tags:
    - zookeeper
---
# zookeeper-1-基本概念

初次接触zookeeper大概是在2015年左右，一直都没有亲自把玩过。现如今大家都转向etcd等新鲜玩意了，才回过头来到故纸堆里挖掘挖掘，大概是因为自己喜欢历史



闲话少说，要了解zookeeper，首先要知道它提供哪些功能，用来干嘛的，然后再了解它的内部原理，是如何实现所提供的功能的



以下主要参考：

[Apache ZooKeeper Tutorial – ZooKeeper Guide for Beginners](https://data-flair.training/blogs/zookeeper-tutorial/)

## zookeeper架构

![arch](https://user-images.githubusercontent.com/2216435/117949201-73fc6180-b344-11eb-9df1-8aa250269148.png)

## 数据模型

非常类似文件系统层级结构

![model](https://user-images.githubusercontent.com/2216435/117949091-58915680-b344-11eb-8414-754481083d8a.png)

## 交互流程

客户端读

客户端写

zk 节点间交互同步等

![model](https://user-images.githubusercontent.com/2216435/117949945-3e0bad00-b345-11eb-9bbb-bc2ed73869d0.png)



## 重要概念

### 1. znode

- 一般节点
- 节点状态(stat)
- 临时节点(Ephemeral Nodes，客户端终止，则节点删除)
- 序列节点(Sequence Nodes)

理解节点的概念非常重要，是后续核心功能的基础

### 2. watch

watch是一次性的，后续要监听的话，需要反复注册

### 3. 选主(Leader Election)

选主流程

### 4. session

客户端会话管理

### 5. 其它

如lock, barrier, queue，还有其它概念可以选择了解

![model](https://user-images.githubusercontent.com/2216435/117951633-eec67c00-b346-11eb-9d61-7af568ef22f7.png)



经过上面这些基本阅读，就能知道zk是干嘛的，自身大概组成，下面就可以搭建集群自己去把玩一下了



## 参考

[ZooKeeper 3.7 Documentation](https://zookeeper.apache.org/doc/r3.7.0/index.html)

[ZooKeeper Recipes and Solutions](https://zookeeper.apache.org/doc/r3.7.0/recipes.html)


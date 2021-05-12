---
layout: post
title:  "zookeeper-2-本地建docker集群+把玩"
date:   2021-05-12
categories: zookeeper
---
# zookeeper-2-本地建docker集群+把玩

可以通过已有或者官方的dockerfile一键compose整个zk集群，方便把玩



## docker source

此处采用的是apache zookeeper官方dockerhub镜像，版本3.7 [Docker Official Images](https://docs.docker.com/docker-hub/official_repos/) 

**compose文件：stack.yml**

```
version: '3.1'

services:
  zoo1:
    image: zookeeper
    restart: always
    hostname: zoo1
    ports:
      - 2181:2181
      - 9081:8080
    volumes:
      - ./data/zoo1:/data
      - ./data/zoo1/log:/datalog
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo2:
    image: zookeeper
    restart: always
    hostname: zoo2
    ports:
      - 2182:2181
      - 9082:8080
    volumes:
      - ./data/zoo2:/data
      - ./data/zoo2/log:/datalog
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo3:
    image: zookeeper
    restart: always
    hostname: zoo3
    ports:
      - 2183:2181
      - 9083:8080
    volumes:
      - ./data/zoo3:/data
      - ./data/zoo3/log:/datalog
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
```

**2181～2183** 客户端连接端口

**2888** zk节点间同步端口

**3888** zk节点间leader election端口

**9081～9083** 各节点web页面访问端口

**volumes**挂载到本地是为了方便查看zk数据



## cluster control

**启动集群**

`docker-compose -f stack.yml up`

**停止集群**

`docker-compose -f stack.yml down`



## cli登录zk节点

有两种方式

#### **1. 命令方式**

本地命令，根据端口选择登录不同节点

`zkCli.sh -server 127.0.0.1:2182`

#### **2. docker方式**

1) 首先确认zk集群使用的network

`docker network ls`

2) 连接某台server

`docker run -it --network=bitnami-docker-zookeeper_default --rm --link bitnami-docker-zookeeper_zoo3_1:zookeeper zookeeper zkCli.sh -server bitnami-docker-zookeeper_zoo3_1
:2183`

**bitnami-docker-zookeeper_default**为network名称

**bitnami-docker-zookeeper_zoo3_1**为要登录的节点容器名称



## 玩一玩

#### 各节点admin

![admin](https://user-images.githubusercontent.com/2216435/117957480-b3c74700-b34c-11eb-9b30-9b14137d45fc.png)

可以访问各节点，查看相关信息

#### 操作

对节点的create,delete,stat,addWatch等进行测试，尤其是临时节点，序列节点等

![admin](https://user-images.githubusercontent.com/2216435/117958146-51227b00-b34d-11eb-934f-5d76c71ccb5b.png)

![admin](https://user-images.githubusercontent.com/2216435/117958287-73b49400-b34d-11eb-800f-ccbd1ef01e57.png)



## tcpdump数据分析

#### 创建tcpdump镜像

为什么不直接使用本地命令呢？是为了方便通过“--net=container:your id” attach到其它container上

**建立镜像**

```
docker build -t tcpdump - <<EOF 
FROM ubuntu 
RUN apt-get update && apt-get install -y tcpdump 
VOLUME ["/tcpdump"]
CMD tcpdump -i eth0 
EOF
```

#### 启动命令

```
docker run -v /your/local/folder:/tcpdump --tty --net=container:"your container id" tcpdump tcpdump -tttt -s0 -X -vv tcp port 2181 -w /tcpdump/captcha.cap

docker run --tty --net=container:6628d9a4e3e5 tcpdump

docker run --tty --net=container:6628d9a4e3e5 tcpdump tcpdump -N -A 'port 8080'
```

增加volume是为了将文件dump到本地便于wireshark分析



因为是tcp通信，所以具体数据还需要进一步开发脚本解析

#### ref

[How to TCPdump effectively in Docker](https://xxradar.medium.com/how-to-tcpdump-effectively-in-docker-2ed0a09b5406)

[Using tcpdump With Docker](https://rmoff.net/2019/11/29/using-tcpdump-with-docker/)
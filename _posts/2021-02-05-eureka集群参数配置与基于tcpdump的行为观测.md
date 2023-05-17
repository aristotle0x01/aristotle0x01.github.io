---
layout: post
title:  "eureka集群参数配置与基于tcpdump的行为观测"
date:   2021-02-05
categories: eureka 注册中心 spring-cloud tcpdump
---
# eureka集群参数配置与基于tcpdump的行为观测

## 1.背景

​        Eureka作为SpringCloud全家桶中的注册中心，一旦挂了，依赖它的微服务就无法找到彼此而不可用，可谓关键组件。因此有必要探究一下它的基本原理和配置，不至于救火的时候两眼一抹黑。

​        其大体作用和交互可参考：[eureka详解](https://www.sakuratears.top/blog/Eureka%E8%AF%A6%E8%A7%A3.html)，下图即引用自该🔗。

![server-client interaction](https://user-images.githubusercontent.com/2216435/106833632-d3f0c580-66ce-11eb-9de9-a312e1f1f273.png)

​        **本文主要关注:**

- 高可用集群配置
- 利用tcpdump进行行为分析   



​        spring cloud 里面的**eureka**这个词有点意思，eureka  ~= aha moment! 

![image](https://user-images.githubusercontent.com/2216435/106893799-07603e00-6729-11eb-896d-85af2359efa0.png)



## 2. 集群配置

### 2.1 Eureka Server集群启动

```
eureka:
  client:
    registerWithEureka: true
    fetchRegistry: true

docker run -d --name peer1 -e SPRING_OPTS="--eureka.environment=prod --eureka.instance.appname=server-eureka --spring.application.name=server-eureka --eureka.instance.hostname=10.185.55.81 --eureka.instance.preferIpAddress=true --eureka.instance.ip-address=10.185.55.81 --eureka.client.serviceUrl.defaultZone=http://10.185.55.82:8501/eureka/,http://10.185.55.83:8501/eureka/"

docker run -d --name peer2 -e SPRING_OPTS="--eureka.environment=prod --eureka.instance.appname=server-eureka --spring.application.name=server-eureka --eureka.instance.hostname=10.185.55.82 --eureka.instance.preferIpAddress=true --eureka.instance.ip-address=10.185.55.82 --eureka.client.serviceUrl.defaultZone=http://10.185.55.81:8501/eureka/,http://10.185.55.83:8501/eureka/"

docker run -d --name peer3 -e SPRING_OPTS="--eureka.environment=prod --eureka.instance.appname=server-eureka --spring.application.name=server-eureka --eureka.instance.hostname=10.185.55.83 --eureka.instance.preferIpAddress=true --eureka.instance.ip-address=10.185.55.83 --eureka.client.serviceUrl.defaultZone=http://10.185.55.81:8501/eureka/,http://10.185.55.82:8501/eureka/"
```

![image](https://user-images.githubusercontent.com/2216435/106833173-106ff180-66ce-11eb-8c0a-88afa1a8bcee.png)

[Spring Cloud: High Availability for Eureka]( https://medium.com/swlh/spring-cloud-high-availability-for-eureka-b5b7abcefb32)

[High Availability, Zones and Regions](https://cloud.spring.io/spring-cloud-netflix/reference/html/#spring-cloud-eureka-server)

### 2.2 Eureka client配置

```
eureka:
  instance:
    // 不同服务可以配置不同参数，查看meta data即可知晓
    lease-renewal-interval-in-seconds: 4
    lease-expiration-duration-in-seconds: 12
  client:
    fetch-registry: true
    registry-fetch-interval-seconds: 8
    serviceUrl:
      defaultZone: http://10.185.55.81:8501/eureka/,http://10.185.55.82:8501/eureka/,http://10.185.55.83:8501/eureka/
```

对于client端，<u>第一个注册中心服务挂了，会尝试第二个，依次类推</u>

[Eureka Server: How to achieve high availability](https://stackoverflow.com/questions/38549902/eureka-server-how-to-achieve-high-availability)

[Spring Cloud Netflix Eureka - The Hidden Manual](https://blog.asarkar.com/technical/netflix-eureka)

### 2.3 重要参数及其影响

#### 2.3.1 eureka.instance.appname

**eureka.instance.appname**必须一致，否则会出现下面的情况(*<u>2.3.4</u>*)。也就是说，页面上"Application"列使用该参数

![](https://user-images.githubusercontent.com/2216435/236419426-848ed167-4057-4d1d-8975-81160402e038.png)

#### 2.3.2 **spring.application.name**

**spring.application.name**可以不一致，但作为同一服务的不同实例，建议统一

![](https://user-images.githubusercontent.com/2216435/236419484-90acfda5-c556-45bd-9f2f-486ab0e3b825.png)

#### 2.3.3 **eureka.instance.preferIpAddress**

> 默认情况下，eureka client使用主机名(hostName)向注册中心注册。 
>
> 当prefer-ip-address: true时 ，client使用的是ip向服务中心注册 ，但是默认获取的ip是 127.0.0.1。默认情况下当获取的ip 和 hostName 不同时 ，则产生不可用分片提示信息(unavailable-replicas)，并且集群间的数据不会同步

prefer-ip-address必须为true时, 可做如下设置

```
SPRING_OPTS="
 --eureka.instance.preferIpAddress=true 
 --eureka.instance.hostname=ip1
 --eureka.instance.ip-address=ip1"
 
 SPRING_OPTS="
 --eureka.instance.preferIpAddress=true 
 --eureka.instance.hostname=ip2
 --eureka.instance.ip-address=ip2"
```

#### 2.3.4 unavailable-replicas现象

<img src="https://user-images.githubusercontent.com/2216435/148346887-95a81e04-ea09-4b93-b2fd-df8f3ab14b6e.png" style="zoom:50%;" />



## 3. Eureka集群间交互准则

### 3.1 集群间通信(peer to peer communication)

- 等同于eureka client -> eureka server 通信
- 增量更新 delta update
- timestamp 冲突化解
- 容忍不同server间数据不一致
- self-preservation during network outages

[Understanding Eureka Peer to Peer Communication](https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication)

[Understanding eureka client server communication](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)

[The Mystery of Eureka Self-Preservation](https://dzone.com/articles/the-mystery-of-eurekas-self-preservation)



### 3.2 peer to peer  VS  master/slave

集群节点之间不分主从，任何节点都可以接受写数据，节点之间进行数据同步。



### 3.3 持久化与客户端使用

Eureka的注册信息都在内存，未持久化，比较适合中小规模应用。客户端支持**<u>负载均衡算法及失败重试</u>**

![round robin](https://user-images.githubusercontent.com/2216435/106844334-394eb180-66e3-11eb-933c-33be3aa2c912.png)

[服务发现与注册 Eureka 设计理念](https://www.linuxprobe.com/service-eureka.html)



## 4. 行为分析

### 4.0 手段

#### 4.0.1 tcpdump

在10.x.x.21，即eureka所在机器

```
tcpdump -tttt -s0 -X -vv tcp port 8501 -w captcha.cap
```

#### 4.0.2 wireshark分析

![image](https://user-images.githubusercontent.com/2216435/115950997-01e8e780-a511-11eb-97a0-508c50be3be3.png)

#### 4.0.3 SimpleHTTPServer文件访问

```
// 文件所在端执行
python -m SimpleHTTPServer 12345

// 下载端浏览器访问即可
ip:12345

// wget
wget ip:12345/file_to_download
```

#### 4.1 特定实例信息(meta data)

```
curl http://10.182.50.79:8501/eureka/apps/BOP-ZUUL-GATEWAY

// 三个注册实例的详情
<application>
  <name>BOP-ZUUL-GATEWAY</name>
  <instance>
    <instanceId>cat.bop.weibo.com:bop-zuul-gateway:8800</instanceId>
    <hostName>cat.bop.weibo.com</hostName>
    <app>BOP-ZUUL-GATEWAY</app>
    <ipAddr>10.182.50.79</ipAddr>
    <status>UP</status>
    <port enabled="true">8800</port>
    <leaseInfo>
      <renewalIntervalInSecs>4</renewalIntervalInSecs>
      <durationInSecs>12</durationInSecs>
      <registrationTimestamp>1683790603401</registrationTimestamp>
      <lastRenewalTimestamp>1684229441428</lastRenewalTimestamp>
      <evictionTimestamp>0</evictionTimestamp>
      <serviceUpTimestamp>1683790603401</serviceUpTimestamp>
    </leaseInfo>
    ...
  </instance>
  <instance>
    <instanceId>tiger.bop.weibo.com:bop-zuul-gateway:8800</instanceId>
    <hostName>tiger.bop.weibo.com</hostName>
    <app>BOP-ZUUL-GATEWAY</app>
    <ipAddr>10.185.55.87</ipAddr>
    <status>UP</status>
    <port enabled="true">8800</port>
    ...
  </instance>
  <instance>
    <instanceId>lion.bop.weibo.com:bop-zuul-gateway:8800</instanceId>
    <hostName>lion.bop.weibo.com</hostName>
    <app>BOP-ZUUL-GATEWAY</app>
    <ipAddr>10.182.50.80</ipAddr>
    <status>UP</status>
    <port enabled="true">8800</port>
    ...
  </instance>
</application>
```

### 4.2 实例注册行为

post /eureka/apps/BOP-FMS-QUERY-INFO

![](https://user-images.githubusercontent.com/2216435/238827185-0cc6180f-f27d-4ab7-bea5-5a937b40e2a2.png)

![](https://user-images.githubusercontent.com/2216435/238827383-804acbcd-9b41-4f86-879e-132ab56fcc93.png)

### 4.3 实例心跳

PUT /eureka/apps/BOP-FMS-QUERY-INFO/donkey.bop.weibo.com:bop-fms-query-info:8857

![](https://user-images.githubusercontent.com/2216435/238827822-3fb02702-eb15-4b06-a468-f43ce2279a2f.png)

![](https://user-images.githubusercontent.com/2216435/238828100-8b372ac8-cc4c-4a58-b709-ece1801fd502.png)

### 4.4 实例下线行为

将BOP-FMS-QUERY-INFO实例停止：

![](https://user-images.githubusercontent.com/2216435/238821398-5c806726-a1ea-4cfd-8d87-44ebd7d0cf2b.png)

![](https://user-images.githubusercontent.com/2216435/238821679-9da27065-e1d1-40b8-a37f-40d234bd4cb5.png)

### 4.5 注册中心节点间更新

10.185.55.87 => 10.182.50.80，可见节点间同步数据采用的是主动**push**行为

![](https://user-images.githubusercontent.com/2216435/238814015-33d4488c-b666-4097-86fb-dcc549f517ee.png)

<img src="https://user-images.githubusercontent.com/2216435/238814624-3d303c82-2455-4c9b-b690-785e18169f98.png" style="float: left;zoom:60%;" />

### 4.6 重启特定注册中心节点

重启**10.182.50.79**所在eureka节点，第79帧即位其向另一个注册中心节点注册。此后，第86帧为另一节点向重启节点增量同步注册信息。

![](https://user-images.githubusercontent.com/2216435/238640551-39ca8341-efb8-4440-a0ca-51b817677633.png)



## 5 实践遇到的一些问题
### 5.1 注册中心不更新服务状态
**现象**：关停服务，注册中心不再将该服务状态置为"down"，也不删除该实例

**分析**：

1）测试关停不同服务（spring cloud版本不同，2.1.2及以下），tcpdump抓包，发现低版本在关闭的时候会主动向注册中心发送状态更新数据，而高版本不会

![image](https://user-images.githubusercontent.com/2216435/115952139-fe585f00-a516-11eb-9a7a-b76973a0c991.png)

2）eureka注册中心进入self-preservation状态

此时，lease expiration enabled 为false

![image](https://user-images.githubusercontent.com/2216435/115952082-bc2f1d80-a516-11eb-8ebd-2b9fb6d9d98e.png)

**解决及结论**

最终重启了eureka注册中心，让其走出self-preservation模式，主动靠心跳检测下线已关停服务。两个结论:

a. 重启注册中心会担心注册信息丢失和恢复速度，看到有文章说服务只是在启动的时候才会注册，那就需要重启所有服务造成较大影响，其实不然，每次心跳即可作为注册信息，此次得到验证

b. 服务关停时向注册中心主动发送状态更新消息是一种方式(受spring cloud版本影响)，靠注册中心的心跳探活机制是另外一种机制


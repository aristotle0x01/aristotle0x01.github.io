---
layout: post
title:  "使用tcpdump剖析spring eureka功能"
date:   2021-02-05
categories: eureka 注册中心 spring-cloud tcpdump
---
# spring eureka 高可用探究

为啥要研究这玩意？

需要研究基础设施和组件，才能在救火的时候，出现问题的时候不至于两眼一抹黑



![server-client interaction](https://user-images.githubusercontent.com/2216435/106833632-d3f0c580-66ce-11eb-9de9-a312e1f1f273.png)

[eureka详解](https://www.sakuratears.top/blog/Eureka%E8%AF%A6%E8%A7%A3.html)

 spring cloud 里面的eureka这个词有点意思，eureka moment ~= aha moment! 

![image](https://user-images.githubusercontent.com/2216435/106893799-07603e00-6729-11eb-896d-85af2359efa0.png)



## 1. 鄙司玩法

### 1.1 client端

![image](https://user-images.githubusercontent.com/2216435/115950600-cc42ff00-a50e-11eb-9568-3f182df42bec.png)

### 1.2 eureka 服务端

**注册中心显示：**

![image](https://user-images.githubusercontent.com/2216435/115950703-6f941400-a50f-11eb-9c8d-54e94c7fe49d.png)



**启动命令：**

![image](https://user-images.githubusercontent.com/2216435/115950748-aff39200-a50f-11eb-8685-8c4d80319e4c.png)

### 1.3 等效形式

![spring eureka](https://user-images.githubusercontent.com/2216435/106752387-5cd31700-6665-11eb-93bf-0dd5831ba395.png)

### 1.4 问题

以当前的配置，其实是起不到高可用的作用的。为什么，因为client端只配置了一个eureka服务，该服务挂了，则无可用注册中心



## 2. 单机玩法

**Eureka Server**如何启动

```html
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```



## 3. 集群模式(高可用)

### 3.1 Eureka Server集群启动

```
eureka:
  client:
    serviceUrl:
      defaultZone: https://peer1/eureka/,http://peer2/eureka/,http://peer3/eureka/

---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2

---
spring:
  profiles: peer3
eureka:
  instance:
    hostname: peer3
```

[High Availability, Zones and Regions](https://cloud.spring.io/spring-cloud-netflix/reference/html/#spring-cloud-eureka-server)



### 3.2 Eureka client配置

```
eureka:
  client:   
    serviceUrl:
      defaultZone: http://peer1/eureka, http://peer2/eureka, http://peer3/eureka
```

对于client端，<u>第一个注册中心服务挂了，会尝试第二个，依次类推</u>

[Eureka Server: How to achieve high availability](https://stackoverflow.com/questions/38549902/eureka-server-how-to-achieve-high-availability)



### 3.3 等效形式

![image](https://user-images.githubusercontent.com/2216435/106833173-106ff180-66ce-11eb-8c0a-88afa1a8bcee.png)

[Spring Cloud: High Availability for Eureka]( https://medium.com/swlh/spring-cloud-high-availability-for-eureka-b5b7abcefb32)



### 3.4 集群之间通信(peer to peer communication)

- 等同于eureka client -> eureka server 通信
- 增量更新 delta update
- timestamp 冲突化解
- 容忍不同server间数据不一致
- self-preservation during network outages

[Understanding Eureka Peer to Peer Communication](https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication)

[Understanding eureka client server communication](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication)

[The Mystery of Eureka Self-Preservation](https://dzone.com/articles/the-mystery-of-eurekas-self-preservation)



## 4. 与zookeeper对比

### 4.1 AP优先于CP

![image](https://user-images.githubusercontent.com/2216435/106838757-d86dac00-66d7-11eb-9f24-c58b73e40f9b.png)

eureka是在部署AWS的背景下面设计的，其设计认为，在云端，特别是大规模部署情况下面，失败是不可以避免的，可能是因为eureka自身部署失败或者网络分区等情况导致服务不可用，这些问题是不可以避免的，要解决这个问题就需要eureka在网络分区的时候，还能够正常提供服务，因此eureka选择满足availability这个特性。

eureka选择了A也就必须放弃C，也就是说在eureka中采用最终一致性的方式来保证数据的一致性问题，因此实例的注册信息在集群的所有节点之间的数据都不是强一性的，需要客户端能支持**<u>负载均衡算法及失败重试</u>**等机制



### 4.2 peer to peer  VS  master/slave

副本之间不分主从，任何的副本都可以接受写数据，然后副本之间进行数据更新。在对等复制中，由于每一个副本都可以进行写操作，各个副本之间的数据同步及冲突处理是一个比较难解决的问题



### 4.3 持久化问题

Eureka 对于注册信息都在内存，没有持久化，所以比较适合中小规模应用



### 4.4 客户端支持

客户端支持**<u>负载均衡算法及失败重试</u>**

![round robin](https://user-images.githubusercontent.com/2216435/106844334-394eb180-66e3-11eb-933c-33be3aa2c912.png)

[服务发现与注册 Eureka 设计理念](https://www.linuxprobe.com/service-eureka.html)



## 5. 关键参数

```
eureka:
  instance:
    // 不同服务可以配置不同参数，查看meta data即可知晓
    lease-renewal-interval-in-seconds: 4
    lease-expiration-duration-in-seconds: 12
  client:
    fetch-registry: true
    registry-fetch-interval-seconds: 8
```

[Spring Cloud Netflix Eureka - The Hidden Manual](https://blog.asarkar.com/technical/netflix-eureka)



## 6. meta data

```
curl http://10.xxx.xx.1:8501/eureka/apps/BOP-ZUUL-GATEWAY

<application>
  <name>BOP-ZUUL-GATEWAY</name>
  <instance>
    <instanceId>dante1.com:bop-zuul-gateway:8800</instanceId>
    <hostName>dante1.com</hostName>
    <app>BOP-ZUUL-GATEWAY</app>
    <ipAddr>172.16.181.126</ipAddr>
    <status>UP</status>
    <overriddenstatus>UNKNOWN</overriddenstatus>
    <port enabled="true">8800</port>
    <securePort enabled="false">443</securePort>
    <countryId>1</countryId>
    <dataCenterInfo class="com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo">
      <name>MyOwn</name>
    </dataCenterInfo>
    <leaseInfo>
      <renewalIntervalInSecs>4</renewalIntervalInSecs>
      <durationInSecs>12</durationInSecs>
      <registrationTimestamp>1604564897090</registrationTimestamp>
      <lastRenewalTimestamp>1612409743220</lastRenewalTimestamp>
      <evictionTimestamp>0</evictionTimestamp>
      <serviceUpTimestamp>1604564897090</serviceUpTimestamp>
    </leaseInfo>
    <metadata>
      <management.port>8800</management.port>
    </metadata>
    <homePageUrl>http://dante1.com:8800/</homePageUrl>
    <statusPageUrl>http://dante1.com:8800/actuator/info</statusPageUrl>
    <healthCheckUrl>http://dante1.com:8800/actuator/health</healthCheckUrl>
    <vipAddress>bop-zuul-gateway</vipAddress>
    <secureVipAddress>bop-zuul-gateway</secureVipAddress>
    <isCoordinatingDiscoveryServer>false</isCoordinatingDiscoveryServer>
    <lastUpdatedTimestamp>1604564897090</lastUpdatedTimestamp>
    <lastDirtyTimestamp>1604564897076</lastDirtyTimestamp>
    <actionType>ADDED</actionType>
  </instance>
  </application>
```



## 7. 数据分析

### 7.1 tcpdump拦截

在10.x.x.21，即eureka所在机器

```
tcpdump -tttt -s0 -X -vv tcp port 8501 -w captcha.cap
```

### 7.2 wireshark分析

![image](https://user-images.githubusercontent.com/2216435/115950997-01e8e780-a511-11eb-97a0-508c50be3be3.png)

### 7.3 更新注册信息

![image](https://user-images.githubusercontent.com/2216435/115951097-758af480-a511-11eb-8524-7a2e45d9dec9.png)

### 7.4 server端peer to peer更新

![image](https://user-images.githubusercontent.com/2216435/115951537-d74c5e00-a513-11eb-932e-93441f0038af.png)



## 8 实践遇到的一些问题
### 8.1 注册中心不更新服务状态
**现象**：关停服务，注册中心不再将该服务状态置为"down"，也不删除该实例

**分析**：

1）测试关停不同服务（spring cloud版本不同，2.1.2及以下），tcpdump抓包，发现低版本在关闭的时候会主动向注册中心发送状态更新数据，而高版本不会

![image](https://user-images.githubusercontent.com/2216435/115952139-fe585f00-a516-11eb-9a7a-b76973a0c991.png)

2）eureka注册中心进入self-preservation状态

此时，lease expiration enabled 为false

![image](https://user-images.githubusercontent.com/2216435/115952082-bc2f1d80-a516-11eb-8ebd-2b9fb6d9d98e.png)

**解决及结论**

最终重启了eureka注册中心，让其走出self-preservation模式，主动靠心跳检测下线已关停服务

得出了两个结论

1）重启注册中心会担心注册信息丢失和恢复速度，看到有文章说服务只是在启动的时候才会注册，那就需要重启所有服务造成较大影响，其实不然，每次心跳即可作为注册信息，此次得到验证

2）服务关停时向注册中心主动发送状态更新消息是一种方式(受spring cloud版本影响)，靠注册中心的心跳探活机制是另外一种机制
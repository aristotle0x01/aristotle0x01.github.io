---
layout: post
title:  "Outbox pattern-微服务中kafka消息可靠发送的一种实现"
date:   2020-02-19 11:29:35 +0800
catalog: true
tags:
    - 微服务
    - kafka
    - 消息
    - 可靠性
---
# 一种微服务消息可靠推送的方式
![hot pushup chick](https://user-images.githubusercontent.com/2216435/74813297-230fe000-5330-11ea-9849-82b0a44999b4.jpg)

微服务之间经常会有消息通知的需求，比如上游状态更新，下游需要相应操作。老的代码里面很多实现都是直接通过http接口实现，但是这种方式存在两个问题：

1. 耦合太多，有时候需要通知多个相关方，那就需要调用多次，要了解大量下游细节；
2. 不可靠，如果通知时候对方挂掉了，那怎么办？回滚or重试？业务代码里面需要大量容错逻辑

## 基于binlog的订阅模式
其实消息推送或者状态通知的需求，基于binlog+消息队列，业界早就已经有了canal+kafka connect等多种成熟可靠的玩法。优势非常明显：

1. 解偶，通知方只管完成本系统的流程。至于通知，则几乎与业务代码无关，完全隔离无感知；
2. 高度可靠，基于binlog和消息队列的方式几乎可以做到使命必达。一般公司对于mysql,kakfa集群的运维保障都非常高，这俩挂了，其它的几乎都没法玩了。

其大体原理如下图，不再赘述：

![mysql cdc with kafka connect](https://user-images.githubusercontent.com/2216435/74813946-2b1c4f80-5331-11ea-85ee-79cd3802f1e6.jpg)

## 然鹅
说到这里我要吐槽一下鄙司了，基础设施层面投入太少。有kafka集群，mysql集群和运维，但是却无法提供binlog订阅的功能，只能满足平常业务的需求。嗨，当然如果提供了，也就没有本文了。有点讽刺哈
～_～

## 怎么办？
在微服务越拆越细，需求越来越多的情况下，尤其对于涉及资金结算，财务通知这种情况，接口的模式太痛苦了，运维人力消耗巨大。这个时候，有聪明的研发童鞋就提出了自己的实现，我看着有些精巧的地方，遂不揣浅陋，分析分析。

当然，提下我们的技术栈背景，主体spring cloud全家桶，少量老的php(准备替了，管你是不是最好的语言)。消息队列kafka，数据库mysql，docker化部署，jdk8。

我先上核心代码吧，下一小节再分析其原理：

####0.kafka基本配置

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
       KafkaTemplate kafkaTemplate = new KafkaTemplate(producerFactory(), true);
       kafkaTemplate.setProducerListener(kafkaSendResultHandler);
       return kafkaTemplate;
       }
    
    // 对kafka消息发送状态进行响应，更新数据库消息发送状态
    public class KafkaSendResultHandler implements ProducerListener {

    @Autowired
    private MqMessageProducerService mqMessageProducerService;

    @Override
    public void onSuccess(String topic, Integer partition, Object key, Object value, RecordMetadata recordMetadata) {
        String messageId = key.toString();
        MqMessageProducer producer = new MqMessageProducer();
        producer.setMessageId(messageId);
        producer.setStatus(MessageStatus.SUCCESS.getCode());
        mqMessageProducerService.updateByMessageId(producer);
    }

    @Override
    public void onError(String topic, Integer partition, Object key, Object value, Exception exception) {
        String messageId = key.toString();
        MqMessageProducer producer = new MqMessageProducer();
        producer.setMessageId(messageId);
        if(producer.getTimes().intValue()<MAX_TIMES){
            producer.setStatus(MessageStatus.SENDING.getCode());
        }else {
            producer.setStatus(MessageStatus.FAIL.getCode());
        }
        producer.setTimes(producer.getTimes()+1);
        mqMessageProducerService.updateByMessageId(producer);
    }

    @Override
    public boolean isInterestedInSuccess() {
        return true;
    }
    }


####1.kafka模板

ReliableKafkaTemplate：取代org.springframework.kafka.core.KafkaTemplate

	 @Autowired
    private MqMessageProducerService mqMessageProducerService;

    public void send(String topic, Object data) {
        String key = UUID.randomUUID().toString();
        ProducerRecord producerRecord = new ProducerRecord(topic, key, data);
        MqMessageProducer message = new MqMessageProducer();
        message.setMessageId(producerRecord.key().toString());
        message.setMessage(producerRecord.value().toString());
        message.setType(producerRecord.topic());
        message.setStatus(MessageStatus.SEND.getCode());
        mqMessageProducerService.insert(message);
        
        MessageContext.set(producerRecord);
    }
 
####2.MessageContext
利用threadlocal暂存待发送消息

    private static ThreadLocal<List<ProducerRecord>> instance = new ThreadLocal<>();

    public static void set(ProducerRecord producerRecord){
        if(instance.get() == null || instance.get().isEmpty()){
            List<ProducerRecord> list = new ArrayList<>();
            list.add(producerRecord);
            instance.set(list);
        }else {
            List<ProducerRecord> list = instance.get();
            list.add(producerRecord);
        }
    }

    public static List<ProducerRecord> get(){
        return instance.get();
    }

    public static void remove(){
        instance.remove();
    }

####3.切面拦截


	 // 业务方法注解(需要发送消息者)
    @Target({ElementType.METHOD})
    @Retention(RetentionPolicy.RUNTIME)
    @Inherited
    @Documented
    public @interface KafkaProducer {}
    
    // 切面实现，真正进行kafka发送
    @Aspect
    @Order(value = Integer.MIN_VALUE)
    public class KafkaProducerAspect {
    private KafkaTemplate kafkaTemplate;

    private final ExecutorService executorService = Executors.newFixedThreadPool(10);

    public KafkaProducerAspect(KafkaTemplate kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @Pointcut("@annotation(KafkaProducer)")
    public void pointCuts() {
    }

    @Around(value = "pointCuts()")
    public Object doAround(ProceedingJoinPoint pjp) {
        Object obj = null;
        try {
            obj = pjp.proceed();
        } catch (Throwable throwable) {
            MessageContext.remove();
            throw (BusinessException) throwable;
        }
        List<ProducerRecord> list = MessageContext.get();
        if(list != null && !list.isEmpty()){
            for(ProducerRecord producerRecord : list){
                executorService.execute(()-> kafkaTemplate.send(producerRecord));
            }
        }
        MessageContext.remove();
        return obj;
    }}

####4.异步任务保障可靠发送

    // 重试发送异步任务，定期多次执行
    public class MessageRetryTask {
    @Autowired
    private MqMessageProducerService mqMessageProducerService;

    @Autowired
    private KafkaTemplate kafkaTemplate;

    private final ExecutorService executorService = new ThreadPoolExecutor(10, 10,0L,TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());

    @Scheduled(cron = "0 0 0/1 * * ?")
    @Lock(lockkey = "task.MessageRetryTask.retry", tryLock = true, leaseTime = 60L, waitTime = 0L)
    public void retry() {
        List<MqMessageProducer> list = mqMessageProducerService.getFailMessageList();
        if(list != null && !list.isEmpty()){
            for(MqMessageProducer record : list){
                String key = record.getMessageId();
                String topic = record.getType();
                String message = record.getMessage();
                executorService.execute(()-> kafkaTemplate.send(topic, key, message));
            }
        }
    }}

## 分析分析
别看上面一堆代码，其实就核心步骤用语言描述如下：

1. 初始化kafka模板，需要关心发送成功与否状态(即ProducerListener)
2. 业务方法上加KafkaProducer注解，调用ReliableKafkaTemplate发送消息
   实质是在数据表插入一条kafka消息记录
   同时保存消息到threadlocal变量
3. 切面内部保证事务或者所有业务流程无异常后进行真正消息发送。
   分几种情况：
      业务流程失败(异常)：则回滚，不会发送消息
      业务流程成功，发消息成功onSuccess：更新步骤2中的消息记录状态为成功
      业务流程成功, 发消息失败onError：更新步骤2中的消息记录状态为失败
4. 后台异步任务不停扫描，多次尝试发送状态为的失败消息
5. 多次失败，则邮件，人工处理

上述实现的精巧在于，利用模板伪发送，将消息存入线程变量以及数据库；之后在切面内部待所有业务流程正常完成后，才真正完成消息异步发送，此时不论消息是否发送成功，因为有了数据表消息记录+异步扫描任务，无论如何可以发送成功。而如果业务流程失败，那么因为数据库回滚，则消息记录被删除，不会再尝试。此外利用spring kafka原生模板的监听器，待kafka真正发送消息后实时更新数据库消息发送状态，避免了用户自己去更新的麻烦。

综上，可以达到消息可靠发送的目的。核心原理如下图：
![kafka发送](https://user-images.githubusercontent.com/2216435/74823560-da612280-5341-11ea-8cce-daefeccef0f7.jpg)

注意：

   @Order(value = Integer.MIN_VALUE)保证切面在最外层
   
   业务方法保证插入消息记录，以保证发送失败时通过异步任务重试
   
   需要在数据库建立消息存储表


## 结语
上述代码，核心在于一种机制实现，其实有个标准名称叫做outbox pattern(发件箱模式)。避免业务开发中，每遇到类似情况都要重复容错设计，提升研发效率。但相对于binlog的解耦方式，实属不得已而为之。相较而言，还是有不少缺点：

1. 还是会一定程度跟业务代码耦合
2. 消息可能会重复发送，此时需要消费方具备一次性消费能力





---
title: Spring 整合 Redis Stream 做消息队列
date: 2023-08-05 17:00:07
tags: 
- Spring
- Redis
---
参考 https://www.jianshu.com/p/b95a265f838a
为什么选redis stream
1 不希望引入其它中间件
2 业务规模目前不大 ，可以试水
3 redis stream 比kafka还快？

核心逻辑
1.自定义注解

```java
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    public @interface RedisStreamMqListen {
    String value();
    
        Class type();
    }
```




2.监听器注册和发送方法封装

```java
@Slf4j
public class RedisStreamMqServiceImpl implements RedisStreamMqService {
private final long dataCenterId = getDataCenterId();

private final RedisTemplate<String, String> redisTemplate;
/**
 * 最大长度
 */
long maxLen;
private String group;

public RedisStreamMqServiceImpl(RedisTemplate<String, String> redisTemplate, String group, Long maxLen) {
    this.redisTemplate = redisTemplate;
    this.group = group;
    this.maxLen = maxLen;
}

private static Long getDataCenterId() {
    try {
        String hostName = Inet4Address.getLocalHost().getHostName();
        int[] ints = StringUtils.toCodePoints(hostName);
        int sums = 0;
        for (int b : ints) {
            sums += b;
        }
        return (long) (sums % 32);
    } catch (UnknownHostException e) {
        // 如果获取失败，则使用随机数备用
        return RandomUtils.nextLong(0, 31);
    }
}

public void listener(String event, Class type, StreamListener streamListener) {
    createGroup(event);
    startSubscription(event, type, streamListener);
}

public <V> void coverSend(String event, V val) {
    ObjectRecord<String, V> record = StreamRecords.newRecord().ofObject(val).withId(RecordId.autoGenerate())
            .withStreamKey(event);
    redisTemplate.opsForStream().add(record);
    redisTemplate.opsForStream().trim(event, maxLen, true);
    log.info("event {} send content {}", event, StringUtils.abbreviate(val.toString().trim(), 100));
}

private void startSubscription(String event, Class type, StreamListener streamListener) {
    RedisConnectionFactory redisConnectionFactory = redisTemplate.getConnectionFactory();

    StreamMessageListenerContainer.StreamMessageListenerContainerOptions options = StreamMessageListenerContainer.StreamMessageListenerContainerOptions
            .builder().batchSize(5) // 设置批次大小为 5
            .pollTimeout(Duration.ofSeconds(1)).targetType(type).build();

    StreamMessageListenerContainer listenerContainer = StreamMessageListenerContainer.create(redisConnectionFactory,
            options);

    // redisTemplate 会自动加上前缀,所有在监听的时候也要加上
    event = "frl-stream:" + event;
    listenerContainer.receiveAutoAck(Consumer.from(group, group + dataCenterId),
            StreamOffset.create(event, ReadOffset.lastConsumed()), streamListener);

    listenerContainer.start();
}

private void createGroup(String event) {
    try {
        redisTemplate.opsForStream().createGroup(event, group);
    } catch (RedisSystemException e) {
        if (e.getRootCause() instanceof RedisBusyException) {
            log.info("STREAM - Redis group already exists, skipping Redis group creation: {}", group);
        } else if (e.getRootCause() instanceof RedisCommandExecutionException) {
            log.info("STREAM - Stream does not yet exist, creating empty stream: {}", event);
            boolean streamExists = redisTemplate.hasKey(event);
            if (!streamExists) {
                redisTemplate.opsForStream().add(event, Collections.singletonMap("", ""));
            }
            redisTemplate.opsForStream().createGroup(event, group);
        } else {
            throw e;
        }
    }
}

}
```

其中 核心逻辑为监听方法

```java
public void listener(String event, Class type, StreamListener streamListener) {
createGroup(event);
startSubscription(event, type, streamListener);
}

private void createGroup(String event) {
try {
redisTemplate.opsForStream().createGroup(event, group);
} catch (RedisSystemException e) {
if (e.getRootCause() instanceof RedisBusyException) {
log.info("STREAM - Redis group already exists, skipping Redis group creation: {}", group);
} else if (e.getRootCause() instanceof RedisCommandExecutionException) {
log.info("STREAM - Stream does not yet exist, creating empty stream: {}", event);
boolean streamExists = redisTemplate.hasKey(event);
if (!streamExists) {
redisTemplate.opsForStream().add(event, Collections.singletonMap("", ""));
}
redisTemplate.opsForStream().createGroup(event, group);
} else {
throw e;
}
}
}

private void startSubscription(String event, Class type, StreamListener streamListener) {
RedisConnectionFactory redisConnectionFactory = redisTemplate.getConnectionFactory();

    StreamMessageListenerContainer.StreamMessageListenerContainerOptions options = StreamMessageListenerContainer.StreamMessageListenerContainerOptions
            .builder().batchSize(5) // 设置批次大小为 5
            .pollTimeout(Duration.ofSeconds(1)).targetType(type).build();

    StreamMessageListenerContainer listenerContainer = StreamMessageListenerContainer.create(redisConnectionFactory,
            options);

    // redisTemplate 会自动加上前缀,所有在监听的时候也要加上
    event = "frl-stream:" + event;
    listenerContainer.receiveAutoAck(Consumer.from(group, group + dataCenterId),
            StreamOffset.create(event, ReadOffset.lastConsumed()), streamListener);

    listenerContainer.start();
}
```
核心逻辑
1. 通过 listenerContainer.receiveAutoAck(...) 注册了一个消息监听器，该监听器实现了 StreamListener 接口，定义了 onMessage 方法用于处理接收到的消息。
2. 调用 listenerContainer.start() 启动消息监听。
3. 当 Redis Stream 中有消息到达时，StreamMessageListenerContainer 会调用注册的 StreamListener 的 onMessage 方法，并将消息作为参数传递给这个方法。
4. 在 onMessage 方法中，编写逻辑来处理接收到的消息

这里 event 做了特殊操作，是因为在redisTemplate中设置了 key 的前缀，是用redisTemplate创建的队列，而这里是通过listenerContainer添加监听，所有为了匹配上 key 添加这个操作

项目启动后自动对注解添加监听操作

    public class ListenAnnotation implements ApplicationListener<ContextRefreshedEvent> {
    private static final Logger log = LoggerFactory.getLogger(ListenAnnotation.class);
    
    final RedisStreamMqService redisStreamMqService;
    
    public ListenAnnotation(RedisStreamMqService redisStreamMqService) {
        this.redisStreamMqService = redisStreamMqService;
    }
    
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() == null) {
            Map<String, Object> beans = event.getApplicationContext().getBeansWithAnnotation(RedisStreamMqListen.class);
            for (Object bean : beans.values()) {
                RedisStreamMqListen ca = bean.getClass().getAnnotation(RedisStreamMqListen.class);
                redisStreamMqService.listener(ca.value(), ca.type(), (StreamListener) bean);
                log.info("event {} start listen", ca.value());
            }
    
        }
    }
    
    }
在应用启动过程中，ListenAnnotation会监听 ContextRefreshedEvent 事件，然后扫描应用上下文中的所有 Bean，查找标记了 @RedisStreamMqListen 注解的 Bean，然后将它们注册到RedisStreamMqStartService 中进行消息监听。
event.getApplicationContext().getParent() == null
判断是否为根上下文，确保注册方法只执行一次
3.实现监听类,使用

    @Component
    @Slf4j
    @RedisStreamMqListen(value = "briefReportListener", type = AnalyzeToolDTO.class)
    public class BriefReportListener implements StreamListener<String, ObjectRecord<String, AnalyzeToolDTO>> {
    
    @Resource
    private BriefReportServiceImpl briefReportService;


    @SneakyThrows
    @Override
    public void onMessage(ObjectRecord<String, AnalyzeToolDTO> message) {
    	...
    }

}


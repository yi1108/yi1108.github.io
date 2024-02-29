---
title: redis学习小记
date: 2022-07-09 09:01:55
cover:
categories: 中间件
tags:
- 中间件
- redis
---

### 

# 基础

## 基本数据类型

1. String 字符串
2. Hash  哈希表
3. List 有序集合 可重复
4. Set  无序集合 不重复
5. SortedSet  可排序集合 不重复

### 通用命令

- keys  查询key  keys * 查询所有key  模糊查询占用资源 堵塞进程 生产不使用
- del  删除一个或多个key
- mest 批量添加k -v
- exists 判断一个key 是否存在
- expire 给一个key设置有效期 
- ttl 查询一个key的有效期

### String 类型

```
value 是字符串 根据字符串的格式不同 又可以分为3类：
String：普通字符串
int：整形 可以做自增自减操作
float：浮点 可以做自增自减操作
如果是数字类型 会转为二进制进行存储  节省空间
字符串只能转为字节码
最大512m
```

###  key的结构

```
可以通过多个单词形成层级结构,多个单词之间用：隔开 如
 项目：业务：用户
```

### Hash类型

```
hash类型也叫散列，value是一个无序字典，类似java hashMap
String结构是将对象序列化为json字符串存储，但需要对其某个字段修改时很不方便
Hash结构可以将对象中每个字段独立存储，可针对单个字段crud
```

### List类型

```
与jav中linkedList类似，可以看作一个双向链表 支持正反向检索
有序
元素可重复
插入删除快 查询速度一般
lpush key element 向左侧插入
lpop 从左侧取出
rpush key element 向右侧插入
rpop 
blpop key timeout 指定一个时间内阻塞从左边取出
```

### Set类型

```
与java中HashSet类似 可以看作一个value为null的HashMap
无序
不可重复
查找快
支持 交集 并集 差集 等功能
sadd key member... 向set中添加元素
srem key member... 移除set中元素
scard key 返回set中元素个数
sismember key member 判断set中是否存在一个元素

sinter key1 key2 ... 求交集
sdiff key1 key2... 求交集
sunion key1 key2 求并集
```

### SortedSet类型

```
是一个可排序的set集合  功能上和java中TreeSet类似，但是底层数据结构却差别很大。SortedSet中的每一个元素都带有一个score属性（分数），可以基于score属性对元素排序 底层是一个跳表（SkipList）加Hash表实现。
可排序
不可重复
查询速度快
一般用来实现排行榜功能

排名默认升序 命令后加REV 反转 降序
zscore key member 获取指定元素score值
zrank key member 获取指定元素排名
zrange key min max 获取指定名次内元素
```

## Java使用Redis

```
https://redis.io/clients
Jedis: Jedis实列线程不安全，多线程操作需要基于连接池使用
Lettuce： 基于Netty实现 支持同步,异步和响应式编程 线程安全 支持redis 哨兵模式，集群模式和管道模式
Redisson: 基于redis实现的分布式，在分布式环境下推荐使用

springData Redis 兼容了 Jedis和Lettuce
```

![image-20220910144320566](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/image-20220910144320566.png)



### redis 序列化

```java
afterPropertiesSet(){
	if (defaultSerializer == null) {
		defaultSerializer = new JdkSerializationRedisSerializer(
		classLoader != null ? classLoader : this.getClass().getClassLoader());
		}
}
如果没有指定序列化方式 默认使用jdk的二进制序列化 （可读性差）
```

#### 		redis序列化修改 

```java
 	@Bean
    public RedisTemplate<String,Object> redisTemplate(RedisConnectionFactory redisConnectionFactory){
        //1 创建对象
        RedisTemplate<String , Object> redisTemplate = new RedisTemplate<>();
        //2 设置链接工厂
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        //3 参加json序列化对象
        GenericJackson2JsonRedisSerializer jsonRedisSerializer = new GenericJackson2JsonRedisSerializer();
        //4 设置key序列化格式为字符串
        redisTemplate.setKeySerializer(RedisSerializer.string());
        redisTemplate.setHashKeySerializer(RedisSerializer.string());
        //5 设置value序列化方式为json
        redisTemplate.setValueSerializer(jsonRedisSerializer);
        redisTemplate.setHashValueSerializer(jsonRedisSerializer);

        return redisTemplate;

    }
```


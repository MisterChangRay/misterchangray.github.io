---
layout: post
title:  "springboot cahce使用"
date:   2024-06-28 10:29:20 +0800
categories:
      - 后台
      - springboot
tags:
      - springboot
      - cache
---

springboot 内置cache支持。 可以很方便的对数据进行缓存。 框架的逻辑很简单，在加入` @Cacheable` 注解的方法前，拦截执行参数，然后根据预设的条件进行缓存读取。

这里简单介绍 springboot + redis 对接口进行缓存.

1. 导入以下依赖：
```
	implementation 'org.springframework.boot:spring-boot-starter-data-redis'
	implementation 'org.springframework.boot:spring-boot-starter-cache'
```

2. 配置好redis连接信息：
   
```
# Redis数据库索引（默认为0）
spring.data.redis.database=0
# Redis服务器地址
spring.data.redis.host=127.0.0.1
# Redis服务器连接端口
spring.data.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.data.redis.password=

```

3. 在启动类中启用cache,也就是加上`@EnableCaching`注解


4. 配置redis序列化方式，以及配置cache地址：

```
/**
这里是配置redis序列化为json格式
*/
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new GenericJackson2JsonRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

/**
* 这里配置了两个cache, cache1和cache2. 两个缓存时间略有不同。
*/
    @Bean
    public RedisCacheManagerBuilderCustomizer myRedisCacheManagerBuilderCustomizer(RedisTemplate<Object, Object> template) {
        return (builder) -> builder
                .withCacheConfiguration("cache_10s", RedisCacheConfiguration
                        .defaultCacheConfig().entryTtl(Duration.ofSeconds(10))
                        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(template.getValueSerializer())))
                .withCacheConfiguration("cache_10m", RedisCacheConfiguration
                        .defaultCacheConfig().entryTtl(Duration.ofMinutes(1))
                        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(template.getValueSerializer())));

    }
```


最后只需要在你想要缓存的方法上加上注解` @Cacheable` 可以对方法进行缓存了
示例代码：
```java
package com.example.spiderserver.service;

import com.example.spiderserver.controller.TestReq;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class TestService {

    /**
     * 模仿get函数
     * 缓存结果
     * 这里使用了 tostring 作为缓存的 key, 这样当请求内容发生变更时，可以缓存多个条件
     * @param testReq
     * @return
     */
    @Cacheable(cacheNames = "cache_10m",key = "#p0.toString()")
    public TestReq get(TestReq testReq) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
        }

        return testReq;

    }

    /**
     * 每次都会执行函数。执行完成后缓存结果
     *
     * 也就是起到更新cache的作用
     * @param testReq
     * @return
     */
    @CachePut(cacheNames = "cache_10m" ,key = "#p0.toString()")
    public TestReq cacheput(TestReq testReq) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {

        }
        return testReq;

    }

    /**
     * 删除缓存
     * @param testReq
     * @return
     */
    @CacheEvict(cacheNames = "cache_10m", key = "#p0.toString()", allEntries = true)
    public TestReq delete(TestReq testReq) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {

        }
        return testReq;

    }

}



/**
*
* TestReq 是一个简单的对象
*
* 注意重写了 toString() 方法
*/

public class TestReq {
    private int userid;
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "TestReq{" +
                "userid=" + userid +
                '}';
    }

    public int getUserid() {
        return userid;
    }

    public void setUserid(int userid) {
        this.userid = userid;
    }
}


```

# 使用redis注解的话，返回类R要实现序列化接口
```java
package com.itmhw.common;

import lombok.Data;

import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

/**
 * 通用返回结果，服务端响应的数据最终都会封装成此对象
 * @param <T>
 */
@Data
public class R<T> implements Serializable {
    private Integer code; //编码：1成功，0和其它数字为失败
    private String msg; //错误信息
    private T data; //数据
    private Map map = new HashMap(); //动态数据

    public static <T> R<T> success(T object) {
        R<T> r = new R<T>();
        r.data = object;
        r.code = 1;
        return r;
    }
    public static <T> R<T> error(String msg) {
        R r = new R();
        r.msg = msg;
        r.code = 0;
        return r;
    }
    public R<T> add(String key, Object value) {
        this.map.put(key, value);
        return this;
    }
}
```
# 缓存中的书写也要实现序列化
    如果不实现，缓存中的书写格式不是正常语言
```java
package com.itmhw.config;

import com.alibaba.fastjson.support.spring.GenericFastJsonRedisSerializer;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * Redis配置类
 */

@Configuration
public class RedisConfig extends CachingConfigurerSupport {
/*    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {

        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setDefaultSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        //默认的Key序列化器为：JdkSerializationRedisSerializer
        //改造的是key的序列化方式
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        // 值不要随便字符串格式，否则存入的都是字符串格式的,除非非常确定是字符串
        //改造了序列化方式
        //改造的是value的序列化方式,value的值为string类型
        //redisTemplate.setValueSerializer(new StringRedisSerializer());
        //redisTemplate.setValueSerializer(RedisSerializer.json());
        //redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        //连接池
        redisTemplate.setConnectionFactory(connectionFactory);
        //System.out.println("RedisTemplate");
        return redisTemplate;
    }*/

    // 解决SpringCache(@Cacheable)把数据缓存到redis中的value是乱码问题
    @Bean
    public RedisCacheManager redisCacheManager(LettuceConnectionFactory lettuceConnectionFactory){
        RedisSerializationContext.SerializationPair<Object> objectSerializationPair = RedisSerializationContext.SerializationPair.fromSerializer(new GenericFastJsonRedisSerializer());
        return RedisCacheManager.builder(lettuceConnectionFactory)
                .cacheDefaults(
                        RedisCacheConfiguration.defaultCacheConfig(Thread.currentThread().getContextClassLoader())
                                .serializeValuesWith(objectSerializationPair)
                )
                .build();
    }
        @Bean
        public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
            RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
            //改造的是value的序列化方式
            redisTemplate.setDefaultSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
            // 序列化构造器
            //改造的是key的序列化方式
            redisTemplate.setKeySerializer(new StringRedisSerializer());
            //连接池
            redisTemplate.setConnectionFactory(connectionFactory);
            return redisTemplate;
        }
}

```
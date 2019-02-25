title: SpringBoot redis-cache
author: gslg
tags:
  - springboot
  - spring-cache
  - redis
  - java
categories:
  - SpringBoot
date: 2019-02-25 15:52:00
---
在实际项目中,使用缓存时一般要考虑统一管理key的前缀，超时时间等.在使用spring-boot时，我们可以如下配置redis：

```java
@Configuration
@EnableCaching
//不继承CachingConfigurerSupport则@CachePut会抛出类型转换异常
public class RedisCacheConfig extends CachingConfigurerSupport {

    @Value("${spring.profiles.active}")
    private String env;

    @Value("${spring.application.name}")
    private String app;

    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        //ObjectMapper 反序列化
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        redisTemplate.setKeySerializer(redisTemplate.getStringSerializer());
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);

//        redisTemplate.setValueSerializer(new FastJsonRedisSerializer<>(Object.class));
        redisTemplate.setConnectionFactory(factory);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    /**
     * 统一的前缀: app:env:cacheName
    */
    @Bean
    public RedisCachePrefix redisCachePrefix(){
       return new RedisCachePrefix() {
           private final RedisSerializer serializer = new StringRedisSerializer();
           private final String delimiter = ":";

            @Override
            public byte[] prefix(String cacheName) {
                String prefix = String.join(delimiter, app, env, cacheName,"");
                return serializer.serialize(prefix);
            }
        };
    }

    @Bean
    public CacheManager cacheManager(RedisTemplate redisTemplate) {
        RedisCacheManager redisCacheManager = new RedisCacheManager(redisTemplate);
        //设置缓存失效时长，秒级 30分钟
        redisCacheManager.setDefaultExpiration(60*30);
        redisCacheManager.setCachePrefix(redisCachePrefix());
        redisCacheManager.setUsePrefix(true);

        return redisCacheManager;
    }
}

```

比如我们的app名称为myapp,env是test,使用spring-cache作为缓存框架时:
```java
@Cacheable(cacheNames = "user",key = "'userId:'+#userId")
public User getUser(Long userId){
	//....get a user
}
```
上面的例子中，如果查询userId=1的用户,就会生成`myapp:test:user:userId:1`
如果@Cacheable中不传入key，那么生成`myapp:test:user:1`

此外，因为spring-cache是基于AOP实现的，因此在一个内中自调用另一个有缓存注解的方法是不会生效的。解决这个有2种方式:
- 可以在上一层分开调用这两个方法
- 在调用处通过spring context把该对象取出来，直接用对象方式来调用
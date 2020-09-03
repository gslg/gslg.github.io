---
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
#### redis统一配置
在实际项目中,使用缓存时一般要考虑统一管理key的前缀，超时时间等.在使用spring-boot时，我们可以如下配置redis：
<!--more-->
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

#### 注意事项:
1.因为spring-cache是基于AOP实现的，因此在一个内中自调用另一个有缓存注解的方法是不会生效的。解决这个有2种方式:
- 可以在上一层分开调用这两个方法
- 在调用处通过spring context把该对象取出来，直接用对象方式来调用  

2.同理，如果该方法被其它注解切入，当缓存命中的时候，不会进入该切面;当缓存没有命中的时候，可以进入

我们简单写个demo，首先定义一个启动类和一个rest接口，在方法上使用@Cacheable标识该方法的返回结果将被缓存起来.
```java
//com.example.RedisDemoApplication

@SpringBootApplication
@RestController
public class RedisDemoApplication {
    @GetMapping("/user/{userId}")
    @Cacheable(cacheNames = "user", key = "'userId:'+#userId")
    public User getUser(@PathVariable String userId) {
        //get a user
        System.out.println("get user form db.......");
        User user = new User();
        user.setUserId(userId);
        user.setUsername("test");
        return user;
    }

    public static void main(String[] args) {
        SpringApplication.run(RedisDemoApplication.class, args);
    }

}

```
定义一个切面,配置切入点为我们上面定义的`getUser`这个方法
```java
//com.example.aop.UserAspect

@Component
@Aspect
public class UserAspect {
    /**
     * 配置切入点
     */
    @Pointcut("execution(public * com.example.RedisDemoApplication.getUser(..))")
    public void aspect() {

    }

    @Before("aspect()")
    public void doBefore(JoinPoint jp) throws IOException {
        System.out.println("进入切面....");
    }
}
```

启动后，我们在浏览器测试：

- 首次次访问`http://localhost:8080/user/1`,观察控制台输出:  
>进入切面....  
>get user form db.......
- 再次访问`http://localhost:8080/user/1`，因为缓存命中了(前提是未过期),所以没有进入切面,观察控制台无输出: 
>&nbsp;

#### pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.17.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>redis-demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>redis-demo</name>
	<description>Demo project for Spring Boot</description>

	<properties>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-cache</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```
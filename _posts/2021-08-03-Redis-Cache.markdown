---
layout: post
title:  "Redis 캐시 및 장애대응"
description:  Reids 캐시 적용 및 장애대응 
date:   2021-08-03 00:00:00 +000
categories: Redis JAVA SpringBoot netflix-hystrix
---
## 개요
Java Spring Boot 환경에서 레디스 캐시 설정과 레디스 장애 발생시 대응의 위한 설정 


## Spring Cache 어노테이션 종류
|어노테이션|설명|
|------|---|
|@Cacheable| cache 등록 |
|@CachePut| cache 갱신 |
|@CacheEvict| cache 삭제 |

## dependencies 추가
org.springframework.boot 2.4.2 
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}
```

## properties 추가

```properties
# cache
spring.cache.type=redis
```

## CacheConfig.java - 1분단위 캐시 설정 
```java
@Configuration
public class CacheConfig extends CachingConfigurerSupport {

    @Autowired
    RedisConnectionFactory redisConnectionFactory;

    @Bean
    @Override
    public CacheManager cacheManager(){
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .entryTtl(Duration.ofMinutes(1L));
        return RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory).cacheDefaults(redisCacheConfiguration).build();
    }
}

```

## 사용예시
```java
    
    @Cacheable(value = "testCache",cacheManager="cacheManager")
    public Map<String,Object> testCache(){
            ...
    }

    // 저장된 캐시 정보 
    // testCache::SimpleKey []

    @Cacheable(value = "testCacheKey",key="#keyName",cacheManager="cacheManager")
    public Map<String,Object> testCacheKey(String keyName){
            ...
    }
    
    // 저장된 캐시 정보 
    // testCacheKey::keyName
```

## 문제점 
레디스 서버가 다운되는 경우 서비스 에러가 발생되어 원활한 서비스 지원이 불가능합니다. <br> 
여러 검색내용에서는 로컬 스토리지와 레디스 두개를 다 사용하는 것을 권장하는 부분도 있지만 테스트 진행을 위해 <br>
레디스 서버 장애 발생 시 오류가 발생하지 않고 바로 DB 조회를 할수 있도록 설정을 추가 하였습니다.

#장애대응 설정추가

## dependencies 추가
```gradl
ext {
    set('springCloudVersion', "2020.0.2")
}
dependencies {
    ...    
    implementation group: 'org.springframework.session', name: 'spring-session-data-redis', version: '2.5.0'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-hystrix:2.2.8.RELEASE'
}
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

## properties 추가
```properties
spring.redis.timeout=3000
spring.redis.connect-timeout=3000
spring.redis.database=0
```
## RedisConfig.java
```java
public class RedisConfig {
    @Bean
    public RedisSessionRepository sessionRepository(RedisOperations<String, Object> sessionRedisOperations) {
        RedisSessionRepository redisSessionRepository = new RedisSessionRepository(sessionRedisOperations);
        return redisSessionRepository;
    }

    @Bean
    public static ConfigureRedisAction configureRedisAction() {
        return ConfigureRedisAction.NO_OP;
    }
}
```


## CacheConfig.java
```java
@Configuration
@EnableCaching(proxyTargetClass = true)
public class CacheConfig extends CachingConfigurerSupport{
    @Autowired
    RedisConnectionFactory redisConnectionFactory;

    @Bean
    @Override
    public CacheManager cacheManager(){
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .entryTtl(Duration.ofMinutes(1));
        return new HystrixCacheManager(RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory).cacheDefaults(redisCacheConfiguration).build());
    }
}
```
### Netflix / Hystrix
- https://github.com/Netflix/Hystrix 
- 테스트 진행을 위해 아래의 내용만 우선 사용 

## HystrixCache.java
```java

public class HystrixCache implements Cache {

    private final Cache delegate;

    public HystrixCache(Cache delegate) {
        this.delegate = delegate;
    }

    @Override
    public String getName() {
        return null;
    }

    @Override
    public Object getNativeCache() {
        return null;
    }

    @Override
    public ValueWrapper get(Object key) {
        return new HystrixCacheGetCommand(delegate, key).execute();
    }

    @Override
    public <T> T get(Object key, Class<T> type) {
        return null;
    }

    @Override
    public <T> T get(Object key, Callable<T> valueLoader) {
        return null;
    }

    @Override
    public void put(Object key, Object value) {
        new HystrixCachePutCommand(delegate, key, value).execute();
    }

    @Override
    public void evict(Object key) {
        new HystrixCacheEvictCommand(delegate, key).execute();
    }

    @Override
    public void clear() {

    }
}
```
## HystrixCacheEvictCommand.java
```java
public class HystrixCacheEvictCommand extends HystrixCommand<Object> {

    private final Cache delegate;
    private final Object key;

    public HystrixCacheEvictCommand(Cache delegate, Object key) {

        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("testGroupKey"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("cache-evict"))
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.defaultSetter()
                                .withExecutionTimeoutInMilliseconds(500)
                                .withCircuitBreakerErrorThresholdPercentage(50)
                                .withCircuitBreakerRequestVolumeThreshold(5)
                                .withMetricsRollingStatisticalWindowInMilliseconds(20000)));
        this.delegate = delegate;
        this.key = key;
    }

    @Override
    protected Object run() {
        delegate.evict(key);
        return null;
    }

    @Override
    protected Object getFallback() {
        log.warn("evict fallback called, circuit is {}", super.circuitBreaker.isOpen() ? "opened" : "closed");
        return null;
    }
}
```
## HystrixCacheGetCommand.java
```java
public class HystrixCacheGetCommand extends HystrixCommand<Cache.ValueWrapper> {

    private final Cache delegate;
    private final Object key;

    public HystrixCacheGetCommand(Cache delegate, Object key) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("testGroupKey"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("cache-get"))
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.defaultSetter()
                                .withExecutionTimeoutInMilliseconds(500)
                                .withCircuitBreakerErrorThresholdPercentage(50)
                                .withCircuitBreakerRequestVolumeThreshold(5)
                                .withMetricsRollingStatisticalWindowInMilliseconds(20000)));
        this.delegate = delegate;
        this.key = key;
    }

    @Override
    protected Cache.ValueWrapper run() {
        return delegate.get(key);
    }

    @Override
    protected Cache.ValueWrapper getFallback() {
        log.warn("get fallback called, circuit is {}", super.circuitBreaker.isOpen() ? "opened" : "closed");
        return null;
    }
}
```
## HystrixCacheManager.java
```java
public class HystrixCacheManager implements CacheManager {

    private final CacheManager delegate;
    private final Map<String, Cache> cacheMap = new ConcurrentHashMap<>();

    public HystrixCacheManager(CacheManager delegate) {
        this.delegate = delegate;
    }

    @Override
    public Cache getCache(String name) {
        return cacheMap.computeIfAbsent(name,key->new HystrixCache(delegate.getCache(key)));
    }

    @Override
    public Collection<String> getCacheNames() {
        return delegate.getCacheNames();
    }
}
```
## HystrixCachePutCommand.java
```java
public class HystrixCachePutCommand extends HystrixCommand<Object> {

    private final Cache delegate;
    private final Object key;
    private final Object value;

    public HystrixCachePutCommand(Cache delegate, Object key, Object value) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("testGroupKey"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("cache-put"))
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.defaultSetter()
                                .withExecutionTimeoutInMilliseconds(500)
                                .withCircuitBreakerErrorThresholdPercentage(50)
                                .withCircuitBreakerRequestVolumeThreshold(5)
                                .withMetricsRollingStatisticalWindowInMilliseconds(20000)));
        this.delegate = delegate;
        this.key = key;
        this.value = value;
    }

    @Override
    protected Object run() {
        delegate.put(key, value);
        return null;
    }

    @Override
    protected Object getFallback() {
        log.warn("put fallback called, circuit is {}", super.circuitBreaker.isOpen() ? "opened" : "closed");
        return null;
    }
}
```

### 테스트 결과 
Redis에 캐시 데이터 생성 확인 후 레디스 서버를 다운시킨 결과 DB조회로 대체하여 정상적으로 서비스 유지 확인 완료 


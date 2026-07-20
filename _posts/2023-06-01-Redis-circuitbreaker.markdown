---
layout: post
title:  "Redis 서킷브레이크 적용 - Spring Boot(kotlin)"
description:  Spring Boot(kotlin) 환경에서 DB master / slave 처리 
date:   2023-06-01 00:03:00 +000
categories: Spring boot Redis resilience4j
---

## 개요

Redis 캐시 사용 시 Redis 접속에 문제가 생길 경우 DB 조회를 하기 위해 준비해 봅니다.

기존에는 `netflix/hystrix`를 사용했으나 더 이상 업데이트하지 않는다는 내용이 있고, 2019년이 마지막 버전이어서 이참에 바꿔보았습니다.

## 환경

### build.gradle.kts

```gradle.kts
dependencies {
    ...
    implementation("io.github.resilience4j:resilience4j-spring-boot2:2.0.2")
    ...
}
```

### application.properties

```properties
// 세부 옵션은 구글링으로 찾으면 설명이 잘되어 있다
resilience4j.circuitbreaker.instances.default.failure-rate-threshold=10
resilience4j.circuitbreaker.instances.default.wait-duration-in-open-state=100ms
resilience4j.circuitbreaker.instances.default.sliding-window-size=1
```

### 사용 Service에서 호출 예시

```kotlin
// 캐시설정은 Controller 에 한적도 있고 Service 에 한적도 있는데 개인적으로 Service 에서 관리하는게 난 좀더 편했다 ..
@Service
class CacheService(
    private val testRepository:TestRepository
){
    @Cacheable(cacheNames = ["TestCache"],key="#key",cacheManager="cacheManager")
    @CircuitBreaker(name="TestCache", fallbackMethod = "getTestCall")
    fun testCall(key:Long): Test {
        return this.getTestCall()
    }

    @Transactional(readOnly = true)
    protected fun getTestCall(t:Throwable?=null): Test {
        return testRepository.findById(key)
    }
}
```

## 결과

설정 없이 호출할 경우 Redis 연결이 안 되면 에러가 발생합니다.

하지만 위와 같이 간단하게 설정하고 사용하면 Redis 연결이 실패해도 DB 조회를 하기 때문에 에러가 발생하지 않습니다.

다만 다음 문제가 발생했습니다.

1분 동안 지연이 발생됩니다. 왜지?

Redis 연결 타임아웃도 설정하고, 서킷브레이크도 10%로 설정하고, retry도 1회로 하여 테스트했습니다. 하지만 동일하게 1분 지연이 발생했습니다.

지연이 발생하는 유형은 다음과 같습니다.

- API 기동 시 Redis가 꺼져 있는 경우: 지연 없이 DB 즉시 조회 처리. 이때는 문제가 없습니다.
- API 기동 시 Redis가 연결되어 있다가 서비스 도중 Redis 연결이 끊어진 경우: 1분 딜레이가 걸립니다. 이게 문제입니다.

해당 문제는 이미 이슈로 등록된 내용이 있었습니다. 하지만 그 후 개선된 내용은 확인하지 못했습니다.

그래서 다음과 같이 설정을 일부 더 추가했습니다.

### redisconfig

```kotlin
...
기존 내용
@Bean
fun redisConnectionFactory(): RedisConnectionFactory {
    ....
    return LettuceConnectionFactory(redisStandaloneConfiguration)
}

변경 내용
@Bean
fun redisConnectionFactory(): RedisConnectionFactory {
    ....
    var lettuceClientConfiguration: LettuceClientConfiguration = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofMillis(1000)) // 연결 타임아웃 설정
            .build()
    return LettuceConnectionFactory(redisStandaloneConfiguration,lettuceClientConfiguration)
}
...
```

### application.properties

```properties
pspring.redis.lettuce.pool.enabled=false  // 추가
```

되기는 하는 것 같습니다. 하지만 좀 더 찾아보고 사용할 수 있을지 봐야겠습니다.

## 아쉬운 점

`netflix/hystrix`가 더 잘되던 것 같습니다.

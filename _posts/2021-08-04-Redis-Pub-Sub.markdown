---
layout: post
title:  "Redis Pub/Sub - Spring Boot(Java)"
description:  Redis를 활용한 Pub/Sub 적용 
date:   2021-08-04 00:00:00 +000
categories: Redis JAVA SpringBoot 
---

## 개요

Java Spring Boot 환경에서 Redis를 이용한 Pub/Sub 설정과 사용 방법을 정리합니다.

## Redis 설정 추가

```bash
config set notify-keyspace-events KEA
```

참조: <https://redis.io/topics/notifications>

## Dependencies 추가

Spring Boot 버전은 `2.4.2`를 기준으로 작성했습니다.

## RedisConfig.java

```java
public class RedisConfig {

    ...

    @Bean
    public RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory, RedisKeyExpirationListener expirationListener){
        RedisMessageListenerContainer listenerContainer = new RedisMessageListenerContainer();
        listenerContainer.setConnectionFactory(connectionFactory);
        listenerContainer.addMessageListener(expirationListener, new PatternTopic("__keyevent@*__:expired"));
        listenerContainer.setErrorHandler(e -> log.error("There was an error in redis key expiration listener container", e));
        return listenerContainer;
    }
}
```

## RedisKeyExpirationListener.java

```java
@Component
public class RedisKeyExpirationListener implements MessageListener {

    @Override
    public void onMessage(Message message,byte[] pattern){
        String expiredKey = message.toString();
        System.out.println(expiredKey); // 만료되는 키정보
    }
}
```

## 결과

키 만료 후 서버로 전송 로그가 찍히는 것을 확인했습니다.

## 아쉬운 점

키만 전달되기 때문에 `onMessage` 처리 시 데이터를 사용할 방법이 없습니다.

그래서 임시 활용으로, 만료 시 활용해야 하는 데이터는 다음과 같이 이중으로 처리했습니다.

1. 첫 번째 키를 5초 만료로 등록
2. 두 번째 키를 `5+@`초 만료로 등록

예를 들어 key명이 `abcd`라면 다음처럼 생성합니다.

- 1번: `abcd_expiry`
- 2번: `abcd`

1번이 만료될 경우 `_expiry`를 제외한 `abcd`를 추출해 데이터를 확인합니다.

## 문제점

Redis 의존성을 높이면서 Redis가 죽으면 서버에 reconnect 로그가 남고, 서비스가 정상 작동되지 않는 현상이 발생했습니다.

DB만큼 Redis도 중요해진 상황이 되어 불편했습니다. 결국 장애 빈도를 줄이기 위해 직접 설치가 아닌 AWS ElastiCache를 사용했습니다.

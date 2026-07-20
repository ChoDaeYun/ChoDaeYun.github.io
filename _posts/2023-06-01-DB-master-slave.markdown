---
layout: post
title:  "mysql master/slave - Spring Boot(kotlin)"
description:  Spring Boot(kotlin) 환경에서 DB master / slave 처리 
date:   2023-06-01 00:02:00 +000
categories: Spring boot JPA 
---

## 개요

Spring Boot(Kotlin) 환경에서 MySQL master/slave를 적용한 예시입니다.

## 환경

### build.gradle.kts

```gradle.kts
plugins {
    id("org.springframework.boot") version "2.7.2"
    ...
    kotlin("plugin.spring") version "1.6.21"
    kotlin("plugin.jpa") version "1.6.21"
    kotlin("kapt") version "1.6.21"
}

var querydslVersion = "5.0.0"

dependencies {
    ...
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    ## 테스트 mysql 버전에 맞게 설정 mysql 8 에서 사용중
    implementation("mysql:mysql-connector-java:8.0.31")
    runtimeOnly("mysql:mysql-connector-java")

    ##querydsl -- 어차피 써야 한다......JPA만으로는 불편해서 그럴빠엔 차라리  mybatis를 으음..
    implementation("com.querydsl:querydsl-jpa:$querydslVersion")
    kapt("com.querydsl:querydsl-apt:$querydslVersion:jpa")
    ...
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs = listOf("-Xjsr305=strict")
        jvmTarget = "17"
    }
}
```

### application.properties

```properties
## DB
spring.datasource.test.master.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.test.master.jdbc-url=jdbc:mysq:://localhost/test
spring.datasource.test.master.testname=root
spring.datasource.test.master.password=root
spring.datasource.test.master.pool-name=TESTMasterPool

spring.datasource.test.slave.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.test.slave.jdbc-url=jdbc:mysq:://localhost/test
spring.datasource.test.slave.testname=root
spring.datasource.test.slave.password=root
spring.datasource.test.slave.pool-name=TESTSlavePool
```

### QuerydslConfig.kt

```kotlin
// javax 사용  jakarta로 하려다 속터져서 javax 로 확정

@Configuration
class QuerydslConfig(
    private val em : EntityManager,
){

    @Bean
    fun jpaQueryFactory(): JPAQueryFactory {
        return JPAQueryFactory(em)
    }
}
```

### ReplicationRoutingDataSource.kt

```kotlin
class ReplicationRoutingDataSource : AbstractRoutingDataSource() {
    override fun determineCurrentLookupKey(): Any? {
        return if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) "read" else "write"
    }
}
```

### TestDBConfig

```kotlin
// xxx.repository.test 패키지 하위에 있는 인터페이스들은 해당 설정을 타게 지정
// @Primare 아래의 설정을 여러개 추가 할 경우 추가

@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    entityManagerFactoryRef = "testEntityManager",
    transactionManagerRef = "testTransactionManager",
    basePackages = ["xxx.repository.test"]
)
class TestDbConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.test.master")
    fun masterHikariConfigTest(): HikariConfig {
        return HikariConfig()
    }

    @Bean
    @Primary
    fun masterDataSourceTest(): HikariDataSource {
        return HikariDataSource(masterHikariConfigTest())
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.test.slave")
    fun slaveHikariConfigTest(): HikariConfig {
        return HikariConfig()
    }

    @Bean
    fun slaveDataSourceTest(): HikariDataSource {
        return HikariDataSource(slaveHikariConfigTest())
    }

    @Bean
    fun routingDataSourceTest(): DataSource {
        val routingDataSource = ReplicationRoutingDataSource()
        val dataSourceMap: MutableMap<Any, Any> = HashMap()
        dataSourceMap["write"] = masterDataSourceTest()
        dataSourceMap["read"] = slaveDataSourceTest()
        routingDataSource.setTargetDataSources(dataSourceMap)
        routingDataSource.setDefaultTargetDataSource(masterDataSourceTest())
        return routingDataSource
    }

    @Bean
    fun testDataSource(): DataSource {
        return LazyConnectionDataSourceProxy(routingDataSourceTest())
    }

    @Bean
    @Primary
    fun testTransactionManager(): PlatformTransactionManager {
        val transactionManager = JpaTransactionManager()
        transactionManager.entityManagerFactory = testEntityManager().getObject()
        return transactionManager
    }

    @Bean
    @Primary
    fun testEntityManager(): LocalContainerEntityManagerFactoryBean {
        val em = LocalContainerEntityManagerFactoryBean()
        em.dataSource = testDataSource()
        em.setPackagesToScan("com.photowidget.apis.entity.test")
        val adapter = HibernateJpaVendorAdapter()
        em.jpaVendorAdapter = adapter
        return em
    }
}
```

### Querydsl 개발 시 참고

```kotlin
// 해당 파일들은 별도의 패키지를 구성하여 처리 하였다 그러다 보니.. 위의 설정은 안먹힌다. 그래서 다음과 같이 내용을 추가
class xxxRepositoryImpl(
    var queryFactory: JPAQueryFactory,
    @Qualifier("testEntityManager") em : EntityManager
):xxxRepositoryCustom {
    init {
        this.queryFactory = JPAQueryFactory(em)   // 이걸 안하니.. primary 만 가져온다..
    }
}
```

## 결과

`@Transactional(readOnly = true)`를 추가하니 정상적으로 slave에 접근해 데이터 조회가 가능해졌습니다.

## 아쉬운 점

Jakarta로 하려다 오류가 자꾸 발생되어 `javax`로 변경했습니다. 인내심이 부족했습니다.

몇 년이 되었어도 참고할 만한 내용은 아직 적긴 합니다.

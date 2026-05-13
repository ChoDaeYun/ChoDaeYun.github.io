---
layout: post
title:  "MongoDB - Spring Boot Kotlin"
description: MongoDB 구조와 Spring Boot Kotlin 연동
date:   2026-05-13 00:00:00 +000
categories: MongoDB SpringBoot Kotlin
---

## 개요
실무에서 MongoDB를 사용하면서 헷갈렸던 개념들과 Spring Boot Kotlin 연동 방식을 나중에 다시 찾아보기 쉽게 정리해두고자 작성했습니다.<br>
기본 개념(Document, Collection)부터 객체 설계, Repository 패턴까지 실제 사용했던 구조를 기반으로 예시를 만들었습니다.

---

## MongoDB 구조

### RDB vs MongoDB 비교

| RDB | MongoDB |
|-----|---------|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| JOIN | Embedded Document / $lookup |

### Document
MongoDB에서 데이터의 기본 단위입니다. JSON 형태로 저장되며, 스키마가 유연합니다.

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "metadata": {
    "type": "order",
    "status": "COMPLETED",
    "user": {
      "userId": 1001,
      "name": "홍길동"
    },
    "items": [
      { "productId": "A001", "quantity": 2, "price": 15000 },
      { "productId": "B002", "quantity": 1, "price": 30000 }
    ]
  },
  "createdAt": "2026-05-13T10:00:00"
}
```

**포인트:**
- `_id` 는 각 도큐먼트의 고유 식별자 (미지정 시 ObjectId 자동 생성)
- 중첩 객체(Embedded Document)와 배열을 자유롭게 표현
- RDB의 외래키 JOIN 대신 데이터를 하나의 도큐먼트에 담는 방식을 권장

### Collection
Document의 묶음입니다. RDB의 테이블과 유사하지만, 컬렉션 내 도큐먼트들이 반드시 같은 구조일 필요는 없습니다.

```
orders (collection)
  ├── { _id: ..., type: "order",  user: {...}, items: [...] }
  ├── { _id: ..., type: "order",  user: {...}, items: [...] }
  └── { _id: ..., type: "return", user: {...}, reason: "..." }
```

---

## Spring Boot Kotlin 설정

### build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.25"
    kotlin("plugin.spring") version "1.9.25"
    id("org.springframework.boot") version "3.4.4"
    id("io.spring.dependency-management") version "1.1.7"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}
```

### application.yml

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mydb
```

---

## 객체 구조 설계

### Document 클래스

`@Document` 어노테이션으로 컬렉션 매핑, `@MongoId` 로 ID 필드를 지정합니다.

```kotlin
import com.fasterxml.jackson.annotation.JsonFormat
import org.springframework.data.mongodb.core.mapping.Document
import org.springframework.data.mongodb.core.mapping.MongoId
import java.time.LocalDateTime

@Document(collection = "orders")
data class OrderDocument(
    @MongoId
    val id: String? = null,
    val metadata: OrderMetadata,
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

### Sealed Class로 메타데이터 표현

타입별로 다른 구조의 데이터를 하나의 컬렉션에 저장할 때 `sealed class` 를 활용하면 타입 안전하게 처리할 수 있습니다.

```kotlin
import com.fasterxml.jackson.annotation.JsonIgnoreProperties

@JsonIgnoreProperties(ignoreUnknown = true)
sealed class OrderMetadata {
    abstract val type: OrderType
    abstract val user: UserInfo?
}

data class PurchaseMetadata(
    override val type: OrderType = OrderType.PURCHASE,
    override val user: UserInfo?,
    val items: List<OrderItem>,
    val totalAmount: Long
) : OrderMetadata()

data class ReturnMetadata(
    override val type: OrderType = OrderType.RETURN,
    override val user: UserInfo?,
    val reason: String,
    val refundAmount: Long
) : OrderMetadata()

enum class OrderType {
    PURCHASE, RETURN
}
```

### 중첩 객체 (Embedded Document)

```kotlin
data class UserInfo(
    val userId: Long,
    val name: String,
    val email: String
)

data class OrderItem(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val price: Long
)
```

---

## Custom Converter

`sealed class` 를 MongoDB에 저장/읽기 할 때 타입 정보를 기반으로 역직렬화를 처리해야 합니다.

```kotlin
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import org.bson.Document
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.convert.converter.Converter
import org.springframework.data.convert.ReadingConverter
import org.springframework.data.convert.WritingConverter
import org.springframework.data.mongodb.core.convert.MongoCustomConversions

val objectMapper: ObjectMapper = jacksonObjectMapper()
    .registerModule(JavaTimeModule())

@Configuration
class MongoConverterConfig {
    @Bean
    fun mongoCustomConversions(): MongoCustomConversions {
        return MongoCustomConversions(
            listOf(
                OrderMetadataReadConverter(),
                OrderMetadataWriteConverter()
            )
        )
    }
}

@ReadingConverter
class OrderMetadataReadConverter : Converter<Document, OrderMetadata> {
    override fun convert(source: Document): OrderMetadata {
        val json = source.toJson()
        return when (OrderType.valueOf(source.getString("type"))) {
            OrderType.PURCHASE -> objectMapper.readValue(json, PurchaseMetadata::class.java)
            OrderType.RETURN   -> objectMapper.readValue(json, ReturnMetadata::class.java)
        }
    }
}

@WritingConverter
class OrderMetadataWriteConverter : Converter<OrderMetadata, Document> {
    override fun convert(source: OrderMetadata): Document {
        return Document.parse(objectMapper.writeValueAsString(source))
    }
}
```

---

## Repository 패턴

### MongoTemplate 방식

복잡한 쿼리, Aggregation이 필요할 때 사용합니다.

```kotlin
import org.springframework.data.domain.Sort
import org.springframework.data.mongodb.core.MongoTemplate
import org.springframework.data.mongodb.core.query.Criteria
import org.springframework.data.mongodb.core.query.Query
import org.springframework.stereotype.Repository

@Repository
class OrderRepository(
    private val mongoTemplate: MongoTemplate
) {

    fun save(metadata: OrderMetadata): OrderDocument {
        val document = OrderDocument(metadata = metadata)
        return mongoTemplate.save(document)
    }

    fun findByUserId(userId: Long): List<OrderDocument> {
        val query = Query()
            .addCriteria(Criteria.where("metadata.user.userId").`is`(userId))
            .with(Sort.by(Sort.Direction.DESC, "createdAt"))
        return mongoTemplate.find(query, OrderDocument::class.java)
    }

    fun findLatestByUserId(userId: Long): OrderDocument? {
        val query = Query()
            .addCriteria(Criteria.where("metadata.user.userId").`is`(userId))
            .with(Sort.by(Sort.Direction.DESC, "createdAt"))
            .limit(1)
        return mongoTemplate.findOne(query, OrderDocument::class.java)
    }

    fun findByType(userId: Long, type: OrderType): List<OrderDocument> {
        val query = Query()
            .addCriteria(Criteria.where("metadata.user.userId").`is`(userId))
            .addCriteria(Criteria.where("metadata.type").`is`(type.name))
            .with(Sort.by(Sort.Direction.DESC, "createdAt"))
        return mongoTemplate.find(query, OrderDocument::class.java)
    }
}
```

### Aggregation 활용

날짜별 그룹핑, 집계가 필요한 경우입니다.

```kotlin
import org.springframework.data.mongodb.core.aggregation.Aggregation
import org.springframework.data.mongodb.core.aggregation.DateOperators

fun findMonthlyOrderCount(userId: Long): List<*> {
    val aggr = Aggregation.newAggregation(
        Aggregation.match(
            Criteria.where("metadata.user.userId").`is`(userId)
        ),
        Aggregation.project()
            .and("metadata").`as`("metadata")
            .and("createdAt").`as`("createdAt")
            .and(DateOperators.DateToParts.datePartsOf("createdAt")).`as`("date"),
        Aggregation.group("date.year", "date.month")
            .count().`as`("orderCount")
            .last("createdAt").`as`("lastOrderDate"),
        Aggregation.sort(Sort.Direction.DESC, "lastOrderDate")
    )
    return mongoTemplate.aggregate(aggr, "orders", Any::class.java).mappedResults
}
```

---

## 정리

| 항목 | 내용 |
|------|------|
| `@Document` | 컬렉션 매핑 어노테이션 |
| `@MongoId` | `_id` 필드 지정 |
| `sealed class` | 타입별 다른 구조를 하나의 컬렉션에 저장할 때 유용 |
| `MongoTemplate` | 복잡한 쿼리, Aggregation 처리 |
| `@ReadingConverter` / `@WritingConverter` | sealed class 직렬화/역직렬화 커스터마이징 |

RDB와 비교했을 때 MongoDB의 가장 큰 차별점은 **중첩 구조로 연관 데이터를 한 도큐먼트에 담을 수 있다는 것**입니다.<br>
설계 시 JOIN 대신 Embedded Document를 적극 활용하면 조회 성능을 높일 수 있습니다.

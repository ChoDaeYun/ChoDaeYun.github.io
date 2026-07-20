---
layout: post
title:  "애플로그인 인앱 결제 영수증 검증 , JAVA"
description: 앱에서 결제한 영수증 정보 백엔드 처리  
date:   2020-05-18 14:40:00 +000
categories: Java 
---

# 인앱 결제

앱을 개발하고 서비스하는 분들이라면 잘 아시겠지만, 디지털 상품의 경우 애플에서는 애플스토어 결제 시스템이 아닌 다른 결제 시스템을 허락하지 않습니다.

그래서 간략하게 백엔드에서 영수증을 처리하는 부분을 다뤄보려고 합니다.

우선 사용한 의존성은 다음과 같습니다.

```xml
<dependency>
    <groupId>com.mashape.unirest</groupId>
    <artifactId>unirest-java</artifactId>
    <version>1.4.9</version>
</dependency>
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.0</version>
</dependency>
```

애플에서 제공하는 영수증 API는 다음과 같습니다.

- 라이브 서비스: <https://buy.itunes.apple.com>
- 테스트용 sandbox: <https://sandbox.itunes.apple.com>

애플 API 가이드: <https://developer.apple.com/documentation/appstorereceipts/verifyreceipt>

`buy`는 라이브 서비스 용도이고, `sandbox`는 테스트 용도입니다. iOS 설정에서 샌드박스 아이디를 등록하면 테스트를 편하게 진행할 수 있습니다.

영수증을 검증하는 순서는 간략하게 다음과 같습니다.

1. 영수증 검증 요청
2. `buy.itunes.apple.com`을 통해 영수증 검증 요청
3. 리턴 값이 `21007`인 경우 `sandbox.itunes.apple.com`으로 영수증 검증 재요청
4. 검증 처리 및 완료

영수증을 Decode하는 역할로 수행되는 부분이기 때문에, 인앱 결제상 이미 결제가 되어 있는 상태가 됩니다.

일반적으로 사용하는 변수는 다음과 같이 지정했습니다.

```java
String itunesUrl = "https://buy.itunes.apple.com";
String sandboxUrl = "https://sandbox.itunes.apple.com" ;
/**
* 애플에서 받은 키 정보
*/
String appleSecretKey = "";
/**
* 영수증 정보
*/
String receiptData = "";
```

그리고 영수증을 검증하는 부분은 다음과 같습니다.

```java
/* 영수증 정보 및 애플 키 변수 생성 */
JSONObject bodyData = new JSONObject()
    .put("receipt-data", receiptData)
    .put("password", appleSecretKey)
    .put("exclude-old-transactions", true);

/* 검증 요청 */
HttpResponse<JsonNode> response = Unirest.post(itunesUrl+"/verifyReceipt")
    .header("Content-Type", "application/json")
    .body(bodyData)
    .asJson();

/* sandbox 영수증인 경우 sandbox로 결제 검증 재요청 */
if(response.getStatus() == 200 && response.getBody().getObject().get("status").toString().equals("21007")) {
    response = Unirest.post(sandboxUrl+"/verifyReceipt")
            .header("Content-Type", "application/json")
            .body(bodyData)
            .asJson();
}
/* 영수증 검증이 정상 처리된 경우 */
if(response.getStatus() == 200 && response.getBody().getObject().get("status").toString().equals("0")) {
    JsonParser parser = new JsonParser();
        JsonObject object = (JsonObject)parser.parse(response.getBody().getObject().get("receipt").toString());
        JsonArray array = (JsonArray)parser.parse(object.get("in_app").toString());
        // 최근 결제 내역 가져오기
        String transaction_id = null;
        String original_transaction_id = null;
        String purchase_date_ms = null;
        String product_id = null;
        for(int i = 0 ; i< array.size() ; i ++) {
            object = (JsonObject)parser.parse(array.get(i).toString());
            if(transaction_id == null  || Long.parseLong(purchase_date_ms) < Long.parseLong(object.get("purchase_date_ms").toString().replaceAll("\"",""))) {
                transaction_id  = object.get("transaction_id").toString().replaceAll("\"","");
                original_transaction_id = object.get("original_transaction_id").toString().replaceAll("\"","");
                purchase_date_ms = object.get("purchase_date_ms").toString().replaceAll("\"","");
                product_id = object.get("product_id").toString().replaceAll("\"","");
            }
        }
        System.out.println(transaction_id);
        System.out.println(original_transaction_id);
        System.out.println(purchase_date_ms);
        System.out.println(product_id);
}
```

코드를 보면 결제 검증 이후 `for`문이 의아할 수 있습니다.

애플의 결제 영수증은 결제한 한 건에 대한 영수증 내역이 아닙니다. 해당 서비스를 통해 결제한 내역이라고 보면 될 듯합니다.

그래서 저 같은 경우 `for`문을 통해 최근 처리된 영수증 내역을 가져오도록 했습니다. 부가적으로 더 처리한다면 취소된 내역은 제외하는 등의 요소가 필요합니다.

위 코드는 영수증 검증 시 필요한 내용을 확인하기 위한 코드입니다.

<https://github.com/ChoDaeYun/apple-verifyReceipt>

# 기타

## 소모성, 비소모성 타입 상품의 환불

소모성의 경우 환불 확인이 힘들다는 내용이 많아 비소모성으로 상품을 등록했습니다.

환불 체크는 iOS 클라이언트 레벨에서 영수증을 확인하고, 환불 내역이 있을 경우 서버로 환불 처리 요청을 하도록 개발했습니다.

## 정기 결제의 구독 취소, 환불, 결제

정기 결제에서 구독 취소는 상품은 유효하지만 재결제는 하지 않는다는 내용이므로, 이력 외에는 서버에서 크게 처리할 부분이 없습니다.

문제는 환불과 재결제 이슈인데, 아직 실제로 개발은 진행해본 적이 없어 고민만 하고 있습니다.

## 구글 인앱 결제와의 차이

구글은 48시간 이후 서비스사에 문의하도록 안내되는데, 애플 환불은 애플스토어에서만 가능합니다.

서비스사에서 환불을 처리해주는 게 가능한지도 모르겠습니다. 관련 내용도 찾지 못했습니다.

구글은 재무관리 권한이 있다면 주문 내역 조회가 가능해서 환불 진행이 필요할 때 서비스사에서 직접 환불이 가능한데 말입니다.

그 외 일반적인 결제 서비스를 봐도 별도의 환불 기능이 존재하는데, 애플은 왜 없는 것 같은지...

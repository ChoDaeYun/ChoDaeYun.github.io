---
layout: post
title:  "애플로그인 적용시 자바서버에서 인증 확인"
description: 자바에서 애플로그인 시 전달된 authorization code 값 인증 
date:   2020-04-17 16:40:00 +000
categories: Java 
---
# 애플로그인 적용을 해야 하는가...
IOS 앱의 경우 소셜로그인을 기존에 사용하고 있다면 2020년 4월부터는 애플로그인을 붙여 하는 이슈가 있습니다. <br>
이런 이슈에 대한 준비를 하기 위해 늦었지만 자바 백엔드에서 애플 로그인시 확인이 필요 한 부분에 대해서 한번 볼까 합니다.  <br>
<a href="https://developer.apple.com/kr/news/?id=09122019b" target="_blank">Apple로 로그인에 대한 신규 가이드 ( 2019년 09월 12일)</a><Br>
<a href="https://developer.apple.com/kr/app-store/review/guidelines/" target="_blank">App Strore 심사 지침서 </a><Br>

설정에 대한 부분은 우선 일부 내용을 넘어 갈까 합니다.  <br>
구글검색에서 워낙 많이들 검색이 되고 있어 굳이 언급할 필요성을 못느껴서 입니다.  <br>

그렇다면 먼저 IOS 앱 내에서 로그인 아이디 비밀번호 확인이 끝난뒤에 진행되는 흐름은 다음 이미지와 같습니다. <br>
<img src="/assets/images/884cba8b-78cd-4bb1-91af-909591ece5dd.png" width="452" height="auto">
https://developer.apple.com/documentation/sign_in_with_apple/sign_in_with_apple_rest_api/verifying_a_user 

쉽게 앱에서 API서버로 authorization code 값을 보내고 해당 code 값을 가지고 정상적인 로그인 접근인지를 확인하라는 내용입니다. <br>
애플 가이드대로라면 백엔드 서버에서 두가지 처리요소만 준비 하면됩니다. 
1. Authorization code 값 인증 하기  - 로그인 또는 회원가입시 인증에 사용되는 코드 ( 일회성으로 API호출시 새로운 Authorization code 가 발급됩니다. (api 호출하면 accesstoken 값이 authoriztion code 값을 나타 냅니다. ))
2. refresh token 검증하기  - 서비스 중 애플에서 연동이 된 계정이 맞는지 확인할때 사용된다.  하루에 한번 정도만 호출하라고 가이드가 되어 있습니다. 


Generate and validate tokens URL : <a href="https://developer.apple.com/documentation/sign_in_with_apple/generate_and_validate_tokens">애플 가이드</a><Br>
Apple Api URL : https://appleid.apple.com/auth/token 
# 사용한 의존성 
```xml
    <!-- JJWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.10.7</version>
        </dependency>        
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.10.7</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.10.7</version>
            <scope>runtime</scope>
        </dependency>
        <!-- openssl  -->
        <dependency>
            <groupId>org.bouncycastle</groupId>
            <artifactId>bcpkix-jdk15on</artifactId>
            <version>1.61</version>
        </dependency>
        <!-- unirest-java  -->
        <dependency>
            <groupId>com.mashape.unirest</groupId>
            <artifactId>unirest-java</artifactId>
            <version>1.4.9</version>
        </dependency>
```

# 기본 공통 
```java
    private static String AUTH_TOKEN = "https://appleid.apple.com/auth/token";   
    private static String TEAM_ID = ""; // 애플 개발자 센터에 등록한 TEAM ID 
    private static String CLIENT_ID = ""; //애플 개발자 센터에서 등록시 app , servier id ( ios 와 web&android 각각 발급받아야 서비스 이용 가능합니다.)
    private static String KEY_ID = ""; // 키 ID ( 잘모르면 p8 파일 받을때 AuthKey_{KEY_ID}.p8 파일명 적혀 있습니다. )
    private static PrivateKey pKey;

    private static PrivateKey getPrivateKey() throws Exception {        
         
        final PEMParser pemParser = new PEMParser(new FileReader("애플설정시 다운로드 받은 p8파일 경로"));
        final JcaPEMKeyConverter converter = new JcaPEMKeyConverter();
        final PrivateKeyInfo object = (PrivateKeyInfo) pemParser.readObject();
        final PrivateKey pKey = converter.getPrivateKey(object);
        return pKey;
    }

    private static String generateJWT(Boolean webBool) throws Exception {
      if (pKey == null) {
          pKey = getPrivateKey();
      }
      String token = Jwts.builder().setHeaderParam(JwsHeader.KEY_ID, KEY_ID)
              .setIssuer(TEAM_ID)
              .setAudience("https://appleid.apple.com")
              .setSubject(CLIENT_ID)
              .setExpiration(new Date(System.currentTimeMillis() + (1000 * 60 * 10)))
              .setIssuedAt(new Date(System.currentTimeMillis()))
              .signWith(pKey,SignatureAlgorithm.ES256)
              .compact();
      return token;
    }

    private static String generateJWT() throws Exception {
      if (pKey == null) {
          pKey = getPrivateKey();
      }
      String token = Jwts.builder().setHeaderParam(JwsHeader.KEY_ID, KEY_ID)
              .setIssuer(TEAM_ID)
              .setAudience("https://appleid.apple.com")
              .setSubject(CLIENT_ID)
              .setExpiration(new Date(System.currentTimeMillis() + (1000 * 60 * 10)))
              .setIssuedAt(new Date(System.currentTimeMillis()))
              .signWith(pKey,SignatureAlgorithm.ES256)
              .compact();
      return token;
    }

```
# Authorization code 인증하기 
```java
public static void authToken(String code) throws Exception{
    String token = generateJWT();

    HttpResponse<String> response = Unirest.post(AUTH_TOKEN)
                  .header("Content-Type", "application/x-www-form-urlencoded")
          .field("client_id", CLIENT_ID)
          .field("client_secret", token)
          .field("grant_type", "authorization_code")
          .field("code", code)
              .asString();
    System.out.println(response.getStatus());
    System.out.println(response.getBody());
}
```
이때 리턴된 전달 된 값을 통해 authorization code 그리고 refresh token , apple id ,email 값을 확인 할 수 있습니다.
(이름의 경우 ios에서 전달을 해주어야 합니다. )

# Refresh Token 검증 하기 
```java
public static void authRefreshToken(String refreshToken) throws Exception{
    String token = generateJWT();
    HttpResponse<String> response = Unirest.post(AUTH_TOKEN)
                  .header("Content-Type", "application/x-www-form-urlencoded")
              .field("client_id", CLIENT_ID)
              .field("client_secret", token)
              .field("grant_type", "refresh_token")
              .field("refresh_token", refreshToken)
              .asString();
    System.out.println(response.getStatus());
    System.out.println(response.getBody());
}
```
아무래도 1일 1회 호출이라는 가이드가 계속 눈에 밟히는데 
1일 1회 호출이라하면 결국 로그인이 유지 되는 서비스의 경우 24시간 이후 연결이 끊어 지는게 반영이 된다. 
단 애플 기기를 통하여 로그인 하였다면 즉시 반영이 가능하다.<br>( 애플서버 -> IOS 기기 호출 , 웹과 안드로이드만 했다면 ... 지원이 안되는거 같아 보입니다. ) 

# 웹 애플로그인 
안드로이드와 앱의 경우 IOS와 는 다른  CLIENT ID 값으로 로그인을 진행해주셔야지만 합니다. 
또한 로그인 사전에 로그인후 리다이렉트 URL 에 대한 설정 그리고 도메인 추가 작업이 필요 합니다. 

# 로그임 버튼 적용 하기 
Human Interface Guidelines (Button) : <a href="https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/" target="_blank" >가이드</a><br>
(뭔가 까다롭다.. 거참.. 그러니...직접안만들고 제공해주는것을 사용하자.. )

```html
<style>
    .signin-button {
        width: 240px;
        height: 40px;
    }
</style>
<div class="input-group">
    <div id="appleid-signin" class="signin-button" data-color="black" data-border="false" data-type="sign in"></div>
</div>
<script type="text/javascript" src="https://appleid.cdn-apple.com/appleauth/static/jsapi/appleid/1/en_US/appleid.auth.js"></script>
<script type="text/javascript">
    AppleID.auth.init({
        clientId : '{CLIENT_ID}',
        scope : 'name email',
        redirectURI: '{REDIRECT_URI}',
        state : 'web'
    });
</script>
```

# Authorization code 인증하기 (웹)
리다이렉트로 전달 받을 경우 POST로 내용이 전달 되며 전달되는 내용중 다음과 같은 값이 존재 하며 활용하면됩니다. <br>
code : authorization code 값
user : scope에서 이름과 이메일 요청한 경우 해당 값이 user라는 변수에 통해 전달이 됩니다. 
state : 웹,안드로이드에서 요청시 전달한 값 (안써도 로그인은 잘됩니다.)
전달하는 값이 틀릴뿐 결국 인증을 하는 방식은 동일하게 처리 됩니다.  
코드 생략 .... 


대략적으로 위와 같은 기능들만 있으면 로그인 이나 회원가입시 기초 데이터를 받아 오는 부분에 대한 백엔드 준비는 완료 되는것입니다. 
간략한 소스는 github 에 올려 놓았습니다. 참고용으로 제공되는 부분이니 소스 부분을 보시고 활용하시면 될 듯합니다. 

<a href="https://github.com/ChoDaeYun/apple-authtoken" target="_blank">https://github.com/ChoDaeYun/apple-authtoken</a>

아직까지는 애플로그인개발에 대해서 뭐가 답이다 아니다 나와 있는 부분은 찾지 못했습니다. 
그렇기 때문에 그냥 ... 자신의 서비스에 맞게 잘만들면 되지 않나 생각합니다. 
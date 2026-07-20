---
layout: post
title:  "애플로그인 적용시 자바서버에서 인증 확인"
description: 자바에서 애플로그인 시 전달된 authorization code 값 인증 
date:   2020-04-17 16:40:00 +000
categories: Java 
---

# 애플 로그인을 적용해야 하는가

iOS 앱에서 기존에 소셜 로그인을 사용하고 있다면, 2020년 4월부터는 애플 로그인을 붙여야 하는 이슈가 있었습니다.

늦었지만 이런 이슈에 대비하기 위해, Java 백엔드에서 애플 로그인 시 확인이 필요한 부분을 정리해 봅니다.

- [Apple로 로그인에 대한 신규 가이드 (2019년 09월 12일)](https://developer.apple.com/kr/news/?id=09122019b)
- [App Store 심사 지침서](https://developer.apple.com/kr/app-store/review/guidelines/)

설정에 대한 부분은 우선 일부 내용을 넘어가려고 합니다. 구글 검색으로 워낙 많이 나오고 있어 굳이 언급할 필요성을 느끼지 못했습니다.

그렇다면 먼저 iOS 앱 안에서 로그인 아이디와 비밀번호 확인이 끝난 뒤 진행되는 흐름을 봅니다.

<img src="/assets/images/884cba8b-78cd-4bb1-91af-909591ece5dd.png" width="452" height="auto">

참고: <https://developer.apple.com/documentation/sign_in_with_apple/sign_in_with_apple_rest_api/verifying_a_user>

쉽게 말하면 앱에서 API 서버로 `authorization code` 값을 보내고, 서버는 해당 `code` 값으로 정상적인 로그인 접근인지 확인하라는 내용입니다.

애플 가이드 기준으로 백엔드 서버에서는 두 가지 처리 요소만 준비하면 됩니다.

1. `Authorization code` 값 인증하기: 로그인 또는 회원가입 시 인증에 사용되는 일회성 코드입니다. API 호출 시 새로운 `Authorization code`가 발급됩니다. API를 호출하면 `access token` 값이 `authorization code` 값을 나타냅니다.
2. `refresh token` 검증하기: 서비스 중 애플에서 연동된 계정이 맞는지 확인할 때 사용합니다. 하루에 한 번 정도만 호출하라고 가이드되어 있습니다.

- Generate and validate tokens URL: [애플 가이드](https://developer.apple.com/documentation/sign_in_with_apple/generate_and_validate_tokens)
- Apple API URL: <https://appleid.apple.com/auth/token>

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
<!-- openssl -->
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcpkix-jdk15on</artifactId>
    <version>1.61</version>
</dependency>
<!-- unirest-java -->
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
private static String CLIENT_ID = ""; // 애플 개발자 센터에서 등록한 app, service id (iOS와 web&android 각각 발급받아야 서비스 이용 가능)
private static String KEY_ID = ""; // 키 ID (잘 모르겠다면 p8 파일 받을 때 AuthKey_{KEY_ID}.p8 파일명에 적혀 있습니다.)
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

이때 리턴된 값을 통해 `authorization code`, `refresh token`, `apple id`, `email` 값을 확인할 수 있습니다.

이름의 경우 iOS에서 전달해주어야 합니다.

# Refresh Token 검증하기

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

아무래도 “1일 1회 호출”이라는 가이드가 계속 눈에 밟힙니다.

1일 1회 호출이라 하면, 결국 로그인이 유지되는 서비스의 경우 24시간 이후 연결이 끊어진 것이 반영됩니다. 단, 애플 기기를 통해 로그인했다면 즉시 반영이 가능합니다.

애플 서버에서 iOS 기기를 호출하는 방식으로 보이며, 웹과 안드로이드만 사용했다면 지원이 안 되는 것처럼 보입니다.

# 웹 애플 로그인

안드로이드와 웹 앱의 경우 iOS와는 다른 `CLIENT_ID` 값으로 로그인을 진행해야 합니다.

또한 로그인 전에 로그인 후 리다이렉트 URL 설정과 도메인 추가 작업이 필요합니다.

# 로그인 버튼 적용하기

Human Interface Guidelines (Button): [가이드](https://developer.apple.com/design/human-interface-guidelines/sign-in-with-apple/overview/buttons/)

뭔가 까다롭습니다. 그러니 직접 만들지 말고 제공해주는 것을 사용합시다.

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

리다이렉트로 전달받을 경우 POST로 내용이 전달됩니다. 전달되는 내용 중 다음 값을 활용하면 됩니다.

- `code`: `authorization code` 값
- `user`: `scope`에서 이름과 이메일을 요청한 경우 `user`라는 변수로 전달되는 값
- `state`: 웹, 안드로이드에서 요청 시 전달한 값. 사용하지 않아도 로그인은 잘 됩니다.

전달하는 값이 다를 뿐, 결국 인증 방식은 동일하게 처리됩니다.

코드는 생략합니다.

대략적으로 위와 같은 기능만 있으면 로그인이나 회원가입 시 기초 데이터를 받아오는 부분에 대한 백엔드 준비는 완료됩니다.

간략한 소스는 GitHub에 올려두었습니다. 참고용으로 제공되는 부분이니 소스를 보고 활용하면 될 듯합니다.

<https://github.com/ChoDaeYun/apple-authtoken>

아직까지는 애플 로그인 개발에 대해 “이게 답이다”라고 말할 만한 부분은 찾지 못했습니다.

그렇기 때문에 그냥 자신의 서비스에 맞게 잘 만들면 되지 않나 생각합니다.

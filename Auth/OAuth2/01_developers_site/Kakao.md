# 카카오 개발자 센터 처음 접속할 때 할 일

## 1. 카카오 개발자 앱 등록

1. [Kakao Developers](https://developers.kakao.com) 접속 → 로그인
2. **내 애플리케이션** → **애플리케이션 추가하기**
3. 앱 이름 입력 후 생성
4. **앱 키** 메뉴에서 **REST API 키** 복사해두기


## 2. 카카오 로그인 설정

앱 선택 → **제품 설정** → **카카오 로그인**

1. **활성화 설정**: ON
   - 웹훅은 "카카오에서 우리 서버로 이벤트 알림을 보내는 기능"인데, 나중에 필요하면 설정하면 된다.
2. **Redirect URI 등록**:
   ```
   http://localhost:8080/login/oauth2/code/kakao
   ```
3. **동의항목** → 닉네임 **필수 동의**로 설정
    - 이메일은 비즈 앱으로 전환해야 필수 동의로 설정할 수 있기에
    - 로그인 후 이메일을 받아오는 로직을 서버에서 처리한다.

## 3. Spring Boot 의존성 추가

```groovy
// build.gradle
dependencies {
    // Security + OAuth2
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

## 4. application.yml 설정

```yaml
server:
  port: 8080
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            redirect-uri: "{baseUrl}/login/oauth2/code/kakao"
            authorization-grant-type: authorization_code
            client-authentication-method: client_secret_post
            scope:
              - profile_nickname
            client-name: Kakao
          naver:
            client-id: ${NAVER_CLIENT_ID}
            client-secret: ${NAVER_CLIENT_SECRET}
            redirect-uri: "{baseUrl}/login/oauth2/code/naver"
            authorization-grant-type: authorization_code
            scope:
              - name
            client-name: Naver
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
          naver:
            authorization-uri: https://nid.naver.com/oauth2.0/authorize
            token-uri: https://nid.naver.com/oauth2.0/token
            user-info-uri: https://openapi.naver.com/v1/nid/me
            user-name-attribute: response

  data:
    redis:
      host: localhost
      port: 6379

jwt:
  secret: ${JWT_SECRET}
  access-token-validity: 1800000      # 30분
  refresh-token-validity: 1209600000  # 14일
```
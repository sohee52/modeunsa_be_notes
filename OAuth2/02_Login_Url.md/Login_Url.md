# OAuth Login Url 생성
- 로그인 화면으로 이동하기 위한 URL 생성
- 예: 사용자가 /oauth/kakao/url 호출 → 카카오 로그인 URL 반환

## 흐름
```
Controller
    ↓
AuthFacade.getOAuthLoginUrl(KAKAO, redirectUri)
    ↓
OAuthUrlUseCase.generateOAuthUrl(KAKAO, redirectUri)
    ↓
OAuthClientFactory.getClient(KAKAO) → KakaoOAuthClient 반환
    ↓
KakaoOAuthClient.generateOAuthUrl(redirectUri)
    ↓
"https://kauth.kakao.com/oauth/authorize?client_id=..."
```

## KakaoOAuthClient

```java
package com.modeunsa.boundedcontext.auth.out.client;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

@Component
public class KakaoOAuthClient implements OAuthClient {
  @Value("${spring.security.oauth2.client.registration.kakao.client-id}")
  private String kakaoClientId;

  @Value("${spring.security.oauth2.client.registration.kakao.redirect-uri}")
  private String kakaoRedirectUri;

  @Override
  public OAuthProvider getProvider() {
    return OAuthProvider.KAKAO;
  }

  @Override
  public String generateOAuthUrl(String redirectUri) {
    String finalRedirectUri = redirectUri != null ? redirectUri : kakaoRedirectUri;

    return UriComponentsBuilder.fromUriString("https://kauth.kakao.com/oauth/authorize")
        .queryParam("client_id", kakaoClientId)
        .queryParam("redirect_uri", finalRedirectUri)
        .queryParam("response_type", "code")
        // 카카오 필수 동의 항목이 있다면 카카오는 scope를 추가해야 한다. 
        .build()
        .toUriString();
  }
}
```

## NaverOAuthClient
```java
package com.modeunsa.boundedcontext.auth.out.client;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import java.util.UUID;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

@Component
public class NaverOAuthClient implements OAuthClient{

  @Value("${spring.security.oauth2.client.registration.naver.client-id}")
  private String naverClientId;

  @Value("${spring.security.oauth2.client.registration.naver.redirect-uri}")
  private String naverRedirectUri;

  @Override
  public OAuthProvider getProvider() {
    return OAuthProvider.NAVER;
  }

  @Override
  public String generateOAuthUrl(String redirectUri) {
    String finalRedirectUri = redirectUri != null ? redirectUri : naverRedirectUri;

    return UriComponentsBuilder.fromUriString("https://nid.naver.com/oauth2.0/authorize")
        .queryParam("client_id", naverClientId)
        .queryParam("redirect_uri", finalRedirectUri)
        .queryParam("response_type", "code")
        .queryParam("state", UUID.randomUUID().toString())  // 네이버는 state 필수
        .build()
        .toUriString();
  }
}
```
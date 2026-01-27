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
import java.time.Duration;
import java.util.UUID;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

@Component
@RequiredArgsConstructor
public class KakaoOAuthClient implements OAuthClient {
  private final StringRedisTemplate redisTemplate;

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
    String state = UUID.randomUUID().toString();

    // Redis에 state 저장 (5분 TTL)
    redisTemplate.opsForValue().set(
        "oauth:state:" + state,
        "KAKAO",  // provider 정보도 같이 저장
        Duration.ofMinutes(5)
    );

    return UriComponentsBuilder.fromUriString("https://kauth.kakao.com/oauth/authorize")
        .queryParam("client_id", kakaoClientId)
        .queryParam("redirect_uri", finalRedirectUri)
        .queryParam("response_type", "code")
        .queryParam("state", state)
        .build()
        .toUriString();
  }
}
```

## NaverOAuthClient
```java
package com.modeunsa.boundedcontext.auth.out.client;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import java.time.Duration;
import java.util.UUID;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

@Component
@RequiredArgsConstructor
public class NaverOAuthClient implements OAuthClient{
  private final StringRedisTemplate redisTemplate;

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
    String state = UUID.randomUUID().toString();

    // Redis에 state 저장 (5분 TTL)
    redisTemplate.opsForValue().set(
        "oauth:state:" + state,
        "NAVER",  // provider 정보도 같이 저장
        Duration.ofMinutes(5)
    );

    return UriComponentsBuilder.fromUriString("https://nid.naver.com/oauth2.0/authorize")
        .queryParam("client_id", naverClientId)
        .queryParam("redirect_uri", finalRedirectUri)
        .queryParam("response_type", "code")
        .queryParam("state", state)
        .build()
        .toUriString();
  }
}
```

## State가 필요한 이유

State는 CSRF(Cross-Site Request Forgery) 공격을 막기 위한 것이다.

**공격 시나리오 (state 없을 때)**

1. 해커가 자기 카카오 계정으로 OAuth 로그인 시작
2. 콜백 URL(`https://modeunsa.com/callback?code=해커코드`)을 얻음
3. 이 링크를 피해자에게 보냄 (메일, 게시글 등)
4. 피해자가 클릭하면, 피해자 브라우저에서 해커 계정으로 로그인됨
5. 피해자가 결제정보나 개인정보 입력하면 → 해커 계정에 저장됨

**State가 있으면**

1. 사용자가 로그인 시작할 때 서버가 랜덤 state 생성 (`abc123`)
2. 카카오로 보낼 때 state 포함 → 콜백에도 state가 돌아옴
3. 서버는 "내가 만든 state인지" 확인
4. 해커가 만든 콜백 URL에는 해커의 state가 있음 → 서버에 저장된 게 아니니까 거부됨

## Redis에 저장하는 이유

State를 "내가 만든 게 맞는지" 검증하려면 어딘가에 저장해둬야 한다.

**왜 Redis인가?**

| 저장소 | 문제점 |
|--------|--------|
| 세션 | 서버 여러 대면 세션 공유 안 됨 |
| DB | 5분 뒤 삭제해야 하는데 DB에 넣기엔 과함 |
| Redis | TTL 자동 만료 + 서버 간 공유 가능 |

Redis의 TTL 기능 덕분에 5분 지나면 자동 삭제되어서, 만료된 state 정리하는 로직을 따로 안 만들어도 된다.

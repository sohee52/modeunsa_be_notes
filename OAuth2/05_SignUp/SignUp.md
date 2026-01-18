# SignUp

## 현재 상태
```
✅ OAuth 로그인 URL 생성 (GET /api/v1/auths/oauth/{provider}/url)
✅ State 파라미터 생성 및 Redis 저장
✅ JWT 토큰 발급/재발급
✅ 로그아웃 + 블랙리스트

❌ OAuth 콜백 처리 (code → 토큰 교환 → 사용자 정보 조회)
❌ 회원가입/로그인 분기 처리
❌ 회원가입 이벤트 발행
```

## 구현 순서
```
1. OAuth 토큰 교환 (code → access_token)
   └─ KakaoOAuthClient, NaverOAuthClient에 메서드 추가

2. OAuth 사용자 정보 조회
   └─ 카카오/네이버 API로 사용자 정보 가져오기

3. 회원가입/로그인 처리 UseCase
   └─ 소셜 계정 존재 → 로그인
   └─ 소셜 계정 없음 → 회원가입 + 이벤트 발행

4. 회원가입 이벤트
   └─ MemberSignupEvent 발행
   └─ @EventListener로 후속 처리
```


좋아! OAuth 토큰 교환 + 사용자 정보 조회 구현하자.

---

## 구현할 파일 목록

| 파일 | 설명 |
|------|------|
| `OAuthClient.java` | 인터페이스에 메서드 추가 |
| `OAuthTokenResponse.java` | 토큰 응답 DTO (새 파일) |
| `OAuthUserInfo.java` | 사용자 정보 DTO (새 파일) |
| `KakaoOAuthClient.java` | 토큰 교환 + 사용자 정보 조회 |
| `NaverOAuthClient.java` | 토큰 교환 + 사용자 정보 조회 |

---

## 1. OAuthClient.java (인터페이스 수정)

```java
package com.modeunsa.boundedcontext.auth.out.client;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthTokenResponse;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthUserInfo;

public interface OAuthClient {

    OAuthProvider getProvider();

    // OAuth2 인가 URL 생성
    String generateOAuthUrl(String redirectUri);

    // 인가 코드로 토큰 교환
    OAuthTokenResponse getToken(String code, String redirectUri);

    // 토큰으로 사용자 정보 조회
    OAuthUserInfo getUserInfo(String accessToken);
}
```

---

## 2. OAuthTokenResponse.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.out.client.dto;

import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class OAuthTokenResponse {

    private String accessToken;
    private String refreshToken;
    private String tokenType;
    private Long expiresIn;

    public static OAuthTokenResponse of(String accessToken, String refreshToken, 
                                         String tokenType, Long expiresIn) {
        return OAuthTokenResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .tokenType(tokenType)
                .expiresIn(expiresIn)
                .build();
    }
}
```

---

## 3. OAuthUserInfo.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.out.client.dto;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class OAuthUserInfo {

    private OAuthProvider provider;
    private String providerId;      // 소셜 서비스의 고유 ID
    private String email;
    private String nickname;
    private String profileImageUrl;

    public static OAuthUserInfo of(OAuthProvider provider, String providerId, 
                                    String email, String nickname, String profileImageUrl) {
        return OAuthUserInfo.builder()
                .provider(provider)
                .providerId(providerId)
                .email(email)
                .nickname(nickname)
                .profileImageUrl(profileImageUrl)
                .build();
    }
}
```

---

## 4. KakaoOAuthClient.java (전체 수정)

```java
package com.modeunsa.boundedcontext.auth.out.client;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthTokenResponse;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthUserInfo;
import com.modeunsa.global.exception.GeneralException;
import com.modeunsa.global.status.ErrorStatus;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestClient;
import org.springframework.web.util.UriComponentsBuilder;

@Slf4j
@Component
@RequiredArgsConstructor
public class KakaoOAuthClient implements OAuthClient {

    private final StringRedisTemplate redisTemplate;
    private final OAuthClientProperties properties;
    private final RestClient restClient = RestClient.create();

    private static final String TOKEN_URL = "https://kauth.kakao.com/oauth/token";
    private static final String USER_INFO_URL = "https://kapi.kakao.com/v2/user/me";

    @Override
    public OAuthProvider getProvider() {
        return OAuthProvider.KAKAO;
    }

    @Override
    public String generateOAuthUrl(String redirectUri) {
        OAuthClientProperties.Registration kakaoProps = properties.registration().get("kakao");

        String finalRedirectUri = redirectUri != null ? redirectUri : kakaoProps.redirectUri();
        String state = UUID.randomUUID().toString();

        redisTemplate.opsForValue().set("oauth:state:" + state, "KAKAO", Duration.ofMinutes(5));

        return UriComponentsBuilder.fromUriString("https://kauth.kakao.com/oauth/authorize")
                .queryParam("client_id", kakaoProps.clientId())
                .queryParam("redirect_uri", finalRedirectUri)
                .queryParam("response_type", "code")
                .queryParam("state", state)
                .build()
                .toUriString();
    }

    @Override
    public OAuthTokenResponse getToken(String code, String redirectUri) {
        OAuthClientProperties.Registration kakaoProps = properties.registration().get("kakao");
        String finalRedirectUri = redirectUri != null ? redirectUri : kakaoProps.redirectUri();

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("grant_type", "authorization_code");
        params.add("client_id", kakaoProps.clientId());
        params.add("client_secret", kakaoProps.clientSecret());
        params.add("redirect_uri", finalRedirectUri);
        params.add("code", code);

        try {
            Map<String, Object> response = restClient.post()
                    .uri(TOKEN_URL)
                    .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                    .body(params)
                    .retrieve()
                    .body(new ParameterizedTypeReference<>() {});

            return OAuthTokenResponse.of(
                    (String) response.get("access_token"),
                    (String) response.get("refresh_token"),
                    (String) response.get("token_type"),
                    ((Number) response.get("expires_in")).longValue()
            );
        } catch (Exception e) {
            log.error("카카오 토큰 요청 실패: {}", e.getMessage());
            throw new GeneralException(ErrorStatus.OAUTH_TOKEN_REQUEST_FAILED);
        }
    }

    @Override
    public OAuthUserInfo getUserInfo(String accessToken) {
        try {
            Map<String, Object> response = restClient.get()
                    .uri(USER_INFO_URL)
                    .header("Authorization", "Bearer " + accessToken)
                    .retrieve()
                    .body(new ParameterizedTypeReference<>() {});

            String id = String.valueOf(response.get("id"));

            Map<String, Object> kakaoAccount = (Map<String, Object>) response.get("kakao_account");
            Map<String, Object> profile = kakaoAccount != null 
                    ? (Map<String, Object>) kakaoAccount.get("profile") 
                    : null;

            String email = kakaoAccount != null ? (String) kakaoAccount.get("email") : null;
            String nickname = profile != null ? (String) profile.get("nickname") : null;
            String profileImage = profile != null ? (String) profile.get("profile_image_url") : null;

            return OAuthUserInfo.of(OAuthProvider.KAKAO, id, email, nickname, profileImage);
        } catch (Exception e) {
            log.error("카카오 사용자 정보 요청 실패: {}", e.getMessage());
            throw new GeneralException(ErrorStatus.OAUTH_USER_INFO_FAILED);
        }
    }
}
```

---

## 5. NaverOAuthClient.java (전체 수정)

```java
package com.modeunsa.boundedcontext.auth.out.client;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthTokenResponse;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthUserInfo;
import com.modeunsa.global.exception.GeneralException;
import com.modeunsa.global.status.ErrorStatus;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;
import org.springframework.web.util.UriComponentsBuilder;

@Slf4j
@Component
@RequiredArgsConstructor
public class NaverOAuthClient implements OAuthClient {

    private final StringRedisTemplate redisTemplate;
    private final OAuthClientProperties properties;
    private final RestClient restClient = RestClient.create();

    private static final String TOKEN_URL = "https://nid.naver.com/oauth2.0/token";
    private static final String USER_INFO_URL = "https://openapi.naver.com/v1/nid/me";

    @Override
    public OAuthProvider getProvider() {
        return OAuthProvider.NAVER;
    }

    @Override
    public String generateOAuthUrl(String redirectUri) {
        OAuthClientProperties.Registration naverProps = properties.registration().get("naver");

        String finalRedirectUri = redirectUri != null ? redirectUri : naverProps.redirectUri();
        String state = UUID.randomUUID().toString();

        redisTemplate.opsForValue().set("oauth:state:" + state, "NAVER", Duration.ofMinutes(5));

        return UriComponentsBuilder.fromUriString("https://nid.naver.com/oauth2.0/authorize")
                .queryParam("client_id", naverProps.clientId())
                .queryParam("redirect_uri", finalRedirectUri)
                .queryParam("response_type", "code")
                .queryParam("state", state)
                .build()
                .toUriString();
    }

    @Override
    public OAuthTokenResponse getToken(String code, String redirectUri) {
        OAuthClientProperties.Registration naverProps = properties.registration().get("naver");
        String finalRedirectUri = redirectUri != null ? redirectUri : naverProps.redirectUri();

        String uri = UriComponentsBuilder.fromUriString(TOKEN_URL)
                .queryParam("grant_type", "authorization_code")
                .queryParam("client_id", naverProps.clientId())
                .queryParam("client_secret", naverProps.clientSecret())
                .queryParam("redirect_uri", finalRedirectUri)
                .queryParam("code", code)
                .build()
                .toUriString();

        try {
            Map<String, Object> response = restClient.get()
                    .uri(uri)
                    .retrieve()
                    .body(new ParameterizedTypeReference<>() {});

            return OAuthTokenResponse.of(
                    (String) response.get("access_token"),
                    (String) response.get("refresh_token"),
                    (String) response.get("token_type"),
                    Long.parseLong((String) response.get("expires_in"))
            );
        } catch (Exception e) {
            log.error("네이버 토큰 요청 실패: {}", e.getMessage());
            throw new GeneralException(ErrorStatus.OAUTH_TOKEN_REQUEST_FAILED);
        }
    }

    @Override
    public OAuthUserInfo getUserInfo(String accessToken) {
        try {
            Map<String, Object> response = restClient.get()
                    .uri(USER_INFO_URL)
                    .header("Authorization", "Bearer " + accessToken)
                    .retrieve()
                    .body(new ParameterizedTypeReference<>() {});

            Map<String, Object> naverResponse = (Map<String, Object>) response.get("response");

            String id = (String) naverResponse.get("id");
            String email = (String) naverResponse.get("email");
            String nickname = (String) naverResponse.get("nickname");
            String profileImage = (String) naverResponse.get("profile_image");

            return OAuthUserInfo.of(OAuthProvider.NAVER, id, email, nickname, profileImage);
        } catch (Exception e) {
            log.error("네이버 사용자 정보 요청 실패: {}", e.getMessage());
            throw new GeneralException(ErrorStatus.OAUTH_USER_INFO_FAILED);
        }
    }
}
```

---

## 파일 구조

```
auth/out/client/
├── dto/
│   ├── OAuthTokenResponse.java  (새 파일)
│   └── OAuthUserInfo.java       (새 파일)
├── KakaoOAuthClient.java        (수정)
├── NaverOAuthClient.java        (수정)
├── OAuthClient.java             (수정)
├── OAuthClientFactory.java
└── OAuthClientProperties.java
```

---

여기까지 추가하고, 다음은 **회원가입/로그인 처리 UseCase** 구현할게. 질문 있으면 말해!

좋아! 이제 **회원가입/로그인 처리 UseCase**와 **회원가입 이벤트**를 구현하자.

---

## 구현할 파일 목록

| 파일 | 설명 |
|------|------|
| `AuthSocialAccountRepository.java` | 소셜 계정 Repository (새 파일) |
| `OAuthLoginUseCase.java` | OAuth 로그인/회원가입 처리 (새 파일) |
| `MemberSignupEvent.java` | 회원가입 이벤트 (새 파일) |
| `MemberSignupEventListener.java` | 이벤트 리스너 (새 파일) |
| `AuthFacade.java` | oauthLogin 메서드 추가 |
| `ApiV1AuthController.java` | login API 수정 |

---

## 1. AuthSocialAccountRepository.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.out.repository;

import com.modeunsa.boundedcontext.auth.domain.entity.AuthSocialAccount;
import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AuthSocialAccountRepository extends JpaRepository<AuthSocialAccount, Long> {

    Optional<AuthSocialAccount> findByOauthProviderAndProviderAccountId(
            OAuthProvider oauthProvider, String providerAccountId);

    boolean existsByOauthProviderAndProviderAccountId(
            OAuthProvider oauthProvider, String providerAccountId);
}
```

---

## 2. MemberSignupEvent.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.domain.event;

import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import lombok.Getter;
import lombok.RequiredArgsConstructor;

@Getter
@RequiredArgsConstructor
public class MemberSignupEvent {

    private final Long memberId;
    private final String email;
    private final String nickname;
    private final OAuthProvider provider;

    public static MemberSignupEvent of(Long memberId, String email, 
                                        String nickname, OAuthProvider provider) {
        return new MemberSignupEvent(memberId, email, nickname, provider);
    }
}
```

---

## 3. MemberSignupEventListener.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.app.listener;

import com.modeunsa.boundedcontext.auth.domain.event.MemberSignupEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Slf4j
@Component
@RequiredArgsConstructor
public class MemberSignupEventListener {

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleMemberSignup(MemberSignupEvent event) {
        log.info("회원가입 이벤트 수신 - memberId: {}, email: {}, provider: {}",
                event.getMemberId(), event.getEmail(), event.getProvider());

        // TODO: 후속 처리 구현
        // - 환영 메시지 발송
        // - 신규 가입 포인트 지급
        // - 가입 통계 기록
        // - 알림 서비스 연동
    }
}
```

---

## 4. OAuthLoginUseCase.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.app.usecase;

import com.modeunsa.boundedcontext.auth.domain.entity.AuthSocialAccount;
import com.modeunsa.boundedcontext.auth.domain.event.MemberSignupEvent;
import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import com.modeunsa.boundedcontext.auth.out.client.OAuthClient;
import com.modeunsa.boundedcontext.auth.out.client.OAuthClientFactory;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthTokenResponse;
import com.modeunsa.boundedcontext.auth.out.client.dto.OAuthUserInfo;
import com.modeunsa.boundedcontext.auth.out.repository.AuthSocialAccountRepository;
import com.modeunsa.boundedcontext.member.domain.entity.Member;
import com.modeunsa.boundedcontext.member.domain.types.MemberRole;
import com.modeunsa.boundedcontext.member.out.repository.MemberRepository;
import com.modeunsa.shared.auth.dto.TokenResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class OAuthLoginUseCase {

    private final OAuthClientFactory oauthClientFactory;
    private final AuthSocialAccountRepository socialAccountRepository;
    private final MemberRepository memberRepository;
    private final AuthTokenIssueUseCase authTokenIssueUseCase;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public TokenResponse execute(OAuthProvider provider, String code, String redirectUri) {
        // 1. OAuth 토큰 교환
        OAuthClient oauthClient = oauthClientFactory.getClient(provider);
        OAuthTokenResponse tokenResponse = oauthClient.getToken(code, redirectUri);

        // 2. 사용자 정보 조회
        OAuthUserInfo userInfo = oauthClient.getUserInfo(tokenResponse.getAccessToken());
        log.info("OAuth 사용자 정보 조회 완료 - provider: {}, providerId: {}", 
                provider, userInfo.getProviderId());

        // 3. 소셜 계정 조회 또는 신규 가입
        AuthSocialAccount socialAccount = socialAccountRepository
                .findByOauthProviderAndProviderAccountId(provider, userInfo.getProviderId())
                .orElseGet(() -> registerNewMember(userInfo));

        Member member = socialAccount.getMember();

        // 4. JWT 토큰 발급
        return authTokenIssueUseCase.execute(member.getId(), member.getRole());
    }

    private AuthSocialAccount registerNewMember(OAuthUserInfo userInfo) {
        log.info("신규 회원 가입 - provider: {}, email: {}", 
                userInfo.getProvider(), userInfo.getEmail());

        // 1. Member 생성
        Member member = Member.builder()
                .email(userInfo.getEmail())
                .nickname(userInfo.getNickname())
                .profileImageUrl(userInfo.getProfileImageUrl())
                .role(MemberRole.MEMBER)
                .build();

        memberRepository.save(member);

        // 2. 소셜 계정 연동
        AuthSocialAccount socialAccount = AuthSocialAccount.builder()
                .member(member)
                .oauthProvider(userInfo.getProvider())
                .providerAccountId(userInfo.getProviderId())
                .build();

        socialAccountRepository.save(socialAccount);

        // 3. 회원가입 이벤트 발행
        eventPublisher.publishEvent(MemberSignupEvent.of(
                member.getId(),
                member.getEmail(),
                member.getNickname(),
                userInfo.getProvider()
        ));

        return socialAccount;
    }
}
```

---

## 5. AuthFacade.java (수정)

```java
package com.modeunsa.boundedcontext.auth.app.facade;

import com.modeunsa.boundedcontext.auth.app.usecase.AuthLogoutUseCase;
import com.modeunsa.boundedcontext.auth.app.usecase.AuthTokenIssueUseCase;
import com.modeunsa.boundedcontext.auth.app.usecase.AuthTokenReissueUseCase;
import com.modeunsa.boundedcontext.auth.app.usecase.OAuthLoginUseCase;
import com.modeunsa.boundedcontext.auth.app.usecase.OAuthUrlUseCase;
import com.modeunsa.boundedcontext.auth.domain.types.OAuthProvider;
import com.modeunsa.boundedcontext.member.domain.types.MemberRole;
import com.modeunsa.shared.auth.dto.TokenResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class AuthFacade {

    private final OAuthUrlUseCase oauthUrlUseCase;
    private final OAuthLoginUseCase oauthLoginUseCase;
    private final AuthTokenIssueUseCase authTokenIssueUseCase;
    private final AuthTokenReissueUseCase authTokenReissueUseCase;
    private final AuthLogoutUseCase authLogoutUseCase;

    /** OAuth2 로그인 URL 생성 */
    public String getOAuthLoginUrl(OAuthProvider provider, String redirectUri) {
        return oauthUrlUseCase.generateOAuthUrl(provider, redirectUri);
    }

    /** OAuth 로그인/회원가입 처리 */
    public TokenResponse oauthLogin(OAuthProvider provider, String code, String redirectUri) {
        return oauthLoginUseCase.execute(provider, code, redirectUri);
    }

    /** 로그인 성공 후 토큰 발급 (테스트용) */
    public TokenResponse login(Long memberId, MemberRole role) {
        return authTokenIssueUseCase.execute(memberId, role);
    }

    /** 토큰 재발급 */
    public TokenResponse reissueToken(String refreshToken) {
        return authTokenReissueUseCase.execute(refreshToken);
    }

    /** 로그아웃 */
    public void logout(String accessToken) {
        authLogoutUseCase.execute(accessToken);
    }
}
```

---

## 6. ApiV1AuthController.java (login 메서드 수정)

```java
@Operation(summary = "소셜 로그인", description = "소셜 로그인 인증 코드를 사용하여 로그인 및 토큰을 발급합니다.")
@PostMapping("/login/{provider}")
public ResponseEntity<ApiResponse> login(
        @Parameter(description = "OAuth 제공자", example = "kakao") @PathVariable String provider,
        @Parameter(description = "인증 코드", required = true) @RequestParam String code,
        @Parameter(description = "리다이렉트 URI") @RequestParam(required = false) String redirectUri) {

    OAuthProvider oauthProvider = findProvider(provider);
    TokenResponse tokenResponse = authFacade.oauthLogin(oauthProvider, code, redirectUri);

    return ApiResponse.onSuccess(SuccessStatus.AUTH_LOGIN_SUCCESS, tokenResponse);
}
```

---

## 7. @Async 활성화 (Application 클래스)

`MemberSignupEventListener`가 `@Async`를 사용하므로 메인 클래스에 추가:

```java
@EnableAsync  // 추가
@SpringBootApplication
public class ModeUnsaApplication {
    public static void main(String[] args) {
        SpringApplication.run(ModeUnsaApplication.class, args);
    }
}
```

---

## 파일 구조 정리

```
auth/
├── app/
│   ├── facade/
│   │   └── AuthFacade.java (수정)
│   ├── listener/
│   │   └── MemberSignupEventListener.java (새 파일)
│   └── usecase/
│       ├── OAuthLoginUseCase.java (새 파일)
│       └── ...
├── domain/
│   ├── event/
│   │   └── MemberSignupEvent.java (새 파일)
│   └── ...
├── out/
│   ├── client/
│   │   ├── dto/
│   │   │   ├── OAuthTokenResponse.java (새 파일)
│   │   │   └── OAuthUserInfo.java (새 파일)
│   │   └── ...
│   └── repository/
│       ├── AuthSocialAccountRepository.java (새 파일)
│       └── ...
```

---

## 확인 필요

`OAuthLoginUseCase`에서 `Member` 엔티티와 `MemberRepository`를 사용하는데, 현재 Member 엔티티 구조를 확인해야 해.

**Member.java**와 **MemberRepository.java** 공유해줄 수 있어? 필드명 맞춰서 수정해줄게!


잘했어! 🎉

---

## 현재 완료된 것

| 항목 | 상태 |
|------|------|
| OAuth 로그인 URL 생성 | ✅ |
| OAuth 토큰 교환 | ✅ |
| OAuth 사용자 정보 조회 | ✅ |
| 회원가입/로그인 처리 | ✅ |
| 회원가입 이벤트 발행 | ✅ |
| JWT 토큰 발급/재발급 | ✅ |
| 로그아웃 + 블랙리스트 | ✅ |

---

## 남은 선택지

**1. State 검증 로직 추가 (보안 강화)**
```
OAuth 콜백 시 state 파라미터 검증 → CSRF 공격 방지
(TODO로 남겨둔 부분)
```

**2. 실제 테스트**
```
카카오/네이버 실제 로그인 테스트
```

**3. 커밋 먼저**
```
지금까지 작업한 것 커밋/PR 후 다음 기능으로
```

---

어떻게 할래?

- State 검증까지 마무리하고 커밋할지
- 일단 커밋하고 테스트해볼지
- 바로 테스트해볼지
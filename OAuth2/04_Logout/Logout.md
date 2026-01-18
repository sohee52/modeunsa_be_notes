코드 구조 파악 완료! UseCase 패턴을 따라서 구현할게요.

---

## 구현할 파일 목록

| 파일 | 설명 |
|------|------|
| `AuthAccessTokenBlacklist.java` | 블랙리스트 Redis 엔티티 |
| `AuthAccessTokenBlacklistRepository.java` | Redis Repository |
| `AuthLogoutUseCase.java` | 로그아웃 비즈니스 로직 |
| `JwtTokenProvider.java` | 남은 만료시간 계산 메서드 추가 |
| `JwtAuthenticationFilter.java` | 블랙리스트 체크 로직 추가 |
| `AuthFacade.java` | logout 메서드 추가 |
| `ApiV1AuthController.java` | logout API 추가 |

---

## 1. AuthAccessTokenBlacklist.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.domain.entity;

import java.util.concurrent.TimeUnit;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.RedisHash;
import org.springframework.data.redis.core.TimeToLive;

@Getter
@RedisHash(value = "blacklist")
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class AuthAccessTokenBlacklist {

    @Id
    private String accessToken;

    private Long memberId;

    @TimeToLive(unit = TimeUnit.MILLISECONDS)
    private Long expiration;

    public static AuthAccessTokenBlacklist of(String accessToken, Long memberId, Long remainingExpiration) {
        return AuthAccessTokenBlacklist.builder()
                .accessToken(accessToken)
                .memberId(memberId)
                .expiration(remainingExpiration)
                .build();
    }
}
```

---

## 2. AuthAccessTokenBlacklistRepository.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.out.repository;

import com.modeunsa.boundedcontext.auth.domain.entity.AuthAccessTokenBlacklist;
import org.springframework.data.repository.CrudRepository;

public interface AuthAccessTokenBlacklistRepository extends CrudRepository<AuthAccessTokenBlacklist, String> {
}
```

---

## 3. AuthLogoutUseCase.java (새 파일)

```java
package com.modeunsa.boundedcontext.auth.app.usecase;

import com.modeunsa.boundedcontext.auth.domain.entity.AuthAccessTokenBlacklist;
import com.modeunsa.boundedcontext.auth.out.repository.AuthAccessTokenBlacklistRepository;
import com.modeunsa.boundedcontext.auth.out.repository.AuthRefreshTokenRepository;
import com.modeunsa.global.security.jwt.JwtTokenProvider;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuthLogoutUseCase {

    private final JwtTokenProvider jwtTokenProvider;
    private final AuthRefreshTokenRepository refreshTokenRepository;
    private final AuthAccessTokenBlacklistRepository blacklistRepository;

    public void execute(String accessToken) {
        // 1. Access Token에서 정보 추출
        Long memberId = jwtTokenProvider.getMemberIdFromToken(accessToken);
        long remainingExpiration = jwtTokenProvider.getRemainingExpiration(accessToken);

        // 2. Refresh Token 삭제
        refreshTokenRepository.deleteById(memberId);
        log.info("Refresh Token 삭제 완료 - memberId: {}", memberId);

        // 3. Access Token 블랙리스트 등록 (남은 만료시간만큼 TTL 설정)
        if (remainingExpiration > 0) {
            AuthAccessTokenBlacklist blacklist = AuthAccessTokenBlacklist.of(
                    accessToken,
                    memberId,
                    remainingExpiration
            );
            blacklistRepository.save(blacklist);
            log.info("Access Token 블랙리스트 등록 완료 - memberId: {}, TTL: {}ms", memberId, remainingExpiration);
        }
    }
}
```

---

## 4. JwtTokenProvider.java (메서드 추가)

기존 코드 하단에 추가:

```java
/** 토큰의 남은 만료시간 계산 (밀리초) */
public long getRemainingExpiration(String token) {
    Claims claims = parseClaims(token);
    Date expiration = claims.getExpiration();
    long remainingTime = expiration.getTime() - System.currentTimeMillis();
    return Math.max(0, remainingTime);
}
```

---

## 5. JwtAuthenticationFilter.java (블랙리스트 체크 추가)

```java
package com.modeunsa.global.security.jwt;

import com.modeunsa.boundedcontext.auth.out.repository.AuthAccessTokenBlacklistRepository;
import com.modeunsa.boundedcontext.member.domain.types.MemberRole;
import com.modeunsa.global.exception.GeneralException;
import com.modeunsa.global.status.ErrorStatus;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.List;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

@Slf4j
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final AuthAccessTokenBlacklistRepository blacklistRepository;

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";
    private static final String ROLE_PREFIX = "ROLE_";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String token = resolveToken(request);

        if (StringUtils.hasText(token)) {
            try {
                jwtTokenProvider.validateTokenOrThrow(token);

                if (!jwtTokenProvider.isAccessToken(token)) {
                    throw new GeneralException(ErrorStatus.AUTH_INVALID_ACCESS_TOKEN);
                }

                // 블랙리스트 체크
                if (blacklistRepository.existsById(token)) {
                    throw new GeneralException(ErrorStatus.AUTH_BLACKLISTED_TOKEN);
                }

                Long memberId = jwtTokenProvider.getMemberIdFromToken(token);
                MemberRole role = jwtTokenProvider.getRoleFromToken(token);

                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(
                                memberId, null, List.of(new SimpleGrantedAuthority(ROLE_PREFIX + role.name())));

                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);

                log.debug("인증 정보 저장 완료 - memberId: {}, role: {}", memberId, role);
            } catch (GeneralException e) {
                request.setAttribute("exception", e);
                log.warn("토큰 검증 실패: {}", e.getMessage());
            }
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);

        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith(BEARER_PREFIX)) {
            return bearerToken.substring(BEARER_PREFIX.length());
        }

        return null;
    }
}
```

---

## 6. AuthFacade.java (logout 추가)

```java
package com.modeunsa.boundedcontext.auth.app.facade;

import com.modeunsa.boundedcontext.auth.app.usecase.AuthLogoutUseCase;
import com.modeunsa.boundedcontext.auth.app.usecase.AuthTokenIssueUseCase;
import com.modeunsa.boundedcontext.auth.app.usecase.AuthTokenReissueUseCase;
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
    private final AuthTokenIssueUseCase authTokenIssueUseCase;
    private final AuthTokenReissueUseCase authTokenReissueUseCase;
    private final AuthLogoutUseCase authLogoutUseCase;

    public String getOAuthLoginUrl(OAuthProvider provider, String redirectUri) {
        return oauthUrlUseCase.generateOAuthUrl(provider, redirectUri);
    }

    public TokenResponse login(Long memberId, MemberRole role) {
        return authTokenIssueUseCase.execute(memberId, role);
    }

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

## 7. ApiV1AuthController.java (logout API 추가)

기존 메서드들 아래에 추가:

```java
@Operation(summary = "로그아웃", description = "Access Token을 블랙리스트에 등록하고 Refresh Token을 삭제합니다.")
@PostMapping("/logout")
public ResponseEntity<ApiResponse> logout(
        @Parameter(description = "Access Token", required = true) 
        @RequestHeader("Authorization") String authorizationHeader) {
    
    String accessToken = authorizationHeader.replace("Bearer ", "");
    authFacade.logout(accessToken);
    
    return ApiResponse.onSuccess(SuccessStatus.AUTH_LOGOUT_SUCCESS);
}
```

---

## 8. ErrorStatus에 추가

```java
// Auth 관련 에러에 추가
AUTH_BLACKLISTED_TOKEN(HttpStatus.UNAUTHORIZED, "AUTH_401_007", "로그아웃된 토큰입니다."),
```

## 9. SuccessStatus에 추가

```java
AUTH_LOGOUT_SUCCESS(HttpStatus.OK, "AUTH_200_002", "로그아웃에 성공했습니다."),
```

---

| 파일 | 작업 |
|------|------|
| `AuthAccessTokenBlacklist.java` | 새로 생성 |
| `AuthAccessTokenBlacklistRepository.java` | 새로 생성 |
| `AuthLogoutUseCase.java` | 새로 생성 |
| `JwtTokenProvider.java` | `getRemainingExpiration()` 메서드 추가 |
| `JwtAuthenticationFilter.java` | `blacklistRepository` 주입 + 블랙리스트 체크 로직 추가 |
| `AuthFacade.java` | `AuthLogoutUseCase` 주입 + `logout()` 메서드 추가 |
| `ApiV1AuthController.java` | `logout` API 추가 |
| `ErrorStatus.java` | `AUTH_BLACKLISTED_TOKEN` 추가 |

```
1. 로그인 → Access Token, Refresh Token 발급
2. 로그아웃 호출 (POST /api/v1/auths/logout + Authorization: Bearer {accessToken})
   └─ Refresh Token 삭제 (Redis)
   └─ Access Token 블랙리스트 등록 (Redis, TTL = 남은 만료시간)
3. 같은 Access Token으로 API 호출 → 401 AUTH_401_007 응답
```

당연하지! Swagger로 테스트해도 충분해.

---

## Swagger 테스트 순서

**1. 로그인 → 토큰 발급**
```
POST /api/v1/auths/login/{provider}
→ accessToken, refreshToken 복사
```

**2. 로그아웃**
```
POST /api/v1/auths/logout
Authorization: Bearer {accessToken}
→ 200 AUTH_200_002 응답 확인
```

**3. 같은 토큰으로 인증 필요한 API 호출**
```
Authorization: Bearer {동일한 accessToken}
→ 401 AUTH_401_007 (로그아웃된 토큰입니다) 응답 확인
```

**4. (선택) Redis 확인**
```bash
redis-cli
> KEYS blacklist:*
> TTL blacklist:{accessToken}
```
# JWT 구현

## 전체 아키텍처 개요

```
[클라이언트] → [Controller] → [Facade] → [UseCase] → [Repository/Provider]
                                              ↓
                                    [Redis - RefreshToken 저장]
```

---

## 1. 토큰 발급 흐름 (로그인)

**ApiV1AuthController → AuthFacade → AuthTokenUseCase**

```
POST /api/v1/auths/login/{provider}
     ↓
AuthFacade.login(memberId, role)
     ↓
AuthTokenUseCase.issueTokens()
     ↓
1) JwtTokenProvider로 Access/Refresh Token 생성
2) Redis에 RefreshToken 저장 (TTL 7일)
3) TokenResponse 반환
```

**JwtTokenProvider.createAccessToken()**에서 JWT 구조:
- subject: memberId
- claim: role
- expiration: 30분 (Access) / 7일 (Refresh)
- 서명: HMAC-SHA256

---

## 2. 인증 필터 흐름 (매 요청마다)

**JwtAuthenticationFilter** (OncePerRequestFilter)

```
모든 HTTP 요청
     ↓
Authorization 헤더에서 "Bearer {token}" 추출
     ↓
JwtTokenProvider.validateToken() - 서명/만료 검증
     ↓
memberId, role 추출
     ↓
SecurityContext에 Authentication 저장
     ↓
Controller에서 인증된 사용자로 처리
```

---

## 3. 토큰 재발급 흐름 (Refresh)

**POST /api/v1/auths/reissue**

```
RefreshToken 헤더로 전달
     ↓
AuthTokenUseCase.reissueTokens()
     ↓
1) JWT 자체 유효성 검증 (서명, 만료)
2) Redis에서 저장된 토큰 조회
3) 요청 토큰 == 저장 토큰 비교
4) 기존 토큰 삭제 (Rotation)
5) 새 토큰 세트 발급
```

**Refresh Token Rotation** 구현되어 있어서 재발급 시 기존 토큰은 무효화돼.

---

## 4. Redis 저장 구조

**AuthRefreshToken** 엔티티:
```java
@RedisHash(value = "refresh")
- Key: refresh:{memberId}
- Value: refreshToken 문자열
- TTL: 7일 (자동 만료)
```

memberId를 키로 사용해서 **한 사용자당 하나의 RefreshToken**만 유효함.

---

## 5. Security 설정

**SecurityConfig**:
- CSRF 비활성화 (Stateless)
- 세션 사용 안 함 (STATELESS)
- permitUrls로 화이트리스트 관리 (`/swagger-ui/**`, `/api/v1/auths/oauth/**` 등)
- JwtAuthenticationFilter를 UsernamePasswordAuthenticationFilter 앞에 추가

**JwtAuthenticationEntryPoint**: 인증 실패 시 JSON 에러 응답
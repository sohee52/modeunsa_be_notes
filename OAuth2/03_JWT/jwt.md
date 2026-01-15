# JWT (JSON Web Token)

**JWT는 인증·인가 정보를 JSON 형태로 안전하게 전달하기 위한 토큰 표준**이야.
주로 **로그인 인증**과 **API 접근 제어**에 사용돼.

---

## JWT의 핵심 개념

### 1. 왜 쓰는가?

* 서버가 **세션을 저장하지 않아도(stateless)** 인증 가능
* 모바일 / 웹 / MSA 환경에서 공통으로 사용 가능
* 토큰 하나로 **사용자 신원 + 권한**을 전달 가능

---

### 2. JWT의 구조 (중요)

JWT는 점(`.`)으로 구분된 **3부분**으로 구성돼.

```
Header.Payload.Signature
```

#### (1) Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

* 어떤 알고리즘으로 서명했는지 정보

#### (2) Payload

```json
{
  "sub": "123",
  "role": "USER",
  "exp": 1700000000
}
```

* **사용자 정보(Claim)**
* ⚠️ 암호화 아님 → **누구나 디코딩 가능**

#### (3) Signature

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secretKey
)
```

* 위조 방지용 서명
* 서버만 secretKey를 알고 있음

---

## JWT 인증 흐름 (로그인 기준)

1. 사용자가 로그인 요청
2. 서버가 로그인 성공 시 **JWT 발급**
3. 클라이언트가 JWT 저장

   * 보통 `Authorization: Bearer <JWT>`
4. 이후 요청마다 JWT 전송
5. 서버는 JWT 검증 후 사용자 식별

---

## JWT의 장단점

### ✅ 장점

* 서버 상태 저장 필요 없음 (Stateless)
* 확장성 좋음 (MSA, API 서버)
* 다양한 플랫폼에서 사용 가능

### ❌ 단점

* 토큰 탈취 시 위험
* 서버에서 **강제 로그아웃 처리 어려움**
* Payload가 노출됨 (민감 정보 ❌)

---

## 한 줄 정리

> **JWT는 로그인 후 사용자의 신원과 권한을 서버 상태 없이 증명하기 위한 토큰 방식이다.**
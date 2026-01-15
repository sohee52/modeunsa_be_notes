# JWT_SECRET

JWT_SECRET은 직접 생성해야 해. 256비트(32바이트) 이상의 랜덤 문자열이면 돼.

---

## 생성 방법

**1. OpenSSL (Git Bash)**
```bash
openssl rand -base64 32
```

**2. 온라인 생성기**
- https://generate-secret.vercel.app/32

**3. Java 코드로 생성**
```java
import java.security.SecureRandom;
import java.util.Base64;

SecureRandom random = new SecureRandom();
byte[] bytes = new byte[32];
random.nextBytes(bytes);
String secret = Base64.getEncoder().encodeToString(bytes);
System.out.println(secret);
```

---

## 적용 방법

**로컬 개발**: 환경변수 설정
```bash
# .env 파일 또는 IDE 환경변수
JWT_SECRET=K7gNU3sdo+OL0wNhqoVWhr3g6s1xYv72ol/pe/Unols=
```

**운영 환경**: 서버 환경변수나 Secret Manager 사용
```bash
export JWT_SECRET=K7gNU3sdo+OL0wNhqoVWhr3g6s1xYv72ol/pe/Unols=
```

---

## 주의사항

- **절대 Git에 커밋하면 안 돼** (`.gitignore`에 `.env` 추가)
- 운영/개발 환경별로 다른 키 사용
- 키가 노출되면 즉시 변경해야 함

현재 yml에 있는 기본값(`your-256-bit-secret-key-here...`)은 개발용이니까 운영 배포 전에 꼭 변경해야 해.
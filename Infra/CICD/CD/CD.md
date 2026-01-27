| 단계 | 상태 |
| --- | --- |
| EC2 생성 및 세팅 | ✅ |
| Docker 설치 | ✅ |
| Dockerfile 작성 | ✅ |
| GitHub Secrets 설정 | ✅ |
| CD 파이프라인 작성 | ✅ |
| 배포 테스트 | ✅ |
| 환경변수 설정 (Redis) | ✅ |
| 무중단 배포 구성 | ✅ |
| 문서화 | ✅ |

## EC2 생성, Elastic IP 생성

- AWS

## SSH 접속 → Docker 설치

1. 로컬 PowerShell 열기
2. pem 파일이 있는 곳으로 이동
3. SSH 접속
    - SSH(Secure Shell) = 암호화된 원격 로그인 + 원격 명령 실행 프로토콜
    - 주 용도는 서버 원격 접속, 파일 전송, 포트 터널링이다.
    
    ```bash
    ssh -i modeunsa_pem.pem ec2-user@[Elastic IP]
    # 예시: ssh -i modeunsa_pem.pem ec2-user@52.79.155.221
    ```
    
4. Docker 설치
    
    ```bash
    # Docker 설치
    sudo yum update -y
    sudo yum install docker -y
    
    # Docker 시작 및 자동 실행 설정
    sudo systemctl start docker
    sudo systemctl enable docker
    
    # ec2-user가 docker 명령어 쓸 수 있게 권한 추가
    sudo usermod -aG docker ec2-user
    ```
    
    여기까지 다 하면 **한 번 나갔다가 다시 접속**해야 권한이 적용돼:
    
    ```bash
    exit
    ```
    
    그다음 다시 SSH 접속:
    
    ```bash
    ssh -i modeunsa_pem.pem ec2-user@52.79.155.221
    ```
    
    재접속 후 확인:
    
    ```bash
    docker --version
    ```
    
    버전 나오면 성공
    
    이제 **Docker Compose 설치**
    
    - Docker Compose는 여러 개의 Docker 컨테이너를 한 번에 정의하고 실행하기 위한 도구
    - `docker-compose.yml` 파일 하나로 **컨테이너 구성 + 실행 방법**을 선언
    
    ```bash
    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```
    
    설치 확인:
    
    ```bash
    docker-compose --version
    ```
    
    버전 나오면 완료!
    

## EC2에 8080 포트 열기

이제 **8080 포트 열기**가 필요해. 아까 보안 그룹에서 22, 80, 443만 열었거든.

보안 그룹에 8080 포트 추가

AWS 콘솔에서:

1. **EC2 → 인스턴스 → team01 선택**
2. 아래 **보안 탭** 클릭
3. 보안 그룹 링크 클릭 (launch-wizard-7 같은 이름)
4. **인바운드 규칙 편집** 클릭
5. **규칙 추가**:
    - 유형: 사용자 지정 TCP
    - 포트 범위: `8080`
    - 소스: `0.0.0.0/0`
6. **규칙 저장**

## Dockerfile 작성

프로젝트 루트에 `Dockerfile` 파일 만들고:

```docker
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "app.jar"]
```

- `build/libs/` 밑에서 .jar 파일 확인 가능

그다음 `.dockerignore` 파일도 만들어:

```
.git
.gradle
build
!build/libs
*.md
```

(선택) 순수 컴파일 클래스 생성하는 plain.jar 안 생기게 하려면?

`build.gradle` 에 추가

```bash
jar {
    enabled = false
}
```

## GitHub Secrets 등록

GitHub 레포지토리 → **Settings → Secrets and variables → Actions → New repository secret**

하나씩 추가해:

| Name | Value |
| --- | --- |
| `DOCKER_USERNAME` | Docker Hub 아이디 |
| `DOCKER_PASSWORD` | Docker Hub 비밀번호 |
| `EC2_HOST` | `52.79.155.221` |
| `EC2_KEY` | pem 파일 내용 전체 |

## EC2_KEY 등록 방법

pem 파일을 메모장으로 열어서 **전체 내용** 복사:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
(중간 내용 전부)
...
-----END RSA PRIVATE KEY-----

```

이거 통째로 복사해서 EC2_KEY 값에 붙여넣기.

---

4개 다 등록하면 알려줘!

좋아! 이제 **CD 파이프라인 작성**이야.

## deploy.yml 생성

`.github/workflows/deploy.yml` 파일 만들어:

```yaml
name: CD Pipeline

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew build -x test

      - name: Docker login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_KEY }}
          script: |
            docker pull ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest
            docker stop modeunsa || true
            docker rm modeunsa || true
            docker run -d --name modeunsa -p 8080:8080 ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest
            docker image prune -f

```

파일 생성하면 알려줘!

이제 **테스트 배포**해볼 차례야.

main 브랜치에 push하면 자동으로 CD가 실행돼.

## 테스트 방법

1. 지금까지 작업한 거 commit & push:

```bash
git add .
git commit -m "feat: add Dockerfile and CD pipeline"
git push origin main

```

1. GitHub 레포지토리 → **Actions** 탭에서 실행 확인

---

그 전에 한 가지 확인할 게 있어. `application-prod.yml` 파일 있어? 없으면 만들어야 해.

cf. main에서 테스트하기 어렵다면?

**별도 테스트 브랜치 만들기**

- develop에서 `feature/cd-test` 브랜치 생성
- 거기서 테스트하고 확인되면 develop에 머지

**deploy.yml 수정**

```yaml
on:
  push:
    branches: [main, feature/26]
```

이렇게 `feature/26` 추가하면 돼.

테스트 성공하면 나중에 `feature/26` 지우고 `main`만 남기면 돼.

🎉 축하해!

이제 **서버 접속 테스트** 해보자.

브라우저에서:

```
http://52.79.155.221:8080
```

열리면 배포 성공이야! 결과 알려줘!

좋아! 다음은 **환경변수 설정**이야.

지금 Redis가 없어서 Redis 관련 기능은 안 될 거야. EC2에 Redis 컨테이너 띄우자.

## EC2에서 Redis 실행

SSH 접속해서:

```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

그다음 앱 컨테이너가 Redis에 접근할 수 있게 **네트워크 연결**이 필요해.

```bash
# 기존 컨테이너 중지 및 삭제
docker stop modeunsa
docker rm modeunsa

# Redis와 같은 네트워크로 앱 실행
docker network create modeunsa-net
docker network connect modeunsa-net redis
docker run -d --name modeunsa --network modeunsa-net -p 8080:8080 -e REDIS_HOST=redis sohee52/modeunsa:latest
```

## 남은 작업

| 단계 | 상태 |
| --- | --- |
| 무중단 배포 구성 | ⬜ |
| 문서화 | ⬜ |

좋아! EC2 1대로 무중단 배포하려면 **Blue-Green 방식**을 쓸 거야.

## 구조

```
Nginx (80) → App-Blue (8081) 또는 App-Green (8082)
```

새 버전 배포 시:

1. Green에 새 버전 띄움
2. 헬스체크 통과하면 Nginx가 Green으로 전환
3. Blue 종료

사전 작업

## AWS 콘솔에서 보안 그룹 수정

1. EC2 → 인스턴스 → team01 선택
2. 보안 탭 → 보안 그룹 클릭
3. 인바운드 규칙 편집
4. 규칙 추가:
    - 유형: 사용자 지정 TCP
    - 포트 범위: `8081`
    - 소스: `0.0.0.0/0`
5. 규칙 추가:
    - 유형: 사용자 지정 TCP
    - 포트 범위: `8082`
    - 소스: `0.0.0.0/0`
6. 규칙 저장

## 1단계: docker-compose.yml 생성

프로젝트 루트에 `docker-compose.yml` 파일 만들어:

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app-blue
    networks:
      - modeunsa-net

  app-blue:
    image: ${DOCKER_IMAGE}
    container_name: app-blue
    ports:
      - "8081:8080"
    environment:
      - REDIS_HOST=redis
    networks:
      - modeunsa-net

  app-green:
    image: ${DOCKER_IMAGE}
    container_name: app-green
    ports:
      - "8082:8080"
    environment:
      - REDIS_HOST=redis
    networks:
      - modeunsa-net
    profiles:
      - green

  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - modeunsa-net

networks:
  modeunsa-net:
    driver: bridge

```

무중단 배포 시 Nginx가 blue ↔ green 전환해서 끊김 없이 배포!

좋아! 프로젝트 루트에 `nginx.conf` 파일 만들어:

```
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app-blue:8080;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            access_log off;
            return 200 "OK";
            add_header Content-Type text/plain;
        }
    }
}

```

만들면 알려줘!

좋아! 이제 **배포 스크립트** 만들자.

프로젝트 루트에 `deploy.sh` 파일 만들어:

```bash
#!/bin/bash

DOCKER_IMAGE=$1
COMPOSE_FILE="/home/ec2-user/app/docker-compose.yml"
NGINX_CONF="/home/ec2-user/app/nginx.conf"

# 현재 실행 중인 앱 확인
CURRENT=$(docker ps --format '{{.Names}}' | grep -E 'app-(blue|green)' | head -1)

if [ "$CURRENT" == "app-blue" ]; then
    NEW="green"
    OLD="blue"
    NEW_PORT=8082
else
    NEW="blue"
    OLD="green"
    NEW_PORT=8081
fi

echo "현재: $OLD → 새로운: $NEW"

# 새 컨테이너 실행
DOCKER_IMAGE=$DOCKER_IMAGE docker-compose -f $COMPOSE_FILE --profile $NEW up -d app-$NEW

# 헬스체크 (최대 30초 대기)
echo "헬스체크 중..."
for i in {1..30}; do
    if curl -s http://localhost:$NEW_PORT/swagger-ui/index.html > /dev/null 2>&1; then
        echo "헬스체크 성공!"
        break
    fi
    if [ $i -eq 30 ]; then
        echo "헬스체크 실패. 롤백."
        docker stop app-$NEW || true
        docker rm app-$NEW || true
        exit 1
    fi
    sleep 1
done

# Nginx 설정 변경
sed -i "s/app-$OLD/app-$NEW/g" $NGINX_CONF
docker exec nginx nginx -s reload

echo "Nginx 전환 완료: app-$NEW"

# 이전 컨테이너 종료
docker stop app-$OLD || true
docker rm app-$OLD || true

# 이미지 정리
docker image prune -f

echo "배포 완료!"
```

만들면 알려줘!

좋아! 이제 **deploy.yml 수정**이야.

`.github/workflows/deploy.yml` 파일을 이렇게 수정해:

```yaml
name: CD Pipeline

on:
  push:
    branches: [main, feature/26]

permissions:
  contents: read

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew build -x test

      - name: Docker login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest

      - name: Copy files to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_KEY }}
          source: "docker-compose.yml,nginx.conf,deploy.sh"
          target: "/home/ec2-user/app"

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_KEY }}
          script: |
            cd /home/ec2-user/app
            chmod +x deploy.sh
            
            # 기존 단일 컨테이너 정리
            docker stop modeunsa || true
            docker rm modeunsa || true
            
            # 네트워크 생성 (없으면)
            docker network create modeunsa-net || true
            
            # Redis 실행 (없으면)
            docker ps | grep redis || docker run -d --name redis --network modeunsa-net -p 6379:6379 redis:7-alpine
            docker network connect modeunsa-net redis || true
            
            # 배포 실행
            ./deploy.sh ${{ secrets.DOCKER_USERNAME }}/modeunsa:latest
```

새 deploy.yml이 하는 일

1. `docker-compose.yml`, `nginx.conf`, `deploy.sh`를 EC2로 복사
2. `deploy.sh` 실행해서 Blue-Green 전환

좋아! 이제 **Nginx 띄우자**.

EC2에서:

```bash
cd /home/ec2-user/app
docker run -d --name nginx --network modeunsa-net -p 80:80 -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx:alpine
```

네트워크 연결해주자:

```bash
docker network connect modeunsa-net app-blue
docker rm -f nginx
docker run -d --name nginx --network modeunsa-net -p 80:80 -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf nginx:alpine
docker ps
```

그다음 확인:

```bash
docker ps
```

nginx, app-blue, redis 3개 다 뜨면 `http://52.79.155.221` 접속해봐!
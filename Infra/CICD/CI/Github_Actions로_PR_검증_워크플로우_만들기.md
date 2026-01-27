# GitHub Actions로 PR 검증 워크플로우 만들기

GitHub Actions로 PR 검증 워크플로우 만드는 방법 알려줄게!

## 기본 구조

`.github/workflows/` 디렉토리에 YAML 파일을 만들면 돼.

```yaml
name: PR Check

on:
  pull_request:
    branches: [ main, develop ]
    types: [ opened, synchronize, reopened ]

permissions:
  contents: read

jobs:
  code-style:
    name: Code Style Check
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

      - name: Spotless Check
        run: ./gradlew spotlessCheck

      - name: Checkstyle
        run: ./gradlew checkstyleMain checkstyleTest

      - name: Upload Checkstyle Report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: checkstyle-report
          path: build/reports/checkstyle/

  build:
    name: Build (without tests & style)
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

      - name: Build (without tests)
        run: ./gradlew build -x test -x checkstyleMain -x checkstyleTest -x spotlessCheck

  test:
    name: Test
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

      - name: Run tests
        run: ./gradlew test -Dspring.profiles.active=test --no-daemon

      - name: Upload Test Report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-report
          path: build/reports/tests/
```

## 트리거 설명

- `opened`: PR 처음 열릴 때
- `synchronize`: PR에 새 커밋이 푸시될 때 (네가 원하는 거!)
- `reopened`: 닫았다가 다시 열 때

## 컨벤션 체크 옵션

팀에서 어떤 컨벤션 도구 쓰는지에 따라 달라져:

**Checkstyle** (Google/Sun 스타일)
```yaml
- name: Checkstyle
  run: ./gradlew checkstyleMain checkstyleTest
```

**Spotless** (코드 포맷팅)
```yaml
- name: Spotless check
  run: ./gradlew spotlessCheck
```

**ktlint** (Kotlin인 경우)
```yaml
- name: Lint check
  run: ./gradlew ktlintCheck
```

## build.gradle에 추가할 내용

Checkstyle 쓴다면:
```groovy
plugins {
    id 'checkstyle'
}

checkstyle {
    toolVersion = '10.12.0'
    configFile = file("${rootDir}/config/checkstyle/checkstyle.xml")
}
```

Spotless 쓴다면:
```groovy
plugins {
    id 'com.diffplug.spotless' version '6.22.0'
}

spotless {
    java {
        googleJavaFormat()
        // 또는 팀 설정
    }
}
```

## PR 머지 조건 설정

GitHub 레포 Settings → Branches → Branch protection rules에서:
1. `main` 브랜치 선택
2. "Require status checks to pass before merging" 체크
3. 위에서 만든 job 이름(`build-and-test`, `code-style`) 선택

이렇게 하면 체크 통과 안 하면 머지 버튼이 비활성화돼!
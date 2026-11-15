# AWS Elastic Beanstalk + RDS(MySQL) 배포 가이드 (무료 계정 / Spring Boot JAR)

Spring Boot 애플리케이션(JAR)을 **AWS 무료 플랜**으로 **Elastic Beanstalk(EB)** 에 올리고,
**분리형(decoupled) RDS(MySQL)** 에 연결하는 전체 과정을 정리한 문서입니다.

> 이 문서는 로컬 개발은 H2, 운영(EB)은 RDS(MySQL)를 쓰도록 **prod 프로파일**로 분리하는 방식을 기준으로 합니다.

---

## 0. 준비물 / 전제

- Spring Boot 프로젝트 (JAR 빌드 가능, Gradle 기준)
- AWS 계정 (무료 플랜)
- 이미 생성된 EB 웹 서버 환경 (플랫폼: Corretto / Java SE, Amazon Linux)
- 리전은 하나로 통일 (예: `ap-northeast-2` 서울). **RDS와 EB는 같은 리전·같은 VPC**여야 함

---

## 1. 애플리케이션 설정 (코드/설정 파일)

### 1-1. MySQL 드라이버 추가 — `build.gradle`

```groovy
dependencies {
    // ... 기존 의존성 ...
    runtimeOnly 'com.h2database:h2'            // 로컬 개발용
    runtimeOnly 'com.mysql:mysql-connector-j'  // RDS(MySQL) 접속용 (버전은 Spring Boot BOM이 관리)
}
```

### 1-2. 운영 전용 설정 — `src/main/resources/application-prod.yaml` (신규)

EB가 RDS를 연결하거나 환경변수로 주입하는 `RDS_*` 값을 읽습니다.

```yaml
# 운영(EB + RDS) 전용. EB 환경변수 SPRING_PROFILES_ACTIVE=prod 일 때 적용됨
spring:
  datasource:
    url: jdbc:mysql://${RDS_HOSTNAME}:${RDS_PORT}/${RDS_DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&characterEncoding=UTF-8
    username: ${RDS_USERNAME}
    password: ${RDS_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update      # 운영에서는 스키마를 지우지 않도록 update (create-drop 금지)
    show-sql: false
  sql:
    init:
      mode: never           # data.sql은 H2 전용 초기화였으므로 운영에서는 실행 안 함
  h2:
    console:
      enabled: false        # 운영에서는 H2 콘솔 비활성화
```

> 기존 `application.yaml`(H2 인메모리 설정)은 그대로 두면 로컬 개발은 계속 H2로 동작합니다.
> 운영에서는 prod 프로파일이 datasource만 덮어씁니다.

### 1-3. (Spring Security 사용 시) H2 콘솔 매처 주의 — `WebSecurityConfig`

`toH2Console()` 은 요청 시 `H2ConsoleProperties` 빈을 찾는데, 운영에서 H2 콘솔을 끄면 그 빈이 없어
**매 요청마다 500 에러**가 납니다. 경로 문자열로 바꾸세요.

```java
// 잘못됨 (운영 500 유발)
// import static ...PathRequest.toH2Console;
// web.ignoring().requestMatchers(toH2Console())

// 올바름 — 빈 조회 없는 일반 경로 매처
web.ignoring()
   .requestMatchers("/h2-console/**")
   .requestMatchers("/static/**");
```

### 1-4. (Spring Security 사용 시) 로그인 성공 후 이동 경로

브라우저가 자동으로 요청하는 `/favicon.ico` 가 "저장된 요청(SavedRequest)"으로 잡혀,
로그인 성공 후 엉뚱한 곳으로 리다이렉트되는 경우가 있습니다. `true`(alwaysUse)를 주면
**항상 지정한 경로로** 이동합니다.

```java
.formLogin(form -> form
    .loginPage("/login")
    .defaultSuccessUrl("/articles", true)   // 두 번째 인자 true 중요
)
```

---

## 2. RDS(MySQL) 인스턴스 생성

RDS 콘솔 → **데이터베이스 생성**

- 생성 방식: **표준 생성**
- 엔진: **MySQL** (Aurora ❌ — 프리 티어 없음)
- 템플릿: **프리 티어**
- 인스턴스: `db.t3.micro` / `db.t4g.micro` (프리 티어 자동)
- 자격 증명: 마스터 사용자명(예: `admin`) + **비밀번호(반드시 메모)**
- 연결:
  - VPC: **EB와 같은 VPC** (보통 Default VPC)
  - 컴퓨팅 리소스: **EC2에 연결 안 함**
  - 퍼블릭 액세스: **아니요** (EB가 같은 VPC라 접속 가능. 로컬에서 직접 붙을 거면 "예")
  - VPC 보안 그룹: **새로 생성** (예: `rds-blog-sg`)
- **추가 구성 → 초기 데이터베이스 이름: `blogdb`** ← **반드시 입력!** (없으면 `Unknown database` 에러)

생성 후 **"사용 가능"** 상태가 되면 **엔드포인트 주소**를 확인
(RDS → 해당 DB → 연결 & 보안 → 엔드포인트). 예:
`blog-db.xxxxxxxx.ap-northeast-2.rds.amazonaws.com`

---

## 3. 보안 그룹 — EB가 RDS에 접속하도록 열기 (⭐ 연결 타임아웃 1순위 원인)

> **Elastic Beanstalk은 내부적으로 EC2 인스턴스를 만들어 앱을 실행**합니다.
> 그 EC2의 보안 그룹이 RDS에 접속하도록 RDS 쪽에서 허용해줘야 합니다.

1. EB EC2 보안 그룹 ID 확인: EC2 콘솔 → 보안 그룹 → 이름이 `awseb-e-...` 인 것 (예: `sg-0aaa...`)
2. EC2 콘솔 → 보안 그룹 → **`rds-blog-sg`** 선택 → **인바운드 규칙 편집** → **규칙 추가**
   - 유형: **MySQL/Aurora** (포트 3306)
   - 소스: **사용자 지정** → EB 보안 그룹(`sg-0aaa...`) 선택
   - 저장

---

## 4. EB 환경변수 설정

EB 콘솔 → 환경 → **구성 → 소프트웨어(또는 환경 세부 정보) 편집 → 환경 속성**에 추가:

| 키 | 값 |
|---|---|
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `RDS_HOSTNAME` | RDS 엔드포인트 주소 |
| `RDS_PORT` | `3306` |
| `RDS_DB_NAME` | `blogdb` |
| `RDS_USERNAME` | `admin` |
| `RDS_PASSWORD` | (RDS 마스터 비밀번호) |
| `SERVER_PORT` | `5000` |

> **`SERVER_PORT=5000` 이 중요합니다.** EB의 nginx는 앱이 **5000 포트**에 있다고 보고 요청을
> 넘기는데, Spring Boot 기본값은 8080이라 그대로 두면 **502 Bad Gateway**가 납니다.
> Spring Boot는 `SERVER_PORT` 환경변수를 읽어 `server.port=5000`으로 뜹니다.

적용하면 환경이 자동 재시작됩니다.

---

## 5. JAR 빌드 & 수동 배포

### 5-1. 빌드 (⭐ 반드시 clean — stale JAR 방지)

```powershell
# PowerShell은 현재 폴더 실행파일에 .\ 필요
.\gradlew clean bootJar
```
또는 IntelliJ: Gradle 패널 → Tasks → build → **clean** → **bootJar**

- 결과물: `build/libs/블로그이름-0.0.1-SNAPSHOT.jar`
- **JAR 파일의 수정 시각이 "방금"인지 반드시 확인** (예전 시각이면 빌드가 안 된 것)

### 5-2. 배포

EB 콘솔 → 환경 → **업로드 및 배포** → 방금 만든 `build/libs/....jar` 선택 → **배포**

### 5-3. 수정 후 재배포 프로세스 (수동)

1. 코드 수정 & 저장
2. `.\gradlew clean bootJar` (반드시 clean)
3. `build/libs`의 JAR **수정 시각 확인**
4. EB 업로드 및 배포
5. 상태 Ok 확인 + 로그 확인

> **환경변수만 바꿀 때**는 재빌드 불필요 — EB 환경 속성에서 값만 바꾸면 재시작되며 반영됩니다.
> **엔티티(테이블) 변경**은 `ddl-auto: update`라 재배포 시 스키마가 자동 반영됩니다(컬럼 추가만, 삭제/변경은 수동).

---

## 6. 확인

- EB 환경 상태가 **Ok(초록)** 인지
- 환경 도메인(`...elasticbeanstalk.com`)으로 접속해 앱이 뜨는지
- 로그: EB → **로그 → 로그 요청 → 마지막 100줄** → `/var/log/web.stdout.log` 에서
  - `HikariPool-1 - Start completed` + `Database JDBC URL [jdbc:mysql://...blogdb...]` → RDS 연결 성공
  - `Started ...Application in N seconds` → 정상 기동

---

## 7. 자주 겪는 에러 & 원인

| 증상 | 원인 | 해결 |
|---|---|---|
| **502 Bad Gateway** (nginx) | 앱이 8080에서 뜸, EB는 5000 기대 | `SERVER_PORT=5000` 환경변수 추가 |
| **500 / `H2ConsoleProperties` 빈 없음** | prod에서 H2 콘솔 껐는데 `toH2Console()` 사용 | `toH2Console()` → `/h2-console/**` |
| **로그인 후 블로그로 안 넘어감** | favicon SavedRequest 리다이렉트 | `defaultSuccessUrl("/articles", true)` |
| **연결 타임아웃 / Communications link failure** | RDS 보안 그룹이 EB를 안 막음 | RDS SG 인바운드 3306에 EB SG 허용 |
| **Access denied for user** | 계정/비밀번호 오타 | `RDS_USERNAME` / `RDS_PASSWORD` 확인 |
| **Unknown database 'blogdb'** | RDS 생성 시 초기 DB 이름 미지정 | RDS에 DB 생성 or 이름 확인 |
| **앱이 여전히 H2로 뜸** | 프로파일 미적용 | `SPRING_PROFILES_ACTIVE=prod` 확인 |
| **수정했는데 반영 안 됨 (에러 그대로)** | **stale JAR 배포** | `clean` 빌드 + JAR 시각 확인 후 재업로드 |

> **핵심 교훈:** 코드/설정을 바꿨는데 증상이 그대로면, 십중팔구 **예전 JAR이 올라간 것**입니다.
> 항상 `.\gradlew clean bootJar` → **JAR 수정 시각 확인** → 업로드 순서를 지키세요.

---

## 8. (선택) 로컬에서 RDS에 직접 접속 (MySQL Workbench 등)

1. RDS **퍼블릭 액세스: 예** (수정 → 연결 → 추가 구성 → 퍼블릭 액세스 → 예 → 즉시 적용)
2. RDS 보안 그룹 인바운드 3306에 **내 IP** 허용
3. Workbench: Host=엔드포인트, Port=3306, User=admin, Password=…, Default Schema=blogdb

> 인터넷 IP가 바뀌면 인바운드 규칙의 IP도 갱신해야 합니다.

---

## 9. (선택) CI/CD 자동 배포 — GitHub Actions

수동 "빌드 → JAR 확인 → 업로드"를 자동화합니다. **코드를 push하면 자동으로 EB에 배포**됩니다.

```
코드 push → GitHub Actions가 JDK 설치 → gradlew clean bootJar → EB에 자동 배포
```

### 9-0. CI/CD란? (개념)

**CI/CD**는 코드를 고친 뒤 "빌드하고 테스트해서 서버에 올리는" 반복 작업을 자동화하는 개발 방식입니다.
크게 두 단계로 나뉩니다.

- **CI (Continuous Integration, 지속적 통합)**
  개발자가 코드를 저장소에 push하면 **자동으로 빌드하고 테스트**합니다.
  "내 PC에선 되는데 서버에선 안 돼" 같은 문제를, 합치는 순간 바로 잡아내는 것이 목적입니다.
  → 이 프로젝트에서는 `./gradlew clean bootJar` 로 JAR을 만드는 단계가 CI에 해당합니다.

- **CD (Continuous Delivery / Deployment, 지속적 배포)**
  CI를 통과한 결과물(JAR)을 **자동으로 실제 서버(EB)에 배포**합니다.
  → 이 프로젝트에서는 EB에 JAR을 업로드하는 단계가 CD에 해당합니다.

**왜 쓰나요? — 수동 배포와 비교**

| | 수동 배포 (5장) | CI/CD 자동 배포 |
|---|---|---|
| 과정 | 코드 수정 → `clean bootJar` → JAR 시각 확인 → EB 콘솔 업로드 | 코드 수정 → **git push** (끝) |
| 실수 여지 | stale JAR 업로드, clean 빼먹기 등 사람 실수 | 항상 같은 순서로 깨끗이 빌드 |
| 기록 | 누가 언제 뭘 올렸는지 흐릿함 | push·배포 이력이 저장소에 남음 |
| 협업 | 각자 로컬에서 올려 충돌 위험 | 정해진 브랜치 push로 일원화 |

> 핵심: **"반복되는 배포 작업을 사람이 매번 손으로 하지 않고, push 한 번으로 기계가 똑같이 수행"** 하게 만드는 것.
> 이 가이드 내내 겪은 *stale JAR 문제*(4장·7장)도, 빌드를 기계가 매번 `clean`부터 하니 근본적으로 사라집니다.

**파이프라인(pipeline)** 이란 이 일련의 자동 단계(빌드 → 테스트 → 배포)를 연결한 흐름을 말하며,
GitHub Actions에서는 `.github/workflows/deploy.yml` 파일 하나가 그 파이프라인 정의가 됩니다.

### 9-1. AWS 자격증명(IAM 사용자) 준비

- AWS IAM → 사용자 생성 → **프로그래밍 방식 액세스(액세스 키)**
- 권한: 학습용이면 `AdministratorAccess-AWSElasticBeanstalk` + S3 접근 (실서비스는 최소 권한으로 좁히기)
- **액세스 키 ID / 시크릿 키** 발급받아 메모

### 9-2. GitHub Secrets에 키 저장 (⭐ 절대 코드에 넣지 말 것)

GitHub 저장소 → Settings → Secrets and variables → Actions → New repository secret
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

### 9-3. 워크플로 파일 — `.github/workflows/deploy.yml`

```yaml
name: Deploy to Elastic Beanstalk

on:
  push:
    branches: [ main ]   # main 브랜치에 push되면 실행

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: 'corretto'
          java-version: '21'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build JAR
        run: ./gradlew clean bootJar

      - name: Deploy to Elastic Beanstalk
        uses: einaregilsson/beanstalk-deploy@v22
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          region: ap-northeast-2
          application_name: springboot-developer
          environment_name: Springboot-developer-env
          version_label: ${{ github.run_number }}-${{ github.sha }}
          deployment_package: build/libs/blog-0.0.1-SNAPSHOT.jar
```

이제 `git push` 하면 GitHub 저장소의 **Actions** 탭에서 빌드·배포가 자동으로 돌아갑니다.

### 9-4. 확인/주의할 점

- `application_name`, `environment_name`은 EB 콘솔의 **실제 이름과 일치**해야 함
  (현재: 앱 `springboot-developer`, 환경 `Springboot-developer-env`)
- `deployment_package` 경로의 JAR 이름 확인 (`build/libs/blog-0.0.1-SNAPSHOT.jar`)
- **git 저장소 루트가 `blog` 폴더**여야 위 경로가 맞음. `blog`가 하위 폴더면 각 step에
  `working-directory: blog`를 추가하고 JAR 경로를 `blog/build/libs/...`로 조정
- **DB 비밀번호 등은 CI에 넣을 필요 없음** — `RDS_*`, `SPRING_PROFILES_ACTIVE`, `SERVER_PORT`는
  이미 EB 환경변수에 있으므로 CI는 JAR만 올리면 됨

### 9-5. 대안 — AWS 네이티브 (CodePipeline + CodeBuild)

- 소스(GitHub 연결) → 빌드(CodeBuild, `buildspec.yml`에 `gradlew bootJar`) → 배포(Elastic Beanstalk)
- 콘솔에서 파이프라인 시각화 가능하나 설정이 더 복잡하고 CodeBuild는 빌드 시간당 과금(프리 티어 한도 있음)
- **학습·개인 프로젝트는 GitHub Actions 추천** (무료 한도 넉넉, 설정 간단)

---

## 10. 마무리 / 비용 관리

- 임시로 넣었던 디버깅용 환경변수(예: `SPRING_H2_CONSOLE_ENABLED`)는 정리
- 로컬 접속 위해 켰던 RDS **퍼블릭 액세스는 실서비스면 다시 "아니요"** 로
- **학습이 끝나면 RDS 인스턴스와 EB 환경을 삭제** (프리 티어라도 방치 시 과금될 수 있음)
- 코드/저장소에 노출된 API 키·비밀번호·AWS 액세스 키가 있었다면 **폐기(rotate)**
```

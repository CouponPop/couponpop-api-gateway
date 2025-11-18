# couponpop-api-gateway

CouponPop 마이크로서비스 아키텍처의 모든 외부 요청에 대한 라우팅을 담당하는 **API 게이트웨이** 서비스입니다.

---

## 1. 주요 역할

* **[라우팅]** Spring Cloud Gateway를 사용하여 `local` 및 `prod` 환경에 맞게 백엔드 서비스로 요청을 분산합니다.
* **[모니터링]** 백엔드 서비스의 `health check` 및 `prometheus` 메트릭 수집 엔드포인트를 외부에 노출합니다.
* **[CI/CD]** Jenkins 파이프라인(`Jenkinsfile`)을 통해 SonarQube 분석, Docker 이미지 빌드 및 AWS ECS Blue/Green 배포가 자동화되어 있습니다.

## 2. 기술 스택

* **Language**: Java 17
* **Framework**: Spring Boot 3.x, Spring Cloud Gateway
* **CI/CD**: Jenkins, Docker, SonarQube, Jacoco
* **Monitoring**: Micrometer (Prometheus)

## 3. 라우팅 테이블

프로덕션 환경(`prod`) 기준 라우팅 경로입니다. (AWS Service Connect Namespace 기준)

| Path Prefix | Target Service (Internal DNS) | 비고 |
| :--- | :--- | :--- |
| `/api/v1/auth/**` | `http://member.couponpop.internal:8080` | 인증/회원가입 |
| `/api/v1/members/**` | `http://member.couponpop.internal:8080` | 회원 정보 |
| `/api/v1/owner/stores/**` | `http://store.couponpop.internal:8080` | (점주) 매장 관리 |
| `/api/v1/stores/**` | `http://store.couponpop.internal:8080` | (고객) 매장 조회/검색 |
| `/api/v1/owner/coupons/**`| `http://coupon.couponpop.internal:8080` | (점주) 쿠폰 이벤트 관리 |
| `/api/v1/coupons/**` | `http://coupon.couponpop.internal:8080` | (고객) 쿠폰 발급/조회 |
| `/api/v1/fcm-token` | `http://notification.couponpop.internal:8080` | FCM 토큰 등록 |
| `/api/v1/jobs/**` | `http://batch.couponpop.internal:8080` | 배치 외부 수동 트리거 |
| `/health/{service}` | `http://{service}.couponpop.internal:8080` | 서비스 헬스 체크 |
| `/prometheus/{service}` | `http://{service}.couponpop.internal:8080` | 모니터링 메트릭 수집 |

## 4. 로컬 개발 실행 방법

API 게이트웨이는 단독으로 실행되지 않으며, 요청을 전달할 대상 서비스들이 로컬에 실행되어 있어야 합니다.
로컬에서의 서비스 디스커버리는 port를 활용합니다.

1.  (필수) **JDK 17**을 설치합니다.
2.  (필수) **모든 마이크로 서비스를 로컬에서 먼저 실행합니다.**
    * `member-service` (Port 8081)
    * `store-service` (Port 8082)
    * `coupon-service` (Port 8083)
    * `notification-service` (Port 8084)
    * (`batch-service`는 API 테스트에 필수적이지 않을 수 있습니다.)

3.  이 저장소를 클론합니다:
    ```bash
    git clone https://github.com/CouponPop/couponpop-api-gateway.git
    cd couponpop-api-gateway
    ```

4.  IntelliJ 또는 선호하는 IDE에서 프로젝트를 엽니다.

5.  **'local' 프로필**을 활성화하여 애플리케이션을 실행합니다.
    * IDE의 실행 설정(Run Configuration)에서 Active Profile을 `local`로 설정합니다.
    * `application-local.yml` 파일이 로드되어야 합니다. (이 파일에 `localhost:8081`, `localhost:8082` 등 로컬 라우팅 정보가 포함되어 있습니다.)

6.  (또는) 터미널에서 Gradle로 직접 실행합니다:
    ```bash
    ./gradlew bootRun --args='--spring.profiles.active=local'
    ```

7.  게이트웨이가 `http://localhost:8080` 에서 실행됩니다.

8.  Postman 등을 통해 원하는 API를 호출합니다.

## 5. 운영 시 참고 사항

* **서비스 디스커버리**: 이 게이트웨이는 Eureka 등 별도 Discovery Client를 사용하지 않습니다. 대신 `application-prod.yml`에 정의된 **AWS Service Connect의 내부 DNS 이름**(예: `http://member.couponpop.internal:8080`)을 사용하여 서비스를 직접 호출합니다.
* **헬스 체크**: AWS Target Group의 헬스 체크는 게이트웨이 자체의 헬스 체크 엔드포인트인 `/actuator/health`를 사용해야 합니다. (`/health/member` 등은 백엔드 서비스의 상태를 확인하는 프록시 경로입니다.)
* **인증 정책**: 본 게이트웨이는 인증/인가를 직접 처리하지 않습니다. 클라이언트의 `Authorization` 헤더를 변경 없이 그대로 백엔드 서비스로 전달(pass-through)하며, 실제 JWT 검증은 각 서비스(예: `member-service`)에서 `security-module`을 의존하여 수행됩니다.

# Spring Boot Project Structure

## 개념
- 상용 프로젝트에서 검증된 디렉토리 구조
- [[Enterprise Design Patterns]]을 반영한 패키지 설계
- [[Spring Boot Project Setup]]의 실무 확장 버전
- 확장성과 유지보수성을 고려한 계층형 아키텍처

## 1. 표준 Maven/Gradle 프로젝트 구조

## 개요
```
com.company.ecommerce/
├── domain/          // JPA 엔티티, VO, Repository
├── dto/            // Request/Response/Internal DTO
├── controller/     // REST 컨트롤러 (v1, v2, admin, internal)
├── service/        // 비즈니스 로직 (인터페이스/구현체 분리)
├── repository/     // 데이터 접근 (Custom Repository, Specification)
├── security/       // JWT, OAuth2, 보안 설정
├── exception/      // 글로벌/도메인별 예외 처리
├── config/         // 각종 설정 클래스들
├── util/          // 유틸리티 클래스들
├── aspect/        // AOP 관련
├── event/         // 이벤트 처리
├── integration/   // 외부 시스템 연동
└── monitoring/    // 헬스체크, 메트릭
```

### **멀티모듈 프로젝트 구조**

- 대규모 프로젝트를 위한 모듈 분리 전략
- api, core, common, security, integration 모듈
- 각 모듈별 책임과 의존관계

### **설정 파일 구조**

- 환경별 application.yml 분리
- DB 마이그레이션 스크립트 구조
- 정적 리소스, 템플릿, 국제화 파일 조직화

### **테스트 구조**

- integration, unit, fixture, container, mock 분리
- Testcontainers 활용법
- 테스트 데이터 관리 전략

### **운영 및 배포 파일**

- Docker, Kubernetes 매니페스트
- Terraform Infrastructure as Code
- GitHub Actions CI/CD 파이프라인
- 각종 운영 스크립트들

### **Gradle 멀티모듈 구조**

- build.gradle 파일 구성
- 모듈 의존성 관리
### 기본 디렉토리 레이아웃
```
ecommerce-api/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── company/
│   │   │           └── ecommerce/
│   │   │               ├── EcommerceApplication.java
│   │   │               ├── config/
│   │   │               ├── controller/
│   │   │               ├── service/
│   │   │               ├── repository/
│   │   │               ├── domain/
│   │   │               ├── dto/
│   │   │               ├── exception/
│   │   │               ├── security/
│   │   │               └── util/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/
│   │       │   └── migration/
│   │       ├── static/
│   │       ├── templates/
│   │       └── i18n/
│   └── test/
│       ├── java/
│       │   └── com/company/ecommerce/
│       │       ├── integration/
│       │       ├── unit/
│       │       └── fixture/
│       └── resources/
├── docs/
├── scripts/
├── docker/
├── .github/
│   └── workflows/
├── README.md
├── pom.xml (또는 build.gradle)
└── Dockerfile
```

## 2. 패키지별 상세 구조
### A. 도메인 중심 구조 (Domain-Driven Design)
```java
com.company.ecommerce/
├── EcommerceApplication.java                    // 메인 애플리케이션
│
├── config/                                      // 설정 클래스들
│   ├── DatabaseConfig.java
│   ├── SecurityConfig.java
│   ├── WebConfig.java
│   ├── RedisConfig.java
│   ├── OpenApiConfig.java
│   └── AsyncConfig.java
│
├── domain/                                      // 도메인 엔티티 및 VO
│   ├── user/
│   │   ├── User.java                           // JPA Entity
│   │   ├── UserRole.java                       // Enum
│   │   ├── UserStatus.java                     // Enum  
│   │   └── UserRepository.java                 // Repository Interface
│   ├── product/
│   │   ├── Product.java
│   │   ├── Category.java
│   │   ├── ProductRepository.java
│   │   └── CategoryRepository.java
│   ├── order/
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   ├── OrderStatus.java
│   │   └── OrderRepository.java
│   └── common/
│       ├── BaseEntity.java                     // 공통 엔티티 필드
│       ├── Address.java                        // Value Object
│       └── Money.java                          // Value Object
│
├── dto/                                         // 데이터 전송 객체
│   ├── request/
│   │   ├── user/
│   │   │   ├── CreateUserRequest.java
│   │   │   ├── UpdateUserRequest.java
│   │   │   └── UserSearchRequest.java
│   │   ├── product/
│   │   │   ├── CreateProductRequest.java
│   │   │   └── ProductSearchRequest.java
│   │   └── order/
│   │       ├── CreateOrderRequest.java
│   │       └── UpdateOrderRequest.java
│   ├── response/
│   │   ├── user/
│   │   │   ├── UserResponse.java
│   │   │   └── UserSummaryResponse.java
│   │   ├── product/
│   │   │   ├── ProductResponse.java
│   │   │   └── ProductListResponse.java
│   │   ├── order/
│   │   │   ├── OrderResponse.java
│   │   │   └── OrderSummaryResponse.java
│   │   └── common/
│   │       ├── ApiResponse.java                // 공통 응답 래퍼
│   │       ├── PageResponse.java               // 페이징 응답
│   │       └── ErrorResponse.java              // 에러 응답
│   └── internal/                               // 내부 서비스 간 DTO
│       ├── UserServiceDto.java
│       └── OrderServiceDto.java
│
├── controller/                                  // REST 컨트롤러
│   ├── api/
│   │   ├── v1/
│   │   │   ├── UserController.java
│   │   │   ├── ProductController.java
│   │   │   ├── OrderController.java
│   │   │   └── AuthController.java
│   │   └── v2/                                 // API 버전 관리
│   │       └── UserControllerV2.java
│   ├── admin/                                  // 관리자 전용 API
│   │   ├── AdminUserController.java
│   │   ├── AdminProductController.java
│   │   └── AdminOrderController.java
│   └── internal/                               // 내부 서비스 간 통신
│       └── InternalUserController.java
│
├── service/                                     // 비즈니스 로직
│   ├── user/
│   │   ├── UserService.java                    // 인터페이스
│   │   ├── UserServiceImpl.java                // 구현체
│   │   ├── UserRegistrationService.java       // 특화된 서비스
│   │   └── UserValidationService.java
│   ├── product/
│   │   ├── ProductService.java
│   │   ├── ProductServiceImpl.java
│   │   ├── ProductSearchService.java
│   │   └── ProductInventoryService.java
│   ├── order/
│   │   ├── OrderService.java
│   │   ├── OrderServiceImpl.java
│   │   ├── OrderProcessingService.java
│   │   └── OrderNotificationService.java
│   └── common/
│       ├── EmailService.java
│       ├── NotificationService.java
│       ├── FileStorageService.java
│       └── AuditService.java
│
├── repository/                                  // 데이터 접근 계층
│   ├── user/
│   │   ├── UserRepositoryImpl.java             // Custom Repository 구현
│   │   └── UserRepositoryCustom.java          // Custom Repository 인터페이스
│   ├── product/
│   │   ├── ProductRepositoryImpl.java
│   │   └── ProductRepositoryCustom.java
│   ├── order/
│   │   ├── OrderRepositoryImpl.java
│   │   └── OrderRepositoryCustom.java
│   └── specification/                          // JPA Specification
│       ├── UserSpecification.java
│       ├── ProductSpecification.java
│       └── OrderSpecification.java
│
├── security/                                    // 보안 관련
│   ├── jwt/
│   │   ├── JwtTokenProvider.java
│   │   ├── JwtAuthenticationFilter.java
│   │   └── JwtAuthenticationEntryPoint.java
│   ├── oauth/
│   │   ├── OAuth2UserService.java
│   │   ├── OAuth2SuccessHandler.java
│   │   └── OAuth2FailureHandler.java
│   ├── UserPrincipal.java
│   ├── SecurityService.java
│   └── PasswordService.java
│
├── exception/                                   // 예외 처리
│   ├── GlobalExceptionHandler.java             // 글로벌 예외 처리
│   ├── BusinessException.java                  // 비즈니스 예외
│   ├── ErrorCode.java                          // 에러 코드 정의
│   ├── user/
│   │   ├── UserNotFoundException.java
│   │   ├── DuplicateEmailException.java
│   │   └── InvalidPasswordException.java
│   ├── product/
│   │   ├── ProductNotFoundException.java
│   │   └── InsufficientStockException.java
│   └── order/
│       ├── OrderNotFoundException.java
│       └── InvalidOrderStatusException.java
│
├── util/                                        // 유틸리티 클래스
│   ├── DateUtils.java
│   ├── StringUtils.java
│   ├── ValidationUtils.java
│   ├── FileUtils.java
│   ├── CryptoUtils.java
│   └── JsonUtils.java
│
├── aspect/                                      // AOP 관련
│   ├── LoggingAspect.java
│   ├── SecurityAuditAspect.java
│   ├── PerformanceAspect.java
│   └── TransactionAspect.java
│
├── event/                                       // 이벤트 처리
│   ├── user/
│   │   ├── UserRegisteredEvent.java
│   │   ├── UserUpdatedEvent.java
│   │   └── UserRegisteredEventHandler.java
│   ├── order/
│   │   ├── OrderCreatedEvent.java
│   │   ├── OrderCompletedEvent.java
│   │   └── OrderEventHandler.java
│   └── common/
│       └── DomainEvent.java
│
├── integration/                                 // 외부 시스템 연동
│   ├── payment/
│   │   ├── PaymentGateway.java
│   │   ├── PaymentService.java
│   │   └── PaymentWebhookController.java
│   ├── notification/
│   │   ├── EmailProvider.java
│   │   ├── SmsProvider.java
│   │   └── PushNotificationProvider.java
│   └── external/
│       ├── WeatherApiClient.java
│       └── GeolocationApiClient.java
│
└── monitoring/                                  // 모니터링 및 헬스체크
    ├── health/
    │   ├── CustomHealthIndicator.java
    │   ├── DatabaseHealthIndicator.java
    │   └── RedisHealthIndicator.java
    ├── metrics/
    │   ├── CustomMetrics.java
    │   └── BusinessMetrics.java
    └── actuator/
        └── CustomEndpoint.java
```

## 3. 멀티모듈 프로젝트 구조
### 대규모 프로젝트를 위한 모듈 분리
```
ecommerce-platform/
├── ecommerce-api/                              // API 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/api/
│   │       ├── controller/
│   │       ├── config/
│   │       └── EcommerceApiApplication.java
│   └── pom.xml
│
├── ecommerce-core/                             // 핵심 비즈니스 로직
│   ├── src/main/java/
│   │   └── com/company/ecommerce/core/
│   │       ├── service/
│   │       ├── domain/
│   │       └── repository/
│   └── pom.xml
│
├── ecommerce-common/                           // 공통 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/common/
│   │       ├── dto/
│   │       ├── exception/
│   │       ├── util/
│   │       └── constant/
│   └── pom.xml
│
├── ecommerce-security/                         // 보안 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/security/
│   │       ├── jwt/
│   │       ├── oauth/
│   │       └── config/
│   └── pom.xml
│
├── ecommerce-integration/                      // 외부 연동 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/integration/
│   │       ├── payment/
│   │       ├── notification/
│   │       └── external/
│   └── pom.xml
│
├── ecommerce-admin/                            // 관리자 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/admin/
│   │       ├── controller/
│   │       ├── service/
│   │       └── EcommerceAdminApplication.java
│   └── pom.xml
│
├── ecommerce-batch/                            // 배치 작업 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/batch/
│   │       ├── job/
│   │       ├── tasklet/
│   │       └── EcommerceBatchApplication.java
│   └── pom.xml
│
├── ecommerce-test-support/                     // 테스트 지원 모듈
│   ├── src/main/java/
│   │   └── com/company/ecommerce/test/
│   │       ├── fixture/
│   │       ├── container/
│   │       └── mock/
│   └── pom.xml
│
└── pom.xml                                     // 루트 POM
```

## 4. 설정 파일 구조
### resources 디렉토리 상세
```
src/main/resources/
├── application.yml                             // 기본 설정
├── application-local.yml                      // 로컬 개발 환경
├── application-dev.yml                        // 개발 환경
├── application-staging.yml                    // 스테이징 환경  
├── application-prod.yml                       // 운영 환경
├── application-test.yml                       // 테스트 환경
│
├── db/
│   ├── migration/                              // Flyway/Liquibase 마이그레이션
│   │   ├── V001__Create_user_table.sql
│   │   ├── V002__Create_product_table.sql
│   │   ├── V003__Create_order_table.sql
│   │   └── V004__Add_indexes.sql
│   └── data/                                   // 초기 데이터
│       ├── users.sql
│       ├── products.sql
│       └── categories.sql
│
├── static/                                     // 정적 리소스
│   ├── css/
│   ├── js/
│   ├── images/
│   └── favicon.ico
│
├── templates/                                  // 템플릿 파일 (Thymeleaf 등)
│   ├── email/
│   │   ├── welcome.html
│   │   ├── order-confirmation.html
│   │   └── password-reset.html
│   └── admin/
│       ├── dashboard.html
│       └── reports.html
│
├── i18n/                                       // 국제화 메시지
│   ├── messages.properties
│   ├── messages_ko.properties
│   ├── messages_en.properties
│   └── messages_ja.properties
│
├── config/                                     // 추가 설정 파일
│   ├── logback-spring.xml                     // 로깅 설정
│   ├── ehcache.xml                            // 캐시 설정  
│   └── elastic-apm.properties                 // APM 설정
│
├── certificates/                               // 인증서 파일
│   ├── keystore.p12
│   └── truststore.jks
│
└── META-INF/
    ├── spring.factories                        // Auto Configuration
    └── additional-spring-configuration-metadata.json
```

## 5. 테스트 구조
### test 디렉토리 구성
```
src/test/
├── java/
│   └── com/company/ecommerce/
│       ├── integration/                        // 통합 테스트
│       │   ├── controller/
│       │   │   ├── UserControllerIntegrationTest.java
│       │   │   ├── ProductControllerIntegrationTest.java
│       │   │   └── OrderControllerIntegrationTest.java
│       │   ├── service/
│       │   │   ├── UserServiceIntegrationTest.java
│       │   │   └── OrderServiceIntegrationTest.java
│       │   └── repository/
│       │       ├── UserRepositoryIntegrationTest.java
│       │       └── ProductRepositoryIntegrationTest.java
│       │
│       ├── unit/                               // 단위 테스트
│       │   ├── service/
│       │   │   ├── UserServiceTest.java
│       │   │   ├── ProductServiceTest.java
│       │   │   └── OrderServiceTest.java
│       │   ├── util/
│       │   │   ├── DateUtilsTest.java
│       │   │   └── ValidationUtilsTest.java
│       │   └── security/
│       │       ├── JwtTokenProviderTest.java
│       │       └── PasswordServiceTest.java
│       │
│       ├── fixture/                            // 테스트 데이터 팩토리
│       │   ├── UserFixture.java
│       │   ├── ProductFixture.java
│       │   ├── OrderFixture.java
│       │   └── TestDataBuilder.java
│       │
│       ├── container/                          // Testcontainers
│       │   ├── DatabaseContainer.java
│       │   ├── RedisContainer.java
│       │   └── ElasticsearchContainer.java
│       │
│       ├── mock/                               // Mock 객체
│       │   ├── MockEmailService.java
│       │   ├── MockPaymentGateway.java
│       │   └── MockExternalApiClient.java
│       │
│       └── config/                             // 테스트 설정
│           ├── TestConfig.java
│           ├── TestSecurityConfig.java
│           └── TestDatabaseConfig.java
│
└── resources/
    ├── application-test.yml                    // 테스트 전용 설정
    ├── test-data/                              // 테스트 데이터
    │   ├── users.json
    │   ├── products.json
    │   └── orders.json
    ├── contracts/                              // Contract 테스트
    │   └── user-api-contract.yml
    └── db/
        └── test-data.sql                       // 테스트 DB 초기 데이터
```

## 6. 운영 및 배포 파일
### 프로젝트 루트의 운영 관련 파일들
```
project-root/
├── docker/
│   ├── Dockerfile                              // 애플리케이션 도커파일
│   ├── Dockerfile.multi-stage                  // 멀티스테이지 빌드
│   ├── docker-compose.yml                      // 로컬 개발용
│   ├── docker-compose.prod.yml                 // 운영용
│   └── nginx/
│       ├── nginx.conf
│       └── ssl/
│
├── k8s/                                        // Kubernetes 매니페스트
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
│
├── terraform/                                  // Infrastructure as Code
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
│
├── scripts/                                    // 유틸리티 스크립트
│   ├── deploy.sh
│   ├── backup.sh
│   ├── health-check.sh
│   └── db-migration.sh
│
├── .github/                                    // GitHub Actions
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       ├── security-scan.yml
│       └── release.yml
│
├── docs/                                       // 문서
│   ├── api/
│   │   └── openapi.yaml
│   ├── architecture/
│   │   ├── system-design.md
│   │   └── database-schema.md
│   ├── deployment/
│   │   └── deployment-guide.md
│   └── development/
│       ├── coding-standards.md
│       └── setup-guide.md
│
├── .env.example                                // 환경변수 예시
├── .gitignore
├── .dockerignore
├── README.md
├── CHANGELOG.md
├── LICENSE
└── sonar-project.properties                    // SonarQube 설정
```

## 7. Gradle 기반 멀티모듈 구조
### build.gradle 파일 구성
```
ecommerce-platform/
├── build.gradle                                // 루트 빌드 스크립트
├── settings.gradle                             // 모듈 설정
├── gradle/
│   └── wrapper/
├── gradle.properties                           // Gradle 속성
│
├── ecommerce-api/
│   └── build.gradle                            // API 모듈 빌드
├── ecommerce-core/  
│   └── build.gradle                            // Core 모듈 빌드
├── ecommerce-common/
│   └── build.gradle                            // Common 모듈 빌드
└── ecommerce-security/
    └── build.gradle                            // Security 모듈 빌드
```

## 8. 패키지 명명 규칙
### 일관된 네이밍 컨벤션
```java
// 도메인별 패키지 구조
com.company.projectname.domain.subdomain

// 예시
com.company.ecommerce.user.controller.UserController
com.company.ecommerce.user.service.UserService  
com.company.ecommerce.user.repository.UserRepository
com.company.ecommerce.user.dto.request.CreateUserRequest
com.company.ecommerce.user.dto.response.UserResponse

// 공통 패키지
com.company.ecommerce.common.exception.BusinessException
com.company.ecommerce.common.dto.ApiResponse
com.company.ecommerce.common.util.DateUtils
```

## 관련 패턴
- [[Enterprise Design Patterns]]
- [[Spring Boot Project Setup]]
- [[Spring Boot Project Structure]]
- [[Configuration Properties]]
- [[API Design Patterns]]
- [[Testing]]
- [[Security Patterns]]
- [[Spring Boot CI CD with GitLab and ArgoCD]]

#project-structure #directory-layout #package-organization #multi-module
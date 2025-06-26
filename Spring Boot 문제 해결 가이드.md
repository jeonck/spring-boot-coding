# Spring Boot 문제 해결 가이드

> Spring Boot 개발 중 자주 발생하는 문제들과 해결 방법을 정리한 트러블슈팅 가이드

## 📋 목차

- [시작 및 설정 문제](#시작-및-설정-문제)
- [의존성 및 버전 충돌](#의존성-및-버전-충돌)
- [데이터베이스 연결 문제](#데이터베이스-연결-문제)
- [보안 관련 문제](#보안-관련-문제)
- [성능 및 메모리 문제](#성능-및-메모리-문제)
- [테스트 관련 문제](#테스트-관련-문제)
- [배포 및 운영 문제](#배포-및-운영-문제)
- [로깅 및 모니터링 문제](#로깅-및-모니터링-문제)

## 🚀 시작 및 설정 문제

### 애플리케이션 시작 실패

#### 문제: `Port already in use`
```bash
# 에러 메시지
The Tomcat connector configured to listen on port 8080 failed to start. 
The port may already be in use or the connector may be misconfigured.
```

**해결 방법:**
```yaml
# application.yml
server:
  port: 0  # 랜덤 포트 사용
  # 또는
  port: 8081  # 다른 포트 사용

# 또는 실행 시 포트 지정
java -jar app.jar --server.port=8081
```

**포트 사용 중인 프로세스 확인:**
```bash
# Windows
netstat -ano | findstr :8080
taskkill /F /PID <PID>

# Linux/Mac
lsof -i :8080
kill -9 <PID>
```

#### 문제: `Bean creation failed`
```bash
# 에러 메시지
Error creating bean with name 'dataSource': 
Bean instantiation via factory method failed
```

**해결 방법:**
```java
// 1. 의존성 확인
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'  // 또는 적절한 DB 드라이버
}

// 2. 설정 확인
@Configuration
@ConditionalOnProperty(name = "spring.datasource.url")
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  jpa:
    hibernate:
      ddl-auto: create-drop
```

#### 문제: `Auto-configuration classes not found`
```bash
# 에러 메시지
Unable to start embedded container; 
nested exception is org.springframework.context.ApplicationContextException
```

**해결 방법:**
```java
// 1. 메인 클래스 확인
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// 2. 패키지 구조 확인 - 메인 클래스는 최상위 패키지에 위치
com.example.app
├── Application.java (메인 클래스)
├── controller/
├── service/
└── repository/

// 3. 명시적 컴포넌트 스캔
@SpringBootApplication(scanBasePackages = "com.example.app")
public class Application {
    // ...
}
```

### 설정 파일 문제

#### 문제: `application.yml` 파싱 오류
```bash
# 에러 메시지
while parsing a block mapping; expected <block end>, but found '<scalar>'
```

**해결 방법:**
```yaml
# 잘못된 YAML 구조
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
  username: user  # 잘못된 들여쓰기
    password: pass

# 올바른 YAML 구조
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
```

#### 문제: 프로파일별 설정이 적용되지 않음
```bash
# 에러: 개발 환경 설정이 운영에 적용됨
```

**해결 방법:**
```yaml
# application.yml (공통 설정)
spring:
  profiles:
    active: local

---
# application-local.yml
spring:
  profiles: local
  datasource:
    url: jdbc:h2:mem:testdb

---
# application-dev.yml
spring:
  profiles: dev
  datasource:
    url: jdbc:postgresql://dev-db:5432/mydb

---
# application-prod.yml
spring:
  profiles: prod
  datasource:
    url: jdbc:postgresql://prod-db:5432/mydb
```

```java
// 프로그래밍 방식으로 프로파일 설정
@Component
@Profile("!test")  // 테스트가 아닌 경우에만 실행
public class ProductionComponent {
    // ...
}

// 환경변수로 프로파일 설정
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

## 📦 의존성 및 버전 충돌

### 의존성 충돌 문제

#### 문제: `ClassNotFoundException`
```bash
# 에러 메시지
java.lang.ClassNotFoundException: org.springframework.web.bind.annotation.RestController
```

**해결 방법:**
```gradle
// 1. 의존성 확인
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // RestController는 spring-boot-starter-web에 포함됨
}

// 2. 의존성 트리 확인
./gradlew dependencies

// 3. 충돌하는 의존성 제외
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }
    implementation 'org.springframework.boot:spring-boot-starter-jetty'
}
```

#### 문제: `NoSuchMethodError`
```bash
# 에러 메시지
java.lang.NoSuchMethodError: org.springframework.util.StringUtils.hasText(Ljava/lang/String;)Z
```

**해결 방법:**
```gradle
// 1. Spring Boot BOM 사용
dependencyManagement {
    imports {
        mavenBom "org.springframework.boot:spring-boot-dependencies:3.2.0"
    }
}

// 2. 버전 충돌 해결
configurations.all {
    resolutionStrategy {
        force 'org.springframework:spring-core:6.1.0'
        force 'org.springframework:spring-web:6.1.0'
    }
}

// 3. 의존성 버전 확인
./gradlew dependencyInsight --dependency spring-core
```

#### 문제: Jackson 버전 충돌
```bash
# 에러 메시지
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: 
Cannot construct instance of `java.time.LocalDateTime`
```

**해결 방법:**
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // Jackson 버전은 Spring Boot가 관리하도록 버전 명시하지 않음
}
```

```java
@Configuration
public class JacksonConfig {
    
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .build();
    }
}
```

### Gradle/Maven 빌드 문제

#### 문제: `Could not resolve dependencies`
```bash
# 에러 메시지
Could not resolve org.springframework.boot:spring-boot-starter-web:3.2.0
```

**해결 방법:**
```gradle
// 1. 저장소 설정 확인
repositories {
    mavenCentral()
    gradlePluginPortal()
    // 회사 내부 저장소가 있다면
    maven {
        url "https://repo.company.com/maven"
        credentials {
            username = project.findProperty("repoUser") ?: "user"
            password = project.findProperty("repoPassword") ?: "pass"
        }
    }
}

// 2. 프록시 설정 (필요한 경우)
systemProp.http.proxyHost=proxy.company.com
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=proxy.company.com
systemProp.https.proxyPort=8080

// 3. 캐시 정리
./gradlew clean build --refresh-dependencies
```

## 🗄 데이터베이스 연결 문제

### 데이터베이스 연결 실패

#### 문제: `Connection refused`
```bash
# 에러 메시지
java.net.ConnectException: Connection refused: connect
```

**해결 방법:**
```yaml
# 1. 연결 정보 확인
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: password
    driver-class-name: org.postgresql.Driver

# 2. 연결 풀 설정
  hikari:
    connection-timeout: 30000
    maximum-pool-size: 10
    minimum-idle: 5
    idle-timeout: 300000
    max-lifetime: 900000
```

```java
// 3. 데이터베이스 연결 테스트
@Component
@RequiredArgsConstructor
@Slf4j
public class DatabaseConnectionTest {
    
    private final DataSource dataSource;
    
    @PostConstruct
    public void testConnection() {
        try (Connection connection = dataSource.getConnection()) {
            log.info("Database connection successful: {}", connection.getMetaData().getURL());
        } catch (SQLException e) {
            log.error("Database connection failed", e);
        }
    }
}
```

#### 문제: `Too many connections`
```bash
# 에러 메시지
PSQLException: FATAL: too many connections for role "user"
```

**해결 방법:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 5  # 연결 수 줄이기
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000  # 연결 누수 감지
```

```java
// 연결 모니터링
@Component
@RequiredArgsConstructor
@Slf4j
public class ConnectionPoolMonitor {
    
    private final HikariDataSource dataSource;
    
    @Scheduled(fixedRate = 30000)
    public void logConnectionStats() {
        HikariPoolMXBean poolBean = dataSource.getHikariPoolMXBean();
        log.info("Active connections: {}, Total connections: {}, Threads awaiting: {}",
            poolBean.getActiveConnections(),
            poolBean.getTotalConnections(),
            poolBean.getThreadsAwaitingConnection());
    }
}
```

### JPA/Hibernate 문제

#### 문제: `LazyInitializationException`
```bash
# 에러 메시지
org.hibernate.LazyInitializationException: 
failed to lazily initialize a collection of role: com.example.User.orders, 
could not initialize proxy - no Session
```

**해결 방법:**
```java
// 1. 즉시 로딩 사용
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER, mappedBy = "user")
    private List<Order> orders;
}

// 2. JOIN FETCH 사용
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);

// 3. @Transactional 추가
@Service
@Transactional(readOnly = true)
public class UserService {
    public UserDto getUser(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        user.getOrders().size(); // 트랜잭션 내에서 컬렉션 초기화
        return convertToDto(user);
    }
}

// 4. DTO 패턴 사용
@Query("SELECT new com.example.dto.UserDto(u.id, u.name, COUNT(o)) " +
       "FROM User u LEFT JOIN u.orders o " +
       "WHERE u.id = :id GROUP BY u.id, u.name")
UserDto findUserDtoById(@Param("id") Long id);
```

#### 문제: `Table doesn't exist`
```bash
# 에러 메시지
Table 'mydb.users' doesn't exist
```

**해결 방법:**
```yaml
# 1. DDL 자동 생성 설정
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop  # 개발용
      # ddl-auto: validate   # 운영용
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

```java
// 2. 스키마 초기화 스크립트
// src/main/resources/schema.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

// 3. 데이터 초기화 스크립트
// src/main/resources/data.sql
INSERT INTO users (username, email) VALUES 
('admin', 'admin@example.com'),
('user', 'user@example.com');
```

```yaml
# 스크립트 실행 설정
spring:
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
      data-locations: classpath:data.sql
```

## 🔐 보안 관련 문제

### Spring Security 문제

#### 문제: 모든 요청이 401 Unauthorized
```bash
# 에러: 인증이 필요한 모든 엔드포인트에 접근할 수 없음
```

**해결 방법:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**", "/health", "/actuator/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable())  // REST API의 경우
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .httpBasic(Customizer.withDefaults());
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### 문제: CSRF 토큰 관련 오류
```bash
# 에러 메시지
Invalid CSRF Token 'null' was found on the request parameter '_csrf'
```

**해결 방법:**
```java
// 1. REST API의 경우 CSRF 비활성화
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable());
        return http.build();
    }
}

// 2. 웹 애플리케이션의 경우 CSRF 토큰 포함
@Controller
public class WebController {
    
    @GetMapping("/form")
    public String showForm(Model model, HttpServletRequest request) {
        CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
        model.addAttribute("_csrf", csrfToken);
        return "form";
    }
}
```

```html
<!-- Thymeleaf 템플릿 -->
<form th:action="@{/submit}" method="post">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
    <!-- 폼 필드들 -->
</form>
```

### JWT 토큰 문제

#### 문제: JWT 토큰 검증 실패
```bash
# 에러 메시지
JWT signature does not match locally computed signature
```

**해결 방법:**
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration:86400000}")
    private long expiration;
    
    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));
        
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }
}
```

```yaml
# application.yml
jwt:
  secret: mySecretKey1234567890123456789012345678901234567890
  expiration: 86400000  # 24시간
```

## ⚡ 성능 및 메모리 문제

### OutOfMemoryError

#### 문제: `java.lang.OutOfMemoryError: Java heap space`
```bash
# 에러 메시지
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

**해결 방법:**
```bash
# 1. JVM 힙 크기 증가
java -Xms512m -Xmx2g -jar app.jar

# 2. GC 튜닝
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar

# 3. 힙 덤프 생성 설정
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof -jar app.jar
```

```java
// 메모리 사용량 모니터링
@Component
@Slf4j
public class MemoryMonitor {
    
    @Scheduled(fixedRate = 60000)
    public void logMemoryUsage() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        
        long used = heapUsage.getUsed() / 1024 / 1024;
        long max = heapUsage.getMax() / 1024 / 1024;
        double utilization = (double) heapUsage.getUsed() / heapUsage.getMax() * 100;
        
        log.info("Memory usage: {} MB / {} MB ({:.2f}%)", used, max, utilization);
        
        if (utilization > 85) {
            log.warn("High memory usage detected: {:.2f}%", utilization);
        }
    }
}
```

#### 문제: 메모리 누수
```java
// 메모리 누수 예방
@Service
public class ServiceWithCache {
    
    // 잘못된 예: 무제한 캐시
    private final Map<String, Object> cache = new HashMap<>();
    
    // 올바른 예: 제한된 캐시
    private final Cache<String, Object> cache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .softValues()
        .build();
    
    // ThreadLocal 사용 시 정리
    private final ThreadLocal<Context> contextThreadLocal = new ThreadLocal<>();
    
    @PreDestroy
    public void cleanup() {
        contextThreadLocal.remove();
    }
}
```

### 느린 응답 시간

#### 문제: API 응답이 느림
**해결 방법:**
```java
// 1. 데이터베이스 쿼리 최적화
@Query("SELECT p FROM Product p LEFT JOIN FETCH p.category WHERE p.id IN :ids")
List<Product> findByIdsWithCategory(@Param("ids") List<Long> ids);

// 2. 비동기 처리
@Service
public class AsyncService {
    
    @Async
    public CompletableFuture<String> processAsync(String data) {
        // 비동기 처리 로직
        return CompletableFuture.completedFuture("result");
    }
}

// 3. 캐싱 적용
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return productRepository.findById(id).orElse(null);
}

// 4. 페이징 적용
@GetMapping("/api/products")
public Page<Product> getProducts(Pageable pageable) {
    return productService.getAllProducts(pageable);
}
```

```java
// 응답 시간 모니터링
@Component
public class ResponseTimeInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                              HttpServletResponse response, 
                              Object handler, Exception ex) throws Exception {
        Long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        
        if (duration > 1000) {  // 1초 초과
            log.warn("Slow request: {} {} - {}ms", 
                    request.getMethod(), request.getRequestURI(), duration);
        }
    }
}
```

## 🧪 테스트 관련 문제

### 테스트 실행 실패

#### 문제: `@SpringBootTest` 실행 시 컨텍스트 로드 실패
```bash
# 에러 메시지
Unable to start embedded container; nested exception is 
org.springframework.boot.web.server.WebServerException
```

**해결 방법:**
```java
// 1. 테스트용 설정 분리
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    classes = TestApplication.class
)
public class IntegrationTest {
    // 테스트 코드
}

@TestConfiguration
public class TestApplication {
    
    @Bean
    @Primary
    public DataSource testDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

// 2. 테스트 프로파일 사용
@ActiveProfiles("test")
@SpringBootTest
public class ServiceTest {
    // 테스트 코드
}
```

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  security:
    user:
      name: test
      password: test
```

#### 문제: Mock 객체가 주입되지 않음
```java
// 잘못된 예
@SpringBootTest
public class ServiceTest {
    
    @Mock
    private ExternalService externalService;  // 주입되지 않음
    
    @Autowired
    private MyService myService;
}

// 올바른 예
@SpringBootTest
public class ServiceTest {
    
    @MockBean
    private ExternalService externalService;  // Spring 컨텍스트에서 관리
    
    @Autowired
    private MyService myService;
    
    @Test
    public void testMethod() {
        when(externalService.getData()).thenReturn("test");
        // 테스트 로직
    }
}

// 또는 단위 테스트
@ExtendWith(MockitoExtension.class)
public class ServiceUnitTest {
    
    @Mock
    private ExternalService externalService;
    
    @InjectMocks
    private MyService myService;
}
```

### 테스트 데이터 문제

#### 문제: 테스트 간 데이터 간섭
```java
// 해결 방법 1: @Transactional + @Rollback
@SpringBootTest
@Transactional
@Rollback
public class DataTest {
    
    @Test
    public void testMethod() {
        // 테스트 후 자동 롤백
    }
}

// 해결 방법 2: @DirtiesContext
@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
public class ContextTest {
    // 각 테스트 후 컨텍스트 재생성
}

// 해결 방법 3: 테스트 데이터 정리
@TestMethodOrder(OrderAnnotation.class)
public class OrderedTest {
    
    @BeforeEach
    public void setUp() {
        // 테스트 데이터 초기화
        testDataRepository.deleteAll();
    }
    
    @AfterEach
    public void tearDown() {
        // 테스트 데이터 정리
        testDataRepository.deleteAll();
    }
}
```

## 🚀 배포 및 운영 문제

### Docker 관련 문제

#### 문제: Docker 컨테이너 실행 실패
```dockerfile
# Dockerfile 최적화
FROM openjdk:17-jre-slim

# 시간대 설정
ENV TZ=Asia/Seoul
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 애플리케이션 사용자 생성
RUN groupadd -r appuser && useradd -r -g appuser appuser

# 작업 디렉토리 설정
WORKDIR /app

# JAR 파일 복사
COPY target/app.jar app.jar

# 소유자 변경
RUN chown appuser:appuser app.jar

# 사용자 전환
USER appuser

# 헬스체크 추가
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# 애플리케이션 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### 문제: Kubernetes 배포 실패
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: app
        image: spring-boot-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### 환경별 설정 문제

#### 문제: 환경별 설정이 제대로 적용되지 않음
```java
// ConfigMap을 통한 외부 설정
@Component
@ConfigurationProperties(prefix = "app")
@Data
public class AppProperties {
    private String environment;
    private Database database = new Database();
    private External external = new External();
    
    @Data
    public static class Database {
        private String url;
        private String username;
        private String password;
        private int maxPoolSize;
    }
    
    @Data
    public static class External {
        private String apiUrl;
        private String apiKey;
        private int timeout;
    }
}
```

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yml: |
    app:
      environment: production
      database:
        url: jdbc:postgresql://prod-db:5432/mydb
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}
        max-pool-size: 20
      external:
        api-url: https://api.production.com
        api-key: ${API_KEY}
        timeout: 5000
    
    logging:
      level:
        com.example: INFO
        org.springframework.security: DEBUG
```

## 📊 로깅 및 모니터링 문제

### 로깅 설정 문제

#### 문제: 로그가 출력되지 않음
```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="!prod">
        <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
        <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
        
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
        
        <logger name="com.example" level="DEBUG"/>
        <logger name="org.springframework.security" level="DEBUG"/>
    </springProfile>
    
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>/var/log/app/application.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>/var/log/app/application.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
                <maxFileSize>100MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>3GB</totalSizeCap>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        
        <root level="INFO">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
</configuration>
```

#### 문제: 구조화된 로깅이 필요함
```java
// 구조화된 로깅
@Component
@RequiredArgsConstructor
@Slf4j
public class StructuredLogger {
    
    private final ObjectMapper objectMapper;
    
    public void logUserAction(String userId, String action, Object details) {
        try {
            Map<String, Object> logEntry = Map.of(
                "timestamp", Instant.now().toString(),
                "userId", userId,
                "action", action,
                "details", details,
                "traceId", getCurrentTraceId()
            );
            
            log.info("USER_ACTION: {}", objectMapper.writeValueAsString(logEntry));
        } catch (Exception e) {
            log.error("Failed to log user action", e);
        }
    }
    
    private String getCurrentTraceId() {
        // Sleuth나 Micrometer Tracing을 통한 TraceId 획득
        return "trace-id";
    }
}
```

### 메트릭 수집 문제

#### 문제: Actuator 엔드포인트가 노출되지 않음
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      show-components: always
    metrics:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
        resilience4j.circuitbreaker.calls: 0.5, 0.95, 0.99
```

```java
// 커스텀 메트릭 추가
@Component
@RequiredArgsConstructor
public class CustomMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;
    
    public CustomMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.orderCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .register(meterRegistry);
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing time")
            .register(meterRegistry);
    }
    
    public void recordOrderCreated(String orderType) {
        orderCounter.increment(Tags.of("type", orderType));
    }
    
    public void recordOrderProcessingTime(Duration duration) {
        orderProcessingTimer.record(duration);
    }
}
```

### 분산 추적 문제

#### 문제: 마이크로서비스 간 요청 추적이 안됨
```java
// Spring Cloud Sleuth 설정
@Configuration
public class TracingConfig {
    
    @Bean
    public Sender sender() {
        return OkHttpSender.create("http://zipkin:9411/api/v2/spans");
    }
    
    @Bean
    public AsyncReporter<Span> spanReporter() {
        return AsyncReporter.create(sender());
    }
    
    @Bean
    public Tracer tracer() {
        return Tracing.newBuilder()
            .localServiceName("spring-boot-app")
            .spanReporter(spanReporter())
            .sampler(Sampler.create(1.0f)) // 100% 샘플링 (개발용)
            .build()
            .tracer();
    }
}
```

```java
// 수동 트레이싱
@Service
@RequiredArgsConstructor
public class TracedService {
    
    private final Tracer tracer;
    
    public String processOrder(String orderId) {
        Span span = tracer.nextSpan()
            .name("process-order")
            .tag("order.id", orderId)
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // 비즈니스 로직 실행
            return doProcessOrder(orderId);
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    private String doProcessOrder(String orderId) {
        // 실제 처리 로직
        return "processed";
    }
}
```

## 🔧 일반적인 문제 해결 방법

### 디버깅 도구 활용

```java
// 1. Spring Boot DevTools 활용
dependencies {
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
}

// 2. Actuator를 통한 런타임 정보 확인
@RestController
public class DebugController {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    @GetMapping("/debug/beans")
    public String[] getBeans() {
        return applicationContext.getBeanDefinitionNames();
    }
    
    @GetMapping("/debug/config")
    public Map<String, Object> getConfig(Environment env) {
        Map<String, Object> config = new HashMap<>();
        config.put("activeProfiles", Arrays.toString(env.getActiveProfiles()));
        config.put("defaultProfiles", Arrays.toString(env.getDefaultProfiles()));
        return config;
    }
}
```

### 성능 프로파일링

```java
// JProfiler, VisualVM 연동
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

// 또는 Flight Recorder 사용
-XX:+FlightRecorder
-XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr
```

### 로그 분석 도구

```bash
# ELK Stack을 통한 로그 분석
# Logstash 설정 예제
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][service] == "spring-boot-app" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{DATA:thread}\] %{LOGLEVEL:level} %{DATA:logger} - %{GREEDYDATA:message}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "spring-boot-logs-%{+YYYY.MM.dd}"
  }
}
```

## 📚 관련 노트

- [[Spring Boot Project Setup]] - 프로젝트 초기 설정
- [[Spring Boot Performance Tuning]] - 성능 최적화
- [[Spring Security 6]] - 보안 설정
- [[Testing]] - 테스트 전략
- [[Observability]] - 모니터링 및 로깅

## 🔗 유용한 리소스

- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Boot Common Application Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html)
- [Spring Boot Troubleshooting Guide](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html)
- [Stack Overflow Spring Boot Tags](https://stackoverflow.com/questions/tagged/spring-boot)

---

*문제 해결은 체계적인 접근이 중요합니다. 로그를 자세히 확인하고, 단계별로 원인을 분석해 나가세요.*
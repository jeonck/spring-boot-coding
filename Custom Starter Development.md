# Custom Starter Development

## 개념
- 사용자 정의 Spring Boot Starter 제작
- [[Spring Boot Starter]]의 확장 개념
- [[Auto Configuration]]과 완전 통합
- 재사용 가능한 모듈화 패턴

## 핵심 구성 요소
- AutoConfiguration 클래스
- Properties 바인딩
- 조건부 Bean 등록
- Starter POM 설정

## 1. Starter 프로젝트 구조
### 표준 디렉토리 레이아웃
```
custom-notification-spring-boot-starter/
├── src/
│   └── main/
│       ├── java/
│       │   └── com/
│       │       └── company/
│       │           └── notification/
│       │               ├── autoconfigure/
│       │               │   ├── NotificationAutoConfiguration.java
│       │               │   ├── NotificationProperties.java
│       │               │   └── ConditionalOnNotificationEnabled.java
│       │               ├── service/
│       │               │   ├── NotificationService.java
│       │               │   ├── EmailNotificationProvider.java
│       │               │   ├── SmsNotificationProvider.java
│       │               │   └── PushNotificationProvider.java
│       │               └── model/
│       │                   ├── NotificationMessage.java
│       │                   ├── NotificationType.java
│       │                   └── NotificationResult.java
│       └── resources/
│           └── META-INF/
│               ├── spring.factories
│               ├── spring-configuration-metadata.json
│               └── additional-spring-configuration-metadata.json
├── pom.xml
└── README.md
```

## 2. Maven POM 설정
### Starter POM 구성
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.company</groupId>
    <artifactId>notification-spring-boot-starter</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>Notification Spring Boot Starter</name>
    <description>Spring Boot Starter for unified notification service</description>

    <properties>
        <java.version>21</java.version>
        <spring-boot.version>3.5.0</spring-boot.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Spring Boot Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>${spring-boot.version}</version>
        </dependency>
        
        <!-- Configuration Processor -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <version>${spring-boot.version}</version>
            <optional>true</optional>
        </dependency>
        
        <!-- AutoConfigure -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>${spring-boot.version}</version>
        </dependency>
        
        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
            <version>${spring-boot.version}</version>
        </dependency>
        
        <!-- Web (선택적) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring-boot.version}</version>
            <optional>true</optional>
        </dependency>
        
        <!-- Email 지원 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
            <version>${spring-boot.version}</version>
            <optional>true</optional>
        </dependency>
        
        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <version>${spring-boot.version}</version>
            <scope>test</scope>
        </dependency>
        
        <!-- Test Autoconfiguration -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure-processor</artifactId>
            <version>${spring-boot.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## 3. Configuration Properties
### 타입 안전한 설정 클래스
```java
@ConfigurationProperties(prefix = "notification")
@Validated
@Data
@AllArgsConstructor
@NoArgsConstructor
public class NotificationProperties {
    
    /**
     * 알림 서비스 활성화 여부
     */
    private boolean enabled = true;
    
    /**
     * 기본 알림 제공자
     */
    @NotNull
    private ProviderType defaultProvider = ProviderType.EMAIL;
    
    /**
     * 비동기 처리 여부
     */
    private boolean async = true;
    
    /**
     * 재시도 설정
     */
    private Retry retry = new Retry();
    
    /**
     * 이메일 설정
     */
    private Email email = new Email();
    
    /**
     * SMS 설정
     */
    private Sms sms = new Sms();
    
    /**
     * 푸시 알림 설정
     */
    private Push push = new Push();
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class Retry {
        /**
         * 최대 재시도 횟수
         */
        @Min(0)
        @Max(10)
        private int maxAttempts = 3;
        
        /**
         * 재시도 간격 (밀리초)
         */
        @Min(100)
        @Max(60000)
        private long delay = 1000;
        
        /**
         * 백오프 배율
         */
        @DecimalMin("1.0")
        @DecimalMax("10.0")
        private double multiplier = 2.0;
    }
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class Email {
        /**
         * 이메일 서비스 활성화
         */
        private boolean enabled = true;
        
        /**
         * SMTP 서버 호스트
         */
        private String host = "localhost";
        
        /**
         * SMTP 서버 포트
         */
        @Min(1)
        @Max(65535)
        private int port = 587;
        
        /**
         * 사용자명
         */
        private String username;
        
        /**
         * 비밀번호
         */
        private String password;
        
        /**
         * 기본 발신자 이메일
         */
        @Email
        private String from = "noreply@company.com";
        
        /**
         * SSL 사용 여부
         */
        private boolean ssl = true;
        
        /**
         * TLS 사용 여부
         */
        private boolean tls = true;
    }
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class Sms {
        /**
         * SMS 서비스 활성화
         */
        private boolean enabled = false;
        
        /**
         * SMS 제공자
         */
        private SmsProvider provider = SmsProvider.TWILIO;
        
        /**
         * API 키
         */
        private String apiKey;
        
        /**
         * API 시크릿
         */
        private String apiSecret;
        
        /**
         * 기본 발신 번호
         */
        private String from;
    }
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class Push {
        /**
         * 푸시 알림 활성화
         */
        private boolean enabled = false;
        
        /**
         * FCM 프로젝트 ID
         */
        private String fcmProjectId;
        
        /**
         * FCM 서비스 계정 키 파일 경로
         */
        private String fcmServiceAccountKeyPath;
        
        /**
         * APNS 키 ID
         */
        private String apnsKeyId;
        
        /**
         * APNS 팀 ID
         */
        private String apnsTeamId;
        
        /**
         * APNS 키 파일 경로
         */
        private String apnsKeyPath;
    }
    
    public enum ProviderType {
        EMAIL, SMS, PUSH
    }
    
    public enum SmsProvider {
        TWILIO, AWS_SNS, AZURE_SMS
    }
}
```

## 4. Auto Configuration
### 핵심 자동 설정 클래스
```java
@Configuration
@EnableConfigurationProperties(NotificationProperties.class)
@ConditionalOnProperty(prefix = "notification", name = "enabled", havingValue = "true", matchIfMissing = true)
@AutoConfigureAfter({MailSenderAutoConfiguration.class, WebMvcAutoConfiguration.class})
@Slf4j
public class NotificationAutoConfiguration {
    
    private final NotificationProperties properties;
    
    public NotificationAutoConfiguration(NotificationProperties properties) {
        this.properties = properties;
    }
    
    // 핵심 알림 서비스
    @Bean
    @ConditionalOnMissingBean
    public NotificationService notificationService(List<NotificationProvider> providers) {
        log.info("Creating NotificationService with {} providers", providers.size());
        return new NotificationServiceImpl(providers, properties);
    }
    
    // 이메일 제공자
    @Bean
    @ConditionalOnProperty(prefix = "notification.email", name = "enabled", havingValue = "true")
    @ConditionalOnBean(JavaMailSender.class)
    public EmailNotificationProvider emailNotificationProvider(JavaMailSender mailSender) {
        log.info("Creating EmailNotificationProvider");
        return new EmailNotificationProvider(mailSender, properties.getEmail());
    }
    
    // SMS 제공자
    @Bean
    @ConditionalOnProperty(prefix = "notification.sms", name = "enabled", havingValue = "true")
    @ConditionalOnClass(name = "com.twilio.Twilio")
    public SmsNotificationProvider smsNotificationProvider() {
        log.info("Creating SmsNotificationProvider with provider: {}", properties.getSms().getProvider());
        return new SmsNotificationProvider(properties.getSms());
    }
    
    // 푸시 알림 제공자
    @Bean
    @ConditionalOnProperty(prefix = "notification.push", name = "enabled", havingValue = "true")
    @ConditionalOnClass(name = "com.google.firebase.messaging.FirebaseMessaging")
    public PushNotificationProvider pushNotificationProvider() {
        log.info("Creating PushNotificationProvider");
        return new PushNotificationProvider(properties.getPush());
    }
    
    // 비동기 실행기
    @Bean
    @ConditionalOnProperty(prefix = "notification", name = "async", havingValue = "true")
    @ConditionalOnMissingBean(name = "notificationTaskExecutor")
    public TaskExecutor notificationTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("notification-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        
        log.info("Created notification task executor with core pool size: {}", 5);
        return executor;
    }
    
    // 재시도 템플릿
    @Bean
    @ConditionalOnMissingBean
    public RetryTemplate notificationRetryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();
        
        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
        backOffPolicy.setBackOffPeriod(properties.getRetry().getDelay());
        
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(properties.getRetry().getMaxAttempts());
        
        retryTemplate.setBackOffPolicy(backOffPolicy);
        retryTemplate.setRetryPolicy(retryPolicy);
        
        log.info("Created retry template with max attempts: {} and delay: {}ms", 
            properties.getRetry().getMaxAttempts(), properties.getRetry().getDelay());
        
        return retryTemplate;
    }
    
    // 헬스 인디케이터
    @Bean
    @ConditionalOnClass(HealthIndicator.class)
    @ConditionalOnProperty(prefix = "notification", name = "health.enabled", havingValue = "true", matchIfMissing = true)
    public NotificationHealthIndicator notificationHealthIndicator(NotificationService notificationService) {
        return new NotificationHealthIndicator(notificationService);
    }
    
    // 메트릭
    @Bean
    @ConditionalOnClass(MeterRegistry.class)
    @ConditionalOnProperty(prefix = "notification", name = "metrics.enabled", havingValue = "true", matchIfMissing = true)
    public NotificationMetrics notificationMetrics(MeterRegistry meterRegistry) {
        return new NotificationMetrics(meterRegistry);
    }
    
    // 웹 자동 설정
    @Configuration
    @ConditionalOnWebApplication
    @ConditionalOnClass({DispatcherServlet.class, WebMvcConfigurer.class})
    @ConditionalOnProperty(prefix = "notification.web", name = "enabled", havingValue = "true", matchIfMissing = true)
    static class NotificationWebConfiguration {
        
        @Bean
        public NotificationController notificationController(NotificationService notificationService) {
            return new NotificationController(notificationService);
        }
        
        @Bean
        public NotificationWebMvcConfigurer notificationWebMvcConfigurer() {
            return new NotificationWebMvcConfigurer();
        }
    }
}
```

## 5. 조건부 어노테이션
### 커스텀 조건부 설정
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnNotificationEnabledCondition.class)
public @interface ConditionalOnNotificationEnabled {
    
    /**
     * 알림 제공자 타입
     */
    NotificationProperties.ProviderType[] value() default {};
    
    /**
     * 매칭 전략
     */
    MatchStrategy matchStrategy() default MatchStrategy.ANY;
    
    enum MatchStrategy {
        ANY, ALL
    }
}

public class OnNotificationEnabledCondition extends SpringBootCondition {
    
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        ConditionMessage.Builder message = ConditionMessage.forCondition("NotificationEnabled");
        
        Map<String, Object> attributes = metadata.getAnnotationAttributes(
            ConditionalOnNotificationEnabled.class.getName());
        
        if (attributes == null) {
            return ConditionOutcome.match(message.because("@ConditionalOnNotificationEnabled annotation not present"));
        }
        
        NotificationProperties.ProviderType[] providers = 
            (NotificationProperties.ProviderType[]) attributes.get("value");
        ConditionalOnNotificationEnabled.MatchStrategy strategy = 
            (ConditionalOnNotificationEnabled.MatchStrategy) attributes.get("matchStrategy");
        
        Environment environment = context.getEnvironment();
        boolean notificationEnabled = environment.getProperty("notification.enabled", Boolean.class, true);
        
        if (!notificationEnabled) {
            return ConditionOutcome.noMatch(message.because("notification.enabled is false"));
        }
        
        if (providers.length == 0) {
            return ConditionOutcome.match(message.because("notification is enabled"));
        }
        
        boolean anyMatch = false;
        boolean allMatch = true;
        
        for (NotificationProperties.ProviderType provider : providers) {
            String propertyKey = String.format("notification.%s.enabled", provider.name().toLowerCase());
            boolean providerEnabled = environment.getProperty(propertyKey, Boolean.class, false);
            
            if (providerEnabled) {
                anyMatch = true;
            } else {
                allMatch = false;
            }
        }
        
        boolean matches = (strategy == ConditionalOnNotificationEnabled.MatchStrategy.ANY) ? anyMatch : allMatch;
        
        if (matches) {
            return ConditionOutcome.match(message.foundExactly("enabled providers"));
        } else {
            return ConditionOutcome.noMatch(message.because("required providers not enabled"));
        }
    }
}
```

## 6. Spring Factories 등록
### META-INF/spring.factories
```properties
# Auto Configuration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.company.notification.autoconfigure.NotificationAutoConfiguration

# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
com.company.notification.autoconfigure.NotificationFailureAnalyzer

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
com.company.notification.autoconfigure.NotificationEnvironmentPostProcessor
```

## 7. Configuration Metadata
### spring-configuration-metadata.json
```json
{
  "groups": [
    {
      "name": "notification",
      "type": "com.company.notification.autoconfigure.NotificationProperties",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties"
    },
    {
      "name": "notification.email",
      "type": "com.company.notification.autoconfigure.NotificationProperties$Email",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties",
      "sourceMethod": "getEmail()"
    },
    {
      "name": "notification.sms",
      "type": "com.company.notification.autoconfigure.NotificationProperties$Sms",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties",
      "sourceMethod": "getSms()"
    }
  ],
  "properties": [
    {
      "name": "notification.enabled",
      "type": "java.lang.Boolean",
      "description": "알림 서비스 활성화 여부",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties",
      "defaultValue": true
    },
    {
      "name": "notification.default-provider",
      "type": "com.company.notification.autoconfigure.NotificationProperties$ProviderType",
      "description": "기본 알림 제공자",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties",
      "defaultValue": "email"
    },
    {
      "name": "notification.async",
      "type": "java.lang.Boolean",
      "description": "비동기 처리 여부",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties",
      "defaultValue": true
    },
    {
      "name": "notification.email.enabled",
      "type": "java.lang.Boolean",
      "description": "이메일 알림 활성화",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties$Email",
      "defaultValue": true
    },
    {
      "name": "notification.email.host",
      "type": "java.lang.String",
      "description": "SMTP 서버 호스트",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties$Email",
      "defaultValue": "localhost"
    },
    {
      "name": "notification.email.port",
      "type": "java.lang.Integer",
      "description": "SMTP 서버 포트",
      "sourceType": "com.company.notification.autoconfigure.NotificationProperties$Email",
      "defaultValue": 587
    }
  ],
  "hints": [
    {
      "name": "notification.default-provider",
      "values": [
        {
          "value": "email",
          "description": "이메일을 기본 제공자로 사용"
        },
        {
          "value": "sms",
          "description": "SMS를 기본 제공자로 사용"
        },
        {
          "value": "push",
          "description": "푸시 알림을 기본 제공자로 사용"
        }
      ]
    },
    {
      "name": "notification.sms.provider",
      "values": [
        {
          "value": "twilio",
          "description": "Twilio SMS 서비스 사용"
        },
        {
          "value": "aws_sns",
          "description": "AWS SNS 서비스 사용"
        },
        {
          "value": "azure_sms",
          "description": "Azure Communication Services 사용"
        }
      ]
    }
  ]
}
```

## 8. 서비스 구현
### 핵심 알림 서비스 인터페이스
```java
public interface NotificationService {
    
    /**
     * 알림 전송
     * @param message 알림 메시지
     * @return 전송 결과
     */
    CompletableFuture<NotificationResult> send(NotificationMessage message);
    
    /**
     * 벌크 알림 전송
     * @param messages 알림 메시지 목록
     * @return 전송 결과 목록
     */
    CompletableFuture<List<NotificationResult>> sendBulk(List<NotificationMessage> messages);
    
    /**
     * 특정 제공자로 알림 전송
     * @param message 알림 메시지
     * @param providerType 제공자 타입
     * @return 전송 결과
     */
    CompletableFuture<NotificationResult> sendWithProvider(
        NotificationMessage message, 
        NotificationProperties.ProviderType providerType);
    
    /**
     * 사용 가능한 제공자 목록
     * @return 제공자 타입 목록
     */
    List<NotificationProperties.ProviderType> getAvailableProviders();
    
    /**
     * 제공자 상태 확인
     * @param providerType 제공자 타입
     * @return 상태 정보
     */
    ProviderStatus getProviderStatus(NotificationProperties.ProviderType providerType);
}

// 구현체
@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationServiceImpl implements NotificationService {
    
    private final List<NotificationProvider> providers;
    private final NotificationProperties properties;
    private final NotificationMetrics metrics;
    private final RetryTemplate retryTemplate;
    
    @Async("notificationTaskExecutor")
    @Override
    public CompletableFuture<NotificationResult> send(NotificationMessage message) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                metrics.incrementSendAttempt(message.getType());
                
                NotificationResult result = retryTemplate.execute(context -> {
                    NotificationProvider provider = selectProvider(message.getType());
                    return provider.send(message);
                });
                
                if (result.isSuccess()) {
                    metrics.incrementSendSuccess(message.getType());
                } else {
                    metrics.incrementSendFailure(message.getType());
                }
                
                return result;
                
            } catch (Exception e) {
                log.error("Failed to send notification: {}", e.getMessage(), e);
                metrics.incrementSendError(message.getType());
                return NotificationResult.failure(e.getMessage());
            }
        });
    }
    
    private NotificationProvider selectProvider(NotificationType type) {
        return providers.stream()
            .filter(provider -> provider.supports(type))
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("No provider found for type: " + type));
    }
}
```

## 9. 테스트 지원
### 자동 설정 테스트
```java
@TestConfiguration
public class NotificationTestAutoConfiguration {
    
    @Bean
    @Primary
    public NotificationService mockNotificationService() {
        return Mockito.mock(NotificationService.class);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public TestNotificationProvider testNotificationProvider() {
        return new TestNotificationProvider();
    }
}

// 테스트용 어노테이션
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@ImportAutoConfiguration(NotificationTestAutoConfiguration.class)
@TestPropertySource(properties = {
    "notification.enabled=true",
    "notification.email.enabled=false",
    "notification.sms.enabled=false",
    "notification.push.enabled=false"
})
public @interface EnableNotificationTest {
}

// 사용 예시
@SpringBootTest
@EnableNotificationTest
class NotificationServiceTest {
    
    @Autowired
    private NotificationService notificationService;
    
    @Test
    void shouldSendEmailNotification() {
        // 테스트 코드
    }
}
```

## 10. 사용법 및 문서화
### README.md 예시
```markdown
# Notification Spring Boot Starter

이메일, SMS, 푸시 알림을 위한 통합 Spring Boot Starter입니다.

## 설치

```xml
<dependency>
    <groupId>com.company</groupId>
    <artifactId>notification-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

## 설정

```yaml
notification:
  enabled: true
  default-provider: email
  async: true
  
  email:
    enabled: true
    host: smtp.gmail.com
    port: 587
    username: your-email@gmail.com
    password: your-password
    from: noreply@company.com
    
  sms:
    enabled: false
    provider: twilio
    api-key: your-twilio-key
    
  retry:
    max-attempts: 3
    delay: 1000
```

## 사용법

```java
@Service
@RequiredArgsConstructor
public class UserService {
    
    private final NotificationService notificationService;
    
    public void registerUser(User user) {
        // 사용자 등록 로직
        
        // 환영 이메일 발송
        NotificationMessage message = NotificationMessage.builder()
            .type(NotificationType.EMAIL)
            .recipient(user.getEmail())
            .subject("환영합니다!")
            .content("회원가입을 축하드립니다.")
            .build();
            
        notificationService.send(message);
    }
}
```
```

## 관련 개념
- [[Spring Boot Starter]]
- [[Auto Configuration]]
- [[Configuration Properties]]
- [[Component Scanning]]
- [[Spring Bean Management]]
- [[Enterprise Design Patterns]]

#custom-starter #auto-configuration #spring-boot #modular-design

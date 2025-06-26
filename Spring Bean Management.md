# Spring Bean Management

## 개념
- Spring Container가 Bean의 생명주기를 관리
- [[Spring IoC와 DI]]의 핵심 기능
- [[Auto Configuration]]과 연동하여 자동 관리

## Bean 생명주기
```java
1. Bean 정의 스캔 (Component Scanning or Configuration)
2. Bean 인스턴스 생성
3. 의존성 주입 (Dependency Injection)
4. 초기화 콜백 (@PostConstruct)
5. Bean 사용
6. 소멸 콜백 (@PreDestroy)
```

## Bean 스코프
### Singleton (기본값)
```java
@Service  // 기본적으로 Singleton
public class ItemService {
    // Spring Container에서 하나의 인스턴스만 관리
}
```

### Prototype
```java
@Service
@Scope("prototype")
public class TaskProcessor {
    // 요청할 때마다 새로운 인스턴스 생성
}
```

### 웹 관련 스코프
```java
@Service
@Scope("request")  // HTTP 요청마다 새 인스턴스
public class RequestScopedService { ... }

@Service
@Scope("session")  // HTTP 세션마다 새 인스턴스
public class SessionScopedService { ... }
```

## 초기화와 소멸 콜백
```java
@Service
public class DatabaseService {
    
    @PostConstruct
    public void init() {
        // Bean 초기화 후 실행
        System.out.println("Database connection initialized");
    }
    
    @PreDestroy
    public void cleanup() {
        // Bean 소멸 전 실행
        System.out.println("Database connection closed");
    }
}
```

## Bean 조건부 등록
```java
@Service
@ConditionalOnProperty(name = "feature.email.enabled", havingValue = "true")
public class EmailService {
    // feature.email.enabled=true일 때만 Bean 등록
}

@Service
@ConditionalOnMissingBean(EmailService.class)
public class MockEmailService {
    // EmailService Bean이 없을 때만 등록
}
```

## Bean 우선순위
```java
@Primary  // 같은 타입의 Bean이 여러 개일 때 우선 선택
@Service
public class PrimaryEmailService implements EmailService { ... }

@Service
@Qualifier("backup")  // 명시적으로 특정 Bean 선택
public class BackupEmailService implements EmailService { ... }

// 사용 시
@Autowired
@Qualifier("backup")
private EmailService emailService;
```

## Lazy 초기화
```java
@Service
@Lazy  // 실제 사용될 때까지 Bean 생성 지연
public class HeavyService {
    public HeavyService() {
        System.out.println("Heavy service created");
    }
}
```

## 프로파일별 Bean 관리
```java
@Service
@Profile("dev")
public class DevDatabaseService implements DatabaseService { ... }

@Service
@Profile("prod")
public class ProdDatabaseService implements DatabaseService { ... }
```

## Bean 팩토리 메서드
```java
@Configuration
public class ServiceConfig {
    
    @Bean
    public EmailService emailService(@Value("${email.provider}") String provider) {
        switch (provider) {
            case "gmail":
                return new GmailService();
            case "sendgrid":
                return new SendGridService();
            default:
                return new MockEmailService();
        }
    }
}
```

## 실제 프로젝트에서의 Bean 관리
```java
// 1. 자동 등록된 Bean들
@Repository
public interface ItemRepository extends JpaRepository<Item, Long> {
    // Spring Data JPA가 자동으로 구현체 생성하여 Bean 등록
}

@Service
public class ItemService {
    // @Service로 자동 Bean 등록
    // Singleton 스코프로 하나의 인스턴스만 관리
}

// 2. 설정으로 등록된 Bean들
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI().info(new Info()
                .title("Spring Boot Boilerplate API")
                .version("1.0.0"));
    }
}
```

## Bean 순환 참조 해결
```java
// ❌ 순환 참조 발생
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;
}

@Service  
public class ServiceB {
    @Autowired
    private ServiceA serviceA;  // 순환 참조!
}

// ✅ 해결 방법 1: @Lazy 사용
@Service
public class ServiceA {
    @Autowired
    @Lazy
    private ServiceB serviceB;
}

// ✅ 해결 방법 2: 설계 개선
@Service
public class ServiceA {
    @Autowired
    private CommonService commonService;
}

@Service
public class ServiceB {
    @Autowired  
    private CommonService commonService;
}
```

## 관련 개념
- [[Spring IoC와 DI]]
- [[Component Scanning]]
- [[Configuration Classes]]
- [[Dependency Injection Patterns]]
- [[Profile-based Configuration]]

#bean-lifecycle #singleton #prototype #spring-container
# Dependency Injection Patterns

## 개념
- [[Spring IoC와 DI]]의 구체적인 구현 패턴
- 객체 간 의존성을 외부에서 주입하는 방식들
- [[Spring Boot 3.5 개요]]에서 사용되는 핵심 패턴

## 주입 방식의 종류

### 1. 생성자 주입 (Constructor Injection) - 권장
```java
@RestController
public class ItemController {
    private final ItemService itemService;  // final로 불변성 보장

    @Autowired  // 생성자가 하나면 생략 가능
    public ItemController(ItemService itemService) {
        this.itemService = itemService;
    }
}
```

**장점:**
- 불변성 보장 (final 키워드)
- 필수 의존성 보장
- 테스트 용이성
- 순환 참조 조기 발견

### 2. 세터 주입 (Setter Injection) - 선택적 의존성
```java
@Service
public class ItemService {
    private EmailService emailService;  // 선택적 의존성

    @Autowired(required = false)
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

### 3. 필드 주입 (Field Injection) - 비권장
```java
@Service
public class ItemService {
    @Autowired
    private ItemRepository itemRepository;  // 테스트하기 어려움
}
```

## 의존성 주입 체인 분석
```java
// 완전한 의존성 체인 예시
@RestController
public class ItemController {
    private final ItemService itemService;  // ← Spring이 주입

    @GetMapping
    public ResponseEntity<List<Item>> getAllItems() {
        List<Item> items = itemService.getAllItems();  // ← Service 계층 호출
        return ResponseEntity.ok(items);
    }
}

@Service  
public class ItemService {
    private final ItemRepository itemRepository;  // ← Spring이 주입
    
    public List<Item> getAllItems() {
        return itemRepository.findAll();  // ← Repository 계층 호출
    }
}
```

## 인터페이스 기반 의존성
```java
// Repository는 인터페이스만 정의
@Repository
public interface ItemRepository extends JpaRepository<Item, Long> {
    List<Item> findByActiveTrue();
}

// Service는 구체적인 구현체를 몰라도 됨
@Service
public class ItemService {
    private final ItemRepository itemRepository;  // ← 인터페이스 타입
    
    public ItemService(ItemRepository itemRepository) {
        this.itemRepository = itemRepository;  // Spring이 실제 구현체(Proxy) 주입
    }
}
```

## 조건부 주입
```java
@Service
public class NotificationService {
    private final List<NotificationProvider> providers;
    
    // 같은 타입의 여러 Bean들을 List로 주입
    public NotificationService(List<NotificationProvider> providers) {
        this.providers = providers;
    }
}

@Primary  // 기본 구현체 지정
@Service
public class EmailNotificationProvider implements NotificationProvider { ... }

@Service
public class SmsNotificationProvider implements NotificationProvider { ... }
```

## 프로파일별 주입
```java
@Service
@Profile("dev")
public class MockEmailService implements EmailService { ... }

@Service
@Profile("prod")
public class RealEmailService implements EmailService { ... }
```

## 테스트에서의 Mock 주입
```java
@Test
class ItemServiceTest {
    @Mock
    ItemRepository mockRepository;  // 가짜 구현체
    
    @InjectMocks
    ItemService itemService;  // Mock이 주입됨
    
    @Test
    void testGetAllItems() {
        when(mockRepository.findAll()).thenReturn(mockItems);
        
        List<Item> items = itemService.getAllItems();
        // 실제 DB 없이도 테스트 가능!
    }
}
```

## 설정값 주입
```java
@Service
public class ItemService {
    
    @Value("${app.items.default-active:true}")  // 설정값 주입
    private boolean defaultActive;
    
    public Item createItem(String name) {
        return Item.builder()
            .name(name)
            .active(defaultActive)  // 외부 설정값 사용
            .build();
    }
}
```

## Bean 관계도
```
Spring IoC Container
├── ItemController (Singleton)
│   └── depends on → ItemService
├── ItemService (Singleton) 
│   └── depends on → ItemRepository
├── ItemRepository (Singleton, JPA Proxy)
├── OpenAPI (Singleton)
└── CommandLineRunner (Singleton)
    └── depends on → ItemRepository
```

## 관련 개념
- [[Spring IoC와 DI]]
- [[Component Scanning]]
- [[Configuration Classes]]
- [[Testing]] Mock 활용
- [[Spring Bean Management]]

#dependency-injection #constructor-injection #interface-based #loose-coupling
# Spring IoC DI 실습 예제

## 프로젝트 구조 분석
실제 Spring Boot 프로젝트에서 [[Spring IoC와 DI]]가 어떻게 구현되는지 단계별로 분석

## 1. 메인 애플리케이션 - IoC 컨테이너 시작
```java
@SpringBootApplication  // ← IoC 컨테이너 활성화
public class BoilerplateApplication {
    public static void main(String[] args) {
        SpringApplication.run(BoilerplateApplication.class, args);  // Spring 컨테이너 생성
    }
}
```

## 2. 의존성 주입 체인 구현

### Controller → Service → Repository
```java
// Controller 계층
@RestController
@RequestMapping("/api/items")
public class ItemController {
    private final ItemService itemService;  // final로 불변성 보장

    @Autowired  // 생성자 주입
    public ItemController(ItemService itemService) {
        this.itemService = itemService;  // Spring이 ItemService Bean 주입
    }
    
    @GetMapping
    public ResponseEntity<List<Item>> getAllItems() {
        List<Item> items = itemService.getAllItems();
        return ResponseEntity.ok(items);
    }
}

// Service 계층
@Service
public class ItemService {
    private final ItemRepository itemRepository;

    @Autowired
    public ItemService(ItemRepository itemRepository) {
        this.itemRepository = itemRepository;  // Spring이 Repository Bean 주입
    }
    
    public List<Item> getAllItems() {
        return itemRepository.findAll();
    }
}

// Repository 계층
@Repository
public interface ItemRepository extends JpaRepository<Item, Long> {
    List<Item> findByActiveTrue();
    // Spring Data JPA가 구현체 자동 생성
}
```

## 3. Bean 등록 방식 비교

### A. 스테레오타입 어노테이션 (자동 등록)
```java
@RestController  // = @Controller + @ResponseBody
public class ItemController { ... }

@Service  // = @Component + 비즈니스 계층 명시
public class ItemService { ... }

@Repository  // = @Component + 데이터 접근 계층 명시
public interface ItemRepository { ... }
```

### B. Configuration 클래스 (수동 등록)
```java
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Spring Boot Boilerplate API")
                        .version("1.0.0"));
    }
}

@Configuration
public class DataInitializer {
    
    @Bean
    public CommandLineRunner initData(ItemRepository itemRepository) {
        return args -> {
            if (itemRepository.count() == 0) {
                // 초기 데이터 생성
                itemRepository.save(Item.builder()
                        .name("Sample Item 1")
                        .description("This is the first sample item")
                        .active(true)
                        .build());
            }
        };
    }
}
```

## 4. 의존성 주입 흐름도
```
Spring Container 시작
    ↓
패키지 스캔 (@SpringBootApplication)
    ↓
Bean 후보 발견 (@Service, @Repository, @Configuration)
    ↓
의존성 분석 (ItemRepository → ItemService → ItemController)
    ↓
Bean 생성 및 의존성 주입
    ↓
애플리케이션 준비 완료
```

## 5. 런타임 요청 처리 과정
```java
// HTTP GET /api/items 요청 시:
// 1. DispatcherServlet이 ItemController Bean 찾기
// 2. ItemController.getAllItems() 호출
// 3. 주입된 ItemService.getAllItems() 호출
// 4. 주입된 ItemRepository.findAll() 호출
// 5. 결과 반환

// 모든 객체는 Spring이 관리하는 싱글톤!
```

## 6. 테스트에서의 Mock 주입
```java
@ExtendWith(MockitoExtension.class)
class ItemServiceTest {
    
    @Mock
    ItemRepository mockRepository;  // 가짜 구현체
    
    @InjectMocks
    ItemService itemService;  // Mock이 주입됨
    
    @Test
    void testGetAllItems() {
        // given
        List<Item> mockItems = Arrays.asList(
            Item.builder().name("Test Item").build()
        );
        when(mockRepository.findAll()).thenReturn(mockItems);
        
        // when
        List<Item> result = itemService.getAllItems();
        
        // then
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("Test Item");
    }
}
```

## 7. 설정값 외부화 예제
```java
@Service
public class ItemService {
    
    @Value("${app.items.default-active:true}")
    private boolean defaultActive;  // application.properties에서 주입
    
    public Item createItem(String name, String description) {
        return Item.builder()
            .name(name)
            .description(description)
            .active(defaultActive)  // 외부 설정값 사용
            .build();
    }
}
```

## 8. Bean 생명주기 콜백
```java
@Service
public class DatabaseService {
    
    @PostConstruct
    public void init() {
        System.out.println("DatabaseService 초기화 완료");
        // 데이터베이스 연결 풀 초기화 등
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("DatabaseService 정리 작업");
        // 리소스 정리
    }
}
```

## 9. 전체 Bean 관계도
```
Spring IoC Container
├── ItemController (Singleton)
│   └── depends on → ItemService
├── ItemService (Singleton) 
│   └── depends on → ItemRepository
├── ItemRepository (Singleton, JPA Proxy)
├── OpenAPI (Singleton)
├── CommandLineRunner (Singleton)
│   └── depends on → ItemRepository
└── DatabaseService (Singleton)
```

## 핵심 포인트 정리
1. **제어의 역전**: Spring이 객체 생성과 관계 설정 담당
2. **생성자 주입**: 불변성과 필수 의존성 보장
3. **인터페이스 기반**: 느슨한 결합으로 테스트 용이성 향상
4. **싱글톤 관리**: Spring이 Bean 생명주기 관리
5. **설정 중앙화**: Configuration 클래스로 설정 집중 관리

## 관련 노트
- [[Spring IoC와 DI]]
- [[Component Scanning]]
- [[Configuration Classes]]
- [[Dependency Injection Patterns]]
- [[Spring Bean Management]]
- [[Spring Boot Project Structure]]
- [[Testing]]

#ioc-di #실습예제 #spring-boot #dependency-injection
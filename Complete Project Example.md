# Complete Project Example

> 완전한 Spring Boot 프로젝트 예제 - 실제 운영 환경에서 사용할 수 있는 완성된 프로젝트

## 📋 프로젝트 개요

### 프로젝트 명: E-Commerce API 시스템
- **목적**: 온라인 쇼핑몰 백엔드 API
- **기술 스택**: Spring Boot 3.3, Spring Security 6, Spring Data JPA, PostgreSQL
- **아키텍처**: 레이어드 아키텍처 + 도메인 주도 설계(DDD)

## 🏗 프로젝트 구조

```
ecommerce-api/
├── src/main/java/com/example/ecommerce/
│   ├── EcommerceApplication.java
│   ├── config/
│   │   ├── SecurityConfig.java
│   │   ├── JpaConfig.java
│   │   └── WebConfig.java
│   ├── domain/
│   │   ├── user/
│   │   │   ├── User.java
│   │   │   ├── UserRepository.java
│   │   │   └── UserService.java
│   │   ├── product/
│   │   │   ├── Product.java
│   │   │   ├── ProductRepository.java
│   │   │   └── ProductService.java
│   │   └── order/
│   │       ├── Order.java
│   │       ├── OrderRepository.java
│   │       └── OrderService.java
│   ├── web/
│   │   ├── dto/
│   │   │   ├── UserDto.java
│   │   │   ├── ProductDto.java
│   │   │   └── OrderDto.java
│   │   └── controller/
│   │       ├── UserController.java
│   │       ├── ProductController.java
│   │       └── OrderController.java
│   ├── infrastructure/
│   │   ├── security/
│   │   │   ├── JwtTokenProvider.java
│   │   │   └── CustomUserDetailsService.java
│   │   └── exception/
│   │       ├── GlobalExceptionHandler.java
│   │       └── BusinessException.java
│   └── common/
│       ├── BaseEntity.java
│       └── ResponseWrapper.java
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   ├── application-prod.yml
│   └── db/migration/
│       ├── V1__Create_users_table.sql
│       ├── V2__Create_products_table.sql
│       └── V3__Create_orders_table.sql
└── src/test/java/
    ├── integration/
    └── unit/
```

## 💼 핵심 구현 코드

### 메인 애플리케이션 클래스

```java
@SpringBootApplication
@EnableJpaAuditing
@EnableScheduling
public class EcommerceApplication {
    public static void main(String[] args) {
        SpringApplication.run(EcommerceApplication.class, args);
    }
}
```

### 사용자 도메인

```java
@Entity
@Table(name = "users")
@EntityListeners(AuditingEntityListener.class)
public class User extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private String password;
    
    @Column(nullable = false)
    private String name;
    
    @Enumerated(EnumType.STRING)
    private UserRole role;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
    
    // constructors, getters, setters
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name%")
    List<User> findByNameContaining(@Param("name") String name);
}

@Service
@Transactional(readOnly = true)
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
    
    @Transactional
    public UserDto createUser(CreateUserRequest request) {
        validateUserRequest(request);
        
        User user = User.builder()
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .name(request.getName())
            .role(UserRole.USER)
            .build();
            
        User savedUser = userRepository.save(user);
        return UserDto.from(savedUser);
    }
    
    public UserDto getUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new BusinessException("사용자를 찾을 수 없습니다."));
        return UserDto.from(user);
    }
    
    private void validateUserRequest(CreateUserRequest request) {
        if (userRepository.findByEmail(request.getEmail()).isPresent()) {
            throw new BusinessException("이미 존재하는 이메일입니다.");
        }
    }
}
```

### 상품 도메인

```java
@Entity
@Table(name = "products")
public class Product extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    @Column(nullable = false)
    private Integer stockQuantity;
    
    @Enumerated(EnumType.STRING)
    private ProductStatus status;
    
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();
    
    // business methods
    public void decreaseStock(int quantity) {
        if (this.stockQuantity < quantity) {
            throw new BusinessException("재고가 부족합니다.");
        }
        this.stockQuantity -= quantity;
    }
}

@Service
@Transactional(readOnly = true)
public class ProductService {
    private final ProductRepository productRepository;
    
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
    
    @Transactional
    public ProductDto createProduct(CreateProductRequest request) {
        Product product = Product.builder()
            .name(request.getName())
            .description(request.getDescription())
            .price(request.getPrice())
            .stockQuantity(request.getStockQuantity())
            .status(ProductStatus.ACTIVE)
            .build();
            
        Product savedProduct = productRepository.save(product);
        return ProductDto.from(savedProduct);
    }
    
    public Page<ProductDto> getProducts(Pageable pageable) {
        return productRepository.findByStatus(ProductStatus.ACTIVE, pageable)
            .map(ProductDto::from);
    }
}
```

### 주문 도메인

```java
@Entity
@Table(name = "orders")
public class Order extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();
    
    @Column(nullable = false)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    // business methods
    public void addOrderItem(Product product, int quantity) {
        OrderItem orderItem = new OrderItem(this, product, quantity, product.getPrice());
        orderItems.add(orderItem);
        product.decreaseStock(quantity);
        calculateTotalAmount();
    }
    
    private void calculateTotalAmount() {
        this.totalAmount = orderItems.stream()
            .map(OrderItem::getTotalPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

@Service
@Transactional(readOnly = true)
public class OrderService {
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    
    @Transactional
    public OrderDto createOrder(Long userId, CreateOrderRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new BusinessException("사용자를 찾을 수 없습니다."));
            
        Order order = new Order(user);
        
        for (OrderItemRequest itemRequest : request.getOrderItems()) {
            Product product = productRepository.findById(itemRequest.getProductId())
                .orElseThrow(() -> new BusinessException("상품을 찾을 수 없습니다."));
            order.addOrderItem(product, itemRequest.getQuantity());
        }
        
        Order savedOrder = orderRepository.save(order);
        return OrderDto.from(savedOrder);
    }
}
```

### REST 컨트롤러

```java
@RestController
@RequestMapping("/api/v1/users")
@Validated
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @PostMapping
    public ResponseEntity<ResponseWrapper<UserDto>> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserDto userDto = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ResponseWrapper.success(userDto));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ResponseWrapper<UserDto>> getUser(@PathVariable Long id) {
        UserDto userDto = userService.getUserById(id);
        return ResponseEntity.ok(ResponseWrapper.success(userDto));
    }
}

@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    private final ProductService productService;
    
    @GetMapping
    public ResponseEntity<ResponseWrapper<Page<ProductDto>>> getProducts(
            @PageableDefault(size = 20) Pageable pageable) {
        Page<ProductDto> products = productService.getProducts(pageable);
        return ResponseEntity.ok(ResponseWrapper.success(products));
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ResponseWrapper<ProductDto>> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        ProductDto productDto = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ResponseWrapper.success(productDto));
    }
}
```

## ⚙️ 설정 파일

### application.yml

```yaml
spring:
  profiles:
    active: dev
  
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce
    username: ${DB_USERNAME:ecommerce}
    password: ${DB_PASSWORD:password}
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  flyway:
    enabled: true
    locations: classpath:db/migration
  
  security:
    jwt:
      secret: ${JWT_SECRET:mySecretKey}
      expiration: 86400000  # 24 hours

logging:
  level:
    com.example.ecommerce: DEBUG
    org.springframework.security: DEBUG

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

### 보안 설정

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    private final JwtTokenProvider jwtTokenProvider;
    private final CustomUserDetailsService userDetailsService;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers("/actuator/**").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), 
                UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

## 🧪 테스트 코드

### 통합 테스트

```java
@SpringBootTest
@Testcontainers
@Transactional
class UserServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserService userService;
    
    @Test
    void createUser_성공() {
        // given
        CreateUserRequest request = new CreateUserRequest(
            "test@example.com", "password", "Test User");
        
        // when
        UserDto result = userService.createUser(request);
        
        // then
        assertThat(result.getEmail()).isEqualTo("test@example.com");
        assertThat(result.getName()).isEqualTo("Test User");
    }
}
```

### API 테스트

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void createUser_성공() throws Exception {
        // given
        CreateUserRequest request = new CreateUserRequest(
            "test@example.com", "password", "Test User");
        UserDto userDto = new UserDto(1L, "test@example.com", "Test User", UserRole.USER);
        
        when(userService.createUser(any(CreateUserRequest.class)))
            .thenReturn(userDto);
        
        // when & then
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.data.email").value("test@example.com"))
                .andExpect(jsonPath("$.data.name").value("Test User"));
    }
}
```

## 🚀 빌드 및 배포

### Dockerfile

```dockerfile
FROM openjdk:21-jdk-slim as builder

WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .
COPY src src

RUN ./gradlew build -x test

FROM openjdk:21-jdk-slim

WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### docker-compose.yml

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: ecommerce
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  app:
    build: .
    depends_on:
      - postgres
    environment:
      DB_USERNAME: ecommerce
      DB_PASSWORD: password
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce
    ports:
      - "8080:8080"

volumes:
  postgres_data:
```

## 📚 관련 노트

- [[Spring Boot Project Structure]] - 프로젝트 구조 설계
- [[Spring Data JPA]] - 데이터 계층 구현
- [[Spring Security 6]] - 보안 구현
- [[Testing]] - 테스트 전략
- [[Service Layer Pattern]] - 서비스 계층 패턴
- [[DTO and VO Patterns]] - DTO 패턴 적용

## 🔗 외부 리소스

- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)

---

*이 예제는 실제 운영 환경에서 사용할 수 있는 수준의 코드 품질과 아키텍처를 제공합니다.*
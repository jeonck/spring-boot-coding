# Spring Boot Code Templates

> 자주 사용되는 Spring Boot 코드 패턴과 템플릿 모음 - 복사해서 바로 사용 가능

## 📁 목차

- [엔티티 템플릿](#엔티티-템플릿)
- [리포지토리 템플릿](#리포지토리-템플릿)
- [서비스 템플릿](#서비스-템플릿)
- [컨트롤러 템플릿](#컨트롤러-템플릿)
- [DTO 템플릿](#dto-템플릿)
- [설정 클래스 템플릿](#설정-클래스-템플릿)
- [예외 처리 템플릿](#예외-처리-템플릿)
- [테스트 템플릿](#테스트-템플릿)

## 🗄 엔티티 템플릿

### 기본 엔티티 템플릿

```java
@Entity
@Table(name = "entity_name")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class EntityName extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Enumerated(EnumType.STRING)
    private Status status;
    
    // 생성자, 비즈니스 메서드
    protected EntityName(String name, String description) {
        this.name = name;
        this.description = description;
        this.status = Status.ACTIVE;
    }
    
    public void updateName(String name) {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("이름은 필수입니다.");
        }
        this.name = name;
    }
}
```

### 연관관계 엔티티 템플릿

```java
@Entity
@Table(name = "parent_entity")
public class ParentEntity extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    // 일대다 관계
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<ChildEntity> children = new ArrayList<>();
    
    // 연관관계 편의 메서드
    public void addChild(ChildEntity child) {
        children.add(child);
        child.setParent(this);
    }
    
    public void removeChild(ChildEntity child) {
        children.remove(child);
        child.setParent(null);
    }
}

@Entity
@Table(name = "child_entity")
public class ChildEntity extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    // 다대일 관계
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private ParentEntity parent;
}
```

### BaseEntity 템플릿

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String lastModifiedBy;
    
    @Version
    private Long version;
}
```

## 🏪 리포지토리 템플릿

### 기본 JPA 리포지토리

```java
@Repository
public interface EntityNameRepository extends JpaRepository<EntityName, Long> {
    
    // 기본 메서드들은 자동 제공
    
    // 커스텀 쿼리 메서드
    Optional<EntityName> findByName(String name);
    
    List<EntityName> findByStatus(Status status);
    
    @Query("SELECT e FROM EntityName e WHERE e.name LIKE %:keyword%")
    List<EntityName> findByNameContaining(@Param("keyword") String keyword);
    
    @Query("SELECT e FROM EntityName e WHERE e.createdAt BETWEEN :start AND :end")
    Page<EntityName> findByCreatedAtBetween(
        @Param("start") LocalDateTime start, 
        @Param("end") LocalDateTime end, 
        Pageable pageable);
    
    @Modifying
    @Query("UPDATE EntityName e SET e.status = :status WHERE e.id = :id")
    void updateStatus(@Param("id") Long id, @Param("status") Status status);
    
    // 네이티브 쿼리
    @Query(value = "SELECT * FROM entity_name WHERE status = ?1", nativeQuery = true)
    List<EntityName> findByStatusNative(String status);
}
```

### 커스텀 리포지토리 구현

```java
public interface EntityNameRepositoryCustom {
    List<EntityName> findWithComplexConditions(SearchCriteria criteria);
}

@Repository
public class EntityNameRepositoryImpl implements EntityNameRepositoryCustom {
    
    private final JPAQueryFactory queryFactory;
    
    public EntityNameRepositoryImpl(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }
    
    @Override
    public List<EntityName> findWithComplexConditions(SearchCriteria criteria) {
        QEntityName entity = QEntityName.entityName;
        
        BooleanBuilder builder = new BooleanBuilder();
        
        if (criteria.getName() != null) {
            builder.and(entity.name.containsIgnoreCase(criteria.getName()));
        }
        
        if (criteria.getStatus() != null) {
            builder.and(entity.status.eq(criteria.getStatus()));
        }
        
        if (criteria.getStartDate() != null) {
            builder.and(entity.createdAt.goe(criteria.getStartDate()));
        }
        
        return queryFactory
            .selectFrom(entity)
            .where(builder)
            .orderBy(entity.createdAt.desc())
            .fetch();
    }
}
```

## 🔧 서비스 템플릿

### 기본 서비스 클래스

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class EntityNameService {
    
    private final EntityNameRepository repository;
    private final EntityNameMapper mapper;
    
    @Transactional
    public EntityNameDto create(CreateEntityNameRequest request) {
        log.debug("Creating entity with name: {}", request.getName());
        
        validateCreateRequest(request);
        
        EntityName entity = EntityName.builder()
            .name(request.getName())
            .description(request.getDescription())
            .build();
        
        EntityName savedEntity = repository.save(entity);
        
        log.info("Created entity with id: {}", savedEntity.getId());
        return mapper.toDto(savedEntity);
    }
    
    public EntityNameDto findById(Long id) {
        EntityName entity = repository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Entity not found with id: " + id));
        
        return mapper.toDto(entity);
    }
    
    public Page<EntityNameDto> findAll(Pageable pageable) {
        return repository.findAll(pageable)
            .map(mapper::toDto);
    }
    
    @Transactional
    public EntityNameDto update(Long id, UpdateEntityNameRequest request) {
        log.debug("Updating entity with id: {}", id);
        
        EntityName entity = repository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Entity not found with id: " + id));
        
        entity.updateName(request.getName());
        entity.setDescription(request.getDescription());
        
        log.info("Updated entity with id: {}", id);
        return mapper.toDto(entity);
    }
    
    @Transactional
    public void delete(Long id) {
        log.debug("Deleting entity with id: {}", id);
        
        if (!repository.existsById(id)) {
            throw new EntityNotFoundException("Entity not found with id: " + id);
        }
        
        repository.deleteById(id);
        log.info("Deleted entity with id: {}", id);
    }
    
    private void validateCreateRequest(CreateEntityNameRequest request) {
        if (repository.findByName(request.getName()).isPresent()) {
            throw new DuplicateResourceException("Entity already exists with name: " + request.getName());
        }
    }
}
```

### 복잡한 비즈니스 로직을 가진 서비스

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public OrderDto createOrder(Long userId, CreateOrderRequest request) {
        // 1. 사용자 조회
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        // 2. 상품 재고 확인
        List<OrderItem> orderItems = validateAndCreateOrderItems(request.getItems());
        
        // 3. 주문 생성
        Order order = Order.builder()
            .user(user)
            .orderItems(orderItems)
            .build();
        
        order.calculateTotalAmount();
        
        // 4. 재고 차감
        decreaseProductStock(orderItems);
        
        // 5. 주문 저장
        Order savedOrder = orderRepository.save(order);
        
        // 6. 이벤트 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(savedOrder.getId()));
        
        // 7. 이메일 발송 (비동기)
        emailService.sendOrderConfirmationAsync(user.getEmail(), savedOrder);
        
        return OrderDto.from(savedOrder);
    }
    
    private List<OrderItem> validateAndCreateOrderItems(List<OrderItemRequest> requests) {
        return requests.stream()
            .map(this::createOrderItem)
            .collect(Collectors.toList());
    }
    
    private OrderItem createOrderItem(OrderItemRequest request) {
        Product product = productRepository.findById(request.getProductId())
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
        
        if (product.getStockQuantity() < request.getQuantity()) {
            throw new InsufficientStockException("Insufficient stock for product: " + product.getName());
        }
        
        return OrderItem.builder()
            .product(product)
            .quantity(request.getQuantity())
            .price(product.getPrice())
            .build();
    }
    
    private void decreaseProductStock(List<OrderItem> orderItems) {
        orderItems.forEach(item -> 
            item.getProduct().decreaseStock(item.getQuantity()));
    }
}
```

## 🎮 컨트롤러 템플릿

### REST API 컨트롤러

```java
@RestController
@RequestMapping("/api/v1/entities")
@RequiredArgsConstructor
@Validated
@Slf4j
public class EntityNameController {
    
    private final EntityNameService service;
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseWrapper<EntityNameDto> create(
            @Valid @RequestBody CreateEntityNameRequest request) {
        
        log.info("Creating entity with name: {}", request.getName());
        EntityNameDto dto = service.create(request);
        
        return ResponseWrapper.success(dto, "Entity created successfully");
    }
    
    @GetMapping("/{id}")
    public ResponseWrapper<EntityNameDto> getById(@PathVariable Long id) {
        EntityNameDto dto = service.findById(id);
        return ResponseWrapper.success(dto);
    }
    
    @GetMapping
    public ResponseWrapper<Page<EntityNameDto>> getAll(
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) 
            Pageable pageable) {
        
        Page<EntityNameDto> result = service.findAll(pageable);
        return ResponseWrapper.success(result);
    }
    
    @PutMapping("/{id}")
    public ResponseWrapper<EntityNameDto> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateEntityNameRequest request) {
        
        log.info("Updating entity with id: {}", id);
        EntityNameDto dto = service.update(id, request);
        
        return ResponseWrapper.success(dto, "Entity updated successfully");
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        log.info("Deleting entity with id: {}", id);
        service.delete(id);
    }
    
    @GetMapping("/search")
    public ResponseWrapper<List<EntityNameDto>> search(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) Status status) {
        
        SearchCriteria criteria = SearchCriteria.builder()
            .name(name)
            .status(status)
            .build();
        
        List<EntityNameDto> result = service.search(criteria);
        return ResponseWrapper.success(result);
    }
}
```

### 파일 업로드 컨트롤러

```java
@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
public class FileUploadController {
    
    private final FileStorageService fileStorageService;
    
    @PostMapping("/upload")
    public ResponseWrapper<FileUploadResponse> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam(value = "category", defaultValue = "general") String category) {
        
        validateFile(file);
        
        String fileName = fileStorageService.storeFile(file, category);
        String fileDownloadUri = ServletUriComponentsBuilder.fromCurrentContextPath()
            .path("/api/v1/files/download/")
            .path(fileName)
            .toUriString();
        
        FileUploadResponse response = FileUploadResponse.builder()
            .fileName(fileName)
            .fileDownloadUri(fileDownloadUri)
            .fileType(file.getContentType())
            .size(file.getSize())
            .build();
        
        return ResponseWrapper.success(response);
    }
    
    @GetMapping("/download/{fileName:.+}")
    public ResponseEntity<Resource> downloadFile(@PathVariable String fileName) {
        Resource resource = fileStorageService.loadFileAsResource(fileName);
        
        String contentType;
        try {
            contentType = Files.probeContentType(Paths.get(resource.getFile().getAbsolutePath()));
        } catch (IOException ex) {
            contentType = "application/octet-stream";
        }
        
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType))
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                "attachment; filename=\"" + resource.getFilename() + "\"")
            .body(resource);
    }
    
    private void validateFile(MultipartFile file) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("파일이 비어있습니다.");
        }
        
        String contentType = file.getContentType();
        if (!isValidContentType(contentType)) {
            throw new IllegalArgumentException("지원되지 않는 파일 형식입니다.");
        }
        
        if (file.getSize() > 10 * 1024 * 1024) { // 10MB
            throw new IllegalArgumentException("파일 크기가 10MB를 초과합니다.");
        }
    }
    
    private boolean isValidContentType(String contentType) {
        return contentType != null && (
            contentType.startsWith("image/") ||
            contentType.equals("application/pdf") ||
            contentType.startsWith("text/")
        );
    }
}
```

## 📝 DTO 템플릿

### Request DTO 템플릿

```java
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CreateEntityNameRequest {
    
    @NotBlank(message = "이름은 필수입니다.")
    @Size(min = 2, max = 100, message = "이름은 2-100자 사이여야 합니다.")
    private String name;
    
    @Size(max = 500, message = "설명은 500자를 초과할 수 없습니다.")
    private String description;
    
    @Email(message = "올바른 이메일 형식이 아닙니다.")
    private String email;
    
    @NotNull(message = "상태는 필수입니다.")
    private Status status;
    
    @Valid
    @NotEmpty(message = "항목은 최소 1개 이상이어야 합니다.")
    private List<SubItemRequest> items;
    
    @DecimalMin(value = "0.0", inclusive = false, message = "가격은 0보다 커야 합니다.")
    @DecimalMax(value = "999999.99", message = "가격은 999,999.99를 초과할 수 없습니다.")
    private BigDecimal price;
}

@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
public class UpdateEntityNameRequest {
    
    @NotBlank(message = "이름은 필수입니다.")
    @Size(min = 2, max = 100, message = "이름은 2-100자 사이여야 합니다.")
    private String name;
    
    @Size(max = 500, message = "설명은 500자를 초과할 수 없습니다.")
    private String description;
    
    private Status status;
}
```

### Response DTO 템플릿

```java
@Getter @Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class EntityNameDto {
    
    private Long id;
    private String name;
    private String description;
    private Status status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    
    public static EntityNameDto from(EntityName entity) {
        return EntityNameDto.builder()
            .id(entity.getId())
            .name(entity.getName())
            .description(entity.getDescription())
            .status(entity.getStatus())
            .createdAt(entity.getCreatedAt())
            .updatedAt(entity.getUpdatedAt())
            .createdBy(entity.getCreatedBy())
            .build();
    }
}

@Getter @Setter
@Builder
public class ResponseWrapper<T> {
    
    private boolean success;
    private String message;
    private T data;
    private LocalDateTime timestamp;
    
    public static <T> ResponseWrapper<T> success(T data) {
        return ResponseWrapper.<T>builder()
            .success(true)
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ResponseWrapper<T> success(T data, String message) {
        return ResponseWrapper.<T>builder()
            .success(true)
            .message(message)
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ResponseWrapper<T> error(String message) {
        return ResponseWrapper.<T>builder()
            .success(false)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

## ⚙️ 설정 클래스 템플릿

### 데이터베이스 설정

```java
@Configuration
@EnableJpaRepositories(basePackages = "com.example.repository")
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource")
    public DataSourceProperties dataSourceProperties() {
        return new DataSourceProperties();
    }
    
    @Bean
    @Primary
    public DataSource dataSource() {
        return dataSourceProperties()
            .initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
    }
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return new SpringSecurityAuditorAware();
    }
    
    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager entityManager) {
        return new JPAQueryFactory(entityManager);
    }
}

public class SpringSecurityAuditorAware implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication == null || !authentication.isAuthenticated() 
            || authentication instanceof AnonymousAuthenticationToken) {
            return Optional.of("system");
        }
        
        return Optional.of(authentication.getName());
    }
}
```

### 보안 설정

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
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
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), 
                UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new JwtAuthenticationEntryPoint())
                .accessDeniedHandler(new JwtAccessDeniedHandler()));
        
        return http.build();
    }
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("http://localhost:3000", "https://yourdomain.com")
                    .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                    .allowedHeaders("*")
                    .allowCredentials(true);
            }
        };
    }
}
```

## ❌ 예외 처리 템플릿

### 글로벌 예외 핸들러

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResponseWrapper<Void> handleEntityNotFound(EntityNotFoundException ex) {
        log.warn("Entity not found: {}", ex.getMessage());
        return ResponseWrapper.error(ex.getMessage());
    }
    
    @ExceptionHandler(DuplicateResourceException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ResponseWrapper<Void> handleDuplicateResource(DuplicateResourceException ex) {
        log.warn("Duplicate resource: {}", ex.getMessage());
        return ResponseWrapper.error(ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseWrapper<Map<String, String>> handleValidationExceptions(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        log.warn("Validation failed: {}", errors);
        return ResponseWrapper.<Map<String, String>>builder()
            .success(false)
            .message("입력 값 검증에 실패했습니다.")
            .data(errors)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ResponseWrapper<Void> handleAccessDenied(AccessDeniedException ex) {
        log.warn("Access denied: {}", ex.getMessage());
        return ResponseWrapper.error("접근 권한이 없습니다.");
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseWrapper<Void> handleGenericException(Exception ex) {
        log.error("Unexpected error occurred", ex);
        return ResponseWrapper.error("서버 내부 오류가 발생했습니다.");
    }
}
```

### 커스텀 예외 클래스

```java
public class BusinessException extends RuntimeException {
    
    private final String code;
    
    public BusinessException(String message) {
        super(message);
        this.code = "BUSINESS_ERROR";
    }
    
    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }
    
    public BusinessException(String message, Throwable cause) {
        super(message, cause);
        this.code = "BUSINESS_ERROR";
    }
    
    public String getCode() {
        return code;
    }
}

public class EntityNotFoundException extends BusinessException {
    public EntityNotFoundException(String message) {
        super("ENTITY_NOT_FOUND", message);
    }
}

public class DuplicateResourceException extends BusinessException {
    public DuplicateResourceException(String message) {
        super("DUPLICATE_RESOURCE", message);
    }
}

public class InsufficientStockException extends BusinessException {
    public InsufficientStockException(String message) {
        super("INSUFFICIENT_STOCK", message);
    }
}
```

## 🧪 테스트 템플릿

### 단위 테스트 템플릿

```java
@ExtendWith(MockitoExtension.class)
class EntityNameServiceTest {
    
    @Mock
    private EntityNameRepository repository;
    
    @Mock
    private EntityNameMapper mapper;
    
    @InjectMocks
    private EntityNameService service;
    
    @Test
    @DisplayName("엔티티 생성 성공")
    void createEntity_Success() {
        // given
        CreateEntityNameRequest request = new CreateEntityNameRequest("Test Name", "Description");
        EntityName entity = EntityName.builder()
            .id(1L)
            .name("Test Name")
            .description("Description")
            .build();
        EntityNameDto expectedDto = EntityNameDto.from(entity);
        
        when(repository.findByName(request.getName())).thenReturn(Optional.empty());
        when(repository.save(any(EntityName.class))).thenReturn(entity);
        when(mapper.toDto(entity)).thenReturn(expectedDto);
        
        // when
        EntityNameDto result = service.create(request);
        
        // then
        assertThat(result.getName()).isEqualTo("Test Name");
        assertThat(result.getDescription()).isEqualTo("Description");
        
        verify(repository).findByName(request.getName());
        verify(repository).save(any(EntityName.class));
        verify(mapper).toDto(entity);
    }
    
    @Test
    @DisplayName("중복된 이름으로 엔티티 생성 시 예외 발생")
    void createEntity_DuplicateName_ThrowsException() {
        // given
        CreateEntityNameRequest request = new CreateEntityNameRequest("Existing Name", "Description");
        EntityName existingEntity = EntityName.builder()
            .id(1L)
            .name("Existing Name")
            .build();
        
        when(repository.findByName(request.getName())).thenReturn(Optional.of(existingEntity));
        
        // when & then
        assertThatThrownBy(() -> service.create(request))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("Entity already exists with name: Existing Name");
        
        verify(repository).findByName(request.getName());
        verify(repository, never()).save(any(EntityName.class));
    }
}
```

### 통합 테스트 템플릿

```java
@SpringBootTest
@Testcontainers
@Transactional
class EntityNameServiceIntegrationTest {
    
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
    private EntityNameService service;
    
    @Autowired
    private EntityNameRepository repository;
    
    @Test
    @DisplayName("엔티티 생성 및 조회 통합 테스트")
    void createAndRetrieveEntity() {
        // given
        CreateEntityNameRequest request = new CreateEntityNameRequest("Integration Test", "Description");
        
        // when
        EntityNameDto created = service.create(request);
        EntityNameDto retrieved = service.findById(created.getId());
        
        // then
        assertThat(created.getId()).isNotNull();
        assertThat(retrieved.getName()).isEqualTo("Integration Test");
        assertThat(retrieved.getDescription()).isEqualTo("Description");
        
        // 데이터베이스에서 직접 확인
        Optional<EntityName> entityInDb = repository.findById(created.getId());
        assertThat(entityInDb).isPresent();
        assertThat(entityInDb.get().getName()).isEqualTo("Integration Test");
    }
}
```

### 웹 테스트 템플릿

```java
@WebMvcTest(EntityNameController.class)
class EntityNameControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private EntityNameService service;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    @DisplayName("엔티티 생성 API 테스트")
    void createEntity_Success() throws Exception {
        // given
        CreateEntityNameRequest request = new CreateEntityNameRequest("Test Name", "Description");
        EntityNameDto responseDto = EntityNameDto.builder()
            .id(1L)
            .name("Test Name")
            .description("Description")
            .createdAt(LocalDateTime.now())
            .build();
        
        when(service.create(any(CreateEntityNameRequest.class))).thenReturn(responseDto);
        
        // when & then
        mockMvc.perform(post("/api/v1/entities")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.id").value(1))
                .andExpect(jsonPath("$.data.name").value("Test Name"))
                .andExpect(jsonPath("$.data.description").value("Description"));
        
        verify(service).create(any(CreateEntityNameRequest.class));
    }
    
    @Test
    @DisplayName("잘못된 입력으로 엔티티 생성 시 400 에러")
    void createEntity_InvalidInput_BadRequest() throws Exception {
        // given
        CreateEntityNameRequest request = new CreateEntityNameRequest("", "Description"); // 빈 이름
        
        // when & then
        mockMvc.perform(post("/api/v1/entities")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.data.name").exists());
    }
}
```

## 📚 관련 노트

- [[Spring Boot Project Structure]] - 프로젝트 구조 이해
- [[Spring Data JPA]] - JPA 심화 학습
- [[Testing]] - 테스트 전략과 기법
- [[Exception Handling Patterns]] - 예외 처리 패턴
- [[Complete Project Example]] - 완전한 프로젝트 예제

## 🔗 외부 리소스

- [Spring Boot Guides](https://spring.io/guides)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)

---

*이 템플릿들은 실제 프로젝트에서 바로 사용할 수 있도록 최적화되었습니다. 필요에 따라 수정해서 사용하세요.*
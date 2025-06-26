# Spring WebFlux Reactive Programming

> Spring Boot에서 리액티브 프로그래밍과 WebFlux를 활용한 비동기 웹 애플리케이션 개발

## 📋 목차

- [리액티브 프로그래밍 개요](#리액티브-프로그래밍-개요)
- [WebFlux 기초](#webflux-기초)
- [Mono와 Flux](#mono와-flux)
- [리액티브 웹 컨트롤러](#리액티브-웹-컨트롤러)
- [리액티브 데이터 액세스](#리액티브-데이터-액세스)
- [WebClient 사용법](#webclient-사용법)
- [에러 처리](#에러-처리)
- [테스트 전략](#테스트-전략)
- [성능 최적화](#성능-최적화)

## 🌊 리액티브 프로그래밍 개요

### 리액티브 프로그래밍 원칙

```yaml
리액티브 선언문 (Reactive Manifesto):
  응답성 (Responsive): 시스템이 가능한 한 즉시 응답
  탄력성 (Resilient): 장애 상황에서도 응답성 유지
  유연성 (Elastic): 다양한 작업 부하에서 응답성 유지
  메시지 주도 (Message Driven): 비동기 메시지 전달

리액티브 스트림 (Reactive Streams):
  - Publisher: 데이터를 발행
  - Subscriber: 데이터를 구독
  - Subscription: 구독 관계 관리
  - Processor: Publisher이면서 Subscriber

장점:
  - 높은 동시성 처리
  - 적은 리소스 사용
  - 백프레셔 (Backpressure) 지원
  - 논블로킹 I/O

단점:
  - 학습 곡선이 가파름
  - 디버깅의 어려움
  - 생태계의 제한
```

### 의존성 설정

```gradle
dependencies {
    // Spring WebFlux
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    
    // 리액티브 데이터 액세스
    implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb-reactive'
    
    // R2DBC 드라이버
    implementation 'io.r2dbc:r2dbc-postgresql'
    implementation 'io.r2dbc:r2dbc-h2'
    
    // JSON 처리
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
    
    // 검증
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    
    // 모니터링
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    
    // 테스트
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.projectreactor:reactor-test'
    testImplementation 'org.testcontainers:r2dbc'
    testImplementation 'org.testcontainers:postgresql'
}
```

## 🚀 WebFlux 기초

### WebFlux 설정

```java
@Configuration
@EnableWebFlux
public class WebFluxConfig implements WebFluxConfigurer {
    
    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        configurer.defaultCodecs().maxInMemorySize(1024 * 1024); // 1MB
        configurer.defaultCodecs().enableLoggingRequestDetails(true);
    }
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
    
    @Bean
    public RouterFunction<ServerResponse> routerFunction(ProductHandler handler) {
        return RouterFunctions
            .route(GET("/api/products"), handler::getAllProducts)
            .andRoute(GET("/api/products/{id}"), handler::getProduct)
            .andRoute(POST("/api/products"), handler::createProduct)
            .andRoute(PUT("/api/products/{id}"), handler::updateProduct)
            .andRoute(DELETE("/api/products/{id}"), handler::deleteProduct)
            .andRoute(GET("/api/products/stream"), handler::streamProducts);
    }
    
    @Bean
    public WebFilter loggingFilter() {
        return (exchange, chain) -> {
            long startTime = System.currentTimeMillis();
            String path = exchange.getRequest().getPath().value();
            String method = exchange.getRequest().getMethodValue();
            
            return chain.filter(exchange)
                .doFinally(signalType -> {
                    long duration = System.currentTimeMillis() - startTime;
                    log.info("{} {} - {}ms", method, path, duration);
                });
        };
    }
}
```

### 리액티브 데이터베이스 설정

```java
@Configuration
@EnableR2dbcRepositories
public class R2dbcConfig extends AbstractR2dbcConfiguration {
    
    @Value("${spring.r2dbc.url}")
    private String url;
    
    @Value("${spring.r2dbc.username}")
    private String username;
    
    @Value("${spring.r2dbc.password}")
    private String password;
    
    @Bean
    @Override
    public ConnectionFactory connectionFactory() {
        return ConnectionFactories.get(ConnectionFactoryOptions.builder()
            .option(DRIVER, "postgresql")
            .option(HOST, "localhost")
            .option(PORT, 5432)
            .option(USER, username)
            .option(PASSWORD, password)
            .option(DATABASE, "reactive_db")
            .build());
    }
    
    @Bean
    public ReactiveTransactionManager transactionManager(ConnectionFactory connectionFactory) {
        return new R2dbcTransactionManager(connectionFactory);
    }
    
    @Bean
    public R2dbcCustomConversions r2dbcCustomConversions() {
        List<Converter<?, ?>> converters = new ArrayList<>();
        converters.add(new LocalDateTimeToOffsetDateTimeConverter());
        converters.add(new OffsetDateTimeToLocalDateTimeConverter());
        return new R2dbcCustomConversions(getStoreConversions(), converters);
    }
    
    // 타임스탬프 변환기
    @WritingConverter
    static class LocalDateTimeToOffsetDateTimeConverter implements Converter<LocalDateTime, OffsetDateTime> {
        @Override
        public OffsetDateTime convert(LocalDateTime source) {
            return source.atOffset(ZoneOffset.UTC);
        }
    }
    
    @ReadingConverter
    static class OffsetDateTimeToLocalDateTimeConverter implements Converter<OffsetDateTime, LocalDateTime> {
        @Override
        public LocalDateTime convert(OffsetDateTime source) {
            return source.toLocalDateTime();
        }
    }
}
```

## 💫 Mono와 Flux

### 기본 사용법

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ReactiveExampleService {
    
    // Mono: 0-1개 요소를 비동기로 처리
    public Mono<String> getSingleValue() {
        return Mono.just("Hello, Reactive World!")
            .delayElement(Duration.ofSeconds(1))
            .doOnNext(value -> log.info("Processing: {}", value))
            .doOnError(error -> log.error("Error occurred", error))
            .doOnSuccess(result -> log.info("Success: {}", result));
    }
    
    // Flux: 0-N개 요소를 비동기로 처리
    public Flux<Integer> getMultipleValues() {
        return Flux.range(1, 10)
            .delayElements(Duration.ofMillis(100))
            .map(i -> i * 2)
            .filter(i -> i % 4 == 0)
            .doOnNext(value -> log.info("Emitted: {}", value))
            .doOnComplete(() -> log.info("Stream completed"));
    }
    
    // 에러 처리
    public Mono<String> getValueWithErrorHandling() {
        return Mono.fromCallable(() -> {
                if (Math.random() > 0.5) {
                    throw new RuntimeException("Random error occurred");
                }
                return "Success value";
            })
            .onErrorReturn("Default value")
            .timeout(Duration.ofSeconds(5))
            .retry(3);
    }
    
    // 여러 소스 결합
    public Mono<CombinedResult> combineMultipleSources() {
        Mono<String> source1 = Mono.just("Data1").delayElement(Duration.ofMillis(100));
        Mono<String> source2 = Mono.just("Data2").delayElement(Duration.ofMillis(200));
        Mono<String> source3 = Mono.just("Data3").delayElement(Duration.ofMillis(150));
        
        return Mono.zip(source1, source2, source3)
            .map(tuple -> CombinedResult.builder()
                .value1(tuple.getT1())
                .value2(tuple.getT2())
                .value3(tuple.getT3())
                .build());
    }
    
    // 스트림 변환
    public Flux<ProcessedData> processDataStream(Flux<RawData> inputStream) {
        return inputStream
            .buffer(Duration.ofSeconds(1)) // 1초간 수집된 데이터를 배치로 처리
            .flatMap(batch -> Flux.fromIterable(batch)
                .parallel(4) // 4개 스레드로 병렬 처리
                .runOn(Schedulers.parallel())
                .map(this::processData)
                .sequential())
            .onBackpressureBuffer(1000) // 백프레셔 처리
            .share(); // 여러 구독자가 공유
    }
    
    private ProcessedData processData(RawData raw) {
        // 복잡한 처리 로직
        return ProcessedData.builder()
            .id(raw.getId())
            .processedValue(raw.getValue().toUpperCase())
            .processedAt(LocalDateTime.now())
            .build();
    }
}
```

### 고급 연산자 활용

```java
@Service
@RequiredArgsConstructor
public class AdvancedReactiveService {
    
    private final WebClient webClient;
    private final ReactiveRedisTemplate<String, String> redisTemplate;
    
    // 조건부 처리
    public Flux<Data> getDataWithConditions(String criteria) {
        return Flux.defer(() -> {
            if ("fast".equals(criteria)) {
                return getFastData();
            } else if ("slow".equals(criteria)) {
                return getSlowData();
            } else {
                return Flux.error(new IllegalArgumentException("Invalid criteria"));
            }
        });
    }
    
    // 캐싱 구현
    public Mono<String> getCachedData(String key) {
        return redisTemplate.opsForValue()
            .get(key)
            .switchIfEmpty(fetchDataFromDatabase(key)
                .flatMap(data -> redisTemplate.opsForValue()
                    .set(key, data, Duration.ofMinutes(10))
                    .then(Mono.just(data))));
    }
    
    // 재시도 및 폴백
    public Mono<String> getDataWithRetryAndFallback(String id) {
        return webClient.get()
            .uri("/api/data/{id}", id)
            .retrieve()
            .bodyToMono(String.class)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(throwable -> throwable instanceof WebClientResponseException))
            .onErrorResume(throwable -> {
                log.warn("Failed to fetch data for {}, using fallback", id, throwable);
                return getFallbackData(id);
            })
            .timeout(Duration.ofSeconds(10))
            .doOnSuccess(data -> log.info("Successfully fetched data for {}", id));
    }
    
    // 스트림 분기
    public Flux<String> processDataWithBranching(Flux<Integer> input) {
        ConnectableFlux<Integer> sharedStream = input.publish();
        
        Flux<String> evenNumbers = sharedStream
            .filter(i -> i % 2 == 0)
            .map(i -> "Even: " + i);
        
        Flux<String> oddNumbers = sharedStream
            .filter(i -> i % 2 != 0)
            .map(i -> "Odd: " + i);
        
        sharedStream.connect(); // 스트림 시작
        
        return Flux.merge(evenNumbers, oddNumbers);
    }
    
    // 윈도우 처리
    public Flux<List<SensorData>> processSensorDataWindows(Flux<SensorData> sensorStream) {
        return sensorStream
            .window(Duration.ofSeconds(5)) // 5초 윈도우
            .flatMap(window -> window
                .collectList()
                .filter(list -> !list.isEmpty())
                .map(this::aggregateSensorData));
    }
    
    private List<SensorData> aggregateSensorData(List<SensorData> data) {
        // 센서 데이터 집계 로직
        return data.stream()
            .collect(Collectors.groupingBy(SensorData::getSensorId))
            .entrySet().stream()
            .map(entry -> {
                List<SensorData> sensorData = entry.getValue();
                double averageValue = sensorData.stream()
                    .mapToDouble(SensorData::getValue)
                    .average()
                    .orElse(0.0);
                
                return SensorData.builder()
                    .sensorId(entry.getKey())
                    .value(averageValue)
                    .timestamp(LocalDateTime.now())
                    .build();
            })
            .collect(Collectors.toList());
    }
}
```

## 🎮 리액티브 웹 컨트롤러

### 어노테이션 기반 컨트롤러

```java
@RestController
@RequestMapping("/api/reactive/products")
@RequiredArgsConstructor
@Validated
@Slf4j
public class ReactiveProductController {
    
    private final ReactiveProductService productService;
    
    @GetMapping
    public Flux<ProductDto> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String category) {
        
        return productService.getAllProducts(page, size, category)
            .doOnNext(product -> log.debug("Emitting product: {}", product.getId()))
            .doOnComplete(() -> log.info("All products emitted"));
    }
    
    @GetMapping("/{id}")
    public Mono<ResponseEntity<ProductDto>> getProduct(@PathVariable String id) {
        return productService.getProductById(id)
            .map(product -> ResponseEntity.ok(product))
            .defaultIfEmpty(ResponseEntity.notFound().build())
            .doOnSuccess(response -> log.info("Product lookup completed for id: {}", id));
    }
    
    @PostMapping
    public Mono<ResponseEntity<ProductDto>> createProduct(
            @Valid @RequestBody Mono<CreateProductRequest> requestMono) {
        
        return requestMono
            .flatMap(productService::createProduct)
            .map(product -> ResponseEntity.status(HttpStatus.CREATED).body(product))
            .doOnSuccess(response -> log.info("Product created: {}", response.getBody().getId()));
    }
    
    @PutMapping("/{id}")
    public Mono<ResponseEntity<ProductDto>> updateProduct(
            @PathVariable String id,
            @Valid @RequestBody Mono<UpdateProductRequest> requestMono) {
        
        return requestMono
            .flatMap(request -> productService.updateProduct(id, request))
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> deleteProduct(@PathVariable String id) {
        return productService.deleteProduct(id)
            .map(success -> success ? 
                ResponseEntity.noContent().<Void>build() : 
                ResponseEntity.notFound().<Void>build())
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    // 스트리밍 엔드포인트
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ProductDto> streamProducts() {
        return productService.streamProducts()
            .delayElements(Duration.ofSeconds(1))
            .doOnNext(product -> log.info("Streaming product: {}", product.getId()));
    }
    
    // 서버 센트 이벤트 (SSE)
    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<ProductEventDto>> getProductEvents() {
        return productService.getProductEvents()
            .map(event -> ServerSentEvent.<ProductEventDto>builder()
                .id(event.getId())
                .event(event.getType())
                .data(event)
                .build())
            .doOnNext(sse -> log.info("Sending SSE: {}", sse.id()));
    }
}
```

### 함수형 라우터

```java
@Component
@RequiredArgsConstructor
public class ProductHandler {
    
    private final ReactiveProductService productService;
    private final Validator validator;
    
    public Mono<ServerResponse> getAllProducts(ServerRequest request) {
        int page = request.queryParam("page")
            .map(Integer::parseInt)
            .orElse(0);
        int size = request.queryParam("size")
            .map(Integer::parseInt)
            .orElse(10);
        String category = request.queryParam("category").orElse(null);
        
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(productService.getAllProducts(page, size, category), ProductDto.class);
    }
    
    public Mono<ServerResponse> getProduct(ServerRequest request) {
        String id = request.pathVariable("id");
        
        return productService.getProductById(id)
            .flatMap(product -> ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(product))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
    
    public Mono<ServerResponse> createProduct(ServerRequest request) {
        return request.bodyToMono(CreateProductRequest.class)
            .doOnNext(this::validate)
            .flatMap(productService::createProduct)
            .flatMap(product -> ServerResponse.status(HttpStatus.CREATED)
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(product))
            .onErrorResume(ValidationException.class, 
                ex -> ServerResponse.badRequest().bodyValue(ex.getMessage()));
    }
    
    public Mono<ServerResponse> updateProduct(ServerRequest request) {
        String id = request.pathVariable("id");
        
        return request.bodyToMono(UpdateProductRequest.class)
            .doOnNext(this::validate)
            .flatMap(req -> productService.updateProduct(id, req))
            .flatMap(product -> ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(product))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
    
    public Mono<ServerResponse> deleteProduct(ServerRequest request) {
        String id = request.pathVariable("id");
        
        return productService.deleteProduct(id)
            .flatMap(success -> success ? 
                ServerResponse.noContent().build() : 
                ServerResponse.notFound().build());
    }
    
    public Mono<ServerResponse> streamProducts(ServerRequest request) {
        return ServerResponse.ok()
            .contentType(MediaType.TEXT_EVENT_STREAM)
            .body(productService.streamProducts()
                .delayElements(Duration.ofSeconds(1)), ProductDto.class);
    }
    
    private void validate(Object request) {
        Set<ConstraintViolation<Object>> violations = validator.validate(request);
        if (!violations.isEmpty()) {
            String message = violations.stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining(", "));
            throw new ValidationException(message);
        }
    }
}
```

## 💾 리액티브 데이터 액세스

### R2DBC Repository

```java
// 엔티티 클래스
@Table("products")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {
    
    @Id
    private String id;
    
    @Column("name")
    private String name;
    
    @Column("description")
    private String description;
    
    @Column("price")
    private BigDecimal price;
    
    @Column("category_id")
    private String categoryId;
    
    @Column("stock_quantity")
    private Integer stockQuantity;
    
    @Column("created_at")
    private LocalDateTime createdAt;
    
    @Column("updated_at")
    private LocalDateTime updatedAt;
    
    @Transient
    private Category category; // 조인된 데이터
}

// 리액티브 리포지토리
@Repository
public interface ReactiveProductRepository extends ReactiveCrudRepository<Product, String> {
    
    Flux<Product> findByCategory(String category);
    
    Flux<Product> findByPriceBetween(BigDecimal minPrice, BigDecimal maxPrice);
    
    @Query("SELECT * FROM products WHERE name ILIKE :name")
    Flux<Product> findByNameContainingIgnoreCase(@Param("name") String name);
    
    @Query("SELECT * FROM products WHERE stock_quantity > 0 ORDER BY created_at DESC LIMIT :limit OFFSET :offset")
    Flux<Product> findAvailableProducts(@Param("limit") int limit, @Param("offset") int offset);
    
    @Modifying
    @Query("UPDATE products SET stock_quantity = stock_quantity - :quantity WHERE id = :id AND stock_quantity >= :quantity")
    Mono<Integer> decreaseStock(@Param("id") String id, @Param("quantity") int quantity);
    
    @Query("SELECT COUNT(*) FROM products WHERE category_id = :categoryId")
    Mono<Long> countByCategory(@Param("categoryId") String categoryId);
}

// 커스텀 리포지토리 구현
@Repository
@RequiredArgsConstructor
public class CustomReactiveProductRepository {
    
    private final DatabaseClient databaseClient;
    
    public Flux<Product> searchProducts(ProductSearchCriteria criteria) {
        StringBuilder sql = new StringBuilder("SELECT * FROM products WHERE 1=1");
        Map<String, Object> parameters = new HashMap<>();
        
        if (criteria.getName() != null) {
            sql.append(" AND name ILIKE :name");
            parameters.put("name", "%" + criteria.getName() + "%");
        }
        
        if (criteria.getCategoryId() != null) {
            sql.append(" AND category_id = :categoryId");
            parameters.put("categoryId", criteria.getCategoryId());
        }
        
        if (criteria.getMinPrice() != null) {
            sql.append(" AND price >= :minPrice");
            parameters.put("minPrice", criteria.getMinPrice());
        }
        
        if (criteria.getMaxPrice() != null) {
            sql.append(" AND price <= :maxPrice");
            parameters.put("maxPrice", criteria.getMaxPrice());
        }
        
        sql.append(" ORDER BY created_at DESC");
        
        if (criteria.getLimit() != null) {
            sql.append(" LIMIT :limit");
            parameters.put("limit", criteria.getLimit());
        }
        
        if (criteria.getOffset() != null) {
            sql.append(" OFFSET :offset");
            parameters.put("offset", criteria.getOffset());
        }
        
        DatabaseClient.GenericExecuteSpec executeSpec = databaseClient.sql(sql.toString());
        
        for (Map.Entry<String, Object> entry : parameters.entrySet()) {
            executeSpec = executeSpec.bind(entry.getKey(), entry.getValue());
        }
        
        return executeSpec
            .map((row, metadata) -> Product.builder()
                .id(row.get("id", String.class))
                .name(row.get("name", String.class))
                .description(row.get("description", String.class))
                .price(row.get("price", BigDecimal.class))
                .categoryId(row.get("category_id", String.class))
                .stockQuantity(row.get("stock_quantity", Integer.class))
                .createdAt(row.get("created_at", LocalDateTime.class))
                .updatedAt(row.get("updated_at", LocalDateTime.class))
                .build())
            .all();
    }
    
    public Flux<ProductStatistics> getProductStatistics() {
        return databaseClient.sql("""
            SELECT 
                category_id,
                COUNT(*) as product_count,
                AVG(price) as average_price,
                MIN(price) as min_price,
                MAX(price) as max_price,
                SUM(stock_quantity) as total_stock
            FROM products 
            GROUP BY category_id
            """)
            .map((row, metadata) -> ProductStatistics.builder()
                .categoryId(row.get("category_id", String.class))
                .productCount(row.get("product_count", Long.class))
                .averagePrice(row.get("average_price", BigDecimal.class))
                .minPrice(row.get("min_price", BigDecimal.class))
                .maxPrice(row.get("max_price", BigDecimal.class))
                .totalStock(row.get("total_stock", Long.class))
                .build())
            .all();
    }
}
```

### 리액티브 서비스 구현

```java
@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class ReactiveProductService {
    
    private final ReactiveProductRepository productRepository;
    private final CustomReactiveProductRepository customRepository;
    private final CategoryRepository categoryRepository;
    private final ReactiveRedisTemplate<String, Object> redisTemplate;
    
    public Flux<ProductDto> getAllProducts(int page, int size, String category) {
        Flux<Product> products;
        
        if (category != null) {
            products = productRepository.findByCategory(category);
        } else {
            products = customRepository.searchProducts(ProductSearchCriteria.builder()
                .limit(size)
                .offset(page * size)
                .build());
        }
        
        return products
            .flatMap(this::enrichWithCategory)
            .map(this::convertToDto)
            .doOnNext(product -> log.debug("Loaded product: {}", product.getId()));
    }
    
    public Mono<ProductDto> getProductById(String id) {
        String cacheKey = "product:" + id;
        
        return redisTemplate.opsForValue()
            .get(cacheKey)
            .cast(ProductDto.class)
            .switchIfEmpty(productRepository.findById(id)
                .flatMap(this::enrichWithCategory)
                .map(this::convertToDto)
                .flatMap(dto -> redisTemplate.opsForValue()
                    .set(cacheKey, dto, Duration.ofMinutes(10))
                    .then(Mono.just(dto))))
            .doOnSuccess(product -> log.info("Retrieved product: {}", product != null ? product.getId() : "null"));
    }
    
    @Transactional
    public Mono<ProductDto> createProduct(CreateProductRequest request) {
        return validateCreateRequest(request)
            .then(Mono.defer(() -> {
                Product product = Product.builder()
                    .id(UUID.randomUUID().toString())
                    .name(request.getName())
                    .description(request.getDescription())
                    .price(request.getPrice())
                    .categoryId(request.getCategoryId())
                    .stockQuantity(request.getStockQuantity())
                    .createdAt(LocalDateTime.now())
                    .updatedAt(LocalDateTime.now())
                    .build();
                
                return productRepository.save(product);
            }))
            .flatMap(this::enrichWithCategory)
            .map(this::convertToDto)
            .doOnSuccess(product -> {
                log.info("Created product: {}", product.getId());
                invalidateCache(product.getId());
            });
    }
    
    @Transactional
    public Mono<ProductDto> updateProduct(String id, UpdateProductRequest request) {
        return productRepository.findById(id)
            .switchIfEmpty(Mono.error(new ProductNotFoundException("Product not found: " + id)))
            .flatMap(existingProduct -> {
                existingProduct.setName(request.getName());
                existingProduct.setDescription(request.getDescription());
                existingProduct.setPrice(request.getPrice());
                existingProduct.setStockQuantity(request.getStockQuantity());
                existingProduct.setUpdatedAt(LocalDateTime.now());
                
                return productRepository.save(existingProduct);
            })
            .flatMap(this::enrichWithCategory)
            .map(this::convertToDto)
            .doOnSuccess(product -> {
                log.info("Updated product: {}", product.getId());
                invalidateCache(product.getId());
            });
    }
    
    @Transactional
    public Mono<Boolean> deleteProduct(String id) {
        return productRepository.existsById(id)
            .flatMap(exists -> {
                if (exists) {
                    return productRepository.deleteById(id)
                        .then(Mono.just(true))
                        .doOnSuccess(result -> {
                            log.info("Deleted product: {}", id);
                            invalidateCache(id);
                        });
                } else {
                    return Mono.just(false);
                }
            });
    }
    
    public Flux<ProductDto> streamProducts() {
        return Flux.interval(Duration.ofSeconds(1))
            .flatMap(tick -> productRepository.findAll()
                .take(1)
                .flatMap(this::enrichWithCategory)
                .map(this::convertToDto))
            .doOnNext(product -> log.debug("Streaming product: {}", product.getId()));
    }
    
    public Flux<ProductEventDto> getProductEvents() {
        // 실제로는 메시지 브로커나 이벤트 스토어에서 이벤트를 구독
        return Flux.interval(Duration.ofSeconds(2))
            .map(tick -> ProductEventDto.builder()
                .id(UUID.randomUUID().toString())
                .type("PRODUCT_UPDATED")
                .productId("product-" + tick)
                .timestamp(LocalDateTime.now())
                .build());
    }
    
    private Mono<Product> enrichWithCategory(Product product) {
        if (product.getCategoryId() != null) {
            return categoryRepository.findById(product.getCategoryId())
                .map(category -> {
                    product.setCategory(category);
                    return product;
                })
                .defaultIfEmpty(product);
        }
        return Mono.just(product);
    }
    
    private ProductDto convertToDto(Product product) {
        return ProductDto.builder()
            .id(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice())
            .categoryId(product.getCategoryId())
            .categoryName(product.getCategory() != null ? product.getCategory().getName() : null)
            .stockQuantity(product.getStockQuantity())
            .createdAt(product.getCreatedAt())
            .updatedAt(product.getUpdatedAt())
            .build();
    }
    
    private Mono<Void> validateCreateRequest(CreateProductRequest request) {
        return categoryRepository.existsById(request.getCategoryId())
            .flatMap(exists -> {
                if (!exists) {
                    return Mono.error(new IllegalArgumentException("Category not found: " + request.getCategoryId()));
                }
                return Mono.empty();
            });
    }
    
    private void invalidateCache(String productId) {
        redisTemplate.delete("product:" + productId).subscribe();
    }
}
```

## 🌐 WebClient 사용법

### 기본 WebClient 설정

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.USER_AGENT, "ReactiveApp/1.0")
            .filter(logRequest())
            .filter(logResponse())
            .filter(errorHandler())
            .codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(1024 * 1024));
    }
    
    @Bean
    public WebClient externalApiClient(WebClient.Builder builder) {
        return builder
            .baseUrl("https://api.external-service.com")
            .defaultHeaders(headers -> {
                headers.setBearerAuth("your-api-token");
                headers.set("X-API-Version", "v1");
            })
            .build();
    }
    
    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(clientRequest -> {
            log.info("Request: {} {}", clientRequest.method(), clientRequest.url());
            clientRequest.headers().forEach((name, values) -> 
                values.forEach(value -> log.debug("Request header: {}={}", name, value)));
            return Mono.just(clientRequest);
        });
    }
    
    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(clientResponse -> {
            log.info("Response: {}", clientResponse.statusCode());
            return Mono.just(clientResponse);
        });
    }
    
    private ExchangeFilterFunction errorHandler() {
        return ExchangeFilterFunction.ofResponseProcessor(clientResponse -> {
            if (clientResponse.statusCode().isError()) {
                return clientResponse.bodyToMono(String.class)
                    .flatMap(errorBody -> {
                        log.error("Error response: {} - {}", clientResponse.statusCode(), errorBody);
                        return Mono.error(new ExternalApiException(
                            "External API error: " + clientResponse.statusCode(), errorBody));
                    });
            }
            return Mono.just(clientResponse);
        });
    }
}
```

### WebClient 사용 예제

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ExternalApiService {
    
    private final WebClient externalApiClient;
    
    // GET 요청
    public Mono<UserDto> getUser(String userId) {
        return externalApiClient
            .get()
            .uri("/users/{id}", userId)
            .retrieve()
            .bodyToMono(UserDto.class)
            .timeout(Duration.ofSeconds(10))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(this::isRetryableException))
            .doOnSuccess(user -> log.info("Retrieved user: {}", user.getId()))
            .doOnError(error -> log.error("Failed to get user: {}", userId, error));
    }
    
    // POST 요청
    public Mono<OrderDto> createOrder(CreateOrderRequest request) {
        return externalApiClient
            .post()
            .uri("/orders")
            .body(Mono.just(request), CreateOrderRequest.class)
            .retrieve()
            .bodyToMono(OrderDto.class)
            .doOnSuccess(order -> log.info("Created order: {}", order.getId()));
    }
    
    // 파일 업로드
    public Mono<UploadResult> uploadFile(MultipartFile file) {
        MultiValueMap<String, HttpEntity<?>> parts = new LinkedMultiValueMap<>();
        parts.add("file", new FileSystemResource(file.getResource().getFile()));
        parts.add("metadata", new HttpEntity<>(Map.of("filename", file.getOriginalFilename())));
        
        return externalApiClient
            .post()
            .uri("/upload")
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .body(BodyInserters.fromMultipartData(parts))
            .retrieve()
            .bodyToMono(UploadResult.class);
    }
    
    // 스트리밍 응답
    public Flux<DataChunk> streamData(String query) {
        return externalApiClient
            .get()
            .uri(uriBuilder -> uriBuilder
                .path("/stream")
                .queryParam("q", query)
                .build())
            .accept(MediaType.APPLICATION_NDJSON)
            .retrieve()
            .bodyToFlux(DataChunk.class)
            .doOnNext(chunk -> log.debug("Received chunk: {}", chunk.getId()));
    }
    
    // 병렬 요청
    public Mono<CombinedData> getCombinedData(String userId) {
        Mono<UserDto> userMono = getUser(userId);
        Mono<List<OrderDto>> ordersMono = getUserOrders(userId);
        Mono<UserPreferences> preferencesMono = getUserPreferences(userId);
        
        return Mono.zip(userMono, ordersMono, preferencesMono)
            .map(tuple -> CombinedData.builder()
                .user(tuple.getT1())
                .orders(tuple.getT2())
                .preferences(tuple.getT3())
                .build());
    }
    
    // 조건부 요청
    public Mono<ApiResponse<ProductDto>> getProductWithETag(String productId, String etag) {
        return externalApiClient
            .get()
            .uri("/products/{id}", productId)
            .headers(headers -> {
                if (etag != null) {
                    headers.setIfNoneMatch(etag);
                }
            })
            .exchangeToMono(response -> {
                if (response.statusCode() == HttpStatus.NOT_MODIFIED) {
                    return Mono.just(ApiResponse.<ProductDto>builder()
                        .notModified(true)
                        .etag(etag)
                        .build());
                } else {
                    return response.bodyToMono(ProductDto.class)
                        .map(product -> ApiResponse.<ProductDto>builder()
                            .data(product)
                            .etag(response.headers().header("ETag").stream().findFirst().orElse(null))
                            .build());
                }
            });
    }
    
    private boolean isRetryableException(Throwable throwable) {
        return throwable instanceof WebClientResponseException &&
               ((WebClientResponseException) throwable).getStatusCode().is5xxServerError();
    }
    
    private Mono<List<OrderDto>> getUserOrders(String userId) {
        return externalApiClient
            .get()
            .uri("/users/{id}/orders", userId)
            .retrieve()
            .bodyToFlux(OrderDto.class)
            .collectList();
    }
    
    private Mono<UserPreferences> getUserPreferences(String userId) {
        return externalApiClient
            .get()
            .uri("/users/{id}/preferences", userId)
            .retrieve()
            .bodyToMono(UserPreferences.class);
    }
}
```

## ❌ 에러 처리

### 글로벌 에러 핸들러

```java
@Component
@Order(-2)
@RequiredArgsConstructor
@Slf4j
public class GlobalErrorWebExceptionHandler implements WebExceptionHandler {
    
    private final ObjectMapper objectMapper;
    
    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        ServerHttpResponse response = exchange.getResponse();
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        
        ErrorResponse errorResponse;
        HttpStatus status;
        
        if (ex instanceof ValidationException) {
            status = HttpStatus.BAD_REQUEST;
            errorResponse = ErrorResponse.builder()
                .error("VALIDATION_ERROR")
                .message(ex.getMessage())
                .timestamp(LocalDateTime.now())
                .path(exchange.getRequest().getPath().value())
                .build();
                
        } else if (ex instanceof ProductNotFoundException) {
            status = HttpStatus.NOT_FOUND;
            errorResponse = ErrorResponse.builder()
                .error("PRODUCT_NOT_FOUND")
                .message(ex.getMessage())
                .timestamp(LocalDateTime.now())
                .path(exchange.getRequest().getPath().value())
                .build();
                
        } else if (ex instanceof WebExchangeBindException) {
            status = HttpStatus.BAD_REQUEST;
            WebExchangeBindException bindException = (WebExchangeBindException) ex;
            
            Map<String, String> fieldErrors = bindException.getFieldErrors().stream()
                .collect(Collectors.toMap(
                    FieldError::getField,
                    FieldError::getDefaultMessage,
                    (existing, replacement) -> existing
                ));
            
            errorResponse = ErrorResponse.builder()
                .error("VALIDATION_ERROR")
                .message("Validation failed")
                .fieldErrors(fieldErrors)
                .timestamp(LocalDateTime.now())
                .path(exchange.getRequest().getPath().value())
                .build();
                
        } else if (ex instanceof TimeoutException) {
            status = HttpStatus.REQUEST_TIMEOUT;
            errorResponse = ErrorResponse.builder()
                .error("TIMEOUT")
                .message("Request timeout")
                .timestamp(LocalDateTime.now())
                .path(exchange.getRequest().getPath().value())
                .build();
                
        } else {
            status = HttpStatus.INTERNAL_SERVER_ERROR;
            errorResponse = ErrorResponse.builder()
                .error("INTERNAL_SERVER_ERROR")
                .message("An unexpected error occurred")
                .timestamp(LocalDateTime.now())
                .path(exchange.getRequest().getPath().value())
                .build();
        }
        
        response.setStatusCode(status);
        
        try {
            String responseBody = objectMapper.writeValueAsString(errorResponse);
            DataBuffer buffer = response.bufferFactory().wrap(responseBody.getBytes(StandardCharsets.UTF_8));
            
            return response.writeWith(Mono.just(buffer))
                .doOnError(error -> log.error("Error writing response", error));
                
        } catch (JsonProcessingException e) {
            log.error("Error serializing error response", e);
            return response.setComplete();
        }
    }
}

@Data
@Builder
public class ErrorResponse {
    private String error;
    private String message;
    private Map<String, String> fieldErrors;
    private LocalDateTime timestamp;
    private String path;
}
```

### 서비스 레벨 에러 처리

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class RobustReactiveService {
    
    private final WebClient webClient;
    private final ReactiveProductRepository productRepository;
    
    public Mono<String> getDataWithErrorHandling(String id) {
        return webClient.get()
            .uri("/api/data/{id}", id)
            .retrieve()
            .bodyToMono(String.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(this::isRetryable)
                .doBeforeRetry(retrySignal -> 
                    log.warn("Retrying request for id: {}, attempt: {}", 
                            id, retrySignal.totalRetries() + 1)))
            .onErrorResume(TimeoutException.class, ex -> {
                log.warn("Request timeout for id: {}", id);
                return Mono.just("TIMEOUT");
            })
            .onErrorResume(WebClientResponseException.class, ex -> {
                log.error("HTTP error for id: {}, status: {}", id, ex.getStatusCode());
                if (ex.getStatusCode() == HttpStatus.NOT_FOUND) {
                    return Mono.empty();
                }
                return Mono.error(new ServiceException("External service error", ex));
            })
            .onErrorResume(Exception.class, ex -> {
                log.error("Unexpected error for id: {}", id, ex);
                return Mono.error(new ServiceException("Unexpected error", ex));
            });
    }
    
    public Mono<ProductDto> getProductWithFallback(String id) {
        return productRepository.findById(id)
            .map(this::convertToDto)
            .switchIfEmpty(Mono.defer(() -> {
                log.info("Product not found in database, trying cache: {}", id);
                return getCachedProduct(id);
            }))
            .switchIfEmpty(Mono.defer(() -> {
                log.info("Product not found in cache, trying external API: {}", id);
                return getProductFromExternalApi(id);
            }))
            .switchIfEmpty(Mono.error(new ProductNotFoundException("Product not found: " + id)))
            .doOnSuccess(product -> log.info("Successfully retrieved product: {}", product.getId()))
            .doOnError(error -> log.error("Failed to retrieve product: {}", id, error));
    }
    
    // 회로 차단기 패턴 구현
    public Mono<String> getDataWithCircuitBreaker(String id) {
        return CircuitBreaker.ofDefaults("externalService")
            .executeSupplier(() -> callExternalService(id).block())
            .onFailure(throwable -> log.error("Circuit breaker opened for service call", throwable))
            .recover(throwable -> "FALLBACK_VALUE");
    }
    
    private boolean isRetryable(Throwable throwable) {
        return throwable instanceof WebClientResponseException &&
               ((WebClientResponseException) throwable).getStatusCode().is5xxServerError();
    }
    
    private Mono<String> callExternalService(String id) {
        return webClient.get()
            .uri("/api/data/{id}", id)
            .retrieve()
            .bodyToMono(String.class);
    }
    
    private Mono<ProductDto> getCachedProduct(String id) {
        // 캐시에서 조회
        return Mono.empty();
    }
    
    private Mono<ProductDto> getProductFromExternalApi(String id) {
        // 외부 API에서 조회
        return Mono.empty();
    }
    
    private ProductDto convertToDto(Product product) {
        return ProductDto.builder()
            .id(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice())
            .build();
    }
}
```

## 📚 관련 노트

- [[Spring Web MVC]] - 전통적인 웹 MVC와 비교
- [[Spring Data JPA]] - 블로킹 vs 논블로킹 데이터 액세스
- [[API Design Patterns]] - 리액티브 API 설계
- [[Testing]] - 리액티브 코드 테스트
- [[Observability]] - 리액티브 애플리케이션 모니터링

## 🔗 외부 리소스

- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
- [Project Reactor Reference](https://projectreactor.io/docs/core/release/reference/)
- [Reactive Streams Specification](https://www.reactive-streams.org/)
- [R2DBC Documentation](https://r2dbc.io/)

---

*리액티브 프로그래밍은 패러다임의 변화를 요구합니다. 점진적으로 학습하고 적용해 나가세요.*
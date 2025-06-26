# Spring Boot Redis Kafka Elasticsearch Integration

> Spring Boot에서 Redis, Kafka, Elasticsearch를 활용한 고성능 분산 시스템 구축

## 📋 목차

- [통합 아키텍처 개요](#통합-아키텍처-개요)
- [Redis 통합](#redis-통합)
- [Kafka 통합](#kafka-통합)
- [Elasticsearch 통합](#elasticsearch-통합)
- [통합 실습 프로젝트](#통합-실습-프로젝트)
- [성능 최적화](#성능-최적화)
- [모니터링과 운영](#모니터링과-운영)

## 🏗 통합 아키텍처 개요

### 기술 스택 역할

```yaml
Redis:
  용도:
    - 캐싱 레이어
    - 세션 저장소
    - 분산 락
    - 실시간 데이터 저장
    - 메시지 브로커 (Pub/Sub)
  장점:
    - 인메모리 고성능
    - 다양한 데이터 구조 지원
    - 클러스터링 지원

Kafka:
  용도:
    - 이벤트 스트리밍
    - 마이크로서비스 간 통신
    - 로그 수집
    - 실시간 데이터 파이프라인
  장점:
    - 높은 처리량
    - 내구성
    - 확장성

Elasticsearch:
  용도:
    - 전문 검색
    - 로그 분석
    - 실시간 분석
    - 데이터 시각화
  장점:
    - 빠른 검색 성능
    - 분산 아키텍처
    - RESTful API
```

### 의존성 설정

```gradle
dependencies {
    // Spring Boot 기본
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    // Redis
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.session:spring-session-data-redis'
    implementation 'org.redisson:redisson-spring-boot-starter:3.24.3'
    
    // Kafka
    implementation 'org.springframework.kafka:spring-kafka'
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-kafka'
    
    // Elasticsearch
    implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
    implementation 'co.elastic.clients:elasticsearch-java:8.11.0'
    
    // JSON 처리
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
    
    // 모니터링
    implementation 'io.micrometer:micrometer-registry-prometheus'
    
    // 테스트
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.kafka:spring-kafka-test'
    testImplementation 'org.testcontainers:elasticsearch'
    testImplementation 'org.testcontainers:kafka'
    testImplementation 'org.testcontainers:junit-jupiter'
}
```

## 🔴 Redis 통합

### Redis 설정

```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD:}
    timeout: 2000ms
    ssl: false
    cluster:
      nodes:
        - redis-node1:6379
        - redis-node2:6379
        - redis-node3:6379
      max-redirects: 3
    jedis:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
        max-wait: -1ms
    cache:
      type: redis
      redis:
        time-to-live: 600000  # 10분
        cache-null-values: false

  session:
    store-type: redis
    redis:
      namespace: spring:session
      flush-mode: on_save
```

### Redis 설정 클래스

```java
@Configuration
@EnableRedisRepositories
@RequiredArgsConstructor
public class RedisConfig {
    
    @Value("${spring.redis.host}")
    private String redisHost;
    
    @Value("${spring.redis.port}")
    private int redisPort;
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        LettuceClientConfiguration clientConfiguration = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(2))
            .shutdownTimeout(Duration.ZERO)
            .build();
        
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration(redisHost, redisPort),
            clientConfiguration
        );
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        // JSON 직렬화 설정
        GenericJackson2JsonRedisSerializer serializer = new GenericJackson2JsonRedisSerializer();
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        
        template.setDefaultSerializer(serializer);
        template.afterPropertiesSet();
        
        return template;
    }
    
    @Bean
    public StringRedisTemplate stringRedisTemplate() {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory());
        return template;
    }
    
    @Bean
    public RedisCacheManager cacheManager() {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();
        
        return RedisCacheManager.builder(redisConnectionFactory())
            .cacheDefaults(config)
            .transactionAware()
            .build();
    }
}
```

### Redis 캐싱 구현

```java
// 캐시 서비스
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductCacheService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;
    
    private static final String PRODUCT_CACHE_PREFIX = "product:";
    private static final String POPULAR_PRODUCTS_KEY = "popular:products";
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    
    @Cacheable(value = "products", key = "#productId")
    public ProductDto getProduct(String productId) {
        log.info("데이터베이스에서 상품 조회: {}", productId);
        
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        return ProductDto.from(product);
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public void evictProduct(String productId) {
        log.info("상품 캐시 삭제: {}", productId);
    }
    
    @CachePut(value = "products", key = "#result.id")
    public ProductDto updateProduct(String productId, UpdateProductRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.updateDetails(request.getName(), request.getDescription(), request.getPrice());
        Product savedProduct = productRepository.save(product);
        
        return ProductDto.from(savedProduct);
    }
    
    // 수동 캐시 관리
    public void cacheProduct(ProductDto product) {
        String key = PRODUCT_CACHE_PREFIX + product.getId();
        redisTemplate.opsForValue().set(key, product, CACHE_TTL);
    }
    
    public Optional<ProductDto> getCachedProduct(String productId) {
        String key = PRODUCT_CACHE_PREFIX + productId;
        ProductDto product = (ProductDto) redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(product);
    }
    
    // 인기 상품 목록 캐싱 (Redis List 사용)
    public void updatePopularProducts(List<ProductDto> products) {
        redisTemplate.delete(POPULAR_PRODUCTS_KEY);
        
        products.forEach(product -> 
            redisTemplate.opsForList().rightPush(POPULAR_PRODUCTS_KEY, product));
        
        redisTemplate.expire(POPULAR_PRODUCTS_KEY, Duration.ofHours(6));
    }
    
    public List<ProductDto> getPopularProducts() {
        List<Object> cachedProducts = redisTemplate.opsForList()
            .range(POPULAR_PRODUCTS_KEY, 0, -1);
        
        return cachedProducts.stream()
            .map(obj -> (ProductDto) obj)
            .collect(Collectors.toList());
    }
}
```

### Redis 세션 관리

```java
// 세션 기반 장바구니 서비스
@Service
@RequiredArgsConstructor
@Slf4j
public class ShoppingCartService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductService productService;
    
    private static final String CART_PREFIX = "cart:";
    private static final Duration CART_TTL = Duration.ofDays(7);
    
    public void addToCart(String sessionId, String productId, Integer quantity) {
        String cartKey = CART_PREFIX + sessionId;
        
        // 상품 정보 조회
        ProductDto product = productService.getProduct(productId);
        
        CartItem cartItem = CartItem.builder()
            .productId(productId)
            .productName(product.getName())
            .price(product.getPrice())
            .quantity(quantity)
            .addedAt(LocalDateTime.now())
            .build();
        
        // Redis Hash에 저장
        redisTemplate.opsForHash().put(cartKey, productId, cartItem);
        redisTemplate.expire(cartKey, CART_TTL);
        
        log.info("장바구니에 상품 추가: session={}, product={}, quantity={}", 
                sessionId, productId, quantity);
    }
    
    public void removeFromCart(String sessionId, String productId) {
        String cartKey = CART_PREFIX + sessionId;
        redisTemplate.opsForHash().delete(cartKey, productId);
        
        log.info("장바구니에서 상품 제거: session={}, product={}", sessionId, productId);
    }
    
    public Cart getCart(String sessionId) {
        String cartKey = CART_PREFIX + sessionId;
        Map<Object, Object> cartItems = redisTemplate.opsForHash().entries(cartKey);
        
        List<CartItem> items = cartItems.values().stream()
            .map(obj -> (CartItem) obj)
            .collect(Collectors.toList());
        
        BigDecimal totalAmount = items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        return Cart.builder()
            .sessionId(sessionId)
            .items(items)
            .totalAmount(totalAmount)
            .itemCount(items.size())
            .build();
    }
    
    public void clearCart(String sessionId) {
        String cartKey = CART_PREFIX + sessionId;
        redisTemplate.delete(cartKey);
        
        log.info("장바구니 비우기: session={}", sessionId);
    }
}
```

### Redis 분산 락

```java
// Redisson을 이용한 분산 락
@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryService {
    
    private final RedissonClient redissonClient;
    private final ProductRepository productRepository;
    
    public void decreaseStock(String productId, Integer quantity) {
        String lockKey = "product:lock:" + productId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 락 획득 (최대 10초 대기, 30초 후 자동 해제)
            boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException("재고 처리를 위한 락 획득 실패: " + productId);
            }
            
            // 재고 감소 처리
            Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));
            
            if (product.getStock() < quantity) {
                throw new InsufficientStockException("재고 부족: " + productId);
            }
            
            product.decreaseStock(quantity);
            productRepository.save(product);
            
            log.info("재고 감소 완료: product={}, quantity={}, remaining={}", 
                    productId, quantity, product.getStock());
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("락 대기 중 인터럽트 발생", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### Redis Pub/Sub 실시간 알림

```java
// Redis Pub/Sub 설정
@Configuration
public class RedisPubSubConfig {
    
    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory) {
        
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        return container;
    }
    
    @Bean
    public MessageListenerAdapter orderNotificationListener() {
        return new MessageListenerAdapter(new OrderNotificationListener(), "onMessage");
    }
}

// 실시간 알림 발행자
@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationPublisher {
    
    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;
    
    public void publishOrderStatus(String customerId, OrderStatusNotification notification) {
        try {
            String channel = "order:notifications:" + customerId;
            String message = objectMapper.writeValueAsString(notification);
            
            redisTemplate.convertAndSend(channel, message);
            
            log.info("주문 상태 알림 발행: customer={}, order={}", 
                    customerId, notification.getOrderId());
            
        } catch (Exception e) {
            log.error("알림 발행 실패", e);
        }
    }
    
    public void publishGlobalNotification(GlobalNotification notification) {
        try {
            String message = objectMapper.writeValueAsString(notification);
            redisTemplate.convertAndSend("global:notifications", message);
            
            log.info("전역 알림 발행: {}", notification.getTitle());
            
        } catch (Exception e) {
            log.error("전역 알림 발행 실패", e);
        }
    }
}

// 알림 구독자
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderNotificationListener {
    
    private final SimpMessagingTemplate messagingTemplate;
    private final ObjectMapper objectMapper;
    
    public void onMessage(String message, String channel) {
        try {
            if (channel.startsWith("order:notifications:")) {
                String customerId = channel.substring("order:notifications:".length());
                OrderStatusNotification notification = objectMapper.readValue(
                    message, OrderStatusNotification.class);
                
                // WebSocket으로 클라이언트에 전송
                messagingTemplate.convertAndSendToUser(
                    customerId, "/queue/notifications", notification);
                
                log.info("주문 알림 전송 완료: customer={}", customerId);
                
            } else if ("global:notifications".equals(channel)) {
                GlobalNotification notification = objectMapper.readValue(
                    message, GlobalNotification.class);
                
                // 전체 사용자에게 브로드캐스트
                messagingTemplate.convertAndSend("/topic/global", notification);
                
                log.info("전역 알림 전송 완료: {}", notification.getTitle());
            }
            
        } catch (Exception e) {
            log.error("알림 처리 실패: channel={}, message={}", channel, message, e);
        }
    }
}
```

## ⚡ Kafka 통합

### Kafka 설정

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      batch-size: 16384
      linger-ms: 5
      buffer-memory: 33554432
      compression-type: snappy
      enable-idempotence: true
    
    consumer:
      group-id: ecommerce-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "com.example.events"
        max.poll.records: 500
        max.poll.interval.ms: 300000
    
    listener:
      ack-mode: manual_immediate
      concurrency: 3
      poll-timeout: 3000
      
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
          auto-create-topics: true
          auto-add-partitions: true
        bindings:
          order-events-out-0:
            producer:
              configuration:
                acks: all
                retries: 3
          inventory-events-in-0:
            consumer:
              configuration:
                max.poll.records: 100
```

### Kafka 프로듀서 구현

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventProducer {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;
    
    @Value("${app.kafka.topics.order-events}")
    private String orderEventsTopic;
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        try {
            // 파티션 키로 주문 ID 사용 (같은 주문의 이벤트는 순서 보장)
            kafkaTemplate.send(orderEventsTopic, event.getOrderId(), event)
                .addCallback(
                    result -> log.info("주문 생성 이벤트 발행 성공: {}", event.getOrderId()),
                    failure -> log.error("주문 생성 이벤트 발행 실패: {}", event.getOrderId(), failure)
                );
                
        } catch (Exception e) {
            log.error("주문 생성 이벤트 발행 중 예외 발생: {}", event.getOrderId(), e);
            throw new EventPublishException("이벤트 발행 실패", e);
        }
    }
    
    public void publishOrderStatusChanged(OrderStatusChangedEvent event) {
        try {
            kafkaTemplate.send(orderEventsTopic, event.getOrderId(), event);
            log.info("주문 상태 변경 이벤트 발행: {} -> {}", 
                    event.getOrderId(), event.getNewStatus());
                    
        } catch (Exception e) {
            log.error("주문 상태 변경 이벤트 발행 실패: {}", event.getOrderId(), e);
            throw new EventPublishException("이벤트 발행 실패", e);
        }
    }
    
    // 트랜잭션을 통한 안전한 이벤트 발행
    @Transactional
    public void publishOrderEventsTransactionally(List<Object> events) {
        kafkaTemplate.executeInTransaction(operations -> {
            events.forEach(event -> {
                if (event instanceof OrderCreatedEvent) {
                    OrderCreatedEvent orderEvent = (OrderCreatedEvent) event;
                    operations.send(orderEventsTopic, orderEvent.getOrderId(), orderEvent);
                } else if (event instanceof OrderStatusChangedEvent) {
                    OrderStatusChangedEvent statusEvent = (OrderStatusChangedEvent) event;
                    operations.send(orderEventsTopic, statusEvent.getOrderId(), statusEvent);
                }
            });
            return true;
        });
    }
}
```

### Kafka 컨슈머 구현

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventConsumer {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    @KafkaListener(
        topics = "${app.kafka.topics.order-events}",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvents(
            @Payload Object event,
            @Header Map<String, Object> headers,
            Acknowledgment ack) {
        
        try {
            String eventType = (String) headers.get("__TypeId__");
            
            log.info("주문 이벤트 수신: type={}, partition={}, offset={}", 
                    eventType, 
                    headers.get(KafkaHeaders.RECEIVED_PARTITION_ID),
                    headers.get(KafkaHeaders.OFFSET));
            
            switch (eventType) {
                case "OrderCreatedEvent":
                    handleOrderCreated((OrderCreatedEvent) event);
                    break;
                case "OrderStatusChangedEvent":
                    handleOrderStatusChanged((OrderStatusChangedEvent) event);
                    break;
                default:
                    log.warn("알 수 없는 이벤트 타입: {}", eventType);
            }
            
            // 수동 커밋
            ack.acknowledge();
            
        } catch (Exception e) {
            log.error("주문 이벤트 처리 실패", e);
            // 에러 처리 로직 (재시도, DLQ 등)
            handleEventProcessingError(event, e);
        }
    }
    
    private void handleOrderCreated(OrderCreatedEvent event) {
        log.info("주문 생성 이벤트 처리: {}", event.getOrderId());
        
        // 재고 예약
        event.getItems().forEach(item -> 
            inventoryService.reserveStock(item.getProductId(), item.getQuantity()));
        
        // 고객에게 알림
        notificationService.sendOrderCreatedNotification(
            event.getCustomerId(), event.getOrderId());
    }
    
    private void handleOrderStatusChanged(OrderStatusChangedEvent event) {
        log.info("주문 상태 변경 이벤트 처리: {} -> {}", 
                event.getOrderId(), event.getNewStatus());
        
        switch (event.getNewStatus()) {
            case CONFIRMED:
                inventoryService.confirmReservation(event.getOrderId());
                paymentService.capturePayment(event.getOrderId());
                break;
            case CANCELLED:
                inventoryService.releaseReservation(event.getOrderId());
                paymentService.refundPayment(event.getOrderId());
                break;
            case SHIPPED:
                notificationService.sendShippingNotification(event.getOrderId());
                break;
        }
    }
    
    private void handleEventProcessingError(Object event, Exception error) {
        // 에러 이벤트를 DLQ로 전송
        kafkaTemplate.send("order-events-dlq", event);
        
        // 메트릭 기록
        Metrics.counter("kafka.consumer.error", 
            "event_type", event.getClass().getSimpleName()).increment();
    }
}
```

### Kafka Streams 구현

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderAnalyticsStreams {
    
    @Bean
    public KStream<String, OrderCreatedEvent> orderAnalytics(StreamsBuilder streamsBuilder) {
        KStream<String, OrderCreatedEvent> orderStream = streamsBuilder
            .stream("order-events", Consumed.with(Serdes.String(), orderEventSerde()));
        
        // 주문 금액별 분류
        orderStream
            .filter((key, order) -> order.getTotalAmount().compareTo(BigDecimal.valueOf(100)) > 0)
            .mapValues(order -> OrderAnalytics.builder()
                .orderId(order.getOrderId())
                .customerId(order.getCustomerId())
                .totalAmount(order.getTotalAmount())
                .category("HIGH_VALUE")
                .timestamp(order.getOccurredOn())
                .build())
            .to("high-value-orders", Produced.with(Serdes.String(), orderAnalyticsSerde()));
        
        // 시간대별 주문 통계
        orderStream
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofHours(1)))
            .aggregate(
                OrderHourlyStats::new,
                (key, order, stats) -> {
                    stats.incrementOrderCount();
                    stats.addAmount(order.getTotalAmount());
                    return stats;
                },
                Materialized.with(Serdes.String(), orderHourlyStatsSerde())
            )
            .toStream()
            .map((windowedKey, stats) -> {
                String key = windowedKey.key() + "@" + windowedKey.window().start();
                return KeyValue.pair(key, stats);
            })
            .to("order-hourly-stats", Produced.with(Serdes.String(), orderHourlyStatsSerde()));
        
        return orderStream;
    }
    
    private Serde<OrderCreatedEvent> orderEventSerde() {
        return Serdes.serdeFrom(
            new JsonSerializer<>(),
            new JsonDeserializer<>(OrderCreatedEvent.class)
        );
    }
    
    private Serde<OrderAnalytics> orderAnalyticsSerde() {
        return Serdes.serdeFrom(
            new JsonSerializer<>(),
            new JsonDeserializer<>(OrderAnalytics.class)
        );
    }
    
    private Serde<OrderHourlyStats> orderHourlyStatsSerde() {
        return Serdes.serdeFrom(
            new JsonSerializer<>(),
            new JsonDeserializer<>(OrderHourlyStats.class)
        );
    }
}
```

## 🔍 Elasticsearch 통합

### Elasticsearch 설정

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: ${ELASTICSEARCH_USERNAME:}
    password: ${ELASTICSEARCH_PASSWORD:}
    connection-timeout: 1s
    socket-timeout: 30s
    
  data:
    elasticsearch:
      repositories:
        enabled: true

elasticsearch:
  index:
    products: products
    orders: orders
    customers: customers
  settings:
    number-of-shards: 3
    number-of-replicas: 1
    refresh-interval: 1s
```

### Elasticsearch 설정 클래스

```java
@Configuration
@EnableElasticsearchRepositories(basePackages = "com.example.search.repository")
@RequiredArgsConstructor
public class ElasticsearchConfig {
    
    @Value("${spring.elasticsearch.uris}")
    private String elasticsearchUrl;
    
    @Bean
    public ElasticsearchClient elasticsearchClient() {
        RestClient restClient = RestClient.builder(
            HttpHost.create(elasticsearchUrl))
            .setRequestConfigCallback(requestConfigBuilder ->
                requestConfigBuilder
                    .setConnectTimeout(5000)
                    .setSocketTimeout(60000))
            .setHttpClientConfigCallback(httpClientBuilder ->
                httpClientBuilder
                    .setMaxConnTotal(100)
                    .setMaxConnPerRoute(100))
            .build();
        
        ElasticsearchTransport transport = new RestClientTransport(
            restClient, new JacksonJsonpMapper());
        
        return new ElasticsearchClient(transport);
    }
    
    @Bean
    public ElasticsearchOperations elasticsearchOperations() {
        return new ElasticsearchRestTemplate(elasticsearchClient());
    }
}
```

### 검색용 도큐먼트 정의

```java
// 상품 검색 도큐먼트
@Document(indexName = "products")
@Mapping(mappingPath = "/elasticsearch/mappings/product-mapping.json")
@Setting(settingPath = "/elasticsearch/settings/product-settings.json")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ProductDocument {
    
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "korean_analyzer")
    private String name;
    
    @Field(type = FieldType.Text, analyzer = "korean_analyzer")
    private String description;
    
    @Field(type = FieldType.Keyword)
    private String category;
    
    @Field(type = FieldType.Keyword)
    private String brand;
    
    @Field(type = FieldType.Double)
    private BigDecimal price;
    
    @Field(type = FieldType.Integer)
    private Integer stock;
    
    @Field(type = FieldType.Boolean)
    private Boolean available;
    
    @Field(type = FieldType.Keyword)
    private List<String> tags;
    
    @Field(type = FieldType.Object)
    private ProductAttributes attributes;
    
    @Field(type = FieldType.Double)
    private Double rating;
    
    @Field(type = FieldType.Integer)
    private Integer reviewCount;
    
    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second_millis)
    private LocalDateTime createdAt;
    
    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second_millis)
    private LocalDateTime updatedAt;
    
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ProductAttributes {
        private String color;
        private String size;
        private String material;
        private Double weight;
        private Map<String, Object> customAttributes;
    }
    
    public static ProductDocument from(Product product) {
        return ProductDocument.builder()
            .id(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .category(product.getCategory())
            .brand(product.getBrand())
            .price(product.getPrice())
            .stock(product.getStock())
            .available(product.isAvailable())
            .tags(product.getTags())
            .attributes(convertAttributes(product.getAttributes()))
            .rating(product.getAverageRating())
            .reviewCount(product.getReviewCount())
            .createdAt(product.getCreatedAt())
            .updatedAt(product.getUpdatedAt())
            .build();
    }
}

// 주문 검색 도큐먼트
@Document(indexName = "orders")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderDocument {
    
    @Id
    private String id;
    
    @Field(type = FieldType.Keyword)
    private String customerId;
    
    @Field(type = FieldType.Text)
    private String customerName;
    
    @Field(type = FieldType.Keyword)
    private String customerEmail;
    
    @Field(type = FieldType.Keyword)
    private OrderStatus status;
    
    @Field(type = FieldType.Double)
    private BigDecimal totalAmount;
    
    @Field(type = FieldType.Text)
    private String shippingAddress;
    
    @Field(type = FieldType.Nested)
    private List<OrderItemDocument> items;
    
    @Field(type = FieldType.Date)
    private LocalDateTime createdAt;
    
    @Field(type = FieldType.Date)
    private LocalDateTime completedAt;
    
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    public static class OrderItemDocument {
        
        @Field(type = FieldType.Keyword)
        private String productId;
        
        @Field(type = FieldType.Text)
        private String productName;
        
        @Field(type = FieldType.Keyword)
        private String category;
        
        @Field(type = FieldType.Integer)
        private Integer quantity;
        
        @Field(type = FieldType.Double)
        private BigDecimal unitPrice;
        
        @Field(type = FieldType.Double)
        private BigDecimal totalPrice;
    }
}
```

### Elasticsearch Repository

```java
// 상품 검색 리포지토리
@Repository
public interface ProductSearchRepository extends ElasticsearchRepository<ProductDocument, String> {
    
    // 기본 검색 메서드들은 자동 생성
    
    List<ProductDocument> findByCategory(String category);
    
    List<ProductDocument> findByBrand(String brand);
    
    List<ProductDocument> findByPriceBetween(BigDecimal minPrice, BigDecimal maxPrice);
    
    List<ProductDocument> findByAvailableTrue();
    
    @Query("{\"bool\": {\"must\": [{\"match\": {\"name\": \"?0\"}}], \"filter\": [{\"term\": {\"available\": true}}]}}")
    List<ProductDocument> findAvailableProductsByName(String name);
    
    @Query("{\"bool\": {\"must\": [{\"multi_match\": {\"query\": \"?0\", \"fields\": [\"name^2\", \"description\", \"tags\"]}}], \"filter\": [{\"term\": {\"available\": true}}]}}")
    List<ProductDocument> searchProducts(String query);
}

// 커스텀 검색 구현
@Repository
@RequiredArgsConstructor
@Slf4j
public class CustomProductSearchRepository {
    
    private final ElasticsearchOperations elasticsearchOperations;
    private final ElasticsearchClient elasticsearchClient;
    
    public SearchResponse<ProductDocument> searchProducts(ProductSearchCriteria criteria) {
        try {
            SearchRequest.Builder searchBuilder = new SearchRequest.Builder()
                .index("products")
                .query(buildQuery(criteria))
                .sort(buildSort(criteria))
                .from(criteria.getFrom())
                .size(criteria.getSize());
            
            // 집계 추가
            if (criteria.isIncludeAggregations()) {
                searchBuilder.aggregations(buildAggregations());
            }
            
            SearchRequest searchRequest = searchBuilder.build();
            
            return elasticsearchClient.search(searchRequest, ProductDocument.class);
            
        } catch (Exception e) {
            log.error("상품 검색 실패", e);
            throw new SearchException("상품 검색 중 오류 발생", e);
        }
    }
    
    private Query buildQuery(ProductSearchCriteria criteria) {
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();
        
        // 텍스트 검색
        if (StringUtils.hasText(criteria.getKeyword())) {
            boolQuery.must(MultiMatchQuery.of(m -> m
                .query(criteria.getKeyword())
                .fields("name^2", "description", "tags")
                .type(TextQueryType.BestFields)
                .fuzziness("AUTO")
            )._toQuery());
        }
        
        // 카테고리 필터
        if (criteria.getCategories() != null && !criteria.getCategories().isEmpty()) {
            boolQuery.filter(TermsQuery.of(t -> t
                .field("category")
                .terms(TermsQueryField.of(tf -> tf.value(
                    criteria.getCategories().stream()
                        .map(FieldValue::of)
                        .collect(Collectors.toList())
                )))
            )._toQuery());
        }
        
        // 가격 범위 필터
        if (criteria.getMinPrice() != null || criteria.getMaxPrice() != null) {
            RangeQuery.Builder rangeBuilder = new RangeQuery.Builder().field("price");
            
            if (criteria.getMinPrice() != null) {
                rangeBuilder.gte(JsonData.of(criteria.getMinPrice()));
            }
            if (criteria.getMaxPrice() != null) {
                rangeBuilder.lte(JsonData.of(criteria.getMaxPrice()));
            }
            
            boolQuery.filter(rangeBuilder.build()._toQuery());
        }
        
        // 재고 있는 상품만
        if (criteria.isAvailableOnly()) {
            boolQuery.filter(TermQuery.of(t -> t
                .field("available")
                .value(true)
            )._toQuery());
        }
        
        return boolQuery.build()._toQuery();
    }
    
    private List<SortOptions> buildSort(ProductSearchCriteria criteria) {
        List<SortOptions> sortOptions = new ArrayList<>();
        
        if (criteria.getSortBy() != null) {
            switch (criteria.getSortBy()) {
                case PRICE_LOW_TO_HIGH:
                    sortOptions.add(SortOptions.of(s -> s
                        .field(f -> f.field("price").order(SortOrder.Asc))));
                    break;
                case PRICE_HIGH_TO_LOW:
                    sortOptions.add(SortOptions.of(s -> s
                        .field(f -> f.field("price").order(SortOrder.Desc))));
                    break;
                case RATING:
                    sortOptions.add(SortOptions.of(s -> s
                        .field(f -> f.field("rating").order(SortOrder.Desc))));
                    break;
                case CREATED_DATE:
                    sortOptions.add(SortOptions.of(s -> s
                        .field(f -> f.field("createdAt").order(SortOrder.Desc))));
                    break;
                default:
                    // 관련성 점수 기준 정렬 (기본값)
                    sortOptions.add(SortOptions.of(s -> s
                        .score(sc -> sc.order(SortOrder.Desc))));
            }
        }
        
        return sortOptions;
    }
    
    private Map<String, Aggregation> buildAggregations() {
        Map<String, Aggregation> aggregations = new HashMap<>();
        
        // 카테고리별 집계
        aggregations.put("categories", Aggregation.of(a -> a
            .terms(t -> t.field("category").size(20))));
        
        // 브랜드별 집계
        aggregations.put("brands", Aggregation.of(a -> a
            .terms(t -> t.field("brand").size(20))));
        
        // 가격 범위별 집계
        aggregations.put("price_ranges", Aggregation.of(a -> a
            .range(r -> r
                .field("price")
                .ranges(
                    RangeAggregationRange.of(ra -> ra.to(JsonData.of(50)).key("Under $50")),
                    RangeAggregationRange.of(ra -> ra.from(JsonData.of(50)).to(JsonData.of(100)).key("$50-$100")),
                    RangeAggregationRange.of(ra -> ra.from(JsonData.of(100)).to(JsonData.of(200)).key("$100-$200")),
                    RangeAggregationRange.of(ra -> ra.from(JsonData.of(200)).key("Over $200"))
                ))));
        
        return aggregations;
    }
}
```

### 검색 서비스 구현

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductSearchService {
    
    private final ProductSearchRepository productSearchRepository;
    private final CustomProductSearchRepository customProductSearchRepository;
    private final ProductService productService;
    
    public ProductSearchResult searchProducts(ProductSearchCriteria criteria) {
        try {
            SearchResponse<ProductDocument> response = customProductSearchRepository
                .searchProducts(criteria);
            
            List<ProductDocument> products = response.hits().hits().stream()
                .map(hit -> hit.source())
                .collect(Collectors.toList());
            
            Map<String, Object> aggregations = extractAggregations(response);
            
            return ProductSearchResult.builder()
                .products(products)
                .totalHits(response.hits().total().value())
                .aggregations(aggregations)
                .took(response.took())
                .build();
                
        } catch (Exception e) {
            log.error("상품 검색 실패: criteria={}", criteria, e);
            throw new SearchException("상품 검색 중 오류 발생", e);
        }
    }
    
    public List<String> getSearchSuggestions(String prefix) {
        try {
            // 자동완성 쿼리
            SearchRequest searchRequest = SearchRequest.of(s -> s
                .index("products")
                .suggest(suggest -> suggest
                    .suggesters("product_suggest", FieldSuggester.of(fs -> fs
                        .prefix(prefix)
                        .completion(c -> c
                            .field("suggest")
                            .size(10)
                        ))))
                .size(0)
            );
            
            SearchResponse<ProductDocument> response = 
                elasticsearchClient.search(searchRequest, ProductDocument.class);
            
            return response.suggest().get("product_suggest").stream()
                .flatMap(suggest -> suggest.completion().options().stream())
                .map(option -> option.text())
                .collect(Collectors.toList());
                
        } catch (Exception e) {
            log.error("검색 제안 실패: prefix={}", prefix, e);
            return Collections.emptyList();
        }
    }
    
    public void indexProduct(Product product) {
        try {
            ProductDocument document = ProductDocument.from(product);
            productSearchRepository.save(document);
            
            log.info("상품 인덱싱 완료: {}", product.getId());
            
        } catch (Exception e) {
            log.error("상품 인덱싱 실패: {}", product.getId(), e);
            throw new IndexingException("상품 인덱싱 중 오류 발생", e);
        }
    }
    
    public void bulkIndexProducts(List<Product> products) {
        try {
            List<ProductDocument> documents = products.stream()
                .map(ProductDocument::from)
                .collect(Collectors.toList());
            
            productSearchRepository.saveAll(documents);
            
            log.info("벌크 상품 인덱싱 완료: {} 건", products.size());
            
        } catch (Exception e) {
            log.error("벌크 상품 인덱싱 실패", e);
            throw new IndexingException("벌크 인덱싱 중 오류 발생", e);
        }
    }
    
    public void deleteProductIndex(String productId) {
        try {
            productSearchRepository.deleteById(productId);
            log.info("상품 인덱스 삭제 완료: {}", productId);
            
        } catch (Exception e) {
            log.error("상품 인덱스 삭제 실패: {}", productId, e);
        }
    }
    
    private Map<String, Object> extractAggregations(SearchResponse<ProductDocument> response) {
        Map<String, Object> aggregations = new HashMap<>();
        
        if (response.aggregations() != null) {
            response.aggregations().forEach((key, aggregation) -> {
                if (aggregation.isSterms()) {
                    List<Map<String, Object>> buckets = aggregation.sterms().buckets().array()
                        .stream()
                        .map(bucket -> Map.of(
                            "key", bucket.key(),
                            "doc_count", bucket.docCount()
                        ))
                        .collect(Collectors.toList());
                    aggregations.put(key, buckets);
                } else if (aggregation.isRange()) {
                    List<Map<String, Object>> buckets = aggregation.range().buckets().array()
                        .stream()
                        .map(bucket -> Map.of(
                            "key", bucket.key(),
                            "doc_count", bucket.docCount(),
                            "from", bucket.from(),
                            "to", bucket.to()
                        ))
                        .collect(Collectors.toList());
                    aggregations.put(key, buckets);
                }
            });
        }
        
        return aggregations;
    }
}
```

## 🏆 통합 실습 프로젝트

### 실시간 이커머스 추천 시스템

```java
// 통합 추천 서비스
@Service
@RequiredArgsConstructor
@Slf4j
public class RecommendationService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ProductSearchService searchService;
    private final CustomerBehaviorAnalyzer behaviorAnalyzer;
    
    // 실시간 상품 추천
    public List<ProductRecommendation> getRecommendations(String customerId) {
        // 1. Redis에서 캐시된 추천 확인
        String cacheKey = "recommendations:" + customerId;
        List<ProductRecommendation> cached = getCachedRecommendations(cacheKey);
        
        if (cached != null && !cached.isEmpty()) {
            log.info("캐시된 추천 반환: customer={}, count={}", customerId, cached.size());
            return cached;
        }
        
        // 2. 고객 행동 데이터 분석
        CustomerProfile profile = behaviorAnalyzer.getCustomerProfile(customerId);
        
        // 3. Elasticsearch에서 유사 상품 검색
        List<ProductDocument> similarProducts = searchSimilarProducts(profile);
        
        // 4. 추천 점수 계산
        List<ProductRecommendation> recommendations = calculateRecommendationScores(
            similarProducts, profile);
        
        // 5. Redis에 캐시 저장
        cacheRecommendations(cacheKey, recommendations);
        
        // 6. 추천 이벤트 발행 (분석용)
        publishRecommendationEvent(customerId, recommendations);
        
        return recommendations;
    }
    
    // 실시간 검색 기반 추천
    public List<ProductDocument> getSearchBasedRecommendations(String searchQuery, String customerId) {
        // 1. 검색 쿼리 분석
        SearchQueryAnalysis analysis = analyzeSearchQuery(searchQuery);
        
        // 2. 고객 프로필과 결합
        CustomerProfile profile = behaviorAnalyzer.getCustomerProfile(customerId);
        
        // 3. 개인화된 검색 기준 생성
        ProductSearchCriteria criteria = ProductSearchCriteria.builder()
            .keyword(searchQuery)
            .categories(profile.getPreferredCategories())
            .priceRange(profile.getPriceRange())
            .brands(profile.getPreferredBrands())
            .sortBy(profile.getPreferredSortBy())
            .size(20)
            .build();
        
        // 4. Elasticsearch 검색 실행
        ProductSearchResult result = searchService.searchProducts(criteria);
        
        // 5. 검색 행동 Kafka로 전송
        publishSearchBehaviorEvent(customerId, searchQuery, result.getTotalHits());
        
        return result.getProducts();
    }
    
    // 실시간 장바구니 기반 추천
    public List<ProductDocument> getCartBasedRecommendations(String sessionId) {
        // 1. Redis에서 장바구니 조회
        Cart cart = getCartFromRedis(sessionId);
        
        if (cart.getItems().isEmpty()) {
            return Collections.emptyList();
        }
        
        // 2. 장바구니 상품들의 카테고리 분석
        Set<String> categories = cart.getItems().stream()
            .map(item -> getProductCategory(item.getProductId()))
            .collect(Collectors.toSet());
        
        // 3. "함께 구매한 상품" 검색
        List<ProductDocument> recommendations = new ArrayList<>();
        
        for (String category : categories) {
            ProductSearchCriteria criteria = ProductSearchCriteria.builder()
                .categories(List.of(category))
                .availableOnly(true)
                .sortBy(ProductSortType.RATING)
                .size(5)
                .build();
            
            ProductSearchResult result = searchService.searchProducts(criteria);
            recommendations.addAll(result.getProducts());
        }
        
        // 4. 장바구니에 이미 있는 상품 제외
        Set<String> cartProductIds = cart.getItems().stream()
            .map(CartItem::getProductId)
            .collect(Collectors.toSet());
        
        return recommendations.stream()
            .filter(product -> !cartProductIds.contains(product.getId()))
            .distinct()
            .limit(10)
            .collect(Collectors.toList());
    }
    
    private List<ProductRecommendation> getCachedRecommendations(String cacheKey) {
        try {
            List<Object> cached = redisTemplate.opsForList().range(cacheKey, 0, -1);
            return cached.stream()
                .map(obj -> (ProductRecommendation) obj)
                .collect(Collectors.toList());
        } catch (Exception e) {
            log.warn("추천 캐시 조회 실패: {}", cacheKey, e);
            return null;
        }
    }
    
    private void cacheRecommendations(String cacheKey, List<ProductRecommendation> recommendations) {
        try {
            redisTemplate.delete(cacheKey);
            recommendations.forEach(rec -> 
                redisTemplate.opsForList().rightPush(cacheKey, rec));
            redisTemplate.expire(cacheKey, Duration.ofHours(2));
            
        } catch (Exception e) {
            log.warn("추천 캐시 저장 실패: {}", cacheKey, e);
        }
    }
    
    private void publishRecommendationEvent(String customerId, List<ProductRecommendation> recommendations) {
        try {
            RecommendationGeneratedEvent event = RecommendationGeneratedEvent.builder()
                .customerId(customerId)
                .recommendationCount(recommendations.size())
                .productIds(recommendations.stream()
                    .map(ProductRecommendation::getProductId)
                    .collect(Collectors.toList()))
                .timestamp(LocalDateTime.now())
                .build();
            
            kafkaTemplate.send("recommendation-events", customerId, event);
            
        } catch (Exception e) {
            log.warn("추천 이벤트 발행 실패: customer={}", customerId, e);
        }
    }
    
    private void publishSearchBehaviorEvent(String customerId, String query, long resultCount) {
        try {
            SearchBehaviorEvent event = SearchBehaviorEvent.builder()
                .customerId(customerId)
                .searchQuery(query)
                .resultCount(resultCount)
                .timestamp(LocalDateTime.now())
                .build();
            
            kafkaTemplate.send("search-behavior-events", customerId, event);
            
        } catch (Exception e) {
            log.warn("검색 행동 이벤트 발행 실패: customer={}", customerId, e);
        }
    }
}
```

## 📚 관련 노트

- [[Microservices Architecture Patterns]] - 마이크로서비스 아키텍처
- [[Event Driven Architecture]] - 이벤트 드리븐 아키텍처
- [[API Design Patterns]] - API 설계 패턴
- [[Observability]] - 모니터링과 관찰성
- [[Complete Project Example]] - 실제 프로젝트 구현

## 🔗 외부 리소스

- [Redis Documentation](https://redis.io/documentation)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Elasticsearch Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Spring Data Redis Reference](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/docs/current/reference/html/)

---

*이 기술 스택들을 효과적으로 활용하면 확장 가능하고 고성능인 분산 시스템을 구축할 수 있습니다.*
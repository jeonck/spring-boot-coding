# Microservices Architecture Patterns

> Spring Boot 기반 마이크로서비스 아키텍처 패턴과 구현 전략

## 📋 목차

- [마이크로서비스 아키텍처 개요](#마이크로서비스-아키텍처-개요)
- [서비스 분해 패턴](#서비스-분해-패턴)
- [데이터 관리 패턴](#데이터-관리-패턴)
- [통신 패턴](#통신-패턴)
- [서비스 디스커버리](#서비스-디스커버리)
- [API 게이트웨이 패턴](#api-게이트웨이-패턴)
- [회로 차단기 패턴](#회로-차단기-패턴)
- [분산 추적과 로깅](#분산-추적과-로깅)
- [배포 패턴](#배포-패턴)

## 🏗 마이크로서비스 아키텍처 개요

### 마이크로서비스 특징

```yaml
# 마이크로서비스 원칙
원칙:
  - 단일 책임: 각 서비스는 하나의 비즈니스 기능에 집중
  - 독립적 배포: 다른 서비스에 영향 없이 독립적으로 배포
  - 기술 다양성: 서비스별로 최적의 기술 스택 선택 가능
  - 데이터 격리: 각 서비스는 자체 데이터베이스 보유
  - 장애 격리: 한 서비스의 장애가 전체 시스템에 전파되지 않음
  - 팀 자율성: 작은 팀이 서비스를 완전히 소유
```

### Spring Boot 마이크로서비스 스택

```java
// 기본 의존성 구성
dependencies {
    // Spring Boot 기본
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    // Spring Cloud
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    
    // 관찰성
    implementation 'org.springframework.cloud:spring-cloud-starter-sleuth'
    implementation 'org.springframework.cloud:spring-cloud-sleuth-zipkin'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    
    // 메시징
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
    
    // 보안
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
}
```

## 🔄 서비스 분해 패턴

### 도메인 주도 설계 기반 분해

```java
// 주문 서비스 - 경계 컨텍스트
@RestController
@RequestMapping("/api/orders")
public class OrderService {
    
    // 주문 도메인만의 책임
    @PostMapping
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        // 주문 생성 로직
        return ResponseEntity.ok(orderService.createOrder(request));
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable String orderId) {
        return ResponseEntity.ok(orderService.getOrder(orderId));
    }
    
    @PutMapping("/{orderId}/status")
    public ResponseEntity<Void> updateOrderStatus(
            @PathVariable String orderId, 
            @RequestBody UpdateOrderStatusRequest request) {
        orderService.updateOrderStatus(orderId, request.getStatus());
        return ResponseEntity.ok().build();
    }
}

// 상품 서비스 - 별도의 경계 컨텍스트
@RestController
@RequestMapping("/api/products")
public class ProductService {
    
    // 상품 도메인만의 책임
    @GetMapping
    public ResponseEntity<Page<ProductDto>> getProducts(Pageable pageable) {
        return ResponseEntity.ok(productService.getProducts(pageable));
    }
    
    @GetMapping("/{productId}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable String productId) {
        return ResponseEntity.ok(productService.getProduct(productId));
    }
    
    @PutMapping("/{productId}/inventory")
    public ResponseEntity<Void> updateInventory(
            @PathVariable String productId,
            @RequestBody UpdateInventoryRequest request) {
        productService.updateInventory(productId, request.getQuantity());
        return ResponseEntity.ok().build();
    }
}
```

### 서비스별 설정 분리

```yaml
# order-service/application.yml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://localhost:5432/orders_db
    username: ${DB_USERNAME:orders_user}
    password: ${DB_PASSWORD:orders_pass}

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka

---
# product-service/application.yml
spring:
  application:
    name: product-service
  datasource:
    url: jdbc:postgresql://localhost:5432/products_db
    username: ${DB_USERNAME:products_user}
    password: ${DB_PASSWORD:products_pass}

server:
  port: 8082

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

## 🗄 데이터 관리 패턴

### Database per Service 패턴

```java
// 주문 서비스 - 전용 데이터베이스
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private String orderId;
    private String customerId;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    private String itemId;
    private String productId; // 상품 서비스의 ID 참조 (외래키 아님)
    private String productName; // 데이터 복제
    private BigDecimal price;
    private Integer quantity;
    
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
}
```

### Saga 패턴 구현

```java
// 주문 생성 사가 (Orchestration 방식)
@Service
@Slf4j
public class OrderCreationSaga {
    
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final ShippingService shippingService;
    private final OrderRepository orderRepository;
    
    @Transactional
    public OrderDto createOrder(CreateOrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        
        try {
            // 1. 주문 생성 (Pending 상태)
            Order order = createPendingOrder(request, sagaId);
            
            // 2. 재고 예약
            InventoryReservationResult inventoryResult = 
                inventoryService.reserveInventory(order.getItems(), sagaId);
            
            if (!inventoryResult.isSuccess()) {
                throw new SagaException("재고 예약 실패", sagaId);
            }
            
            // 3. 결제 처리
            PaymentResult paymentResult = 
                paymentService.processPayment(order.getTotalAmount(), request.getPaymentInfo(), sagaId);
            
            if (!paymentResult.isSuccess()) {
                // 재고 예약 취소
                inventoryService.cancelReservation(inventoryResult.getReservationId(), sagaId);
                throw new SagaException("결제 처리 실패", sagaId);
            }
            
            // 4. 배송 예약
            ShippingResult shippingResult = 
                shippingService.scheduleShipping(order, request.getShippingAddress(), sagaId);
            
            if (!shippingResult.isSuccess()) {
                // 보상 트랜잭션들 실행
                compensatePayment(paymentResult.getPaymentId(), sagaId);
                compensateInventory(inventoryResult.getReservationId(), sagaId);
                throw new SagaException("배송 예약 실패", sagaId);
            }
            
            // 5. 주문 확정
            order.confirm();
            orderRepository.save(order);
            
            log.info("Order creation saga completed successfully: {}", sagaId);
            return OrderDto.from(order);
            
        } catch (SagaException e) {
            log.error("Order creation saga failed: {}", e.getMessage(), e);
            // 주문 취소
            cancelOrder(e.getSagaId());
            throw e;
        }
    }
    
    private void compensatePayment(String paymentId, String sagaId) {
        try {
            paymentService.refundPayment(paymentId, sagaId);
        } catch (Exception e) {
            log.error("Failed to compensate payment: {}", paymentId, e);
            // 수동 개입 필요 알림
        }
    }
    
    private void compensateInventory(String reservationId, String sagaId) {
        try {
            inventoryService.cancelReservation(reservationId, sagaId);
        } catch (Exception e) {
            log.error("Failed to compensate inventory: {}", reservationId, e);
        }
    }
}
```

### Event Sourcing 패턴

```java
// 이벤트 스토어
@Entity
@Table(name = "event_store")
public class EventStore {
    @Id
    private String eventId;
    private String aggregateId;
    private String aggregateType;
    private String eventType;
    private String eventData;
    private Integer version;
    private LocalDateTime timestamp;
    private String correlationId;
    
    // 생성자, getter, setter
}

// 주문 애그리게이트 (Event Sourcing)
@Component
public class OrderAggregate {
    
    private String orderId;
    private OrderStatus status;
    private List<OrderItem> items;
    private Integer version;
    
    public List<DomainEvent> createOrder(CreateOrderCommand command) {
        List<DomainEvent> events = new ArrayList<>();
        
        // 비즈니스 규칙 검증
        validateCreateOrder(command);
        
        // 이벤트 생성
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(command.getOrderId())
            .customerId(command.getCustomerId())
            .items(command.getItems())
            .timestamp(LocalDateTime.now())
            .build();
        
        events.add(event);
        
        // 상태 변경 (이벤트 적용)
        apply(event);
        
        return events;
    }
    
    public void apply(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.status = OrderStatus.PENDING;
        this.items = event.getItems();
        this.version++;
    }
    
    public void apply(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
        this.version++;
    }
    
    // 이벤트 스트림에서 애그리게이트 재구성
    public static OrderAggregate fromEvents(List<DomainEvent> events) {
        OrderAggregate aggregate = new OrderAggregate();
        
        events.forEach(event -> {
            if (event instanceof OrderCreatedEvent) {
                aggregate.apply((OrderCreatedEvent) event);
            } else if (event instanceof OrderConfirmedEvent) {
                aggregate.apply((OrderConfirmedEvent) event);
            }
            // 다른 이벤트 타입들...
        });
        
        return aggregate;
    }
}
```

## 🔗 통신 패턴

### 동기 통신 - OpenFeign

```java
// 상품 서비스 클라이언트
@FeignClient(name = "product-service", fallback = ProductServiceFallback.class)
public interface ProductServiceClient {
    
    @GetMapping("/api/products/{productId}")
    ProductDto getProduct(@PathVariable("productId") String productId);
    
    @GetMapping("/api/products")
    Page<ProductDto> getProducts(@SpringQueryMap Pageable pageable);
    
    @PutMapping("/api/products/{productId}/inventory")
    void updateInventory(@PathVariable("productId") String productId, 
                        @RequestBody UpdateInventoryRequest request);
}

// Fallback 구현
@Component
public class ProductServiceFallback implements ProductServiceClient {
    
    @Override
    public ProductDto getProduct(String productId) {
        return ProductDto.builder()
            .productId(productId)
            .name("상품 정보를 가져올 수 없음")
            .available(false)
            .build();
    }
    
    @Override
    public Page<ProductDto> getProducts(Pageable pageable) {
        return Page.empty();
    }
    
    @Override
    public void updateInventory(String productId, UpdateInventoryRequest request) {
        throw new ServiceUnavailableException("상품 서비스를 사용할 수 없습니다.");
    }
}

// Feign 설정
@Configuration
@EnableFeignClients
public class FeignConfig {
    
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    
    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            // JWT 토큰 전파
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
            if (authentication != null && authentication.getCredentials() instanceof String) {
                String token = (String) authentication.getCredentials();
                requestTemplate.header("Authorization", "Bearer " + token);
            }
            
            // 추적 ID 전파
            String traceId = MDC.get("traceId");
            if (traceId != null) {
                requestTemplate.header("X-Trace-ID", traceId);
            }
        };
    }
}
```

### 비동기 통신 - Spring Cloud Stream

```java
// 이벤트 발행자
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    
    private final StreamBridge streamBridge;
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        streamBridge.send("orderCreated-out-0", event);
    }
    
    public void publishOrderConfirmed(OrderConfirmedEvent event) {
        streamBridge.send("orderConfirmed-out-0", event);
    }
}

// 이벤트 구독자
@Component
@Slf4j
public class OrderEventHandler {
    
    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Handling order created event: {}", event.getOrderId());
        
        // 재고 예약
        inventoryService.reserveInventory(event.getItems());
        
        // 고객 알림
        notificationService.sendOrderCreatedNotification(event.getCustomerId(), event.getOrderId());
    }
    
    @EventListener
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        log.info("Handling order confirmed event: {}", event.getOrderId());
        
        // 배송 시작
        shippingService.startShipping(event.getOrderId());
        
        // 재고 확정
        inventoryService.confirmReservation(event.getOrderId());
    }
}

// Stream 설정
spring:
  cloud:
    stream:
      bindings:
        orderCreated-out-0:
          destination: order.created
          content-type: application/json
        orderConfirmed-out-0:
          destination: order.confirmed
          content-type: application/json
        orderCreated-in-0:
          destination: order.created
          group: inventory-service
        orderConfirmed-in-0:
          destination: order.confirmed
          group: shipping-service
      rabbit:
        bindings:
          orderCreated-out-0:
            producer:
              exchange-type: topic
              routing-key-expression: headers.eventType
```

## 🔍 서비스 디스커버리

### Eureka 서버 구성

```java
@SpringBootApplication
@EnableEurekaServer
public class ServiceDiscoveryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceDiscoveryApplication.class, args);
    }
}
```

```yaml
# eureka-server/application.yml
spring:
  application:
    name: eureka-server

server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 10000
```

### 서비스 등록

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```yaml
# order-service/application.yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
    healthcheck:
      enabled: true
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

## 🚪 API 게이트웨이 패턴

### Spring Cloud Gateway

```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // 주문 서비스 라우팅
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway", "spring-cloud-gateway")
                    .circuitBreaker(config -> config
                        .setName("order-service-cb")
                        .setFallbackUri("forward:/fallback/orders")))
                .uri("lb://order-service"))
            
            // 상품 서비스 라우팅
            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .rateLimit(config -> config
                        .setRateLimiter(RedisRateLimiter.class)
                        .setKeyResolver(new PrincipalNameKeyResolver())))
                .uri("lb://product-service"))
            
            // 정적 리소스 라우팅
            .route("frontend", r -> r
                .path("/**")
                .and()
                .not(p -> p.path("/api/**"))
                .uri("lb://frontend-service"))
            
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20, 1); // replenishRate, burstCapacity, requestedTokens
    }
}

// 인증 필터
@Component
public class AuthenticationFilter implements GlobalFilter, Ordered {
    
    private final JwtTokenProvider jwtTokenProvider;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // Public 경로는 인증 건너뛰기
        if (isPublicPath(request.getPath().value())) {
            return chain.filter(exchange);
        }
        
        String token = extractToken(request);
        if (token == null || !jwtTokenProvider.validateToken(token)) {
            return unauthorized(exchange);
        }
        
        // 인증 정보를 헤더에 추가
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-User-ID", jwtTokenProvider.getUserId(token))
            .header("X-User-ROLES", String.join(",", jwtTokenProvider.getRoles(token)))
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    private boolean isPublicPath(String path) {
        return path.startsWith("/api/auth/") || 
               path.equals("/api/health") ||
               path.startsWith("/actuator/");
    }
    
    private String extractToken(ServerHttpRequest request) {
        String authorization = request.getHeaders().getFirst("Authorization");
        if (authorization != null && authorization.startsWith("Bearer ")) {
            return authorization.substring(7);
        }
        return null;
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        return response.setComplete();
    }
    
    @Override
    public int getOrder() {
        return -100; // 다른 필터들보다 먼저 실행
    }
}
```

## ⚡ 회로 차단기 패턴

### Resilience4j 구현

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final ProductServiceClient productServiceClient;
    private final PaymentServiceClient paymentServiceClient;
    
    @CircuitBreaker(name = "product-service", fallbackMethod = "createOrderFallback")
    @Retry(name = "product-service")
    @TimeLimiter(name = "product-service")
    @Bulkhead(name = "product-service")
    public CompletableFuture<OrderDto> createOrder(CreateOrderRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // 상품 정보 조회
            List<ProductDto> products = request.getItems().stream()
                .map(item -> productServiceClient.getProduct(item.getProductId()))
                .collect(Collectors.toList());
            
            // 재고 확인
            validateInventory(products, request.getItems());
            
            // 주문 생성
            return createOrderInternal(request, products);
        });
    }
    
    public CompletableFuture<OrderDto> createOrderFallback(CreateOrderRequest request, Exception ex) {
        log.error("Order creation failed, using fallback", ex);
        
        // 기본 주문 생성 (제한된 기능)
        return CompletableFuture.completedFuture(
            createBasicOrder(request)
        );
    }
}
```

```yaml
# resilience4j 설정
resilience4j:
  circuitbreaker:
    instances:
      product-service:
        register-health-indicator: true
        sliding-window-size: 10
        permitted-number-of-calls-in-half-open-state: 3
        sliding-window-type: TIME_BASED
        minimum-number-of-calls: 5
        wait-duration-in-open-state: 30s
        failure-rate-threshold: 50
        event-consumer-buffer-size: 10
  
  retry:
    instances:
      product-service:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
  
  timelimiter:
    instances:
      product-service:
        timeout-duration: 2s
        cancel-running-future: true
  
  bulkhead:
    instances:
      product-service:
        max-concurrent-calls: 20
        max-wait-duration: 1s
```

## 📊 분산 추적과 로깅

### Spring Cloud Sleuth 구성

```yaml
spring:
  sleuth:
    zipkin:
      base-url: http://zipkin-server:9411
    sampler:
      probability: 1.0
  application:
    name: order-service

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### 구조화된 로깅

```java
@Component
@Slf4j
public class StructuredLogger {
    
    private final ObjectMapper objectMapper;
    
    public void logOrderCreated(String orderId, String customerId, BigDecimal amount) {
        Map<String, Object> logData = Map.of(
            "event", "order_created",
            "orderId", orderId,
            "customerId", customerId,
            "amount", amount,
            "timestamp", Instant.now(),
            "service", "order-service"
        );
        
        try {
            log.info(objectMapper.writeValueAsString(logData));
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize log data", e);
        }
    }
    
    public void logServiceCall(String targetService, String method, String endpoint, long duration) {
        Map<String, Object> logData = Map.of(
            "event", "service_call",
            "targetService", targetService,
            "method", method,
            "endpoint", endpoint,
            "duration", duration,
            "timestamp", Instant.now(),
            "service", "order-service"
        );
        
        try {
            log.info(objectMapper.writeValueAsString(logData));
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize log data", e);
        }
    }
}
```

### 메트릭스 수집

```java
@Component
@RequiredArgsConstructor
public class OrderMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    
    public OrderMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Number of orders created")
            .register(meterRegistry);
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing time")
            .register(meterRegistry);
    }
    
    public void incrementOrderCreated(String customerId, OrderStatus status) {
        orderCreatedCounter.increment(
            Tags.of(
                "customer", customerId,
                "status", status.name()
            )
        );
    }
    
    public Timer.Sample startOrderProcessing() {
        return Timer.start(meterRegistry);
    }
    
    public void recordOrderProcessingTime(Timer.Sample sample, String result) {
        sample.stop(Timer.builder("orders.processing.time")
            .tag("result", result)
            .register(meterRegistry));
    }
}
```

## 🚀 배포 패턴

### 컨테이너화

```dockerfile
# Dockerfile (각 서비스별)
FROM openjdk:21-jdk-slim as builder

WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .
COPY src src

RUN ./gradlew build -x test

FROM openjdk:21-jdk-slim

RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-Xmx512m", "-Djava.security.egd=file:/dev/./urandom", "-jar", "app.jar"]
```

### Kubernetes 배포

```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: v1
  template:
    metadata:
      labels:
        app: order-service
        version: v1
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE
          value: "http://eureka-server:8761/eureka"
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://postgres:5432/orders"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: order-service
```

### Blue-Green 배포

```yaml
# Blue-Green 배포 스크립트
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: order-service-active
      previewService: order-service-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: order-service-preview
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: order-service-active
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:v2
        ports:
        - containerPort: 8080
```

## 📚 관련 노트

- [[Service Layer Pattern]] - 서비스 계층 설계
- [[API Design Patterns]] - API 설계 원칙
- [[Event Driven Architecture]] - 이벤트 기반 아키텍처
- [[CQRS and Event Sourcing]] - CQRS와 이벤트 소싱
- [[Spring Security 6]] - 마이크로서비스 보안
- [[Observability]] - 관찰성과 모니터링

## 🔗 외부 리소스

- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Microservices Patterns by Chris Richardson](https://microservices.io/patterns/)
- [Building Microservices by Sam Newman](https://samnewman.io/books/building_microservices/)

---

*마이크로서비스는 복잡성을 수반하므로, 조직의 성숙도와 요구사항을 신중히 고려하여 도입하세요.*
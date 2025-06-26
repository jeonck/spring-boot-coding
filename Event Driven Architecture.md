# Event Driven Architecture

> Spring Boot 기반 이벤트 드리븐 아키텍처 설계와 구현 패턴

## 📋 목차

- [이벤트 드리븐 아키텍처 개요](#이벤트-드리븐-아키텍처-개요)
- [도메인 이벤트 설계](#도메인-이벤트-설계)
- [이벤트 발행 패턴](#이벤트-발행-패턴)
- [이벤트 구독 패턴](#이벤트-구독-패턴)
- [이벤트 스토어](#이벤트-스토어)
- [이벤트 스트리밍](#이벤트-스트리밍)
- [Outbox 패턴](#outbox-패턴)
- [이벤트 처리 전략](#이벤트-처리-전략)
- [모니터링과 관찰성](#모니터링과-관찰성)

## 🌊 이벤트 드리븐 아키텍처 개요

### 핵심 개념

```yaml
# 이벤트 드리븐 아키텍처의 핵심 요소
구성요소:
  이벤트: 시스템에서 발생한 의미있는 변화
  이벤트 프로듀서: 이벤트를 발행하는 서비스
  이벤트 컨슈머: 이벤트를 구독하고 처리하는 서비스
  이벤트 브로커: 이벤트를 라우팅하는 미들웨어
  이벤트 스토어: 이벤트를 영구 저장하는 저장소

장점:
  - 느슨한 결합: 서비스 간 직접적인 의존성 제거
  - 확장성: 비동기 처리로 높은 처리량 달성
  - 복원력: 장애 격리와 복구 능력 향상
  - 유연성: 새로운 기능 추가가 용이

단점:
  - 복잡성: 분산 시스템의 복잡성 증가
  - 결과적 일관성: 강한 일관성 보장 어려움
  - 디버깅 어려움: 분산된 이벤트 플로우 추적 복잡
```

### Spring Boot 이벤트 아키텍처 스택

```java
// 기본 의존성
dependencies {
    // Spring Boot 기본
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    
    // 이벤트 처리
    implementation 'org.springframework.boot:spring-boot-starter-amqp'
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
    implementation 'org.springframework.kafka:spring-kafka'
    
    // 이벤트 스토어
    implementation 'org.axonframework:axon-spring-boot-starter'
    
    // 메시징
    implementation 'org.springframework.cloud:spring-cloud-starter-bus-amqp'
    
    // 관찰성
    implementation 'org.springframework.cloud:spring-cloud-starter-sleuth'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

## 📝 도메인 이벤트 설계

### 기본 이벤트 인터페이스

```java
// 기본 도메인 이벤트 인터페이스
public interface DomainEvent {
    String getEventId();
    String getAggregateId();
    String getAggregateType();
    String getEventType();
    LocalDateTime getOccurredOn();
    Integer getVersion();
    String getCorrelationId();
    String getCausationId();
    Map<String, Object> getMetadata();
}

// 추상 도메인 이벤트 클래스
@Getter
@EqualsAndHashCode
@ToString
public abstract class AbstractDomainEvent implements DomainEvent {
    
    private final String eventId;
    private final String aggregateId;
    private final String aggregateType;
    private final String eventType;
    private final LocalDateTime occurredOn;
    private final Integer version;
    private final String correlationId;
    private final String causationId;
    private final Map<String, Object> metadata;
    
    protected AbstractDomainEvent(String aggregateId, String aggregateType, Integer version) {
        this.eventId = UUID.randomUUID().toString();
        this.aggregateId = aggregateId;
        this.aggregateType = aggregateType;
        this.eventType = this.getClass().getSimpleName();
        this.occurredOn = LocalDateTime.now();
        this.version = version;
        this.correlationId = getCurrentCorrelationId();
        this.causationId = getCurrentCausationId();
        this.metadata = createMetadata();
    }
    
    private String getCurrentCorrelationId() {
        return MDC.get("correlationId");
    }
    
    private String getCurrentCausationId() {
        return MDC.get("causationId");
    }
    
    private Map<String, Object> createMetadata() {
        Map<String, Object> metadata = new HashMap<>();
        metadata.put("userId", getCurrentUserId());
        metadata.put("source", "order-service");
        metadata.put("environment", getActiveProfile());
        return metadata;
    }
}
```

### 구체적인 도메인 이벤트들

```java
// 주문 생성 이벤트
@Getter
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class OrderCreatedEvent extends AbstractDomainEvent {
    
    private final String customerId;
    private final List<OrderItemData> items;
    private final BigDecimal totalAmount;
    private final String shippingAddress;
    private final PaymentMethod paymentMethod;
    
    public OrderCreatedEvent(String orderId, String customerId, 
                           List<OrderItemData> items, BigDecimal totalAmount,
                           String shippingAddress, PaymentMethod paymentMethod, 
                           Integer version) {
        super(orderId, "Order", version);
        this.customerId = customerId;
        this.items = List.copyOf(items);
        this.totalAmount = totalAmount;
        this.shippingAddress = shippingAddress;
        this.paymentMethod = paymentMethod;
    }
    
    @Getter
    @AllArgsConstructor
    @NoArgsConstructor
    public static class OrderItemData {
        private String productId;
        private String productName;
        private Integer quantity;
        private BigDecimal unitPrice;
        private BigDecimal totalPrice;
    }
}

// 주문 확정 이벤트
@Getter
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class OrderConfirmedEvent extends AbstractDomainEvent {
    
    private final String paymentId;
    private final LocalDateTime confirmedAt;
    private final BigDecimal finalAmount;
    
    public OrderConfirmedEvent(String orderId, String paymentId, 
                             BigDecimal finalAmount, Integer version) {
        super(orderId, "Order", version);
        this.paymentId = paymentId;
        this.confirmedAt = LocalDateTime.now();
        this.finalAmount = finalAmount;
    }
}

// 주문 취소 이벤트
@Getter
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class OrderCancelledEvent extends AbstractDomainEvent {
    
    private final String reason;
    private final CancellationType cancellationType;
    private final LocalDateTime cancelledAt;
    private final String cancelledBy;
    
    public OrderCancelledEvent(String orderId, String reason, 
                             CancellationType cancellationType, 
                             String cancelledBy, Integer version) {
        super(orderId, "Order", version);
        this.reason = reason;
        this.cancellationType = cancellationType;
        this.cancelledAt = LocalDateTime.now();
        this.cancelledBy = cancelledBy;
    }
    
    public enum CancellationType {
        CUSTOMER_REQUEST, PAYMENT_FAILED, INVENTORY_UNAVAILABLE, SYSTEM_ERROR
    }
}

// 상품 재고 변경 이벤트
@Getter
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class ProductInventoryChangedEvent extends AbstractDomainEvent {
    
    private final Integer previousQuantity;
    private final Integer newQuantity;
    private final Integer changeAmount;
    private final InventoryChangeReason reason;
    private final String referenceId;
    
    public ProductInventoryChangedEvent(String productId, Integer previousQuantity,
                                      Integer newQuantity, InventoryChangeReason reason,
                                      String referenceId, Integer version) {
        super(productId, "Product", version);
        this.previousQuantity = previousQuantity;
        this.newQuantity = newQuantity;
        this.changeAmount = newQuantity - previousQuantity;
        this.reason = reason;
        this.referenceId = referenceId;
    }
    
    public enum InventoryChangeReason {
        ORDER_PLACED, ORDER_CANCELLED, RESTOCK, DAMAGED, ADJUSTMENT
    }
}
```

## 📤 이벤트 발행 패턴

### Application Events (동일 프로세스 내)

```java
// 이벤트 발행자
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderDto createOrder(CreateOrderRequest request) {
        // 주문 생성
        Order order = Order.create(request);
        Order savedOrder = orderRepository.save(order);
        
        // 이벤트 발행
        OrderCreatedEvent event = new OrderCreatedEvent(
            savedOrder.getId(),
            savedOrder.getCustomerId(),
            savedOrder.getItems().stream()
                .map(this::toOrderItemData)
                .collect(Collectors.toList()),
            savedOrder.getTotalAmount(),
            savedOrder.getShippingAddress(),
            savedOrder.getPaymentMethod(),
            savedOrder.getVersion()
        );
        
        eventPublisher.publishEvent(event);
        
        return OrderDto.from(savedOrder);
    }
    
    private OrderCreatedEvent.OrderItemData toOrderItemData(OrderItem item) {
        return new OrderCreatedEvent.OrderItemData(
            item.getProductId(),
            item.getProductName(),
            item.getQuantity(),
            item.getUnitPrice(),
            item.getTotalPrice()
        );
    }
}

// 이벤트 리스너
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventHandler {
    
    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    private final EmailService emailService;
    
    @EventListener
    @Async("eventExecutor")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Processing order created event: {}", event.getAggregateId());
        
        try {
            // 재고 예약
            inventoryService.reserveInventory(event.getItems());
            
            // 고객 알림
            notificationService.sendOrderCreatedNotification(
                event.getCustomerId(), 
                event.getAggregateId()
            );
            
            // 이메일 발송
            emailService.sendOrderConfirmationEmail(event);
            
        } catch (Exception e) {
            log.error("Failed to process order created event: {}", event.getAggregateId(), e);
            // 재시도 또는 DLQ로 이동
            throw e;
        }
    }
    
    @EventListener
    @Async("eventExecutor")
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        log.info("Processing order confirmed event: {}", event.getAggregateId());
        
        // 배송 시작
        shippingService.initiateShipping(event.getAggregateId());
        
        // 고객 알림
        notificationService.sendOrderConfirmedNotification(
            event.getAggregateId(),
            event.getConfirmedAt()
        );
    }
}
```

### 외부 시스템 이벤트 발행

```java
// Spring Cloud Stream을 이용한 외부 이벤트 발행
@Service
@RequiredArgsConstructor
public class ExternalEventPublisher {
    
    private final StreamBridge streamBridge;
    private final ObjectMapper objectMapper;
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        try {
            EventEnvelope envelope = EventEnvelope.builder()
                .eventId(event.getEventId())
                .eventType(event.getEventType())
                .aggregateId(event.getAggregateId())
                .aggregateType(event.getAggregateType())
                .version(event.getVersion())
                .occurredOn(event.getOccurredOn())
                .correlationId(event.getCorrelationId())
                .causationId(event.getCausationId())
                .payload(objectMapper.writeValueAsString(event))
                .metadata(event.getMetadata())
                .build();
            
            streamBridge.send("order-events", envelope);
            
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize event: {}", event.getEventId(), e);
            throw new EventPublishingException("Failed to publish event", e);
        }
    }
    
    @Getter
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public static class EventEnvelope {
        private String eventId;
        private String eventType;
        private String aggregateId;
        private String aggregateType;
        private Integer version;
        private LocalDateTime occurredOn;
        private String correlationId;
        private String causationId;
        private String payload;
        private Map<String, Object> metadata;
    }
}

// 이벤트 발행을 위한 설정
@Configuration
public class StreamConfig {
    
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }
}
```

## 📥 이벤트 구독 패턴

### Spring Cloud Stream 구독자

```java
// 이벤트 구독자
@Component
@RequiredArgsConstructor
@Slf4j
public class ExternalEventSubscriber {
    
    private final ObjectMapper objectMapper;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    
    @StreamListener("order-events")
    public void handleOrderEvent(ExternalEventPublisher.EventEnvelope envelope) {
        try {
            log.info("Received event: {} for aggregate: {}", 
                envelope.getEventType(), envelope.getAggregateId());
            
            // 상관관계 ID 설정
            MDC.put("correlationId", envelope.getCorrelationId());
            MDC.put("causationId", envelope.getEventId());
            
            switch (envelope.getEventType()) {
                case "OrderCreatedEvent":
                    handleOrderCreated(envelope);
                    break;
                case "OrderConfirmedEvent":
                    handleOrderConfirmed(envelope);
                    break;
                case "OrderCancelledEvent":
                    handleOrderCancelled(envelope);
                    break;
                default:
                    log.warn("Unknown event type: {}", envelope.getEventType());
            }
            
        } catch (Exception e) {
            log.error("Failed to process event: {}", envelope.getEventId(), e);
            throw e; // 재시도 또는 DLQ 처리
        } finally {
            MDC.clear();
        }
    }
    
    private void handleOrderCreated(ExternalEventPublisher.EventEnvelope envelope) 
            throws JsonProcessingException {
        OrderCreatedEvent event = objectMapper.readValue(
            envelope.getPayload(), OrderCreatedEvent.class);
        
        // 결제 준비
        paymentService.preparePayment(
            event.getAggregateId(),
            event.getTotalAmount(),
            event.getPaymentMethod()
        );
    }
    
    private void handleOrderConfirmed(ExternalEventPublisher.EventEnvelope envelope) 
            throws JsonProcessingException {
        OrderConfirmedEvent event = objectMapper.readValue(
            envelope.getPayload(), OrderConfirmedEvent.class);
        
        // 재고 확정
        inventoryService.confirmInventoryReservation(event.getAggregateId());
    }
    
    private void handleOrderCancelled(ExternalEventPublisher.EventEnvelope envelope) 
            throws JsonProcessingException {
        OrderCancelledEvent event = objectMapper.readValue(
            envelope.getPayload(), OrderCancelledEvent.class);
        
        // 재고 해제
        inventoryService.releaseInventoryReservation(event.getAggregateId());
        
        // 결제 취소
        if (event.getCancellationType() != OrderCancelledEvent.CancellationType.PAYMENT_FAILED) {
            paymentService.cancelPayment(event.getAggregateId());
        }
    }
}
```

### 함수형 스타일 이벤트 처리

```java
// 함수형 이벤트 프로세서
@Configuration
public class EventProcessorConfiguration {
    
    @Bean
    public Function<EventEnvelope, EventEnvelope> orderEventProcessor() {
        return event -> {
            log.info("Processing event: {}", event.getEventType());
            
            // 이벤트 처리 로직
            processEvent(event);
            
            // 변환된 이벤트 반환 (필요시)
            return event;
        };
    }
    
    @Bean
    public Consumer<EventEnvelope> inventoryEventConsumer() {
        return event -> {
            log.info("Consuming inventory event: {}", event.getEventType());
            
            switch (event.getEventType()) {
                case "ProductInventoryChangedEvent":
                    handleInventoryChanged(event);
                    break;
                default:
                    log.warn("Unhandled inventory event: {}", event.getEventType());
            }
        };
    }
    
    @Bean
    public Supplier<EventEnvelope> heartbeatEventSupplier() {
        return () -> {
            // 주기적으로 헬스체크 이벤트 발행
            return EventEnvelope.builder()
                .eventId(UUID.randomUUID().toString())
                .eventType("HeartbeatEvent")
                .aggregateId("system")
                .aggregateType("System")
                .version(1)
                .occurredOn(LocalDateTime.now())
                .build();
        };
    }
}
```

## 🗄 이벤트 스토어

### 이벤트 스토어 구현

```java
// 이벤트 스토어 엔티티
@Entity
@Table(name = "event_store")
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class StoredEvent {
    
    @Id
    private String eventId;
    
    @Column(nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private String aggregateType;
    
    @Column(nullable = false)
    private String eventType;
    
    @Column(nullable = false)
    private Integer version;
    
    @Column(nullable = false)
    private LocalDateTime occurredOn;
    
    @Column(columnDefinition = "TEXT")
    private String eventData;
    
    @Column(columnDefinition = "TEXT")
    private String metadata;
    
    private String correlationId;
    private String causationId;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @PrePersist
    public void prePersist() {
        if (createdAt == null) {
            createdAt = LocalDateTime.now();
        }
    }
}

// 이벤트 스토어 리포지토리
@Repository
public interface EventStoreRepository extends JpaRepository<StoredEvent, String> {
    
    List<StoredEvent> findByAggregateIdOrderByVersionAsc(String aggregateId);
    
    List<StoredEvent> findByAggregateTypeAndVersionGreaterThan(
        String aggregateType, Integer version);
    
    @Query("SELECT e FROM StoredEvent e WHERE e.occurredOn BETWEEN :start AND :end ORDER BY e.occurredOn ASC")
    List<StoredEvent> findEventsBetween(
        @Param("start") LocalDateTime start, 
        @Param("end") LocalDateTime end);
    
    @Query("SELECT e FROM StoredEvent e WHERE e.correlationId = :correlationId ORDER BY e.occurredOn ASC")
    List<StoredEvent> findByCorrelationId(@Param("correlationId") String correlationId);
    
    Optional<StoredEvent> findTopByAggregateIdOrderByVersionDesc(String aggregateId);
}

// 이벤트 스토어 서비스
@Service
@RequiredArgsConstructor
@Transactional
public class EventStoreService {
    
    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    
    public void saveEvent(DomainEvent event) {
        try {
            StoredEvent storedEvent = StoredEvent.builder()
                .eventId(event.getEventId())
                .aggregateId(event.getAggregateId())
                .aggregateType(event.getAggregateType())
                .eventType(event.getEventType())
                .version(event.getVersion())
                .occurredOn(event.getOccurredOn())
                .eventData(objectMapper.writeValueAsString(event))
                .metadata(objectMapper.writeValueAsString(event.getMetadata()))
                .correlationId(event.getCorrelationId())
                .causationId(event.getCausationId())
                .build();
            
            repository.save(storedEvent);
            
        } catch (JsonProcessingException e) {
            throw new EventStoreException("Failed to serialize event", e);
        }
    }
    
    public List<DomainEvent> getEventsForAggregate(String aggregateId) {
        return repository.findByAggregateIdOrderByVersionAsc(aggregateId)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
    
    public List<DomainEvent> getEventsSince(String aggregateType, Integer sinceVersion) {
        return repository.findByAggregateTypeAndVersionGreaterThan(aggregateType, sinceVersion)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
    
    public List<DomainEvent> getEventsInTimeRange(LocalDateTime start, LocalDateTime end) {
        return repository.findEventsBetween(start, end)
            .stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }
    
    private DomainEvent deserializeEvent(StoredEvent storedEvent) {
        try {
            Class<?> eventClass = Class.forName(
                "com.example.events." + storedEvent.getEventType());
            
            return (DomainEvent) objectMapper.readValue(
                storedEvent.getEventData(), eventClass);
                
        } catch (Exception e) {
            throw new EventStoreException("Failed to deserialize event", e);
        }
    }
}
```

## 📨 Outbox 패턴

### Outbox 패턴 구현

```java
// Outbox 이벤트 엔티티
@Entity
@Table(name = "outbox_events")
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OutboxEvent {
    
    @Id
    private String id;
    
    @Column(nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private String eventType;
    
    @Column(columnDefinition = "TEXT")
    private String payload;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProcessingStatus status;
    
    private LocalDateTime processedAt;
    private Integer retryCount;
    private String errorMessage;
    
    public enum ProcessingStatus {
        PENDING, PROCESSED, FAILED
    }
    
    @PrePersist
    public void prePersist() {
        if (createdAt == null) {
            createdAt = LocalDateTime.now();
        }
        if (status == null) {
            status = ProcessingStatus.PENDING;
        }
        if (retryCount == null) {
            retryCount = 0;
        }
    }
    
    public void markAsProcessed() {
        this.status = ProcessingStatus.PROCESSED;
        this.processedAt = LocalDateTime.now();
    }
    
    public void markAsFailed(String errorMessage) {
        this.status = ProcessingStatus.FAILED;
        this.errorMessage = errorMessage;
        this.retryCount++;
    }
}

// Outbox 서비스
@Service
@RequiredArgsConstructor
@Transactional
public class OutboxService {
    
    private final OutboxEventRepository outboxRepository;
    private final ExternalEventPublisher eventPublisher;
    private final ObjectMapper objectMapper;
    
    public void saveOutboxEvent(DomainEvent event) {
        try {
            OutboxEvent outboxEvent = OutboxEvent.builder()
                .id(UUID.randomUUID().toString())
                .aggregateId(event.getAggregateId())
                .eventType(event.getEventType())
                .payload(objectMapper.writeValueAsString(event))
                .status(OutboxEvent.ProcessingStatus.PENDING)
                .build();
            
            outboxRepository.save(outboxEvent);
            
        } catch (JsonProcessingException e) {
            throw new OutboxException("Failed to save outbox event", e);
        }
    }
    
    @Scheduled(fixedDelay = 5000) // 5초마다 실행
    public void processOutboxEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository
            .findByStatusOrderByCreatedAtAsc(OutboxEvent.ProcessingStatus.PENDING);
        
        for (OutboxEvent outboxEvent : pendingEvents) {
            try {
                // 이벤트 발행
                DomainEvent event = deserializeEvent(outboxEvent);
                eventPublisher.publishEvent(event);
                
                // 처리 완료로 마킹
                outboxEvent.markAsProcessed();
                outboxRepository.save(outboxEvent);
                
                log.info("Published outbox event: {}", outboxEvent.getId());
                
            } catch (Exception e) {
                log.error("Failed to process outbox event: {}", outboxEvent.getId(), e);
                
                // 실패로 마킹 (최대 재시도 횟수 확인)
                if (outboxEvent.getRetryCount() < 3) {
                    outboxEvent.markAsFailed(e.getMessage());
                    outboxRepository.save(outboxEvent);
                } else {
                    // DLQ로 이동 또는 수동 처리 필요 알림
                    handleMaxRetriesExceeded(outboxEvent, e);
                }
            }
        }
    }
    
    private DomainEvent deserializeEvent(OutboxEvent outboxEvent) 
            throws JsonProcessingException, ClassNotFoundException {
        Class<?> eventClass = Class.forName(
            "com.example.events." + outboxEvent.getEventType());
        
        return (DomainEvent) objectMapper.readValue(
            outboxEvent.getPayload(), eventClass);
    }
    
    private void handleMaxRetriesExceeded(OutboxEvent outboxEvent, Exception e) {
        log.error("Max retries exceeded for outbox event: {}", outboxEvent.getId(), e);
        // 알림 서비스로 전송 또는 DLQ로 이동
        notificationService.notifyFailedEvent(outboxEvent, e);
    }
}
```

## 🔄 이벤트 처리 전략

### 멱등성 보장

```java
// 멱등성 키 관리
@Service
@RequiredArgsConstructor
public class IdempotencyService {
    
    private final IdempotencyKeyRepository repository;
    
    @Transactional
    public boolean isProcessed(String idempotencyKey) {
        return repository.existsByKey(idempotencyKey);
    }
    
    @Transactional
    public void markAsProcessed(String idempotencyKey, String result) {
        IdempotencyKey key = IdempotencyKey.builder()
            .key(idempotencyKey)
            .result(result)
            .processedAt(LocalDateTime.now())
            .build();
        
        repository.save(key);
    }
    
    public String generateIdempotencyKey(DomainEvent event) {
        return DigestUtils.sha256Hex(
            event.getEventId() + "_" + 
            event.getAggregateId() + "_" + 
            event.getVersion()
        );
    }
}

// 멱등성을 보장하는 이벤트 핸들러
@Component
@RequiredArgsConstructor
public class IdempotentEventHandler {
    
    private final IdempotencyService idempotencyService;
    private final PaymentService paymentService;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        String idempotencyKey = idempotencyService.generateIdempotencyKey(event);
        
        if (idempotencyService.isProcessed(idempotencyKey)) {
            log.info("Event already processed: {}", event.getEventId());
            return;
        }
        
        try {
            // 실제 비즈니스 로직 처리
            String result = paymentService.preparePayment(
                event.getAggregateId(),
                event.getTotalAmount(),
                event.getPaymentMethod()
            );
            
            // 처리 완료 마킹
            idempotencyService.markAsProcessed(idempotencyKey, result);
            
        } catch (Exception e) {
            log.error("Failed to process event: {}", event.getEventId(), e);
            throw e;
        }
    }
}
```

### 재시도 및 에러 처리

```java
// 재시도 가능한 이벤트 핸들러
@Component
@RequiredArgsConstructor
@Slf4j
public class RetryableEventHandler {
    
    @Retryable(
        value = {TransientException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    @EventListener
    public void handleInventoryChanged(ProductInventoryChangedEvent event) {
        try {
            log.info("Processing inventory change: {}", event.getAggregateId());
            
            // 비즈니스 로직
            processInventoryChange(event);
            
        } catch (TransientException e) {
            log.warn("Transient error occurred, will retry: {}", e.getMessage());
            throw e;
        } catch (PermanentException e) {
            log.error("Permanent error occurred, sending to DLQ: {}", e.getMessage());
            sendToDeadLetterQueue(event, e);
        }
    }
    
    @Recover
    public void recover(TransientException ex, ProductInventoryChangedEvent event) {
        log.error("All retry attempts failed for event: {}", event.getEventId(), ex);
        sendToDeadLetterQueue(event, ex);
    }
    
    private void sendToDeadLetterQueue(DomainEvent event, Exception error) {
        DeadLetterEvent dlqEvent = DeadLetterEvent.builder()
            .originalEventId(event.getEventId())
            .eventType(event.getEventType())
            .aggregateId(event.getAggregateId())
            .errorMessage(error.getMessage())
            .failedAt(LocalDateTime.now())
            .build();
        
        deadLetterQueueService.send(dlqEvent);
    }
}
```

## 📊 모니터링과 관찰성

### 이벤트 메트릭스

```java
// 이벤트 메트릭스 수집
@Component
@RequiredArgsConstructor
public class EventMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordEventPublished(String eventType, String aggregateType) {
        Counter.builder("events.published")
            .description("Number of events published")
            .tag("event_type", eventType)
            .tag("aggregate_type", aggregateType)
            .register(meterRegistry)
            .increment();
    }
    
    public void recordEventProcessed(String eventType, String handler, boolean success) {
        Counter.builder("events.processed")
            .description("Number of events processed")
            .tag("event_type", eventType)
            .tag("handler", handler)
            .tag("status", success ? "success" : "failure")
            .register(meterRegistry)
            .increment();
    }
    
    public Timer.Sample startEventProcessing() {
        return Timer.start(meterRegistry);
    }
    
    public void recordEventProcessingTime(Timer.Sample sample, String eventType, String handler) {
        sample.stop(Timer.builder("events.processing.duration")
            .description("Event processing duration")
            .tag("event_type", eventType)
            .tag("handler", handler)
            .register(meterRegistry));
    }
}

// 메트릭스를 수집하는 이벤트 핸들러
@Component
@RequiredArgsConstructor
public class MetricAwareEventHandler {
    
    private final EventMetrics eventMetrics;
    private final OrderService orderService;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        Timer.Sample sample = eventMetrics.startEventProcessing();
        
        try {
            // 이벤트 처리
            orderService.processOrderCreated(event);
            
            // 성공 메트릭 기록
            eventMetrics.recordEventProcessed(
                event.getEventType(), 
                "MetricAwareEventHandler", 
                true
            );
            
        } catch (Exception e) {
            // 실패 메트릭 기록
            eventMetrics.recordEventProcessed(
                event.getEventType(), 
                "MetricAwareEventHandler", 
                false
            );
            throw e;
            
        } finally {
            eventMetrics.recordEventProcessingTime(
                sample, 
                event.getEventType(), 
                "MetricAwareEventHandler"
            );
        }
    }
}
```

### 이벤트 추적

```java
// 이벤트 플로우 추적
@Component
@RequiredArgsConstructor
@Slf4j
public class EventTracker {
    
    private final EventFlowRepository eventFlowRepository;
    
    @EventListener
    public void trackEvent(DomainEvent event) {
        EventFlow eventFlow = EventFlow.builder()
            .eventId(event.getEventId())
            .correlationId(event.getCorrelationId())
            .causationId(event.getCausationId())
            .eventType(event.getEventType())
            .aggregateId(event.getAggregateId())
            .aggregateType(event.getAggregateType())
            .occurredAt(event.getOccurredOn())
            .serviceName(getServiceName())
            .build();
        
        eventFlowRepository.save(eventFlow);
    }
    
    public List<EventFlow> getEventFlow(String correlationId) {
        return eventFlowRepository.findByCorrelationIdOrderByOccurredAtAsc(correlationId);
    }
    
    private String getServiceName() {
        return environment.getProperty("spring.application.name", "unknown-service");
    }
}
```

## 📚 관련 노트

- [[CQRS and Event Sourcing]] - CQRS와 이벤트 소싱 심화
- [[Microservices Architecture Patterns]] - 마이크로서비스 아키텍처
- [[Enterprise Design Patterns]] - 엔터프라이즈 설계 패턴
- [[Observability]] - 시스템 관찰성
- [[Spring Data JPA]] - 데이터 영속성
- [[API Design Patterns]] - API 설계 패턴

## 🔗 외부 리소스

- [Event Driven Architecture Guide](https://martinfowler.com/articles/201701-event-driven.html)
- [Building Event-Driven Microservices](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)
- [Spring Cloud Stream Documentation](https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/)

---

*이벤트 드리븐 아키텍처는 복잡성을 수반하므로, 간단한 시스템부터 시작하여 점진적으로 발전시키는 것을 권장합니다.*
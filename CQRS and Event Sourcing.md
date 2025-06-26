# CQRS and Event Sourcing

> Command Query Responsibility Segregation과 Event Sourcing 패턴의 Spring Boot 구현

## 📋 목차

- [CQRS 개요](#cqrs-개요)
- [Event Sourcing 개요](#event-sourcing-개요)
- [Command Side 구현](#command-side-구현)
- [Query Side 구현](#query-side-구현)
- [Event Store 구현](#event-store-구현)
- [Projection 구현](#projection-구현)
- [Saga와 Process Manager](#saga와-process-manager)
- [스냅샷 패턴](#스냅샷-패턴)
- [성능 최적화](#성능-최적화)

## 🔄 CQRS 개요

### CQRS 핵심 개념

```yaml
# CQRS (Command Query Responsibility Segregation)
개념:
  정의: 읽기와 쓰기 작업을 별도의 모델로 분리하는 패턴
  목적: 복잡한 도메인에서 읽기와 쓰기 최적화
  
구성요소:
  Command Side: 쓰기 작업 담당 (도메인 모델)
  Query Side: 읽기 작업 담당 (읽기 최적화된 모델)
  Event Bus: 양쪽을 연결하는 이벤트 메커니즘

장점:
  - 읽기/쓰기 독립적 최적화
  - 복잡한 도메인 로직 분리
  - 확장성 향상
  - 다양한 읽기 모델 지원

단점:
  - 아키텍처 복잡성 증가
  - 결과적 일관성
  - 개발 및 운영 복잡도 증가
```

### Spring Boot CQRS 스택

```java
// CQRS + Event Sourcing 의존성
dependencies {
    // Spring Boot 기본
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    
    // Axon Framework (CQRS + Event Sourcing)
    implementation 'org.axonframework:axon-spring-boot-starter:4.8.0'
    implementation 'org.axonframework:axon-mongo:4.8.0'
    
    // 이벤트 스토어
    implementation 'com.eventstore:db-client-java:5.0.0'
    
    // 프로젝션 저장소
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    
    // 메시징
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
    
    // JSON 처리
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
}
```

## 📝 Event Sourcing 개요

### Event Sourcing 핵심 개념

```java
// Event Sourcing 기본 인터페이스
public interface Event {
    String getEventId();
    String getAggregateId();
    LocalDateTime getOccurredOn();
    Integer getVersion();
}

// 기본 이벤트 클래스
@Getter
@EqualsAndHashCode
@ToString
public abstract class BaseEvent implements Event {
    
    private final String eventId;
    private final String aggregateId;
    private final LocalDateTime occurredOn;
    private final Integer version;
    
    protected BaseEvent(String aggregateId, Integer version) {
        this.eventId = UUID.randomUUID().toString();
        this.aggregateId = aggregateId;
        this.occurredOn = LocalDateTime.now();
        this.version = version;
    }
}

// Aggregate Root 인터페이스
public interface AggregateRoot {
    String getId();
    Integer getVersion();
    List<Event> getUncommittedEvents();
    void markEventsAsCommitted();
    void loadFromHistory(List<Event> events);
}
```

## ⚡ Command Side 구현

### Aggregate 구현

```java
// 주문 Aggregate
@Getter
public class OrderAggregate implements AggregateRoot {
    
    private String orderId;
    private String customerId;
    private OrderStatus status;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private Integer version;
    
    private final List<Event> uncommittedEvents = new ArrayList<>();
    
    // 생성자 (새로운 Aggregate)
    public OrderAggregate() {
        this.version = 0;
    }
    
    // Command Handler - 주문 생성
    public OrderAggregate(CreateOrderCommand command) {
        this();
        
        // 비즈니스 규칙 검증
        validateCreateOrderCommand(command);
        
        // 이벤트 생성 및 적용
        OrderCreatedEvent event = new OrderCreatedEvent(
            command.getOrderId(),
            command.getCustomerId(),
            command.getItems(),
            command.getShippingAddress(),
            command.getPaymentMethod(),
            this.version + 1
        );
        
        applyEvent(event);
    }
    
    // Command Handler - 주문 확정
    public void confirmOrder(ConfirmOrderCommand command) {
        // 비즈니스 규칙 검증
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("주문을 확정할 수 없는 상태입니다: " + this.status);
        }
        
        // 이벤트 생성 및 적용
        OrderConfirmedEvent event = new OrderConfirmedEvent(
            this.orderId,
            command.getPaymentId(),
            command.getFinalAmount(),
            this.version + 1
        );
        
        applyEvent(event);
    }
    
    // Command Handler - 주문 취소
    public void cancelOrder(CancelOrderCommand command) {
        // 비즈니스 규칙 검증
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("배송 중이거나 완료된 주문은 취소할 수 없습니다");
        }
        
        // 이벤트 생성 및 적용
        OrderCancelledEvent event = new OrderCancelledEvent(
            this.orderId,
            command.getReason(),
            command.getCancelledBy(),
            this.version + 1
        );
        
        applyEvent(event);
    }
    
    // Event Handler - 주문 생성됨
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getAggregateId();
        this.customerId = event.getCustomerId();
        this.status = OrderStatus.PENDING;
        this.items = event.getItems().stream()
            .map(this::toOrderItem)
            .collect(Collectors.toList());
        this.totalAmount = event.getTotalAmount();
        this.createdAt = event.getOccurredOn();
        this.version = event.getVersion();
    }
    
    // Event Handler - 주문 확정됨
    public void on(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
        this.totalAmount = event.getFinalAmount();
        this.version = event.getVersion();
    }
    
    // Event Handler - 주문 취소됨
    public void on(OrderCancelledEvent event) {
        this.status = OrderStatus.CANCELLED;
        this.version = event.getVersion();
    }
    
    // 이벤트 적용 (내부 상태 변경 + 미커밋 이벤트 추가)
    private void applyEvent(Event event) {
        // 이벤트에 따라 상태 변경
        if (event instanceof OrderCreatedEvent) {
            on((OrderCreatedEvent) event);
        } else if (event instanceof OrderConfirmedEvent) {
            on((OrderConfirmedEvent) event);
        } else if (event instanceof OrderCancelledEvent) {
            on((OrderCancelledEvent) event);
        }
        
        // 미커밋 이벤트에 추가
        this.uncommittedEvents.add(event);
    }
    
    @Override
    public void loadFromHistory(List<Event> events) {
        for (Event event : events) {
            if (event instanceof OrderCreatedEvent) {
                on((OrderCreatedEvent) event);
            } else if (event instanceof OrderConfirmedEvent) {
                on((OrderConfirmedEvent) event);
            } else if (event instanceof OrderCancelledEvent) {
                on((OrderCancelledEvent) event);
            }
        }
    }
    
    @Override
    public List<Event> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }
    
    @Override
    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
    
    private void validateCreateOrderCommand(CreateOrderCommand command) {
        if (command.getItems() == null || command.getItems().isEmpty()) {
            throw new IllegalArgumentException("주문 항목이 없습니다");
        }
        
        if (command.getItems().stream()
                .anyMatch(item -> item.getQuantity() <= 0)) {
            throw new IllegalArgumentException("주문 수량은 0보다 커야 합니다");
        }
    }
    
    private OrderItem toOrderItem(OrderCreatedEvent.OrderItemData itemData) {
        return new OrderItem(
            itemData.getProductId(),
            itemData.getProductName(),
            itemData.getQuantity(),
            itemData.getUnitPrice()
        );
    }
}
```

### Command Handler 구현

```java
// Command Handler
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderCommandHandler {
    
    private final AggregateRepository<OrderAggregate> orderRepository;
    private final EventStore eventStore;
    
    @CommandHandler
    public String handle(CreateOrderCommand command) {
        log.info("처리 중인 주문 생성 명령: {}", command.getOrderId());
        
        // 새로운 Aggregate 생성
        OrderAggregate order = new OrderAggregate(command);
        
        // Aggregate 저장 (이벤트 저장)
        orderRepository.save(order);
        
        log.info("주문 생성 완료: {}", command.getOrderId());
        return command.getOrderId();
    }
    
    @CommandHandler
    public void handle(ConfirmOrderCommand command) {
        log.info("처리 중인 주문 확정 명령: {}", command.getOrderId());
        
        // Aggregate 로드
        OrderAggregate order = orderRepository.load(command.getOrderId());
        
        // Command 처리
        order.confirmOrder(command);
        
        // 변경사항 저장
        orderRepository.save(order);
        
        log.info("주문 확정 완료: {}", command.getOrderId());
    }
    
    @CommandHandler
    public void handle(CancelOrderCommand command) {
        log.info("처리 중인 주문 취소 명령: {}", command.getOrderId());
        
        // Aggregate 로드
        OrderAggregate order = orderRepository.load(command.getOrderId());
        
        // Command 처리
        order.cancelOrder(command);
        
        // 변경사항 저장
        orderRepository.save(order);
        
        log.info("주문 취소 완료: {}", command.getOrderId());
    }
}

// Command 정의
@Getter
@AllArgsConstructor
@ToString
public class CreateOrderCommand {
    private final String orderId;
    private final String customerId;
    private final List<OrderItemCommand> items;
    private final String shippingAddress;
    private final PaymentMethod paymentMethod;
}

@Getter
@AllArgsConstructor
@ToString
public class ConfirmOrderCommand {
    private final String orderId;
    private final String paymentId;
    private final BigDecimal finalAmount;
}

@Getter
@AllArgsConstructor
@ToString
public class CancelOrderCommand {
    private final String orderId;
    private final String reason;
    private final String cancelledBy;
}
```

### Command API 컨트롤러

```java
// Command REST API
@RestController
@RequestMapping("/api/commands/orders")
@RequiredArgsConstructor
@Validated
public class OrderCommandController {
    
    private final CommandGateway commandGateway;
    
    @PostMapping
    public ResponseEntity<CommandResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        
        String orderId = UUID.randomUUID().toString();
        
        CreateOrderCommand command = new CreateOrderCommand(
            orderId,
            request.getCustomerId(),
            request.getItems(),
            request.getShippingAddress(),
            request.getPaymentMethod()
        );
        
        CompletableFuture<String> result = commandGateway.send(command);
        
        return ResponseEntity.accepted()
            .body(new CommandResponse(orderId, "주문 생성 명령이 접수되었습니다"));
    }
    
    @PutMapping("/{orderId}/confirm")
    public ResponseEntity<CommandResponse> confirmOrder(
            @PathVariable String orderId,
            @Valid @RequestBody ConfirmOrderRequest request) {
        
        ConfirmOrderCommand command = new ConfirmOrderCommand(
            orderId,
            request.getPaymentId(),
            request.getFinalAmount()
        );
        
        commandGateway.sendAndWait(command);
        
        return ResponseEntity.ok()
            .body(new CommandResponse(orderId, "주문이 확정되었습니다"));
    }
    
    @PutMapping("/{orderId}/cancel")
    public ResponseEntity<CommandResponse> cancelOrder(
            @PathVariable String orderId,
            @Valid @RequestBody CancelOrderRequest request) {
        
        CancelOrderCommand command = new CancelOrderCommand(
            orderId,
            request.getReason(),
            getCurrentUserId()
        );
        
        commandGateway.sendAndWait(command);
        
        return ResponseEntity.ok()
            .body(new CommandResponse(orderId, "주문이 취소되었습니다"));
    }
    
    private String getCurrentUserId() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null ? auth.getName() : "system";
    }
}
```

## 📖 Query Side 구현

### Query Model (Read Model)

```java
// 주문 조회 모델
@Document(collection = "order_views")
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class OrderView {
    
    @Id
    private String orderId;
    private String customerId;
    private String customerName;
    private String customerEmail;
    private OrderStatus status;
    private List<OrderItemView> items;
    private BigDecimal totalAmount;
    private String shippingAddress;
    private PaymentMethod paymentMethod;
    private String paymentId;
    private LocalDateTime createdAt;
    private LocalDateTime confirmedAt;
    private LocalDateTime cancelledAt;
    private String cancellationReason;
    private Integer version;
    
    @Getter
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public static class OrderItemView {
        private String productId;
        private String productName;
        private String productImageUrl;
        private Integer quantity;
        private BigDecimal unitPrice;
        private BigDecimal totalPrice;
        private String category;
    }
}

// 고객별 주문 요약 모델
@Document(collection = "customer_order_summaries")
@Getter
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class CustomerOrderSummary {
    
    @Id
    private String customerId;
    private String customerName;
    private String customerEmail;
    private Integer totalOrders;
    private Integer pendingOrders;
    private Integer confirmedOrders;
    private Integer cancelledOrders;
    private BigDecimal totalSpent;
    private BigDecimal averageOrderValue;
    private LocalDateTime lastOrderDate;
    private LocalDateTime firstOrderDate;
    private List<String> favoriteCategories;
    private LocalDateTime updatedAt;
}
```

### Query Handler 구현

```java
// Query Handler
@Component
@RequiredArgsConstructor
public class OrderQueryHandler {
    
    private final OrderViewRepository orderViewRepository;
    private final CustomerOrderSummaryRepository summaryRepository;
    
    @QueryHandler
    public OrderView handle(GetOrderByIdQuery query) {
        return orderViewRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
    
    @QueryHandler
    public Page<OrderView> handle(GetOrdersByCustomerQuery query) {
        Pageable pageable = PageRequest.of(
            query.getPage(), 
            query.getSize(),
            Sort.by(Sort.Direction.DESC, "createdAt")
        );
        
        return orderViewRepository.findByCustomerId(query.getCustomerId(), pageable);
    }
    
    @QueryHandler
    public Page<OrderView> handle(GetOrdersByStatusQuery query) {
        Pageable pageable = PageRequest.of(query.getPage(), query.getSize());
        return orderViewRepository.findByStatus(query.getStatus(), pageable);
    }
    
    @QueryHandler
    public CustomerOrderSummary handle(GetCustomerOrderSummaryQuery query) {
        return summaryRepository.findById(query.getCustomerId())
            .orElse(CustomerOrderSummary.builder()
                .customerId(query.getCustomerId())
                .totalOrders(0)
                .totalSpent(BigDecimal.ZERO)
                .build());
    }
    
    @QueryHandler
    public List<OrderView> handle(SearchOrdersQuery query) {
        return orderViewRepository.searchOrders(
            query.getKeyword(),
            query.getStatus(),
            query.getStartDate(),
            query.getEndDate()
        );
    }
}

// Query 정의
@Getter
@AllArgsConstructor
public class GetOrderByIdQuery {
    private final String orderId;
}

@Getter
@AllArgsConstructor
public class GetOrdersByCustomerQuery {
    private final String customerId;
    private final int page;
    private final int size;
}

@Getter
@AllArgsConstructor
public class GetOrdersByStatusQuery {
    private final OrderStatus status;
    private final int page;
    private final int size;
}

@Getter
@AllArgsConstructor
public class GetCustomerOrderSummaryQuery {
    private final String customerId;
}
```

### Query API 컨트롤러

```java
// Query REST API
@RestController
@RequestMapping("/api/queries/orders")
@RequiredArgsConstructor
public class OrderQueryController {
    
    private final QueryGateway queryGateway;
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderView> getOrder(@PathVariable String orderId) {
        GetOrderByIdQuery query = new GetOrderByIdQuery(orderId);
        OrderView order = queryGateway.query(query, OrderView.class).join();
        return ResponseEntity.ok(order);
    }
    
    @GetMapping
    public ResponseEntity<Page<OrderView>> getOrdersByCustomer(
            @RequestParam String customerId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        GetOrdersByCustomerQuery query = new GetOrdersByCustomerQuery(customerId, page, size);
        Page<OrderView> orders = queryGateway.query(query, 
            new ResponseType<Page<OrderView>>() {}).join();
        
        return ResponseEntity.ok(orders);
    }
    
    @GetMapping("/status/{status}")
    public ResponseEntity<Page<OrderView>> getOrdersByStatus(
            @PathVariable OrderStatus status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        GetOrdersByStatusQuery query = new GetOrdersByStatusQuery(status, page, size);
        Page<OrderView> orders = queryGateway.query(query, 
            new ResponseType<Page<OrderView>>() {}).join();
        
        return ResponseEntity.ok(orders);
    }
    
    @GetMapping("/customers/{customerId}/summary")
    public ResponseEntity<CustomerOrderSummary> getCustomerSummary(
            @PathVariable String customerId) {
        
        GetCustomerOrderSummaryQuery query = new GetCustomerOrderSummaryQuery(customerId);
        CustomerOrderSummary summary = queryGateway.query(query, CustomerOrderSummary.class).join();
        
        return ResponseEntity.ok(summary);
    }
    
    @GetMapping("/search")
    public ResponseEntity<List<OrderView>> searchOrders(
            @RequestParam String keyword,
            @RequestParam(required = false) OrderStatus status,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startDate,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endDate) {
        
        SearchOrdersQuery query = new SearchOrdersQuery(keyword, status, startDate, endDate);
        List<OrderView> orders = queryGateway.query(query, 
            new ResponseType<List<OrderView>>() {}).join();
        
        return ResponseEntity.ok(orders);
    }
}
```

## 🗄 Event Store 구현

### Event Store 인터페이스

```java
// Event Store 인터페이스
public interface EventStore {
    void saveEvents(String aggregateId, List<Event> events, Integer expectedVersion);
    List<Event> getEventsForAggregate(String aggregateId);
    List<Event> getEventsForAggregate(String aggregateId, Integer fromVersion);
    List<Event> getAllEvents(Integer fromVersion);
    Optional<AggregateSnapshot> getSnapshot(String aggregateId);
    void saveSnapshot(AggregateSnapshot snapshot);
}

// JPA 기반 Event Store 구현
@Service
@RequiredArgsConstructor
@Transactional
public class JpaEventStore implements EventStore {
    
    private final EventStoreRepository eventStoreRepository;
    private final SnapshotRepository snapshotRepository;
    private final EventSerializer eventSerializer;
    
    @Override
    public void saveEvents(String aggregateId, List<Event> events, Integer expectedVersion) {
        // 낙관적 잠금 검증
        validateConcurrency(aggregateId, expectedVersion);
        
        List<EventStoreEntry> entries = events.stream()
            .map(event -> EventStoreEntry.builder()
                .eventId(event.getEventId())
                .aggregateId(aggregateId)
                .eventType(event.getClass().getSimpleName())
                .eventData(eventSerializer.serialize(event))
                .version(event.getVersion())
                .occurredOn(event.getOccurredOn())
                .build())
            .collect(Collectors.toList());
        
        eventStoreRepository.saveAll(entries);
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<Event> getEventsForAggregate(String aggregateId) {
        return eventStoreRepository.findByAggregateIdOrderByVersionAsc(aggregateId)
            .stream()
            .map(entry -> eventSerializer.deserialize(entry.getEventData(), entry.getEventType()))
            .collect(Collectors.toList());
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<Event> getEventsForAggregate(String aggregateId, Integer fromVersion) {
        return eventStoreRepository.findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
                aggregateId, fromVersion)
            .stream()
            .map(entry -> eventSerializer.deserialize(entry.getEventData(), entry.getEventType()))
            .collect(Collectors.toList());
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<Event> getAllEvents(Integer fromVersion) {
        return eventStoreRepository.findByVersionGreaterThanOrderByOccurredOnAsc(fromVersion)
            .stream()
            .map(entry -> eventSerializer.deserialize(entry.getEventData(), entry.getEventType()))
            .collect(Collectors.toList());
    }
    
    @Override
    @Transactional(readOnly = true)
    public Optional<AggregateSnapshot> getSnapshot(String aggregateId) {
        return snapshotRepository.findByAggregateIdOrderByVersionDesc(aggregateId)
            .stream()
            .findFirst();
    }
    
    @Override
    public void saveSnapshot(AggregateSnapshot snapshot) {
        snapshotRepository.save(snapshot);
    }
    
    private void validateConcurrency(String aggregateId, Integer expectedVersion) {
        Integer currentVersion = eventStoreRepository.findMaxVersionByAggregateId(aggregateId)
            .orElse(0);
        
        if (!currentVersion.equals(expectedVersion)) {
            throw new ConcurrencyException(
                String.format("기대 버전: %d, 현재 버전: %d", expectedVersion, currentVersion));
        }
    }
}
```

### Event Serializer

```java
// Event Serializer
@Component
@RequiredArgsConstructor
public class EventSerializer {
    
    private final ObjectMapper objectMapper;
    private final Map<String, Class<? extends Event>> eventTypes = new HashMap<>();
    
    @PostConstruct
    public void registerEventTypes() {
        eventTypes.put("OrderCreatedEvent", OrderCreatedEvent.class);
        eventTypes.put("OrderConfirmedEvent", OrderConfirmedEvent.class);
        eventTypes.put("OrderCancelledEvent", OrderCancelledEvent.class);
        // 다른 이벤트 타입들 등록
    }
    
    public String serialize(Event event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("이벤트 직렬화 실패", e);
        }
    }
    
    public Event deserialize(String eventData, String eventType) {
        try {
            Class<? extends Event> eventClass = eventTypes.get(eventType);
            if (eventClass == null) {
                throw new UnknownEventTypeException("알 수 없는 이벤트 타입: " + eventType);
            }
            
            return objectMapper.readValue(eventData, eventClass);
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("이벤트 역직렬화 실패", e);
        }
    }
}
```

## 📊 Projection 구현

### Event Handler로 Projection 구축

```java
// Order View Projection Handler
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderViewProjectionHandler {
    
    private final OrderViewRepository orderViewRepository;
    private final CustomerService customerService;
    private final ProductService productService;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        log.info("주문 생성 이벤트 처리 중: {}", event.getAggregateId());
        
        // 고객 정보 조회
        CustomerDto customer = customerService.getCustomer(event.getCustomerId());
        
        // 상품 정보 조회 및 OrderItemView 생성
        List<OrderView.OrderItemView> itemViews = event.getItems().stream()
            .map(item -> {
                ProductDto product = productService.getProduct(item.getProductId());
                return OrderView.OrderItemView.builder()
                    .productId(item.getProductId())
                    .productName(item.getProductName())
                    .productImageUrl(product.getImageUrl())
                    .quantity(item.getQuantity())
                    .unitPrice(item.getUnitPrice())
                    .totalPrice(item.getTotalPrice())
                    .category(product.getCategory())
                    .build();
            })
            .collect(Collectors.toList());
        
        // OrderView 생성
        OrderView orderView = OrderView.builder()
            .orderId(event.getAggregateId())
            .customerId(event.getCustomerId())
            .customerName(customer.getName())
            .customerEmail(customer.getEmail())
            .status(OrderStatus.PENDING)
            .items(itemViews)
            .totalAmount(event.getTotalAmount())
            .shippingAddress(event.getShippingAddress())
            .paymentMethod(event.getPaymentMethod())
            .createdAt(event.getOccurredOn())
            .version(event.getVersion())
            .build();
        
        orderViewRepository.save(orderView);
        
        log.info("주문 뷰 생성 완료: {}", event.getAggregateId());
    }
    
    @EventHandler
    public void on(OrderConfirmedEvent event) {
        log.info("주문 확정 이벤트 처리 중: {}", event.getAggregateId());
        
        OrderView orderView = orderViewRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new ProjectionException("OrderView not found: " + event.getAggregateId()));
        
        OrderView updatedView = orderView.toBuilder()
            .status(OrderStatus.CONFIRMED)
            .paymentId(event.getPaymentId())
            .confirmedAt(event.getOccurredOn())
            .totalAmount(event.getFinalAmount())
            .version(event.getVersion())
            .build();
        
        orderViewRepository.save(updatedView);
        
        log.info("주문 뷰 업데이트 완료: {}", event.getAggregateId());
    }
    
    @EventHandler
    public void on(OrderCancelledEvent event) {
        log.info("주문 취소 이벤트 처리 중: {}", event.getAggregateId());
        
        OrderView orderView = orderViewRepository.findById(event.getAggregateId())
            .orElseThrow(() -> new ProjectionException("OrderView not found: " + event.getAggregateId()));
        
        OrderView updatedView = orderView.toBuilder()
            .status(OrderStatus.CANCELLED)
            .cancelledAt(event.getOccurredOn())
            .cancellationReason(event.getReason())
            .version(event.getVersion())
            .build();
        
        orderViewRepository.save(updatedView);
        
        log.info("주문 뷰 업데이트 완료: {}", event.getAggregateId());
    }
}

// Customer Order Summary Projection Handler
@Component
@RequiredArgsConstructor
@Slf4j
public class CustomerOrderSummaryProjectionHandler {
    
    private final CustomerOrderSummaryRepository summaryRepository;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        updateCustomerSummary(event.getCustomerId(), summary -> {
            summary.setTotalOrders(summary.getTotalOrders() + 1);
            summary.setPendingOrders(summary.getPendingOrders() + 1);
            summary.setLastOrderDate(event.getOccurredOn());
            
            if (summary.getFirstOrderDate() == null) {
                summary.setFirstOrderDate(event.getOccurredOn());
            }
        });
    }
    
    @EventHandler
    public void on(OrderConfirmedEvent event) {
        updateCustomerSummary(getCustomerIdFromOrder(event.getAggregateId()), summary -> {
            summary.setPendingOrders(summary.getPendingOrders() - 1);
            summary.setConfirmedOrders(summary.getConfirmedOrders() + 1);
            summary.setTotalSpent(summary.getTotalSpent().add(event.getFinalAmount()));
            
            // 평균 주문 금액 계산
            if (summary.getConfirmedOrders() > 0) {
                summary.setAverageOrderValue(
                    summary.getTotalSpent().divide(
                        BigDecimal.valueOf(summary.getConfirmedOrders()), 
                        RoundingMode.HALF_UP
                    )
                );
            }
        });
    }
    
    @EventHandler
    public void on(OrderCancelledEvent event) {
        updateCustomerSummary(getCustomerIdFromOrder(event.getAggregateId()), summary -> {
            summary.setPendingOrders(summary.getPendingOrders() - 1);
            summary.setCancelledOrders(summary.getCancelledOrders() + 1);
        });
    }
    
    private void updateCustomerSummary(String customerId, Consumer<CustomerOrderSummary> updater) {
        CustomerOrderSummary summary = summaryRepository.findById(customerId)
            .orElse(CustomerOrderSummary.builder()
                .customerId(customerId)
                .totalOrders(0)
                .pendingOrders(0)
                .confirmedOrders(0)
                .cancelledOrders(0)
                .totalSpent(BigDecimal.ZERO)
                .averageOrderValue(BigDecimal.ZERO)
                .favoriteCategories(new ArrayList<>())
                .build());
        
        updater.accept(summary);
        summary.setUpdatedAt(LocalDateTime.now());
        
        summaryRepository.save(summary);
    }
    
    private String getCustomerIdFromOrder(String orderId) {
        // OrderView에서 고객 ID 조회
        return orderViewRepository.findById(orderId)
            .map(OrderView::getCustomerId)
            .orElseThrow(() -> new ProjectionException("Order not found: " + orderId));
    }
}
```

## 🔄 Saga와 Process Manager

### Order Processing Saga

```java
// Order Processing Saga
@Saga
@RequiredArgsConstructor
@Slf4j
public class OrderProcessingSaga {
    
    @Autowired
    private transient CommandGateway commandGateway;
    
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
    private boolean paymentProcessed = false;
    private boolean inventoryReserved = false;
    
    @SagaOrchestrationStart
    @SagaStart
    public void handle(OrderCreatedEvent event) {
        log.info("Order processing saga 시작: {}", event.getAggregateId());
        
        this.orderId = event.getAggregateId();
        this.customerId = event.getCustomerId();
        this.totalAmount = event.getTotalAmount();
        
        // 1. 재고 예약 명령
        ReserveInventoryCommand reserveCommand = new ReserveInventoryCommand(
            orderId, event.getItems());
        commandGateway.send(reserveCommand);
    }
    
    @SagaOrchestrationContinue
    public void handle(InventoryReservedEvent event) {
        log.info("재고 예약 완료: {}", orderId);
        this.inventoryReserved = true;
        
        // 2. 결제 처리 명령
        ProcessPaymentCommand paymentCommand = new ProcessPaymentCommand(
            orderId, customerId, totalAmount);
        commandGateway.send(paymentCommand);
    }
    
    @SagaOrchestrationContinue
    public void handle(PaymentProcessedEvent event) {
        log.info("결제 처리 완료: {}", orderId);
        this.paymentProcessed = true;
        
        // 3. 주문 확정 명령
        ConfirmOrderCommand confirmCommand = new ConfirmOrderCommand(
            orderId, event.getPaymentId(), event.getAmount());
        commandGateway.send(confirmCommand);
    }
    
    @SagaOrchestrationContinue
    public void handle(OrderConfirmedEvent event) {
        log.info("주문 확정 완료, saga 종료: {}", orderId);
        
        // Saga 종료
        SagaLifecycle.end();
    }
    
    // 보상 트랜잭션들
    @SagaOrchestrationContinue
    public void handle(InventoryReservationFailedEvent event) {
        log.error("재고 예약 실패: {}", orderId);
        
        // 주문 취소
        CancelOrderCommand cancelCommand = new CancelOrderCommand(
            orderId, "재고 부족", "system");
        commandGateway.send(cancelCommand);
    }
    
    @SagaOrchestrationContinue
    public void handle(PaymentFailedEvent event) {
        log.error("결제 실패: {}", orderId);
        
        if (inventoryReserved) {
            // 재고 예약 해제
            ReleaseInventoryCommand releaseCommand = new ReleaseInventoryCommand(orderId);
            commandGateway.send(releaseCommand);
        }
        
        // 주문 취소
        CancelOrderCommand cancelCommand = new CancelOrderCommand(
            orderId, "결제 실패", "system");
        commandGateway.send(cancelCommand);
    }
    
    @SagaOrchestrationContinue
    public void handle(OrderCancelledEvent event) {
        log.info("주문 취소 처리 완료, saga 종료: {}", orderId);
        
        // Saga 종료
        SagaLifecycle.end();
    }
}
```

## 📸 스냅샷 패턴

### 스냅샷 구현

```java
// Aggregate Snapshot
@Entity
@Table(name = "aggregate_snapshots")
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AggregateSnapshot {
    
    @Id
    private String snapshotId;
    
    @Column(nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private String aggregateType;
    
    @Column(nullable = false)
    private Integer version;
    
    @Column(columnDefinition = "TEXT")
    private String snapshotData;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @PrePersist
    public void prePersist() {
        if (snapshotId == null) {
            snapshotId = UUID.randomUUID().toString();
        }
        if (createdAt == null) {
            createdAt = LocalDateTime.now();
        }
    }
}

// 스냅샷 기능이 있는 Aggregate Repository
@Service
@RequiredArgsConstructor
public class SnapshotAggregateRepository<T extends AggregateRoot> {
    
    private final EventStore eventStore;
    private final ObjectMapper objectMapper;
    private final Class<T> aggregateClass;
    
    public T load(String aggregateId) {
        // 1. 최신 스냅샷 조회
        Optional<AggregateSnapshot> snapshot = eventStore.getSnapshot(aggregateId);
        
        T aggregate;
        Integer fromVersion = 0;
        
        if (snapshot.isPresent()) {
            // 스냅샷에서 Aggregate 복원
            aggregate = deserializeAggregate(snapshot.get().getSnapshotData());
            fromVersion = snapshot.get().getVersion();
        } else {
            // 새로운 Aggregate 인스턴스 생성
            aggregate = createNewInstance();
        }
        
        // 2. 스냅샷 이후의 이벤트들 적용
        List<Event> events = eventStore.getEventsForAggregate(aggregateId, fromVersion);
        aggregate.loadFromHistory(events);
        
        return aggregate;
    }
    
    public void save(T aggregate) {
        List<Event> uncommittedEvents = aggregate.getUncommittedEvents();
        
        if (!uncommittedEvents.isEmpty()) {
            // 이벤트 저장
            eventStore.saveEvents(
                aggregate.getId(), 
                uncommittedEvents, 
                aggregate.getVersion() - uncommittedEvents.size()
            );
            
            // 이벤트 커밋 마킹
            aggregate.markEventsAsCommitted();
            
            // 스냅샷 저장 조건 확인 (예: 10개 이벤트마다)
            if (shouldCreateSnapshot(aggregate.getVersion())) {
                createSnapshot(aggregate);
            }
        }
    }
    
    private boolean shouldCreateSnapshot(Integer version) {
        return version % 10 == 0; // 10개 이벤트마다 스냅샷 생성
    }
    
    private void createSnapshot(T aggregate) {
        try {
            String snapshotData = objectMapper.writeValueAsString(aggregate);
            
            AggregateSnapshot snapshot = AggregateSnapshot.builder()
                .aggregateId(aggregate.getId())
                .aggregateType(aggregate.getClass().getSimpleName())
                .version(aggregate.getVersion())
                .snapshotData(snapshotData)
                .build();
            
            eventStore.saveSnapshot(snapshot);
            
        } catch (JsonProcessingException e) {
            log.warn("스냅샷 생성 실패: {}", aggregate.getId(), e);
        }
    }
    
    private T deserializeAggregate(String snapshotData) {
        try {
            return objectMapper.readValue(snapshotData, aggregateClass);
        } catch (JsonProcessingException e) {
            throw new SnapshotDeserializationException("스냅샷 역직렬화 실패", e);
        }
    }
    
    private T createNewInstance() {
        try {
            return aggregateClass.getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            throw new AggregateCreationException("Aggregate 인스턴스 생성 실패", e);
        }
    }
}
```

## ⚡ 성능 최적화

### 읽기 모델 최적화

```java
// 캐시를 사용한 Query Handler
@Component
@RequiredArgsConstructor
public class CachedOrderQueryHandler {
    
    private final OrderViewRepository orderViewRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @QueryHandler
    @Cacheable(value = "orderViews", key = "#query.orderId")
    public OrderView handle(GetOrderByIdQuery query) {
        return orderViewRepository.findById(query.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
    
    @QueryHandler
    @Cacheable(value = "customerOrders", key = "#query.customerId + '_' + #query.page + '_' + #query.size")
    public Page<OrderView> handle(GetOrdersByCustomerQuery query) {
        Pageable pageable = PageRequest.of(query.getPage(), query.getSize());
        return orderViewRepository.findByCustomerId(query.getCustomerId(), pageable);
    }
    
    // 캐시 무효화
    @EventHandler
    @CacheEvict(value = {"orderViews", "customerOrders"}, allEntries = true)
    public void on(OrderUpdatedEvent event) {
        // 캐시 무효화만 수행
    }
}

// 비동기 프로젝션 업데이트
@Component
@RequiredArgsConstructor
@Slf4j
public class AsyncProjectionHandler {
    
    @Async("projectionExecutor")
    @EventHandler
    public void on(OrderCreatedEvent event) {
        try {
            // 무거운 프로젝션 작업
            updateHeavyProjections(event);
        } catch (Exception e) {
            log.error("프로젝션 업데이트 실패: {}", event.getAggregateId(), e);
        }
    }
    
    private void updateHeavyProjections(OrderCreatedEvent event) {
        // 통계 업데이트, 리포트 생성 등 시간이 오래 걸리는 작업
    }
}
```

### 이벤트 스토어 최적화

```java
// 배치로 이벤트 저장
@Service
@RequiredArgsConstructor
public class BatchEventStore implements EventStore {
    
    private final EventStoreRepository repository;
    private final List<EventStoreEntry> pendingEvents = new ArrayList<>();
    private final Object lock = new Object();
    
    @Override
    public void saveEvents(String aggregateId, List<Event> events, Integer expectedVersion) {
        List<EventStoreEntry> entries = events.stream()
            .map(this::toEventStoreEntry)
            .collect(Collectors.toList());
        
        synchronized (lock) {
            pendingEvents.addAll(entries);
        }
        
        // 배치 크기에 도달하면 저장
        if (pendingEvents.size() >= 100) {
            flushPendingEvents();
        }
    }
    
    @Scheduled(fixedDelay = 1000) // 1초마다 플러시
    public void flushPendingEvents() {
        synchronized (lock) {
            if (!pendingEvents.isEmpty()) {
                repository.saveAll(new ArrayList<>(pendingEvents));
                pendingEvents.clear();
            }
        }
    }
}
```

## 📚 관련 노트

- [[Event Driven Architecture]] - 이벤트 드리븐 아키텍처
- [[Microservices Architecture Patterns]] - 마이크로서비스 패턴
- [[Enterprise Design Patterns]] - 엔터프라이즈 설계 패턴
- [[Spring Data JPA]] - 데이터 영속성
- [[API Design Patterns]] - API 설계
- [[Observability]] - 시스템 관찰성

## 🔗 외부 리소스

- [CQRS Journey by Microsoft](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10))
- [Event Sourcing by Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Axon Framework Documentation](https://docs.axoniq.io/reference-guide/)

---

*CQRS와 Event Sourcing은 복잡한 패턴이므로, 도메인의 복잡성이 이를 정당화할 때만 사용하세요.*
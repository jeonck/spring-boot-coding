# Service Layer Pattern

## 개념
- 비즈니스 로직의 중앙 집중화 패턴
- [[Enterprise Design Patterns]]의 핵심 구성요소
- [[Spring IoC와 DI]]를 활용한 느슨한 결합
- 컨트롤러와 데이터 접근 계층 간의 중간 계층

## 서비스 계층의 역할
- **비즈니스 로직 캡슐화**: 핵심 업무 규칙과 프로세스
- **트랜잭션 경계 정의**: 데이터 일관성 보장
- **도메인 객체 조정**: 여러 엔티티 간 상호작용
- **외부 서비스 통합**: API 호출 및 통합 로직

## 1. 기본 서비스 패턴
### 인터페이스 기반 설계
```java
// 서비스 인터페이스 정의
public interface UserService {
    
    // 사용자 생성
    UserResponse createUser(CreateUserRequest request);
    
    // 사용자 조회
    UserResponse getUserById(Long id);
    UserResponse getUserByEmail(String email);
    Page<UserResponse> getUsers(UserSearchCriteria criteria, Pageable pageable);
    
    // 사용자 수정
    UserResponse updateUser(Long id, UpdateUserRequest request);
    UserResponse partialUpdateUser(Long id, Map<String, Object> updates);
    
    // 사용자 삭제
    void deleteUser(Long id);
    void softDeleteUser(Long id);
    
    // 비즈니스 메서드
    void activateUser(Long id);
    void deactivateUser(Long id);
    void resetPassword(String email);
    void changePassword(Long id, ChangePasswordRequest request);
    
    // 통계 및 검색
    UserStatistics getUserStatistics();
    List<UserResponse> searchUsers(String keyword);
    boolean existsByEmail(String email);
}

// 서비스 구현체
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class UserServiceImpl implements UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    private final AuditService auditService;
    private final UserMapper userMapper;
    
    @Override
    public UserResponse createUser(CreateUserRequest request) {
        log.info("Creating user with email: {}", request.getEmail());
        
        // 1. 비즈니스 규칙 검증
        validateUserCreation(request);
        
        // 2. 엔티티 생성 및 변환
        User user = User.builder()
            .name(request.getName())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .role(Role.USER)
            .status(UserStatus.PENDING)
            .emailVerified(false)
            .createdAt(LocalDateTime.now())
            .build();
        
        // 3. 데이터 저장
        User savedUser = userRepository.save(user);
        
        // 4. 후속 처리 (이벤트, 알림 등)
        handlePostUserCreation(savedUser);
        
        // 5. 응답 DTO 변환
        UserResponse response = userMapper.toResponse(savedUser);
        
        log.info("Successfully created user with ID: {}", response.getId());
        return response;
    }
    
    @Override
    @Transactional(readOnly = true)
    public UserResponse getUserById(Long id) {
        log.debug("Fetching user by ID: {}", id);
        
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        // 감사 로그 기록
        auditService.logUserAccess(id);
        
        return userMapper.toResponse(user);
    }
    
    @Override
    public UserResponse updateUser(Long id, UpdateUserRequest request) {
        log.info("Updating user with ID: {}", id);
        
        // 1. 기존 사용자 조회
        User existingUser = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        // 2. 비즈니스 규칙 검증
        validateUserUpdate(existingUser, request);
        
        // 3. 변경 사항 적용
        existingUser.updateProfile(
            request.getName(),
            request.getPhoneNumber(),
            request.getAddress()
        );
        
        // 4. 변경 감지 및 저장 (더티 체킹)
        User updatedUser = userRepository.save(existingUser);
        
        // 5. 후속 처리
        handlePostUserUpdate(updatedUser);
        
        return userMapper.toResponse(updatedUser);
    }
    
    // 비즈니스 로직 메서드들
    @Override
    public void activateUser(Long id) {
        User user = getUserEntity(id);
        
        if (user.isActive()) {
            throw new IllegalStateException("User is already active");
        }
        
        user.activate();
        userRepository.save(user);
        
        // 활성화 알림 이메일 발송
        emailService.sendUserActivationNotification(user.getEmail(), user.getName());
        
        log.info("User activated: {}", id);
    }
    
    @Override
    public void resetPassword(String email) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UserNotFoundException("Email not found: " + email));
        
        // 임시 비밀번호 생성
        String temporaryPassword = generateTemporaryPassword();
        user.setPassword(passwordEncoder.encode(temporaryPassword));
        user.setPasswordResetRequired(true);
        
        userRepository.save(user);
        
        // 임시 비밀번호 이메일 발송
        emailService.sendPasswordResetEmail(email, temporaryPassword);
        
        log.info("Password reset initiated for email: {}", email);
    }
    
    // 검증 메서드들
    private void validateUserCreation(CreateUserRequest request) {
        // 이메일 중복 검사
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }
        
        // 비밀번호 강도 검사
        if (!isStrongPassword(request.getPassword())) {
            throw new WeakPasswordException("Password does not meet security requirements");
        }
        
        // 추가 비즈니스 규칙들...
    }
    
    private void validateUserUpdate(User existingUser, UpdateUserRequest request) {
        // 이메일 변경 시 중복 검사
        if (!existingUser.getEmail().equals(request.getEmail()) &&
            userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }
        
        // 권한 검사
        if (!canUpdateUser(existingUser)) {
            throw new InsufficientPermissionException("Cannot update this user");
        }
    }
    
    // 헬퍼 메서드들
    private User getUserEntity(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    private void handlePostUserCreation(User user) {
        // 환영 이메일 발송
        emailService.sendWelcomeEmail(user.getEmail(), user.getName());
        
        // 사용자 생성 이벤트 발행
        applicationEventPublisher.publishEvent(new UserCreatedEvent(user.getId()));
        
        // 감사 로그 기록
        auditService.logUserCreation(user.getId());
    }
}
```

## 2. 복합 비즈니스 로직 패턴
### 여러 엔티티를 조정하는 서비스
```java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {
    
    private final OrderRepository orderRepository;
    private final ProductService productService;
    private final UserService userService;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    private final OrderMapper orderMapper;
    
    @Override
    public OrderResponse createOrder(CreateOrderRequest request) {
        log.info("Creating order for user: {} with {} items", 
            request.getUserId(), request.getItems().size());
        
        // 1. 사전 검증
        validateOrderCreation(request);
        
        // 2. 사용자 정보 조회
        UserResponse user = userService.getUserById(request.getUserId());
        
        // 3. 주문 항목 처리
        List<OrderItem> orderItems = processOrderItems(request.getItems());
        
        // 4. 주문 총액 계산
        BigDecimal totalAmount = calculateTotalAmount(orderItems);
        
        // 5. 재고 예약
        reserveInventory(orderItems);
        
        try {
            // 6. 주문 엔티티 생성
            Order order = Order.builder()
                .userId(request.getUserId())
                .orderItems(orderItems)
                .totalAmount(totalAmount)
                .status(OrderStatus.PENDING)
                .orderDate(LocalDateTime.now())
                .shippingAddress(request.getShippingAddress())
                .build();
            
            // 7. 주문 저장
            Order savedOrder = orderRepository.save(order);
            
            // 8. 결제 처리
            PaymentResult paymentResult = processPayment(savedOrder, request.getPaymentInfo());
            
            // 9. 결제 성공 시 주문 확정
            if (paymentResult.isSuccess()) {
                savedOrder.confirm(paymentResult.getTransactionId());
                confirmInventoryReservation(orderItems);
            } else {
                savedOrder.fail(paymentResult.getErrorMessage());
                releaseInventoryReservation(orderItems);
                throw new PaymentFailedException(paymentResult.getErrorMessage());
            }
            
            // 10. 후속 처리
            handlePostOrderCreation(savedOrder);
            
            return orderMapper.toResponse(savedOrder);
            
        } catch (Exception e) {
            // 실패 시 재고 예약 해제
            releaseInventoryReservation(orderItems);
            throw new OrderCreationException("Failed to create order", e);
        }
    }
    
    @Override
    public void cancelOrder(Long orderId) {
        Order order = getOrderEntity(orderId);
        
        // 취소 가능 상태 검증
        if (!order.isCancellable()) {
            throw new IllegalOrderStateException("Order cannot be cancelled in current state: " + order.getStatus());
        }
        
        // 주문 취소 처리
        order.cancel();
        
        // 재고 복원
        restoreInventory(order.getOrderItems());
        
        // 환불 처리 (필요 시)
        if (order.isPaid()) {
            processRefund(order);
        }
        
        // 취소 알림
        notificationService.sendOrderCancellationNotification(order);
        
        log.info("Order cancelled: {}", orderId);
    }
    
    // 복잡한 비즈니스 로직 메서드들
    private List<OrderItem> processOrderItems(List<CreateOrderItemRequest> items) {
        return items.stream()
            .map(this::createOrderItem)
            .collect(Collectors.toList());
    }
    
    private OrderItem createOrderItem(CreateOrderItemRequest request) {
        // 상품 정보 조회
        ProductResponse product = productService.getProductById(request.getProductId());
        
        // 재고 확인
        if (!inventoryService.isAvailable(request.getProductId(), request.getQuantity())) {
            throw new InsufficientStockException(request.getProductId(), request.getQuantity());
        }
        
        // 가격 계산 (할인, 쿠폰 등 적용)
        BigDecimal unitPrice = calculateEffectivePrice(product, request);
        BigDecimal totalPrice = unitPrice.multiply(BigDecimal.valueOf(request.getQuantity()));
        
        return OrderItem.builder()
            .productId(request.getProductId())
            .productName(product.getName())
            .quantity(request.getQuantity())
            .unitPrice(unitPrice)
            .totalPrice(totalPrice)
            .build();
    }
    
    private BigDecimal calculateTotalAmount(List<OrderItem> orderItems) {
        BigDecimal subtotal = orderItems.stream()
            .map(OrderItem::getTotalPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        // 배송비 계산
        BigDecimal shippingCost = calculateShippingCost(subtotal);
        
        // 세금 계산
        BigDecimal tax = calculateTax(subtotal);
        
        return subtotal.add(shippingCost).add(tax);
    }
}
```

## 3. 도메인 서비스 패턴
### 도메인별 특화 서비스
```java
// 도메인 특화 서비스
@Service
@RequiredArgsConstructor
@Slf4j
public class UserRegistrationService {
    
    private final UserRepository userRepository;
    private final EmailVerificationService emailVerificationService;
    private final WelcomeService welcomeService;
    private final SecurityService securityService;
    
    @Transactional
    public UserRegistrationResult registerUser(UserRegistrationRequest request) {
        log.info("Starting user registration process for email: {}", request.getEmail());
        
        // 1. 등록 자격 검증
        validateRegistrationEligibility(request);
        
        // 2. 사용자 엔티티 생성
        User user = createUserEntity(request);
        
        // 3. 이메일 인증 프로세스 시작
        EmailVerificationToken token = emailVerificationService.initiateVerification(user.getEmail());
        
        // 4. 사용자 저장
        User savedUser = userRepository.save(user);
        
        // 5. 환영 프로세스 시작
        welcomeService.startWelcomeProcess(savedUser);
        
        // 6. 보안 설정 초기화
        securityService.initializeUserSecurity(savedUser);
        
        return UserRegistrationResult.builder()
            .userId(savedUser.getId())
            .verificationToken(token.getToken())
            .verificationRequired(true)
            .build();
    }
    
    @Transactional
    public void completeRegistration(String verificationToken) {
        // 인증 토큰 검증
        EmailVerificationToken token = emailVerificationService.validateToken(verificationToken);
        
        // 사용자 활성화
        User user = userRepository.findByEmail(token.getEmail())
            .orElseThrow(() -> new UserNotFoundException("Email: " + token.getEmail()));
        
        user.completeRegistration();
        
        // 토큰 만료 처리
        emailVerificationService.markTokenAsUsed(token);
        
        // 환영 프로세스 완료
        welcomeService.completeWelcomeProcess(user);
        
        log.info("User registration completed for: {}", user.getEmail());
    }
}

// 특화된 비즈니스 서비스
@Service
@RequiredArgsConstructor
public class ProductRecommendationService {
    
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final RecommendationEngine recommendationEngine;
    
    public List<ProductResponse> getPersonalizedRecommendations(Long userId, int limit) {
        // 사용자 구매 히스토리 분석
        List<Order> orderHistory = orderRepository.findByUserIdOrderByOrderDateDesc(userId);
        
        // 사용자 선호도 분석
        UserPreference preference = analyzeUserPreference(orderHistory);
        
        // 추천 엔진을 통한 상품 추천
        List<Product> recommendedProducts = recommendationEngine.recommend(preference, limit);
        
        return recommendedProducts.stream()
            .map(this::toProductResponse)
            .collect(Collectors.toList());
    }
    
    public List<ProductResponse> getSimilarProducts(Long productId, int limit) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 유사 상품 찾기 알고리즘
        List<Product> similarProducts = recommendationEngine.findSimilar(product, limit);
        
        return similarProducts.stream()
            .map(this::toProductResponse)
            .collect(Collectors.toList());
    }
}
```

## 4. 서비스 조합 패턴 (Service Composition)
### 여러 서비스를 조합하는 상위 서비스
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EcommerceWorkflowService {
    
    private final UserService userService;
    private final ProductService productService;
    private final OrderService orderService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    private final NotificationService notificationService;
    private final LoyaltyService loyaltyService;
    
    @Transactional
    public CompleteOrderResult processCompleteOrder(CompleteOrderRequest request) {
        log.info("Processing complete order workflow for user: {}", request.getUserId());
        
        try {
            // 1. 주문 생성
            OrderResponse order = orderService.createOrder(request.getOrderRequest());
            
            // 2. 배송 준비
            ShippingResponse shipping = shippingService.prepareShipment(order.getId());
            
            // 3. 고객 알림
            notificationService.sendOrderConfirmation(order);
            
            // 4. 로열티 포인트 적립
            LoyaltyPoints points = loyaltyService.earnPoints(request.getUserId(), order.getTotalAmount());
            
            // 5. 재고 업데이트
            updateInventoryAfterOrder(order);
            
            // 6. 추천 시스템 업데이트
            updateRecommendationModel(request.getUserId(), order);
            
            return CompleteOrderResult.builder()
                .order(order)
                .shipping(shipping)
                .loyaltyPoints(points)
                .estimatedDelivery(shipping.getEstimatedDelivery())
                .build();
                
        } catch (Exception e) {
            log.error("Failed to process complete order: {}", e.getMessage(), e);
            
            // 보상 트랜잭션 (Saga Pattern)
            handleOrderFailure(request);
            
            throw new OrderProcessingException("Order processing failed", e);
        }
    }
    
    @Async
    public CompletableFuture<Void> processPostOrderActivities(Long orderId) {
        return CompletableFuture.runAsync(() -> {
            try {
                OrderResponse order = orderService.getOrderById(orderId);
                
                // 비동기 후속 처리들
                generateOrderAnalytics(order);
                updateCustomerProfile(order);
                triggerCrossSellCampaign(order);
                scheduleFollowUpCommunication(order);
                
            } catch (Exception e) {
                log.error("Failed to process post-order activities for order: {}", orderId, e);
            }
        });
    }
}
```

## 5. 이벤트 기반 서비스 패턴
### 도메인 이벤트를 활용한 느슨한 결합
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserLifecycleService {
    
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void activateUser(Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        UserStatus previousStatus = user.getStatus();
        user.activate();
        
        // 도메인 이벤트 발행
        eventPublisher.publishEvent(UserActivatedEvent.builder()
            .userId(user.getId())
            .email(user.getEmail())
            .previousStatus(previousStatus)
            .activatedAt(LocalDateTime.now())
            .build());
        
        log.info("User activated and event published: {}", userId);
    }
}

// 이벤트 핸들러들
@Component
@RequiredArgsConstructor
@Slf4j
public class UserEventHandler {
    
    private final EmailService emailService;
    private final AuditService auditService;
    private final RecommendationService recommendationService;
    
    @EventListener
    @Async
    public void handleUserActivated(UserActivatedEvent event) {
        log.info("Handling user activation event for user: {}", event.getUserId());
        
        // 환영 이메일 발송
        emailService.sendUserActivationEmail(event.getEmail());
        
        // 감사 로그 기록
        auditService.recordUserActivation(event);
        
        // 개인화 서비스 초기화
        recommendationService.initializeUserProfile(event.getUserId());
    }
    
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        // 즉시 처리가 필요한 작업들
        auditService.recordUserCreation(event);
    }
}
```

## 6. 캐싱 전략이 적용된 서비스
### 성능 최적화를 위한 캐싱 패턴
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductCatalogService {
    
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "products", key = "#id")
    public ProductResponse getProduct(Long id) {
        log.debug("Fetching product from database: {}", id);
        
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        
        return ProductResponse.from(product);
    }
    
    @Cacheable(value = "categories", key = "'all'")
    public List<CategoryResponse> getAllCategories() {
        log.debug("Fetching all categories from database");
        
        return categoryRepository.findAll()
            .stream()
            .map(CategoryResponse::from)
            .collect(Collectors.toList());
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public ProductResponse updateProduct(Long productId, UpdateProductRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.update(request);
        Product savedProduct = productRepository.save(product);
        
        // 관련 캐시들도 무효화
        evictRelatedCaches(productId);
        
        return ProductResponse.from(savedProduct);
    }
    
    private void evictRelatedCaches(Long productId) {
        // 카테고리별 상품 목록 캐시 무효화
        Product product = productRepository.findById(productId).orElse(null);
        if (product != null) {
            redisTemplate.delete("category:products:" + product.getCategoryId());
        }
        
        // 검색 결과 캐시 무효화
        redisTemplate.delete("search:*");
    }
}
```

## 7. 보안이 적용된 서비스 패턴
### 메서드 레벨 보안
```java
@Service
@RequiredArgsConstructor
@Slf4j
@PreAuthorize("hasRole('USER')")
public class SecureUserService {
    
    private final UserRepository userRepository;
    private final SecurityService securityService;
    
    @PreAuthorize("hasRole('ADMIN') or @securityService.isOwner(authentication.name, #userId)")
    public UserResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
            .map(UserResponse::from)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }
    
    @PreAuthorize("@securityService.canModifyUser(authentication.name, #userId)")
    public UserResponse updateUserProfile(Long userId, UpdateUserRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // 민감한 정보 수정 시 추가 검증
        if (request.hasEmailChange()) {
            securityService.validateEmailChange(user, request.getEmail());
        }
        
        user.updateProfile(request);
        return UserResponse.from(userRepository.save(user));
    }
    
    @PostAuthorize("@securityService.filterSensitiveData(returnObject, authentication)")
    public List<UserResponse> searchUsers(String query) {
        return userRepository.findByNameContaining(query)
            .stream()
            .map(UserResponse::from)
            .collect(Collectors.toList());
    }
}
```

## 8. 테스트 가능한 서비스 설계
### 단위 테스트와 통합 테스트를 고려한 설계
```java
@Service
@RequiredArgsConstructor
public class TestableUserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final Clock clock;  // 테스트 가능한 시간 처리
    private final IdGenerator idGenerator;  // 테스트 가능한 ID 생성
    
    public UserResponse createUser(CreateUserRequest request) {
        // 현재 시간을 주입받은 Clock으로 처리
        LocalDateTime now = LocalDateTime.now(clock);
        
        User user = User.builder()
            .id(idGenerator.generateId())  // 테스트에서 예측 가능한 ID
            .name(request.getName())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .createdAt(now)
            .build();
        
        User savedUser = userRepository.save(user);
        return UserResponse.from(savedUser);
    }
}

// 테스트 예시
@ExtendWith(MockitoExtension.class)
class TestableUserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @Mock
    private Clock clock;
    
    @Mock
    private IdGenerator idGenerator;
    
    @InjectMocks
    private TestableUserService userService;
    
    @Test
    void shouldCreateUserWithPredictableValues() {
        // given
        LocalDateTime fixedTime = LocalDateTime.of(2024, 1, 1, 10, 0, 0);
        when(clock.instant()).thenReturn(fixedTime.toInstant(ZoneOffset.UTC));
        when(clock.getZone()).thenReturn(ZoneOffset.UTC);
        when(idGenerator.generateId()).thenReturn(12345L);
        when(passwordEncoder.encode("password")).thenReturn("encoded-password");
        
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com", "password");
        User savedUser = User.builder().id(12345L).name("John").build();
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        
        // when
        UserResponse result = userService.createUser(request);
        
        // then
        assertThat(result.getId()).isEqualTo(12345L);
        verify(userRepository).save(argThat(user -> 
            user.getCreatedAt().equals(fixedTime)));
    }
}
```

## 관련 패턴
- [[Enterprise Design Patterns]]
- [[Spring IoC와 DI]]
- [[Dependency Injection Patterns]]
- [[Exception Handling Patterns]]
- [[Testing]]
- [[Spring Bean Management]]

#service-layer #business-logic #transaction #domain-service

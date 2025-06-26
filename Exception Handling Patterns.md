# Exception Handling Patterns

## 개념
- Spring Boot 애플리케이션의 체계적인 예외 처리
- [[Enterprise Design Patterns]]의 핵심 구성요소
- 사용자 친화적 에러 응답과 로깅 전략

## 1. Global Exception Handler
### @ControllerAdvice를 활용한 중앙 집중식 예외 처리
```java
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    // 비즈니스 예외 처리
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.warn("Business exception occurred: {}", e.getMessage());
        
        ErrorResponse errorResponse = ErrorResponse.builder()
            .code(e.getErrorCode().getCode())
            .message(e.getErrorCode().getMessage())
            .timestamp(LocalDateTime.now())
            .build();
            
        return ResponseEntity
            .status(e.getErrorCode().getHttpStatus())
            .body(errorResponse);
    }
    
    // 엔티티 조회 실패
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleEntityNotFoundException(EntityNotFoundException e) {
        log.warn("Entity not found: {}", e.getMessage());
        
        ErrorResponse errorResponse = ErrorResponse.builder()
            .code("ENTITY_NOT_FOUND")
            .message(e.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
            
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(errorResponse);
    }
    
    // 데이터 검증 실패
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(MethodArgumentNotValidException e) {
        log.warn("Validation failed: {}", e.getMessage());
        
        Map<String, String> fieldErrors = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage,
                (existing, replacement) -> existing
            ));
        
        ErrorResponse errorResponse = ErrorResponse.builder()
            .code("VALIDATION_FAILED")
            .message("입력값 검증에 실패했습니다")
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
            
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }
    
    // 시스템 예외 처리
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception e) {
        log.error("Unexpected error occurred", e);
        
        ErrorResponse errorResponse = ErrorResponse.builder()
            .code("INTERNAL_SERVER_ERROR")
            .message("서버 내부 오류가 발생했습니다")
            .timestamp(LocalDateTime.now())
            .build();
            
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}
```

## 2. 커스텀 Exception 클래스
### 비즈니스 예외 체계
```java
// 기본 비즈니스 예외
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;
    
    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
    
    public BusinessException(ErrorCode errorCode, String customMessage) {
        super(customMessage);
        this.errorCode = errorCode;
    }
    
    public BusinessException(ErrorCode errorCode, Throwable cause) {
        super(errorCode.getMessage(), cause);
        this.errorCode = errorCode;
    }
}

// 구체적인 비즈니스 예외들
public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(Long userId) {
        super(ErrorCode.USER_NOT_FOUND, "사용자를 찾을 수 없습니다. ID: " + userId);
    }
}

public class DuplicateEmailException extends BusinessException {
    public DuplicateEmailException(String email) {
        super(ErrorCode.DUPLICATE_EMAIL, "이미 존재하는 이메일입니다: " + email);
    }
}

public class InsufficientBalanceException extends BusinessException {
    public InsufficientBalanceException(BigDecimal balance, BigDecimal amount) {
        super(ErrorCode.INSUFFICIENT_BALANCE, 
            String.format("잔액이 부족합니다. 현재 잔액: %s, 요청 금액: %s", balance, amount));
    }
}
```

## 3. ErrorCode Enum
### 에러 코드 표준화
```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    // 사용자 관련 에러
    USER_NOT_FOUND("U001", "사용자를 찾을 수 없습니다", HttpStatus.NOT_FOUND),
    DUPLICATE_EMAIL("U002", "이미 존재하는 이메일입니다", HttpStatus.CONFLICT),
    INVALID_PASSWORD("U003", "비밀번호가 올바르지 않습니다", HttpStatus.BAD_REQUEST),
    
    // 주문 관련 에러
    ORDER_NOT_FOUND("O001", "주문을 찾을 수 없습니다", HttpStatus.NOT_FOUND),
    ORDER_ALREADY_CANCELLED("O002", "이미 취소된 주문입니다", HttpStatus.BAD_REQUEST),
    INSUFFICIENT_BALANCE("O003", "잔액이 부족합니다", HttpStatus.BAD_REQUEST),
    
    // 상품 관련 에러
    PRODUCT_NOT_FOUND("P001", "상품을 찾을 수 없습니다", HttpStatus.NOT_FOUND),
    OUT_OF_STOCK("P002", "재고가 부족합니다", HttpStatus.BAD_REQUEST),
    
    // 시스템 에러
    INTERNAL_SERVER_ERROR("S001", "서버 내부 오류", HttpStatus.INTERNAL_SERVER_ERROR),
    EXTERNAL_API_ERROR("S002", "외부 API 호출 실패", HttpStatus.BAD_GATEWAY);
    
    private final String code;
    private final String message;
    private final HttpStatus httpStatus;
}
```

## 4. ErrorResponse DTO
### 표준화된 에러 응답
```java
@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    private String code;
    private String message;
    private Map<String, String> fieldErrors;
    private String path;
    private LocalDateTime timestamp;
    
    // 단순 에러 응답
    public static ErrorResponse of(ErrorCode errorCode) {
        return ErrorResponse.builder()
            .code(errorCode.getCode())
            .message(errorCode.getMessage())
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    // 필드 에러가 있는 응답
    public static ErrorResponse of(ErrorCode errorCode, Map<String, String> fieldErrors) {
        return ErrorResponse.builder()
            .code(errorCode.getCode())
            .message(errorCode.getMessage())
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

## 5. 서비스 계층에서의 예외 처리
### 비즈니스 로직에서의 예외 발생
```java
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public UserResponse createUser(CreateUserRequest request) {
        // 이메일 중복 검사
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }
        
        try {
            User user = User.builder()
                .name(request.getName())
                .email(request.getEmail())
                .password(passwordEncoder.encode(request.getPassword()))
                .build();
                
            User savedUser = userRepository.save(user);
            return UserResponse.from(savedUser);
            
        } catch (DataIntegrityViolationException e) {
            log.error("Database constraint violation", e);
            throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);
        }
    }
    
    public UserResponse getUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
            
        return UserResponse.from(user);
    }
    
    public void deleteUser(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
            
        if (user.hasActiveOrders()) {
            throw new BusinessException(ErrorCode.USER_HAS_ACTIVE_ORDERS);
        }
        
        userRepository.delete(user);
    }
}
```

## 6. 트랜잭션과 예외 처리
### @Transactional과 롤백 전략
```java
@Service
@Transactional
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final ProductService productService;
    private final PaymentService paymentService;
    
    // 체크드 예외는 롤백되지 않으므로 런타임 예외로 변환
    @Transactional(rollbackFor = Exception.class)
    public OrderResponse createOrder(CreateOrderRequest request) {
        try {
            // 1. 상품 재고 확인
            Product product = productService.getProduct(request.getProductId());
            if (product.getStock() < request.getQuantity()) {
                throw new InsufficientStockException(product.getId(), request.getQuantity());
            }
            
            // 2. 주문 생성
            Order order = Order.builder()
                .productId(request.getProductId())
                .quantity(request.getQuantity())
                .totalAmount(product.getPrice().multiply(BigDecimal.valueOf(request.getQuantity())))
                .status(OrderStatus.PENDING)
                .build();
                
            Order savedOrder = orderRepository.save(order);
            
            // 3. 재고 차감
            productService.decreaseStock(request.getProductId(), request.getQuantity());
            
            // 4. 결제 처리
            PaymentResult paymentResult = paymentService.processPayment(
                PaymentRequest.of(savedOrder.getTotalAmount(), request.getPaymentMethod())
            );
            
            if (!paymentResult.isSuccess()) {
                throw new PaymentFailedException(paymentResult.getErrorMessage());
            }
            
            // 5. 주문 완료 처리
            savedOrder.complete(paymentResult.getTransactionId());
            
            return OrderResponse.from(savedOrder);
            
        } catch (ExternalApiException e) {
            // 외부 API 호출 실패 시 비즈니스 예외로 변환
            throw new BusinessException(ErrorCode.EXTERNAL_API_ERROR, e);
        }
    }
}
```

## 7. 비동기 처리에서의 예외 처리
### @Async 메서드의 예외 처리
```java
@Service
@RequiredArgsConstructor
public class NotificationService {
    
    @Async
    @Retryable(value = {Exception.class}, maxAttempts = 3, backoff = @Backoff(delay = 1000))
    public CompletableFuture<Void> sendEmailAsync(String email, String subject, String content) {
        try {
            emailService.send(email, subject, content);
            log.info("Email sent successfully to: {}", email);
            return CompletableFuture.completedFuture(null);
            
        } catch (Exception e) {
            log.error("Failed to send email to: {}", email, e);
            throw new EmailSendException("이메일 발송에 실패했습니다", e);
        }
    }
    
    @Recover
    public CompletableFuture<Void> recover(EmailSendException e, String email, String subject, String content) {
        log.error("All retry attempts failed for email: {}", email);
        // 실패한 이메일을 데드 레터 큐에 저장하거나 다른 처리
        deadLetterService.saveFailedEmail(email, subject, content, e.getMessage());
        return CompletableFuture.completedFuture(null);
    }
}
```

## 관련 패턴
- [[Enterprise Design Patterns]]
- [[Logging Patterns]]
- [[Validation Patterns]]
- [[API Response Patterns]]

#exception-handling #error-handling #global-exception-handler #business-exception
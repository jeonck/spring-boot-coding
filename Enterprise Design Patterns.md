# Enterprise Design Patterns

## 개념
- 상용 프로젝트에서 검증된 설계 패턴들
- [[Spring Boot 3.5 개요]]와 [[Spring IoC와 DI]]를 활용한 실무 패턴
- 확장성과 유지보수성을 고려한 아키텍처 패턴

## 1. 계층형 아키텍처 (Layered Architecture)
### 표준 3계층 구조
```java
// Presentation Layer (Controller)
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {
    private final UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable @Positive Long id) {
        UserResponse user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
}

// Business Layer (Service)
@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final AuditService auditService;
    
    public UserResponse getUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        auditService.logUserAccess(id);
        return UserResponse.from(user);
    }
}

// Data Access Layer (Repository)
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmailAndActiveTrue(String email);
    
    @Query("SELECT u FROM User u WHERE u.lastLoginDate < :date")
    List<User> findInactiveUsers(@Param("date") LocalDateTime date);
}
```

## 2. DTO 패턴 (Data Transfer Object)
### Request/Response 분리
```java
// 요청 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateUserRequest {
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 50, message = "이름은 2-50자 사이여야 합니다")
    private String name;
    
    @Email(message = "유효한 이메일 형식이어야 합니다")
    @NotBlank(message = "이메일은 필수입니다")
    private String email;
    
    @Pattern(regexp = "^\\d{3}-\\d{4}-\\d{4}$", message = "전화번호 형식이 올바르지 않습니다")
    private String phoneNumber;
}

// 응답 DTO
@Data
@Builder
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private String phoneNumber;
    private LocalDateTime createdAt;
    private boolean active;
    
    public static UserResponse from(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .phoneNumber(user.getPhoneNumber())
            .createdAt(user.getCreatedAt())
            .active(user.isActive())
            .build();
    }
}

// 내부 DTO (계층 간 데이터 전달)
@Data
@Builder
public class UserSearchCriteria {
    private String name;
    private String email;
    private Boolean active;
    private LocalDate createdAfter;
    private LocalDate createdBefore;
}
```

## 3. Repository 패턴 확장
### Custom Repository 구현
```java
public interface UserRepositoryCustom {
    Page<User> findByCriteria(UserSearchCriteria criteria, Pageable pageable);
    List<UserStatistics> getUserStatisticsByDepartment();
}

@Repository
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Page<User> findByCriteria(UserSearchCriteria criteria, Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getName() != null) {
            predicates.add(cb.like(cb.lower(root.get("name")), 
                "%" + criteria.getName().toLowerCase() + "%"));
        }
        
        if (criteria.getActive() != null) {
            predicates.add(cb.equal(root.get("active"), criteria.getActive()));
        }
        
        query.where(predicates.toArray(new Predicate[0]));
        
        TypedQuery<User> typedQuery = entityManager.createQuery(query);
        typedQuery.setFirstResult((int) pageable.getOffset());
        typedQuery.setMaxResults(pageable.getPageSize());
        
        List<User> users = typedQuery.getResultList();
        long total = countByCriteria(criteria);
        
        return new PageImpl<>(users, pageable, total);
    }
}
```

## 4. Factory 패턴
### 객체 생성 전략화
```java
@Component
public class NotificationFactory {
    private final Map<NotificationType, NotificationProvider> providers;
    
    public NotificationFactory(List<NotificationProvider> providerList) {
        this.providers = providerList.stream()
            .collect(Collectors.toMap(
                NotificationProvider::getType,
                Function.identity()
            ));
    }
    
    public NotificationProvider getProvider(NotificationType type) {
        NotificationProvider provider = providers.get(type);
        if (provider == null) {
            throw new UnsupportedNotificationTypeException(type);
        }
        return provider;
    }
}

// 구현체들
@Service
public class EmailNotificationProvider implements NotificationProvider {
    @Override
    public NotificationType getType() {
        return NotificationType.EMAIL;
    }
    
    @Override
    public void send(NotificationMessage message) {
        // 이메일 발송 로직
    }
}

@Service
public class SmsNotificationProvider implements NotificationProvider {
    @Override
    public NotificationType getType() {
        return NotificationType.SMS;
    }
    
    @Override
    public void send(NotificationMessage message) {
        // SMS 발송 로직
    }
}
```

## 5. Strategy 패턴
### 비즈니스 로직 전략화
```java
@Service
public class PaymentService {
    private final Map<PaymentMethod, PaymentStrategy> strategies;
    
    public PaymentService(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                PaymentStrategy::getPaymentMethod,
                Function.identity()
            ));
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        PaymentStrategy strategy = strategies.get(request.getPaymentMethod());
        if (strategy == null) {
            throw new UnsupportedPaymentMethodException(request.getPaymentMethod());
        }
        
        return strategy.process(request);
    }
}

// 전략 구현체들
@Service
public class CreditCardPaymentStrategy implements PaymentStrategy {
    @Override
    public PaymentMethod getPaymentMethod() {
        return PaymentMethod.CREDIT_CARD;
    }
    
    @Override
    public PaymentResult process(PaymentRequest request) {
        // 신용카드 결제 로직
        return PaymentResult.success(request.getAmount());
    }
}
```

## 6. Template Method 패턴
### 공통 프로세스 템플릿화
```java
@Service
public abstract class BaseImportService<T> {
    
    @Transactional
    public ImportResult importData(MultipartFile file) {
        try {
            // 1. 파일 검증
            validateFile(file);
            
            // 2. 데이터 파싱
            List<T> data = parseFile(file);
            
            // 3. 데이터 검증
            List<T> validData = validateData(data);
            
            // 4. 데이터 저장
            saveData(validData);
            
            // 5. 후처리
            postProcess(validData);
            
            return ImportResult.success(validData.size());
            
        } catch (Exception e) {
            return ImportResult.failure(e.getMessage());
        }
    }
    
    // Template methods - 하위 클래스에서 구현
    protected abstract List<T> parseFile(MultipartFile file);
    protected abstract List<T> validateData(List<T> data);
    protected abstract void saveData(List<T> data);
    
    // Hook methods - 선택적 구현
    protected void postProcess(List<T> data) {
        // 기본 구현 (아무것도 하지 않음)
    }
    
    // 공통 메서드
    private void validateFile(MultipartFile file) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("파일이 비어있습니다");
        }
        if (file.getSize() > 10 * 1024 * 1024) { // 10MB
            throw new IllegalArgumentException("파일 크기가 너무 큽니다");
        }
    }
}

// 구체적인 구현
@Service
public class UserImportService extends BaseImportService<User> {
    private final UserRepository userRepository;
    
    @Override
    protected List<User> parseFile(MultipartFile file) {
        // CSV 파싱 로직
    }
    
    @Override
    protected List<User> validateData(List<User> users) {
        // 사용자 데이터 검증 로직
    }
    
    @Override
    protected void saveData(List<User> users) {
        userRepository.saveAll(users);
    }
    
    @Override
    protected void postProcess(List<User> users) {
        // 사용자별 웰컴 이메일 발송
        users.forEach(user -> emailService.sendWelcomeEmail(user));
    }
}
```

## 관련 패턴
- [[Service Layer Pattern]]
- [[Repository Pattern]]
- [[DTO Pattern]]
- [[Factory Pattern]]
- [[Strategy Pattern]]
- [[Template Method Pattern]]
- [[Facade Pattern]]

#enterprise-patterns #design-patterns #layered-architecture #dto-pattern
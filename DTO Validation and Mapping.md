# DTO Validation and Mapping

## 개념
- DTO의 데이터 검증과 객체 간 매핑 전문 패턴
- [[DTO and VO Patterns]]의 핵심 구현 기술
- [[Request Response DTO Patterns]]과 연동된 실무 검증 전략
- 타입 안전성과 성능을 보장하는 매핑 기법

## 1. 고급 검증 패턴
### 커스텀 검증 어노테이션
```java
// 커스텀 검증 어노테이션 정의
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidPasswordValidator.class)
@Documented
public @interface ValidPassword {
    
    String message() default "비밀번호가 보안 요구사항을 충족하지 않습니다";
    
    Class<?>[] groups() default {};
    
    Class<? extends Payload>[] payload() default {};
    
    // 커스텀 속성들
    int minLength() default 8;
    boolean requireUppercase() default true;
    boolean requireLowercase() default true;
    boolean requireDigits() default true;
    boolean requireSpecialChars() default true;
    String[] forbiddenPatterns() default {"123456", "password", "qwerty"};
}

// 검증 로직 구현
public class ValidPasswordValidator implements ConstraintValidator<ValidPassword, String> {
    
    private ValidPassword annotation;
    
    @Override
    public void initialize(ValidPassword constraintAnnotation) {
        this.annotation = constraintAnnotation;
    }
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) {
            return false;
        }
        
        List<String> violations = new ArrayList<>();
        
        // 길이 검증
        if (password.length() < annotation.minLength()) {
            violations.add(String.format("최소 %d자 이상이어야 합니다", annotation.minLength()));
        }
        
        // 대문자 검증
        if (annotation.requireUppercase() && !password.matches(".*[A-Z].*")) {
            violations.add("대문자를 포함해야 합니다");
        }
        
        // 소문자 검증
        if (annotation.requireLowercase() && !password.matches(".*[a-z].*")) {
            violations.add("소문자를 포함해야 합니다");
        }
        
        // 숫자 검증
        if (annotation.requireDigits() && !password.matches(".*\\d.*")) {
            violations.add("숫자를 포함해야 합니다");
        }
        
        // 특수문자 검증
        if (annotation.requireSpecialChars() && !password.matches(".*[!@#$%^&*()_+\\-=\\[\\]{};':\"\\\\|,.<>\\/?].*")) {
            violations.add("특수문자를 포함해야 합니다");
        }
        
        // 금지된 패턴 검증
        for (String forbidden : annotation.forbiddenPatterns()) {
            if (password.toLowerCase().contains(forbidden.toLowerCase())) {
                violations.add(String.format("'%s'는 사용할 수 없습니다", forbidden));
            }
        }
        
        if (!violations.isEmpty()) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(String.join(", ", violations))
                   .addConstraintViolation();
            return false;
        }
        
        return true;
    }
}
```

### 조건부 검증 패턴
```java
// 조건부 검증 어노테이션
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ConditionalValidationValidator.class)
@Documented
public @interface ConditionalValidation {
    
    String message() default "조건부 검증 실패";
    
    Class<?>[] groups() default {};
    
    Class<? extends Payload>[] payload() default {};
    
    // 조건과 검증 규칙들
    ConditionalRule[] rules();
}

@Target({})
@Retention(RetentionPolicy.RUNTIME)
public @interface ConditionalRule {
    String condition();  // SpEL 표현식
    String field();
    String[] validations();
}

// 사용 예시
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ConditionalValidation(rules = {
    @ConditionalRule(
        condition = "userType == T(com.example.UserType).PREMIUM",
        field = "phoneNumber",
        validations = {"@NotBlank", "@Pattern(regexp='^\\d{3}-\\d{4}-\\d{4}$')"}
    ),
    @ConditionalRule(
        condition = "age != null && age < 18",
        field = "parentEmail",
        validations = {"@NotBlank", "@Email"}
    )
})
public class UserRegistrationDto {
    
    @NotBlank(message = "이름은 필수입니다")
    private String name;
    
    @Email(message = "유효한 이메일이어야 합니다")
    private String email;
    
    private UserType userType;
    
    private Integer age;
    
    // 조건부로 검증되는 필드들
    private String phoneNumber;
    private String parentEmail;
    
    // 복잡한 비즈니스 규칙 검증
    @AssertTrue(message = "프리미엄 사용자는 성인이어야 합니다")
    private boolean isPremiumUserValid() {
        if (userType == UserType.PREMIUM) {
            return age != null && age >= 18;
        }
        return true;
    }
}
```

### 그룹 기반 검증
```java
// 검증 그룹 인터페이스들
public interface CreateGroup {}
public interface UpdateGroup {}
public interface AdminGroup {}
public interface UserGroup {}

// 계층적 검증 그룹
@GroupSequence({Default.class, CreateGroup.class, AdminGroup.class})
public interface CreateAdminSequence {}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDto {
    
    // ID는 생성 시에는 null, 수정 시에는 필수
    @Null(groups = CreateGroup.class, message = "생성 시 ID는 제공하지 마세요")
    @NotNull(groups = UpdateGroup.class, message = "수정 시 ID는 필수입니다")
    private Long id;
    
    @NotBlank(groups = {CreateGroup.class, UpdateGroup.class}, message = "이름은 필수입니다")
    @Size(min = 2, max = 50, message = "이름은 2-50자 사이여야 합니다")
    private String name;
    
    @Email(groups = {CreateGroup.class, UpdateGroup.class}, message = "유효한 이메일이어야 합니다")
    private String email;
    
    // 관리자만 설정 가능한 필드
    @NotNull(groups = AdminGroup.class, message = "관리자는 역할을 지정해야 합니다")
    private UserRole role;
    
    // 사용자는 설정할 수 없는 필드
    @Null(groups = UserGroup.class, message = "사용자는 상태를 변경할 수 없습니다")
    private UserStatus status;
    
    // 생성 시에만 필요한 필드
    @NotBlank(groups = CreateGroup.class, message = "생성 시 비밀번호는 필수입니다")
    @ValidPassword(groups = CreateGroup.class)
    private String password;
    
    // 중첩 객체의 그룹 검증
    @Valid
    @NotNull(groups = CreateGroup.class)
    private AddressDto address;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class AddressDto {
        
        @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
        private String street;
        
        @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
        private String city;
        
        @Pattern(regexp = "^\\d{5}$", groups = {CreateGroup.class, UpdateGroup.class})
        private String postalCode;
    }
}

// 컨트롤러에서 그룹별 검증 사용
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Validated(CreateGroup.class) @RequestBody UserDto request) {
        // 생성 로직
        return ResponseEntity.ok().build();
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Validated(UpdateGroup.class) @RequestBody UserDto request) {
        // 수정 로직
        return ResponseEntity.ok().build();
    }
    
    @PostMapping("/admin")
    public ResponseEntity<UserResponse> createUserAsAdmin(
            @Validated(CreateAdminSequence.class) @RequestBody UserDto request) {
        // 관리자 생성 로직
        return ResponseEntity.ok().build();
    }
}
```

## 2. MapStruct 고급 매핑 패턴
### 복잡한 매핑 전략
```java
@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE,
    nullValueCheckStrategy = NullValueCheckStrategy.ALWAYS,
    unmappedTargetPolicy = ReportingPolicy.WARN,
    uses = {DateMapper.class, AddressMapper.class}
)
public interface UserMapper {
    
    // 기본 매핑
    @Mapping(source = "firstName", target = "name", qualifiedByName = "formatFullName")
    @Mapping(source = "lastName", target = "name", qualifiedByName = "formatFullName")
    @Mapping(source = "birthDate", target = "age", qualifiedByName = "calculateAge")
    @Mapping(source = "address", target = "addressDto")
    @Mapping(target = "createdAt", expression = "java(java.time.LocalDateTime.now())")
    @Mapping(target = "id", ignore = true)
    UserDto toDto(User entity);
    
    // 역방향 매핑
    @Mapping(source = "name", target = "firstName", qualifiedByName = "extractFirstName")
    @Mapping(source = "name", target = "lastName", qualifiedByName = "extractLastName")
    @Mapping(source = "age", target = "birthDate", qualifiedByName = "calculateBirthDate")
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    User toEntity(UserDto dto);
    
    // 리스트 매핑
    List<UserDto> toDtoList(List<User> entities);
    List<User> toEntityList(List<UserDto> dtos);
    
    // 페이지 매핑
    default Page<UserDto> toDtoPage(Page<User> entityPage) {
        return entityPage.map(this::toDto);
    }
    
    // 부분 업데이트 매핑
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    void updateEntityFromDto(UserDto dto, @MappingTarget User entity);
    
    // 커스텀 매핑 메서드들
    @Named("formatFullName")
    default String formatFullName(String firstName, String lastName) {
        if (firstName == null && lastName == null) {
            return null;
        }
        return Stream.of(firstName, lastName)
                .filter(Objects::nonNull)
                .collect(Collectors.joining(" "));
    }
    
    @Named("extractFirstName")
    default String extractFirstName(String fullName) {
        if (fullName == null || fullName.trim().isEmpty()) {
            return null;
        }
        String[] parts = fullName.trim().split("\\s+");
        return parts[0];
    }
    
    @Named("extractLastName")
    default String extractLastName(String fullName) {
        if (fullName == null || fullName.trim().isEmpty()) {
            return null;
        }
        String[] parts = fullName.trim().split("\\s+");
        return parts.length > 1 ? parts[parts.length - 1] : null;
    }
    
    @Named("calculateAge")
    default Integer calculateAge(LocalDate birthDate) {
        if (birthDate == null) {
            return null;
        }
        return Period.between(birthDate, LocalDate.now()).getYears();
    }
    
    @Named("calculateBirthDate")
    default LocalDate calculateBirthDate(Integer age) {
        if (age == null) {
            return null;
        }
        return LocalDate.now().minusYears(age);
    }
    
    // 조건부 매핑
    @Mapping(source = "email", target = "maskedEmail", qualifiedByName = "maskEmail")
    @Mapping(source = "phoneNumber", target = "maskedPhone", qualifiedByName = "maskPhone")
    UserSensitiveDto toSensitiveDto(User entity);
    
    @Named("maskEmail")
    default String maskEmail(String email) {
        if (email == null || !email.contains("@")) {
            return email;
        }
        String[] parts = email.split("@");
        String localPart = parts[0];
        String domain = parts[1];
        
        if (localPart.length() <= 2) {
            return email;
        }
        
        String masked = localPart.substring(0, 2) + "*".repeat(localPart.length() - 2);
        return masked + "@" + domain;
    }
    
    @Named("maskPhone")
    default String maskPhone(String phone) {
        if (phone == null || phone.length() < 8) {
            return phone;
        }
        return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
    }
}

// 날짜 매핑 전용 유틸리티
@Mapper(componentModel = "spring")
public interface DateMapper {
    
    @Named("timestampToLocalDateTime")
    default LocalDateTime timestampToLocalDateTime(Long timestamp) {
        return timestamp != null ? 
            LocalDateTime.ofEpochSecond(timestamp, 0, ZoneOffset.UTC) : null;
    }
    
    @Named("localDateTimeToTimestamp")
    default Long localDateTimeToTimestamp(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.toEpochSecond(ZoneOffset.UTC) : null;
    }
    
    @Named("dateToString")
    default String dateToString(LocalDate date) {
        return date != null ? date.format(DateTimeFormatter.ISO_LOCAL_DATE) : null;
    }
    
    @Named("stringToDate")
    default LocalDate stringToDate(String dateStr) {
        return dateStr != null ? LocalDate.parse(dateStr, DateTimeFormatter.ISO_LOCAL_DATE) : null;
    }
}
```

### 상속 기반 매핑
```java
// 기본 매퍼 인터페이스
public interface BaseMapper<E, D> {
    
    D toDto(E entity);
    E toEntity(D dto);
    List<D> toDtoList(List<E> entities);
    List<E> toEntityList(List<D> dtos);
    
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntityFromDto(D dto, @MappingTarget E entity);
}

// 구체적인 매퍼 구현
@Mapper(componentModel = "spring")
public interface ProductMapper extends BaseMapper<Product, ProductDto> {
    
    @Override
    @Mapping(source = "category.name", target = "categoryName")
    @Mapping(source = "price", target = "formattedPrice", qualifiedByName = "formatCurrency")
    @Mapping(source = "reviews", target = "averageRating", qualifiedByName = "calculateAverageRating")
    ProductDto toDto(Product entity);
    
    @Override
    @Mapping(target = "reviews", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Product toEntity(ProductDto dto);
    
    @Named("formatCurrency")
    default String formatCurrency(BigDecimal price) {
        if (price == null) {
            return null;
        }
        NumberFormat formatter = NumberFormat.getCurrencyInstance(Locale.KOREA);
        return formatter.format(price);
    }
    
    @Named("calculateAverageRating")
    default Double calculateAverageRating(List<Review> reviews) {
        if (reviews == null || reviews.isEmpty()) {
            return null;
        }
        return reviews.stream()
                .mapToDouble(Review::getRating)
                .average()
                .orElse(0.0);
    }
}
```

## 3. 컴포지트 매핑 패턴
### 여러 엔티티를 하나의 DTO로 매핑
```java
@Mapper(componentModel = "spring")
public interface OrderSummaryMapper {
    
    // 여러 엔티티를 하나의 DTO로 결합
    @Mapping(source = "order.id", target = "orderId")
    @Mapping(source = "order.orderDate", target = "orderDate")
    @Mapping(source = "order.status", target = "status")
    @Mapping(source = "user.name", target = "customerName")
    @Mapping(source = "user.email", target = "customerEmail")
    @Mapping(source = "orderItems", target = "items")
    @Mapping(source = "payment.amount", target = "totalAmount")
    @Mapping(source = "payment.method", target = "paymentMethod")
    OrderSummaryDto toSummaryDto(
        Order order, 
        User user, 
        List<OrderItem> orderItems, 
        Payment payment
    );
    
    // 컬렉션 매핑
    default List<OrderItemDto> mapOrderItems(List<OrderItem> orderItems) {
        if (orderItems == null) {
            return null;
        }
        return orderItems.stream()
                .map(this::mapOrderItem)
                .collect(Collectors.toList());
    }
    
    @Mapping(source = "product.name", target = "productName")
    @Mapping(source = "product.sku", target = "productSku")
    @Mapping(expression = "java(item.getQuantity() * item.getUnitPrice())", target = "totalPrice")
    OrderItemDto mapOrderItem(OrderItem item);
}

// 사용 예시 서비스
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderSummaryService {
    
    private final OrderRepository orderRepository;
    private final OrderSummaryMapper orderSummaryMapper;
    
    public OrderSummaryDto getOrderSummary(Long orderId) {
        // 연관된 엔티티들을 조회
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        User user = order.getUser();
        List<OrderItem> orderItems = order.getOrderItems();
        Payment payment = order.getPayment();
        
        // 여러 엔티티를 하나의 DTO로 매핑
        return orderSummaryMapper.toSummaryDto(order, user, orderItems, payment);
    }
}
```

## 4. 동적 DTO 변환
### 런타임 조건에 따른 다른 DTO 변환
```java
@Component
@RequiredArgsConstructor
public class DynamicUserMapper {
    
    private final UserMapper userMapper;
    private final UserSecurityMapper securityMapper;
    
    public UserDto mapUser(User user, UserContext context) {
        // 컨텍스트에 따라 다른 매핑 전략 사용
        return switch (context.getLevel()) {
            case PUBLIC -> mapPublicUser(user);
            case PRIVATE -> mapPrivateUser(user, context);
            case ADMIN -> mapAdminUser(user);
            case SYSTEM -> mapSystemUser(user);
        };
    }
    
    private UserDto mapPublicUser(User user) {
        UserDto dto = userMapper.toDto(user);
        // 민감한 정보 제거
        dto.setEmail(maskEmail(dto.getEmail()));
        dto.setPhoneNumber(null);
        dto.setAddress(null);
        return dto;
    }
    
    private UserDto mapPrivateUser(User user, UserContext context) {
        UserDto dto = userMapper.toDto(user);
        
        // 자신의 정보가 아닌 경우 일부 정보 마스킹
        if (!Objects.equals(user.getId(), context.getCurrentUserId())) {
            dto.setPhoneNumber(maskPhone(dto.getPhoneNumber()));
            dto.setAddress(maskAddress(dto.getAddress()));
        }
        
        return dto;
    }
    
    private UserDto mapAdminUser(User user) {
        UserDto dto = userMapper.toDto(user);
        // 관리자는 모든 정보 접근 가능
        dto.setInternalNotes(user.getInternalNotes());
        dto.setAuditLog(user.getAuditLog());
        return dto;
    }
    
    private UserDto mapSystemUser(User user) {
        UserDto dto = userMapper.toDto(user);
        // 시스템 레벨에서는 암호화된 정보도 포함
        dto.setEncryptedData(user.getEncryptedData());
        dto.setSystemMetadata(user.getSystemMetadata());
        return dto;
    }
    
    // 마스킹 유틸리티 메서드들...
}

// 컨텍스트 정보
@Data
@Builder
public class UserContext {
    private Long currentUserId;
    private UserRole currentUserRole;
    private AccessLevel level;
    private Set<String> permissions;
    
    public enum AccessLevel {
        PUBLIC, PRIVATE, ADMIN, SYSTEM
    }
}
```

## 5. 성능 최적화 매핑
### 지연 로딩과 배치 매핑
```java
@Component
@RequiredArgsConstructor
public class OptimizedUserMapper {
    
    private final EntityManager entityManager;
    
    // 배치 조회를 통한 N+1 문제 해결
    public List<UserDto> mapUsersWithBatchLoading(List<User> users) {
        if (users.isEmpty()) {
            return Collections.emptyList();
        }
        
        // 연관된 엔티티들을 배치로 미리 로딩
        preloadAssociations(users);
        
        return users.stream()
                .map(this::mapUserWithLoadedAssociations)
                .collect(Collectors.toList());
    }
    
    private void preloadAssociations(List<User> users) {
        Set<Long> userIds = users.stream()
                .map(User::getId)
                .collect(Collectors.toSet());
        
        // 주소 정보 배치 로딩
        entityManager.createQuery(
                "SELECT a FROM Address a WHERE a.user.id IN :userIds", Address.class)
                .setParameter("userIds", userIds)
                .getResultList();
        
        // 프로필 정보 배치 로딩
        entityManager.createQuery(
                "SELECT p FROM Profile p WHERE p.user.id IN :userIds", Profile.class)
                .setParameter("userIds", userIds)
                .getResultList();
    }
    
    private UserDto mapUserWithLoadedAssociations(User user) {
        return UserDto.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .address(mapAddress(user.getAddress()))  // 이미 로딩됨
                .profile(mapProfile(user.getProfile()))  // 이미 로딩됨
                .build();
    }
    
    // 지연 로딩을 활용한 조건부 매핑
    public UserDto mapUserLazily(User user, Set<String> fields) {
        UserDto.UserDtoBuilder builder = UserDto.builder()
                .id(user.getId())
                .name(user.getName());
        
        if (fields.contains("email")) {
            builder.email(user.getEmail());
        }
        
        if (fields.contains("address")) {
            builder.address(mapAddress(user.getAddress()));
        }
        
        if (fields.contains("profile")) {
            builder.profile(mapProfile(user.getProfile()));
        }
        
        if (fields.contains("orders")) {
            builder.orderCount(user.getOrders().size());  // 지연 로딩
        }
        
        return builder.build();
    }
}
```

## 6. 캐시 기반 매핑
### 매핑 결과 캐싱으로 성능 향상
```java
@Component
@RequiredArgsConstructor
public class CachedUserMapper {
    
    private final UserMapper userMapper;
    private final RedisTemplate<String, Object> redisTemplate;
    
    private static final String CACHE_PREFIX = "user:dto:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(30);
    
    @Cacheable(value = "userDtos", key = "#user.id + ':' + #context.level")
    public UserDto mapUserWithCache(User user, UserContext context) {
        return userMapper.toDto(user);
    }
    
    // 배치 캐싱
    public List<UserDto> mapUsersWithBatchCache(List<User> users, UserContext context) {
        Map<Long, UserDto> cachedDtos = getCachedDtos(users, context);
        List<User> uncachedUsers = users.stream()
                .filter(user -> !cachedDtos.containsKey(user.getId()))
                .collect(Collectors.toList());
        
        // 캐시되지 않은 사용자들만 매핑
        List<UserDto> newDtos = uncachedUsers.stream()
                .map(user -> userMapper.toDto(user))
                .collect(Collectors.toList());
        
        // 새로 매핑된 DTO들을 캐시에 저장
        cacheNewDtos(newDtos, context);
        
        // 캐시된 것과 새로 매핑된 것을 합쳐서 반환
        List<UserDto> result = new ArrayList<>(cachedDtos.values());
        result.addAll(newDtos);
        
        return result;
    }
    
    private Map<Long, UserDto> getCachedDtos(List<User> users, UserContext context) {
        List<String> cacheKeys = users.stream()
                .map(user -> createCacheKey(user.getId(), context))
                .collect(Collectors.toList());
        
        List<Object> cachedValues = redisTemplate.opsForValue().multiGet(cacheKeys);
        
        Map<Long, UserDto> cachedDtos = new HashMap<>();
        for (int i = 0; i < users.size(); i++) {
            Object cachedValue = cachedValues.get(i);
            if (cachedValue instanceof UserDto dto) {
                cachedDtos.put(users.get(i).getId(), dto);
            }
        }
        
        return cachedDtos;
    }
    
    private void cacheNewDtos(List<UserDto> dtos, UserContext context) {
        Map<String, Object> cacheMap = dtos.stream()
                .collect(Collectors.toMap(
                    dto -> createCacheKey(dto.getId(), context),
                    Function.identity()
                ));
        
        redisTemplate.opsForValue().multiSet(cacheMap);
        
        // TTL 설정
        cacheMap.keySet().forEach(key -> 
            redisTemplate.expire(key, CACHE_TTL));
    }
    
    private String createCacheKey(Long userId, UserContext context) {
        return CACHE_PREFIX + userId + ":" + context.getLevel();
    }
    
    @CacheEvict(value = "userDtos", key = "#userId + '*'")
    public void evictUserCache(Long userId) {
        // 사용자 정보 변경 시 캐시 무효화
    }
}
```

## 7. 스트리밍 매핑
### 대용량 데이터 처리를 위한 스트리밍 매핑
```java
@Component
@RequiredArgsConstructor
public class StreamingUserMapper {
    
    private final UserMapper userMapper;
    
    // 스트리밍 매핑 (메모리 효율적)
    public Stream<UserDto> mapUsersStream(Stream<User> userStream) {
        return userStream
                .map(userMapper::toDto)
                .onClose(() -> log.info("User stream mapping completed"));
    }
    
    // 병렬 스트리밍 매핑
    public Stream<UserDto> mapUsersParallel(Stream<User> userStream) {
        return userStream
                .parallel()
                .map(user -> {
                    try {
                        return userMapper.toDto(user);
                    } catch (Exception e) {
                        log.error("Error mapping user {}: {}", user.getId(), e.getMessage());
                        return null;
                    }
                })
                .filter(Objects::nonNull);
    }
    
    // 청크 단위 처리
    public Stream<List<UserDto>> mapUsersInChunks(Stream<User> userStream, int chunkSize) {
        AtomicInteger counter = new AtomicInteger();
        Collection<List<User>> chunks = userStream
                .collect(Collectors.groupingBy(
                    user -> counter.getAndIncrement() / chunkSize))
                .values();
        
        return chunks.stream()
                .map(chunk -> chunk.stream()
                        .map(userMapper::toDto)
                        .collect(Collectors.toList()));
    }
    
    // 비동기 매핑
    @Async("mappingTaskExecutor")
    public CompletableFuture<List<UserDto>> mapUsersAsync(List<User> users) {
        return CompletableFuture.supplyAsync(() -> 
            users.stream()
                    .map(userMapper::toDto)
                    .collect(Collectors.toList())
        );
    }
}

// 비동기 설정
@Configuration
@EnableAsync
public class AsyncMappingConfig {
    
    @Bean("mappingTaskExecutor")
    public TaskExecutor mappingTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("mapping-");
        executor.initialize();
        return executor;
    }
}
```

## 관련 패턴
- [[DTO and VO Patterns]]
- [[Request Response DTO Patterns]]
- [[Enterprise Design Patterns]]
- [[Exception Handling Patterns]]
- [[Spring Bean Management]]
- [[Java 21]]

#dto-validation #dto-mapping #mapstruct #bean-validation #performance

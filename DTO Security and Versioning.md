# DTO Security and Versioning

## 개념
- DTO의 보안과 버전 관리 전문 패턴
- [[DTO and VO Patterns]]의 고급 운영 기법
- [[Security Patterns]]과 통합된 데이터 보호 전략
- API 진화와 하위 호환성을 보장하는 버전 관리

## 1. 데이터 마스킹 및 필터링 패턴
### 필드 레벨 보안 어노테이션
```java
// 보안 어노테이션 정의
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Sensitive {
    
    SensitivityLevel level() default SensitivityLevel.CONFIDENTIAL;
    
    MaskingStrategy strategy() default MaskingStrategy.PARTIAL;
    
    String[] allowedRoles() default {};
    
    String customMask() default "***";
    
    enum SensitivityLevel {
        PUBLIC,      // 누구나 볼 수 있음
        INTERNAL,    // 내부 사용자만
        CONFIDENTIAL, // 권한이 있는 사용자만
        RESTRICTED   // 시스템 관리자만
    }
    
    enum MaskingStrategy {
        FULL,        // 완전 마스킹
        PARTIAL,     // 부분 마스킹
        HASH,        // 해시값으로 대체
        REMOVE       // 필드 제거
    }
}

// 보안 적용된 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class SecureUserDto {
    
    private Long id;
    
    private String name;
    
    @Sensitive(level = SensitivityLevel.INTERNAL, strategy = MaskingStrategy.PARTIAL)
    @JsonProperty("email")
    private String email;
    
    @Sensitive(level = SensitivityLevel.CONFIDENTIAL, strategy = MaskingStrategy.PARTIAL)
    @JsonProperty("phone_number")
    private String phoneNumber;
    
    @Sensitive(level = SensitivityLevel.RESTRICTED, strategy = MaskingStrategy.HASH)
    @JsonProperty("ssn")
    private String socialSecurityNumber;
    
    @Sensitive(level = SensitivityLevel.CONFIDENTIAL, strategy = MaskingStrategy.REMOVE)
    @JsonProperty("credit_card")
    private String creditCardNumber;
    
    @Sensitive(level = SensitivityLevel.INTERNAL)
    @JsonProperty("address")
    private AddressDto address;
    
    // 접근 권한에 따른 동적 필드
    @JsonIgnore
    private Set<String> allowedFields = new HashSet<>();
    
    // 보안 컨텍스트
    @JsonIgnore
    private SecurityContext securityContext;
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class AddressDto {
        
        private String street;
        
        @Sensitive(level = SensitivityLevel.INTERNAL, strategy = MaskingStrategy.PARTIAL)
        private String detailAddress;
        
        private String city;
        
        private String postalCode;
    }
}
```

### 동적 보안 필터링
```java
@Component
@RequiredArgsConstructor
public class SecurityAwareMapper {
    
    private final SecurityContextService securityContextService;
    
    public <T> T applySecurity(T dto, Class<T> dtoClass) {
        SecurityContext context = securityContextService.getCurrentContext();
        
        try {
            T securedDto = (T) dtoClass.getDeclaredConstructor().newInstance();
            Field[] fields = dtoClass.getDeclaredFields();
            
            for (Field field : fields) {
                field.setAccessible(true);
                Object value = field.get(dto);
                
                if (value == null) {
                    continue;
                }
                
                Sensitive sensitiveAnnotation = field.getAnnotation(Sensitive.class);
                if (sensitiveAnnotation != null) {
                    value = processSecretiveField(value, sensitiveAnnotation, context);
                }
                
                field.set(securedDto, value);
            }
            
            return securedDto;
            
        } catch (Exception e) {
            throw new SecurityProcessingException("Failed to apply security", e);
        }
    }
    
    private Object processSecretiveField(Object value, Sensitive annotation, SecurityContext context) {
        // 권한 검사
        if (!hasPermission(annotation, context)) {
            return applyMaskingStrategy(value, annotation);
        }
        
        return value;
    }
    
    private boolean hasPermission(Sensitive annotation, SecurityContext context) {
        SensitivityLevel requiredLevel = annotation.level();
        SensitivityLevel userLevel = context.getSensitivityLevel();
        
        // 레벨 기반 권한 확인
        if (userLevel.ordinal() < requiredLevel.ordinal()) {
            return false;
        }
        
        // 역할 기반 권한 확인
        String[] allowedRoles = annotation.allowedRoles();
        if (allowedRoles.length > 0) {
            return Arrays.stream(allowedRoles)
                    .anyMatch(role -> context.hasRole(role));
        }
        
        return true;
    }
    
    private Object applyMaskingStrategy(Object value, Sensitive annotation) {
        if (!(value instanceof String str)) {
            return annotation.strategy() == MaskingStrategy.REMOVE ? null : value;
        }
        
        return switch (annotation.strategy()) {
            case FULL -> annotation.customMask();
            case PARTIAL -> maskPartially(str);
            case HASH -> hashValue(str);
            case REMOVE -> null;
        };
    }
    
    private String maskPartially(String value) {
        if (value == null || value.length() < 3) {
            return "***";
        }
        
        // 이메일 마스킹
        if (value.contains("@")) {
            return maskEmail(value);
        }
        
        // 전화번호 마스킹
        if (value.matches("^\\d{3}-\\d{4}-\\d{4}$")) {
            return maskPhone(value);
        }
        
        // 일반 문자열 마스킹
        int visibleChars = Math.min(2, value.length() / 3);
        return value.substring(0, visibleChars) + 
               "*".repeat(value.length() - visibleChars * 2) + 
               value.substring(value.length() - visibleChars);
    }
    
    private String maskEmail(String email) {
        String[] parts = email.split("@");
        if (parts.length != 2) {
            return "***@***.com";
        }
        
        String localPart = parts[0];
        String domain = parts[1];
        
        String maskedLocal = localPart.length() > 2 ? 
            localPart.substring(0, 2) + "*".repeat(localPart.length() - 2) : 
            "**";
            
        return maskedLocal + "@" + domain;
    }
    
    private String maskPhone(String phone) {
        return phone.substring(0, 3) + "-****-" + phone.substring(phone.length() - 4);
    }
    
    private String hashValue(String value) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(value.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash).substring(0, 8) + "...";
        } catch (NoSuchAlgorithmException e) {
            return "hashed_value";
        }
    }
}
```

## 2. 역할 기반 DTO 변환
### 권한별 다른 DTO 반환
```java
@Component
@RequiredArgsConstructor
public class RoleBasedDtoMapper {
    
    private final UserMapper userMapper;
    private final SecurityContextService securityContextService;
    
    public UserDto mapUserByRole(User user) {
        SecurityContext context = securityContextService.getCurrentContext();
        
        return switch (context.getHighestRole()) {
            case ADMIN -> mapAdminUser(user, context);
            case MANAGER -> mapManagerUser(user, context);
            case USER -> mapRegularUser(user, context);
            case GUEST -> mapGuestUser(user);
        };
    }
    
    private UserDto mapAdminUser(User user, SecurityContext context) {
        AdminUserDto dto = AdminUserDto.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .phoneNumber(user.getPhoneNumber())
            .socialSecurityNumber(user.getSocialSecurityNumber())
            .creditCardNumber(user.getCreditCardNumber())
            .address(mapFullAddress(user.getAddress()))
            .internalNotes(user.getInternalNotes())
            .auditLog(user.getAuditLog())
            .systemMetadata(user.getSystemMetadata())
            .createdAt(user.getCreatedAt())
            .updatedAt(user.getUpdatedAt())
            .lastLoginAt(user.getLastLoginAt())
            .build();
            
        return dto;
    }
    
    private UserDto mapManagerUser(User user, SecurityContext context) {
        ManagerUserDto dto = ManagerUserDto.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .phoneNumber(maskPhone(user.getPhoneNumber()))
            .address(mapPartialAddress(user.getAddress()))
            .department(user.getDepartment())
            .role(user.getRole())
            .status(user.getStatus())
            .createdAt(user.getCreatedAt())
            .lastLoginAt(user.getLastLoginAt())
            .build();
            
        return dto;
    }
    
    private UserDto mapRegularUser(User user, SecurityContext context) {
        // 자신의 정보인지 확인
        boolean isOwnData = Objects.equals(user.getId(), context.getCurrentUserId());
        
        UserDto dto = UserDto.builder()
            .id(user.getId())
            .name(user.getName())
            .email(isOwnData ? user.getEmail() : maskEmail(user.getEmail()))
            .phoneNumber(isOwnData ? user.getPhoneNumber() : null)
            .address(isOwnData ? mapBasicAddress(user.getAddress()) : null)
            .profileImageUrl(user.getProfileImageUrl())
            .status(user.getStatus())
            .createdAt(user.getCreatedAt())
            .build();
            
        return dto;
    }
    
    private UserDto mapGuestUser(User user) {
        return GuestUserDto.builder()
            .id(user.getId())
            .name(user.getName())
            .profileImageUrl(user.getProfileImageUrl())
            .build();
    }
}

// 역할별 전용 DTO 클래스들
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AdminUserDto extends UserDto {
    private String socialSecurityNumber;
    private String creditCardNumber;
    private String internalNotes;
    private List<AuditLogEntry> auditLog;
    private Map<String, Object> systemMetadata;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ManagerUserDto extends UserDto {
    private String department;
    private UserRole role;
    private UserStatus status;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class GuestUserDto extends UserDto {
    // 최소한의 필드만
}
```

## 3. API 버전 관리 패턴
### 스키마 진화와 호환성 관리
```java
// 버전 정보 어노테이션
@Target({ElementType.TYPE, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ApiVersion {
    String since() default "1.0";
    String until() default "";
    boolean deprecated() default false;
    String deprecationMessage() default "";
    String replacedBy() default "";
}

// 기본 버전의 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ApiVersion(since = "1.0")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserDtoV1 {
    
    @ApiVersion(since = "1.0")
    private Long id;
    
    @ApiVersion(since = "1.0")
    private String name;
    
    @ApiVersion(since = "1.0")
    private String email;
    
    @ApiVersion(since = "1.0")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdAt;
}

// 확장된 버전의 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ApiVersion(since = "2.0")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserDtoV2 {
    
    @ApiVersion(since = "1.0")
    private Long id;
    
    @ApiVersion(since = "1.0")
    private String name;
    
    @ApiVersion(since = "1.0")
    private String email;
    
    @ApiVersion(since = "2.0")
    private String phoneNumber;
    
    @ApiVersion(since = "2.0")
    private UserStatus status;
    
    @ApiVersion(since = "2.0")
    private UserRole role;
    
    @ApiVersion(since = "1.0")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  // v2에서 포맷 변경
    private LocalDateTime createdAt;
    
    @ApiVersion(since = "2.0")
    private LocalDateTime updatedAt;
    
    // 하위 호환성을 위한 변환 메서드
    public UserDtoV1 toV1() {
        return UserDtoV1.builder()
            .id(this.id)
            .name(this.name)
            .email(this.email)
            .createdAt(this.createdAt != null ? this.createdAt.toLocalDate() : null)
            .build();
    }
    
    // v1에서 v2로 업그레이드
    public static UserDtoV2 fromV1(UserDtoV1 v1) {
        return UserDtoV2.builder()
            .id(v1.getId())
            .name(v1.getName())
            .email(v1.getEmail())
            .createdAt(v1.getCreatedAt() != null ? 
                v1.getCreatedAt().atStartOfDay() : null)
            .status(UserStatus.ACTIVE)  // 기본값
            .role(UserRole.USER)        // 기본값
            .build();
    }
}

// 최신 버전 (호환성 문제가 있는 변경)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ApiVersion(since = "3.0")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserDtoV3 {
    
    @ApiVersion(since = "1.0")
    private Long id;
    
    // v3에서 firstName, lastName으로 분리
    @ApiVersion(since = "3.0")
    private String firstName;
    
    @ApiVersion(since = "3.0")
    private String lastName;
    
    @ApiVersion(since = "1.0", until = "3.0", deprecated = true, 
               deprecationMessage = "Use firstName and lastName instead")
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String name;
    
    @ApiVersion(since = "1.0")
    private String email;
    
    @ApiVersion(since = "2.0")
    private String phoneNumber;
    
    @ApiVersion(since = "2.0")
    private UserStatus status;
    
    @ApiVersion(since = "2.0")
    private UserRole role;
    
    @ApiVersion(since = "3.0")
    private AddressDto address;
    
    @ApiVersion(since = "1.0")
    private LocalDateTime createdAt;
    
    @ApiVersion(since = "2.0")
    private LocalDateTime updatedAt;
    
    // 하위 호환성 처리
    @JsonIgnore
    public String getName() {
        if (firstName == null && lastName == null) {
            return name;
        }
        return Stream.of(firstName, lastName)
                .filter(Objects::nonNull)
                .collect(Collectors.joining(" "));
    }
    
    public void setName(String name) {
        this.name = name;
        if (name != null && !name.trim().isEmpty()) {
            String[] parts = name.trim().split("\\s+", 2);
            this.firstName = parts[0];
            this.lastName = parts.length > 1 ? parts[1] : null;
        }
    }
}
```

### 동적 버전 변환기
```java
@Component
@RequiredArgsConstructor
public class ApiVersionConverter {
    
    private final ObjectMapper objectMapper;
    
    public <T> T convertToVersion(Object source, Class<T> targetClass, String targetVersion) {
        try {
            // 소스를 JSON으로 변환
            String json = objectMapper.writeValueAsString(source);
            JsonNode jsonNode = objectMapper.readTree(json);
            
            // 버전별 필드 필터링
            JsonNode filteredNode = filterFieldsByVersion(jsonNode, targetClass, targetVersion);
            
            // 대상 클래스로 변환
            return objectMapper.treeToValue(filteredNode, targetClass);
            
        } catch (Exception e) {
            throw new VersionConversionException("Failed to convert to version " + targetVersion, e);
        }
    }
    
    private JsonNode filterFieldsByVersion(JsonNode sourceNode, Class<?> targetClass, String targetVersion) {
        ObjectNode filteredNode = objectMapper.createObjectNode();
        
        Field[] fields = targetClass.getDeclaredFields();
        for (Field field : fields) {
            ApiVersion apiVersion = field.getAnnotation(ApiVersion.class);
            if (apiVersion != null && isFieldAvailableInVersion(apiVersion, targetVersion)) {
                String fieldName = getJsonPropertyName(field);
                JsonNode fieldValue = sourceNode.get(fieldName);
                if (fieldValue != null) {
                    filteredNode.set(fieldName, fieldValue);
                }
            }
        }
        
        return filteredNode;
    }
    
    private boolean isFieldAvailableInVersion(ApiVersion apiVersion, String targetVersion) {
        // 버전 비교 로직
        if (compareVersions(targetVersion, apiVersion.since()) < 0) {
            return false;  // 아직 도입되지 않은 필드
        }
        
        if (!apiVersion.until().isEmpty() && 
            compareVersions(targetVersion, apiVersion.until()) >= 0) {
            return false;  // 이미 제거된 필드
        }
        
        return true;
    }
    
    private int compareVersions(String version1, String version2) {
        String[] v1Parts = version1.split("\\.");
        String[] v2Parts = version2.split("\\.");
        
        int maxLength = Math.max(v1Parts.length, v2Parts.length);
        
        for (int i = 0; i < maxLength; i++) {
            int v1Part = i < v1Parts.length ? Integer.parseInt(v1Parts[i]) : 0;
            int v2Part = i < v2Parts.length ? Integer.parseInt(v2Parts[i]) : 0;
            
            if (v1Part != v2Part) {
                return Integer.compare(v1Part, v2Part);
            }
        }
        
        return 0;
    }
    
    private String getJsonPropertyName(Field field) {
        JsonProperty jsonProperty = field.getAnnotation(JsonProperty.class);
        return jsonProperty != null ? jsonProperty.value() : field.getName();
    }
}
```

## 4. 스키마 검증 및 마이그레이션
### 런타임 스키마 검증
```java
@Component
@RequiredArgsConstructor
public class DtoSchemaValidator {
    
    private final SchemaRepository schemaRepository;
    
    public ValidationResult validateDto(Object dto, String schemaVersion) {
        try {
            Schema schema = schemaRepository.getSchema(dto.getClass(), schemaVersion);
            return validateAgainstSchema(dto, schema);
            
        } catch (Exception e) {
            return ValidationResult.failure("Schema validation failed: " + e.getMessage());
        }
    }
    
    private ValidationResult validateAgainstSchema(Object dto, Schema schema) {
        List<ValidationError> errors = new ArrayList<>();
        
        // 필수 필드 검증
        validateRequiredFields(dto, schema, errors);
        
        // 타입 검증
        validateFieldTypes(dto, schema, errors);
        
        // 제약 조건 검증
        validateConstraints(dto, schema, errors);
        
        // 비즈니스 규칙 검증
        validateBusinessRules(dto, schema, errors);
        
        return errors.isEmpty() ? 
            ValidationResult.success() : 
            ValidationResult.failure(errors);
    }
    
    private void validateRequiredFields(Object dto, Schema schema, List<ValidationError> errors) {
        for (FieldSchema fieldSchema : schema.getRequiredFields()) {
            try {
                Field field = dto.getClass().getDeclaredField(fieldSchema.getName());
                field.setAccessible(true);
                Object value = field.get(dto);
                
                if (value == null) {
                    errors.add(new ValidationError(
                        fieldSchema.getName(), 
                        "Required field is missing"
                    ));
                }
            } catch (Exception e) {
                errors.add(new ValidationError(
                    fieldSchema.getName(), 
                    "Field access error: " + e.getMessage()
                ));
            }
        }
    }
    
    // 자동 스키마 마이그레이션
    public <T> T migrateDto(Object sourceDto, Class<T> targetClass, String fromVersion, String toVersion) {
        MigrationPath path = findMigrationPath(fromVersion, toVersion);
        
        Object currentDto = sourceDto;
        for (MigrationStep step : path.getSteps()) {
            currentDto = applyMigrationStep(currentDto, step);
        }
        
        return (T) currentDto;
    }
    
    private Object applyMigrationStep(Object dto, MigrationStep step) {
        return switch (step.getType()) {
            case FIELD_RENAME -> renameField(dto, step);
            case FIELD_SPLIT -> splitField(dto, step);
            case FIELD_MERGE -> mergeFields(dto, step);
            case FIELD_TYPE_CHANGE -> changeFieldType(dto, step);
            case FIELD_ADD -> addField(dto, step);
            case FIELD_REMOVE -> removeField(dto, step);
            case CUSTOM_TRANSFORMATION -> applyCustomTransformation(dto, step);
        };
    }
}

// 스키마 정의 클래스들
@Data
@Builder
public class Schema {
    private String version;
    private Class<?> dtoClass;
    private List<FieldSchema> requiredFields;
    private List<FieldSchema> optionalFields;
    private List<BusinessRule> businessRules;
    private Map<String, Object> metadata;
}

@Data
@Builder
public class FieldSchema {
    private String name;
    private Class<?> type;
    private List<Constraint> constraints;
    private Object defaultValue;
    private String description;
}

@Data
@Builder
public class MigrationStep {
    private MigrationType type;
    private String sourceField;
    private String targetField;
    private Object parameters;
    private Function<Object, Object> transformer;
    
    public enum MigrationType {
        FIELD_RENAME,
        FIELD_SPLIT,
        FIELD_MERGE,
        FIELD_TYPE_CHANGE,
        FIELD_ADD,
        FIELD_REMOVE,
        CUSTOM_TRANSFORMATION
    }
}
```

## 5. 암호화된 DTO 패턴
### 민감한 데이터의 암호화 처리
```java
// 암호화 어노테이션
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Encrypted {
    EncryptionAlgorithm algorithm() default EncryptionAlgorithm.AES_256;
    boolean storeDecrypted() default false;
    String keyReference() default "default";
    
    enum EncryptionAlgorithm {
        AES_256, RSA_2048, CHACHA20_POLY1305
    }
}

// 암호화된 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EncryptedUserDto {
    
    private Long id;
    
    private String name;
    
    @Encrypted(algorithm = EncryptionAlgorithm.AES_256)
    private String email;
    
    @Encrypted(algorithm = EncryptionAlgorithm.AES_256)
    private String phoneNumber;
    
    @Encrypted(algorithm = EncryptionAlgorithm.AES_256, keyReference = "pii")
    private String socialSecurityNumber;
    
    @Encrypted(algorithm = EncryptionAlgorithm.RSA_2048, keyReference = "financial")
    private String creditCardNumber;
    
    // 암호화 메타데이터
    @JsonIgnore
    private Map<String, EncryptionMetadata> encryptionMetadata = new HashMap<>();
    
    @Data
    @Builder
    public static class EncryptionMetadata {
        private String algorithm;
        private String keyId;
        private String salt;
        private LocalDateTime encryptedAt;
    }
}

// 암호화 프로세서
@Component
@RequiredArgsConstructor
public class DtoEncryptionProcessor {
    
    private final EncryptionService encryptionService;
    
    public <T> T encrypt(T dto) {
        try {
            Field[] fields = dto.getClass().getDeclaredFields();
            
            for (Field field : fields) {
                if (field.isAnnotationPresent(Encrypted.class)) {
                    field.setAccessible(true);
                    Object value = field.get(dto);
                    
                    if (value instanceof String plainText && plainText != null) {
                        Encrypted annotation = field.getAnnotation(Encrypted.class);
                        EncryptionResult result = encryptionService.encrypt(
                            plainText, 
                            annotation.algorithm(), 
                            annotation.keyReference()
                        );
                        
                        field.set(dto, result.getEncryptedData());
                        
                        // 메타데이터 저장
                        storeEncryptionMetadata(dto, field.getName(), result);
                    }
                }
            }
            
            return dto;
            
        } catch (Exception e) {
            throw new EncryptionException("Failed to encrypt DTO", e);
        }
    }
    
    public <T> T decrypt(T dto) {
        try {
            Field[] fields = dto.getClass().getDeclaredFields();
            
            for (Field field : fields) {
                if (field.isAnnotationPresent(Encrypted.class)) {
                    field.setAccessible(true);
                    Object value = field.get(dto);
                    
                    if (value instanceof String encryptedText && encryptedText != null) {
                        Encrypted annotation = field.getAnnotation(Encrypted.class);
                        EncryptionMetadata metadata = getEncryptionMetadata(dto, field.getName());
                        
                        String decryptedText = encryptionService.decrypt(
                            encryptedText,
                            annotation.algorithm(),
                            annotation.keyReference(),
                            metadata
                        );
                        
                        field.set(dto, decryptedText);
                    }
                }
            }
            
            return dto;
            
        } catch (Exception e) {
            throw new DecryptionException("Failed to decrypt DTO", e);
        }
    }
    
    private void storeEncryptionMetadata(Object dto, String fieldName, EncryptionResult result) {
        try {
            Field metadataField = dto.getClass().getDeclaredField("encryptionMetadata");
            metadataField.setAccessible(true);
            Map<String, EncryptedUserDto.EncryptionMetadata> metadata = 
                (Map<String, EncryptedUserDto.EncryptionMetadata>) metadataField.get(dto);
            
            if (metadata == null) {
                metadata = new HashMap<>();
                metadataField.set(dto, metadata);
            }
            
            metadata.put(fieldName, EncryptedUserDto.EncryptionMetadata.builder()
                .algorithm(result.getAlgorithm())
                .keyId(result.getKeyId())
                .salt(result.getSalt())
                .encryptedAt(LocalDateTime.now())
                .build());
                
        } catch (Exception e) {
            // 메타데이터 저장 실패는 로그만 남기고 계속 진행
            log.warn("Failed to store encryption metadata for field: {}", fieldName, e);
        }
    }
}
```

## 6. 감사 로그가 포함된 DTO
### 데이터 변경 추적
```java
// 감사 가능한 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuditableUserDto {
    
    private Long id;
    
    @Auditable
    private String name;
    
    @Auditable
    private String email;
    
    @Auditable(sensitive = true)
    private String phoneNumber;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // 감사 로그
    @JsonProperty("audit_trail")
    private List<FieldChangeLog> auditTrail = new ArrayList<>();
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class FieldChangeLog {
        private String fieldName;
        private Object oldValue;
        private Object newValue;
        private String changedBy;
        private LocalDateTime changedAt;
        private String changeReason;
        private String clientIp;
    }
}

// 감사 어노테이션
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Auditable {
    boolean sensitive() default false;
    boolean trackOldValue() default true;
    String[] excludeFromLog() default {};
}

// 감사 로그 프로세서
@Component
@RequiredArgsConstructor
public class DtoAuditProcessor {
    
    private final AuditContextService auditContextService;
    
    public <T> T trackChanges(T oldDto, T newDto) {
        if (oldDto == null) {
            return newDto;  // 새로 생성된 경우
        }
        
        try {
            Field[] fields = newDto.getClass().getDeclaredFields();
            List<AuditableUserDto.FieldChangeLog> changes = new ArrayList<>();
            
            for (Field field : fields) {
                if (field.isAnnotationPresent(Auditable.class)) {
                    field.setAccessible(true);
                    
                    Object oldValue = field.get(oldDto);
                    Object newValue = field.get(newDto);
                    
                    if (!Objects.equals(oldValue, newValue)) {
                        Auditable annotation = field.getAnnotation(Auditable.class);
                        AuditableUserDto.FieldChangeLog changeLog = createChangeLog(
                            field.getName(), 
                            oldValue, 
                            newValue, 
                            annotation
                        );
                        changes.add(changeLog);
                    }
                }
            }
            
            // 감사 로그를 DTO에 추가
            addAuditTrail(newDto, changes);
            
            return newDto;
            
        } catch (Exception e) {
            throw new AuditException("Failed to track changes", e);
        }
    }
    
    private AuditableUserDto.FieldChangeLog createChangeLog(
            String fieldName, 
            Object oldValue, 
            Object newValue, 
            Auditable annotation) {
        
        AuditContext context = auditContextService.getCurrentContext();
        
        return AuditableUserDto.FieldChangeLog.builder()
            .fieldName(fieldName)
            .oldValue(annotation.sensitive() ? maskValue(oldValue) : oldValue)
            .newValue(annotation.sensitive() ? maskValue(newValue) : newValue)
            .changedBy(context.getUserId())
            .changedAt(LocalDateTime.now())
            .changeReason(context.getChangeReason())
            .clientIp(context.getClientIp())
            .build();
    }
    
    private Object maskValue(Object value) {
        if (value instanceof String str) {
            return str.length() > 3 ? str.substring(0, 3) + "***" : "***";
        }
        return "***";
    }
    
    private void addAuditTrail(Object dto, List<AuditableUserDto.FieldChangeLog> changes) {
        try {
            Field auditTrailField = dto.getClass().getDeclaredField("auditTrail");
            auditTrailField.setAccessible(true);
            List<AuditableUserDto.FieldChangeLog> existingTrail = 
                (List<AuditableUserDto.FieldChangeLog>) auditTrailField.get(dto);
            
            if (existingTrail == null) {
                existingTrail = new ArrayList<>();
                auditTrailField.set(dto, existingTrail);
            }
            
            existingTrail.addAll(changes);
            
        } catch (Exception e) {
            log.warn("Failed to add audit trail", e);
        }
    }
}
```

## 7. 컨트롤러에서의 보안 적용
### 보안과 버전 관리가 통합된 API
```java
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
@Validated
public class SecureVersionedUserController {
    
    private final UserService userService;
    private final SecurityAwareMapper securityMapper;
    private final ApiVersionConverter versionConverter;
    private final DtoEncryptionProcessor encryptionProcessor;
    
    // 버전별 API 엔드포인트
    @GetMapping("/v1/users/{id}")
    @PreAuthorize("hasPermission(#id, 'User', 'READ')")
    public ResponseEntity<UserDtoV1> getUserV1(@PathVariable Long id) {
        User user = userService.getUserById(id);
        UserDtoV2 fullDto = securityMapper.mapUserByRole(user);
        UserDtoV1 v1Dto = versionConverter.convertToVersion(fullDto, UserDtoV1.class, "1.0");
        
        return ResponseEntity.ok(v1Dto);
    }
    
    @GetMapping("/v2/users/{id}")
    @PreAuthorize("hasPermission(#id, 'User', 'READ')")
    public ResponseEntity<UserDtoV2> getUserV2(@PathVariable Long id) {
        User user = userService.getUserById(id);
        UserDtoV2 dto = securityMapper.mapUserByRole(user);
        
        return ResponseEntity.ok(dto);
    }
    
    @GetMapping("/v3/users/{id}")
    @PreAuthorize("hasPermission(#id, 'User', 'READ')")
    public ResponseEntity<UserDtoV3> getUserV3(@PathVariable Long id) {
        User user = userService.getUserById(id);
        UserDtoV3 dto = versionConverter.convertToVersion(user, UserDtoV3.class, "3.0");
        
        return ResponseEntity.ok(dto);
    }
    
    // 암호화된 데이터 API
    @GetMapping("/secure/users/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<EncryptedUserDto> getEncryptedUser(@PathVariable Long id) {
        User user = userService.getUserById(id);
        EncryptedUserDto dto = convertToEncryptedDto(user);
        EncryptedUserDto encryptedDto = encryptionProcessor.encrypt(dto);
        
        return ResponseEntity.ok(encryptedDto);
    }
    
    // 헤더 기반 버전 협상
    @GetMapping("/users/{id}")
    @PreAuthorize("hasPermission(#id, 'User', 'READ')")
    public ResponseEntity<?> getUser(
            @PathVariable Long id,
            @RequestHeader(value = "Accept-Version", defaultValue = "2.0") String version) {
        
        User user = userService.getUserById(id);
        
        return switch (version) {
            case "1.0" -> {
                UserDtoV1 dto = versionConverter.convertToVersion(user, UserDtoV1.class, "1.0");
                yield ResponseEntity.ok(dto);
            }
            case "2.0" -> {
                UserDtoV2 dto = securityMapper.mapUserByRole(user);
                yield ResponseEntity.ok(dto);
            }
            case "3.0" -> {
                UserDtoV3 dto = versionConverter.convertToVersion(user, UserDtoV3.class, "3.0");
                yield ResponseEntity.ok(dto);
            }
            default -> ResponseEntity.badRequest()
                    .body(ErrorResponse.of("UNSUPPORTED_VERSION", "Unsupported API version: " + version));
        };
    }
}
```

## 관련 패턴
- [[DTO and VO Patterns]]
- [[Request Response DTO Patterns]]
- [[DTO Validation and Mapping]]
- [[Security Patterns]]
- [[API Design Patterns]]
- [[Exception Handling Patterns]]

#dto-security #api-versioning #data-masking #encryption #audit-logging

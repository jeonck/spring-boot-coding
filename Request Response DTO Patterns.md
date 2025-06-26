# Request Response DTO Patterns

## 개념
- HTTP 요청과 응답을 위한 전용 DTO 설계 패턴
- [[DTO and VO Patterns]]의 세분화된 실무 적용
- [[API Design Patterns]]와 완전 통합된 데이터 구조
- RESTful API의 일관된 인터페이스 보장

## 1. Request DTO 패턴
### 기본 요청 DTO 구조
```java
// 사용자 생성 요청 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CreateUserRequest {
    
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 50, message = "이름은 2-50자 사이여야 합니다")
    @JsonProperty("name")
    private String name;
    
    @Email(message = "유효한 이메일 형식이어야 합니다")
    @NotBlank(message = "이메일은 필수입니다")
    @JsonProperty("email")
    private String email;
    
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
             message = "비밀번호는 최소 8자 이상, 대소문자, 숫자, 특수문자를 포함해야 합니다")
    @JsonProperty("password")
    private String password;
    
    @Pattern(regexp = "^\\d{3}-\\d{4}-\\d{4}$", message = "전화번호 형식이 올바르지 않습니다")
    @JsonProperty("phone_number")
    private String phoneNumber;
    
    @Valid
    @JsonProperty("address")
    private AddressDto address;
    
    @JsonProperty("preferences")
    private Map<String, Object> preferences;
    
    // 중첩 DTO
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class AddressDto {
        
        @NotBlank(message = "주소는 필수입니다")
        @JsonProperty("street")
        private String street;
        
        @NotBlank(message = "도시는 필수입니다")
        @JsonProperty("city")
        private String city;
        
        @Pattern(regexp = "^\\d{5}$", message = "우편번호는 5자리 숫자여야 합니다")
        @JsonProperty("postal_code")
        private String postalCode;
        
        @JsonProperty("country")
        private String country = "KR";
    }
    
    // 비즈니스 로직 검증
    @AssertTrue(message = "이메일과 전화번호 중 하나는 반드시 제공되어야 합니다")
    private boolean isContactInfoProvided() {
        return StringUtils.hasText(email) || StringUtils.hasText(phoneNumber);
    }
}
```

### 수정 요청 DTO (부분 업데이트)
```java
// 사용자 수정 요청 DTO (PATCH 방식)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UpdateUserRequest {
    
    @Size(min = 2, max = 50, message = "이름은 2-50자 사이여야 합니다")
    @JsonProperty("name")
    private String name;
    
    @Email(message = "유효한 이메일 형식이어야 합니다")
    @JsonProperty("email")
    private String email;
    
    @Pattern(regexp = "^\\d{3}-\\d{4}-\\d{4}$", message = "전화번호 형식이 올바르지 않습니다")
    @JsonProperty("phone_number")
    private String phoneNumber;
    
    @Valid
    @JsonProperty("address")
    private AddressDto address;
    
    @JsonProperty("preferences")
    private Map<String, Object> preferences;
    
    // 변경된 필드만 추적
    @JsonIgnore
    private Set<String> modifiedFields = new HashSet<>();
    
    @JsonAnySetter
    public void setDynamicProperty(String key, Object value) {
        modifiedFields.add(key);
        // 실제 필드 설정 로직
        switch (key) {
            case "name" -> this.name = (String) value;
            case "email" -> this.email = (String) value;
            case "phone_number" -> this.phoneNumber = (String) value;
            // 기타 필드들...
        }
    }
    
    public boolean hasFieldModified(String fieldName) {
        return modifiedFields.contains(fieldName);
    }
    
    // 부분 업데이트를 위한 메서드
    public void applyTo(User user) {
        if (hasFieldModified("name") && name != null) {
            user.setName(name);
        }
        if (hasFieldModified("email") && email != null) {
            user.setEmail(email);
        }
        if (hasFieldModified("phone_number") && phoneNumber != null) {
            user.setPhoneNumber(phoneNumber);
        }
        // 기타 필드들...
    }
}
```

### 검색 요청 DTO
```java
// 사용자 검색 요청 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserSearchRequest {
    
    @Size(max = 100, message = "검색어는 100자를 초과할 수 없습니다")
    @JsonProperty("keyword")
    private String keyword;
    
    @JsonProperty("name")
    private String name;
    
    @Email(message = "유효한 이메일 형식이어야 합니다")
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("status")
    private UserStatus status;
    
    @JsonProperty("role")
    private UserRole role;
    
    @JsonProperty("created_after")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdAfter;
    
    @JsonProperty("created_before")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdBefore;
    
    @JsonProperty("age_min")
    @Min(value = 0, message = "최소 나이는 0 이상이어야 합니다")
    @Max(value = 150, message = "최소 나이는 150 이하여야 합니다")
    private Integer ageMin;
    
    @JsonProperty("age_max")
    @Min(value = 0, message = "최대 나이는 0 이상이어야 합니다")
    @Max(value = 150, message = "최대 나이는 150 이하여야 합니다")
    private Integer ageMax;
    
    // 페이징 정보
    @JsonProperty("page")
    @Min(value = 0, message = "페이지 번호는 0 이상이어야 합니다")
    private Integer page = 0;
    
    @JsonProperty("size")
    @Min(value = 1, message = "페이지 크기는 1 이상이어야 합니다")
    @Max(value = 100, message = "페이지 크기는 100 이하여야 합니다")
    private Integer size = 20;
    
    @JsonProperty("sort")
    private String sort = "id";
    
    @JsonProperty("direction")
    private String direction = "asc";
    
    // Pageable 객체로 변환
    public Pageable toPageable() {
        Sort.Direction sortDirection = Sort.Direction.fromString(direction);
        return PageRequest.of(page, size, Sort.by(sortDirection, sort));
    }
    
    // 검증 메서드
    @AssertTrue(message = "최소 나이는 최대 나이보다 작거나 같아야 합니다")
    private boolean isAgeRangeValid() {
        if (ageMin != null && ageMax != null) {
            return ageMin <= ageMax;
        }
        return true;
    }
    
    @AssertTrue(message = "생성일 범위가 올바르지 않습니다")
    private boolean isDateRangeValid() {
        if (createdAfter != null && createdBefore != null) {
            return !createdAfter.isAfter(createdBefore);
        }
        return true;
    }
}
```

## 2. Response DTO 패턴
### 기본 응답 DTO 구조
```java
// 사용자 응답 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResponse {
    
    @JsonProperty("id")
    private Long id;
    
    @JsonProperty("name")
    private String name;
    
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("phone_number")
    private String phoneNumber;
    
    @JsonProperty("status")
    private UserStatus status;
    
    @JsonProperty("role")
    private UserRole role;
    
    @JsonProperty("address")
    private AddressResponse address;
    
    @JsonProperty("profile_image_url")
    private String profileImageUrl;
    
    @JsonProperty("last_login_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime lastLoginAt;
    
    @JsonProperty("created_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;
    
    @JsonProperty("updated_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime updatedAt;
    
    // 계산된 필드들
    @JsonProperty("age")
    private Integer age;
    
    @JsonProperty("is_active")
    private Boolean isActive;
    
    @JsonProperty("days_since_last_login")
    private Long daysSinceLastLogin;
    
    // HATEOAS 링크들
    @JsonProperty("_links")
    private Map<String, String> links;
    
    // 중첩 응답 DTO
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class AddressResponse {
        
        @JsonProperty("street")
        private String street;
        
        @JsonProperty("city")
        private String city;
        
        @JsonProperty("postal_code")
        private String postalCode;
        
        @JsonProperty("country")
        private String country;
        
        @JsonProperty("formatted")
        private String formatted;
    }
    
    // 팩토리 메서드
    public static UserResponse from(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .phoneNumber(user.getPhoneNumber())
            .status(user.getStatus())
            .role(user.getRole())
            .address(user.getAddress() != null ? AddressResponse.from(user.getAddress()) : null)
            .profileImageUrl(user.getProfileImageUrl())
            .lastLoginAt(user.getLastLoginAt())
            .createdAt(user.getCreatedAt())
            .updatedAt(user.getUpdatedAt())
            .age(user.calculateAge())
            .isActive(user.isActive())
            .daysSinceLastLogin(user.calculateDaysSinceLastLogin())
            .links(createLinks(user))
            .build();
    }
    
    // HATEOAS 링크 생성
    private static Map<String, String> createLinks(User user) {
        Map<String, String> links = new HashMap<>();
        links.put("self", "/api/users/" + user.getId());
        links.put("orders", "/api/users/" + user.getId() + "/orders");
        links.put("preferences", "/api/users/" + user.getId() + "/preferences");
        
        if (user.canBeEdited()) {
            links.put("edit", "/api/users/" + user.getId());
            links.put("delete", "/api/users/" + user.getId());
        }
        
        return links;
    }
}
```

### 요약 응답 DTO (목록용)
```java
// 사용자 요약 응답 DTO (목록 조회용)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserSummaryResponse {
    
    @JsonProperty("id")
    private Long id;
    
    @JsonProperty("name")
    private String name;
    
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("status")
    private UserStatus status;
    
    @JsonProperty("role")
    private UserRole role;
    
    @JsonProperty("created_at")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdAt;
    
    @JsonProperty("is_active")
    private Boolean isActive;
    
    // 경량화된 팩토리 메서드
    public static UserSummaryResponse from(User user) {
        return UserSummaryResponse.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .status(user.getStatus())
            .role(user.getRole())
            .createdAt(user.getCreatedAt().toLocalDate())
            .isActive(user.isActive())
            .build();
    }
    
    // 목록 변환
    public static List<UserSummaryResponse> fromList(List<User> users) {
        return users.stream()
            .map(UserSummaryResponse::from)
            .collect(Collectors.toList());
    }
}
```

## 3. 페이징 응답 패턴
### 표준 페이징 응답 구조
```java
// 페이징 응답 래퍼
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class PagedResponse<T> {
    
    @JsonProperty("content")
    private List<T> content;
    
    @JsonProperty("page")
    private PageInfo page;
    
    @JsonProperty("_links")
    private PageLinks links;
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class PageInfo {
        
        @JsonProperty("number")
        private int number;
        
        @JsonProperty("size")
        private int size;
        
        @JsonProperty("total_elements")
        private long totalElements;
        
        @JsonProperty("total_pages")
        private int totalPages;
        
        @JsonProperty("first")
        private boolean first;
        
        @JsonProperty("last")
        private boolean last;
        
        @JsonProperty("has_next")
        private boolean hasNext;
        
        @JsonProperty("has_previous")
        private boolean hasPrevious;
        
        @JsonProperty("number_of_elements")
        private int numberOfElements;
        
        @JsonProperty("empty")
        private boolean empty;
    }
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class PageLinks {
        
        @JsonProperty("self")
        private String self;
        
        @JsonProperty("first")
        private String first;
        
        @JsonProperty("prev")
        private String prev;
        
        @JsonProperty("next")
        private String next;
        
        @JsonProperty("last")
        private String last;
    }
    
    // Spring Page 객체에서 변환
    public static <T, R> PagedResponse<R> from(Page<T> page, Function<T, R> mapper, String baseUrl) {
        List<R> content = page.getContent().stream()
            .map(mapper)
            .collect(Collectors.toList());
        
        PageInfo pageInfo = PageInfo.builder()
            .number(page.getNumber())
            .size(page.getSize())
            .totalElements(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .first(page.isFirst())
            .last(page.isLast())
            .hasNext(page.hasNext())
            .hasPrevious(page.hasPrevious())
            .numberOfElements(page.getNumberOfElements())
            .empty(page.isEmpty())
            .build();
        
        PageLinks links = createPageLinks(page, baseUrl);
        
        return PagedResponse.<R>builder()
            .content(content)
            .page(pageInfo)
            .links(links)
            .build();
    }
    
    private static <T> PageLinks createPageLinks(Page<T> page, String baseUrl) {
        PageLinks.PageLinksBuilder builder = PageLinks.builder()
            .self(createPageUrl(baseUrl, page.getNumber(), page.getSize()))
            .first(createPageUrl(baseUrl, 0, page.getSize()))
            .last(createPageUrl(baseUrl, page.getTotalPages() - 1, page.getSize()));
        
        if (page.hasPrevious()) {
            builder.prev(createPageUrl(baseUrl, page.getNumber() - 1, page.getSize()));
        }
        
        if (page.hasNext()) {
            builder.next(createPageUrl(baseUrl, page.getNumber() + 1, page.getSize()));
        }
        
        return builder.build();
    }
    
    private static String createPageUrl(String baseUrl, int page, int size) {
        return String.format("%s?page=%d&size=%d", baseUrl, page, size);
    }
}
```

## 4. 에러 응답 패턴
### 표준 에러 응답 구조
```java
// 에러 응답 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    
    @JsonProperty("error")
    private boolean error = true;
    
    @JsonProperty("code")
    private String code;
    
    @JsonProperty("message")
    private String message;
    
    @JsonProperty("details")
    private String details;
    
    @JsonProperty("field_errors")
    private Map<String, String> fieldErrors;
    
    @JsonProperty("path")
    private String path;
    
    @JsonProperty("timestamp")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime timestamp;
    
    @JsonProperty("trace_id")
    private String traceId;
    
    // 간단한 에러 응답
    public static ErrorResponse of(String code, String message) {
        return ErrorResponse.builder()
            .code(code)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    // 검증 에러 응답
    public static ErrorResponse of(String code, String message, Map<String, String> fieldErrors) {
        return ErrorResponse.builder()
            .code(code)
            .message(message)
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    // BindingResult에서 변환
    public static ErrorResponse fromBindingResult(BindingResult bindingResult) {
        Map<String, String> fieldErrors = bindingResult.getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> error.getDefaultMessage() != null ? error.getDefaultMessage() : "Invalid value",
                (existing, replacement) -> existing
            ));
        
        return ErrorResponse.builder()
            .code("VALIDATION_FAILED")
            .message("입력값 검증에 실패했습니다")
            .fieldErrors(fieldErrors)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

## 5. 벌크 작업 DTO 패턴
### 대량 처리 요청/응답 구조
```java
// 벌크 사용자 생성 요청
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BulkCreateUsersRequest {
    
    @Valid
    @NotEmpty(message = "최소 1명의 사용자 정보가 필요합니다")
    @Size(max = 100, message = "한 번에 최대 100명까지 생성 가능합니다")
    @JsonProperty("users")
    private List<CreateUserRequest> users;
    
    @JsonProperty("skip_duplicates")
    private boolean skipDuplicates = false;
    
    @JsonProperty("rollback_on_error")
    private boolean rollbackOnError = true;
    
    @JsonProperty("dry_run")
    private boolean dryRun = false;
}

// 벌크 작업 응답
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class BulkOperationResponse<T> {
    
    @JsonProperty("total_requested")
    private int totalRequested;
    
    @JsonProperty("successful_count")
    private int successfulCount;
    
    @JsonProperty("failed_count")
    private int failedCount;
    
    @JsonProperty("skipped_count")
    private int skippedCount;
    
    @JsonProperty("successful_items")
    private List<T> successfulItems;
    
    @JsonProperty("failed_items")
    private List<FailedItem> failedItems;
    
    @JsonProperty("execution_time_ms")
    private long executionTimeMs;
    
    @JsonProperty("dry_run")
    private boolean dryRun;
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class FailedItem {
        
        @JsonProperty("index")
        private int index;
        
        @JsonProperty("item")
        private Object item;
        
        @JsonProperty("error_code")
        private String errorCode;
        
        @JsonProperty("error_message")
        private String errorMessage;
        
        @JsonProperty("field_errors")
        private Map<String, String> fieldErrors;
    }
}
```

## 6. 실시간 업데이트 DTO 패턴
### WebSocket/SSE용 실시간 데이터 구조
```java
// 실시간 이벤트 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class RealTimeEventDto {
    
    @JsonProperty("event_id")
    private String eventId;
    
    @JsonProperty("event_type")
    private EventType eventType;
    
    @JsonProperty("entity_type")
    private String entityType;
    
    @JsonProperty("entity_id")
    private String entityId;
    
    @JsonProperty("action")
    private EventAction action;
    
    @JsonProperty("data")
    private Object data;
    
    @JsonProperty("metadata")
    private Map<String, Object> metadata;
    
    @JsonProperty("timestamp")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss.SSS")
    private LocalDateTime timestamp;
    
    @JsonProperty("user_id")
    private Long userId;
    
    @JsonProperty("session_id")
    private String sessionId;
    
    public enum EventType {
        ENTITY_CREATED,
        ENTITY_UPDATED,
        ENTITY_DELETED,
        STATUS_CHANGED,
        NOTIFICATION,
        SYSTEM_EVENT
    }
    
    public enum EventAction {
        CREATE,
        UPDATE,
        DELETE,
        ACTIVATE,
        DEACTIVATE,
        NOTIFY
    }
    
    // 사용자 이벤트 생성 팩토리 메서드
    public static RealTimeEventDto userEvent(User user, EventAction action) {
        return RealTimeEventDto.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType(EventType.ENTITY_UPDATED)
            .entityType("USER")
            .entityId(user.getId().toString())
            .action(action)
            .data(UserSummaryResponse.from(user))
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

## 7. API 버전별 DTO 패턴
### 버전 호환성을 고려한 DTO 설계
```java
// API v1 사용자 응답
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResponseV1 {
    
    @JsonProperty("id")
    private Long id;
    
    @JsonProperty("name")
    private String name;
    
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("created_at")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdAt;
}

// API v2 사용자 응답 (확장된 필드)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResponseV2 {
    
    @JsonProperty("id")
    private Long id;
    
    @JsonProperty("name")
    private String name;
    
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("phone_number")  // v2에서 추가
    private String phoneNumber;
    
    @JsonProperty("status")  // v2에서 추가
    private UserStatus status;
    
    @JsonProperty("created_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  // 포맷 변경
    private LocalDateTime createdAt;
    
    @JsonProperty("updated_at")  // v2에서 추가
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime updatedAt;
    
    // v1 호환성을 위한 변환 메서드
    public UserResponseV1 toV1() {
        return UserResponseV1.builder()
            .id(this.id)
            .name(this.name)
            .email(this.email)
            .createdAt(this.createdAt.toLocalDate())
            .build();
    }
}
```

## 8. 컨트롤러에서의 활용 예시
### Request/Response DTO를 활용한 컨트롤러
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
public class UserController {
    
    private final UserService userService;
    
    // 사용자 생성
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        
        UserResponse response = userService.createUser(request);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(response.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(response);
    }
    
    // 사용자 목록 조회
    @GetMapping
    public ResponseEntity<PagedResponse<UserSummaryResponse>> getUsers(
            @Valid @ModelAttribute UserSearchRequest searchRequest) {
        
        Page<User> userPage = userService.searchUsers(searchRequest);
        PagedResponse<UserSummaryResponse> response = PagedResponse.from(
            userPage, 
            UserSummaryResponse::from,
            "/api/v1/users"
        );
        
        return ResponseEntity.ok(response);
    }
    
    // 사용자 상세 조회
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse response = userService.getUserById(id);
        return ResponseEntity.ok(response);
    }
    
    // 사용자 수정
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        
        UserResponse response = userService.updateUser(id, request);
        return ResponseEntity.ok(response);
    }
    
    // 벌크 생성
    @PostMapping("/bulk")
    public ResponseEntity<BulkOperationResponse<UserResponse>> bulkCreateUsers(
            @Valid @RequestBody BulkCreateUsersRequest request) {
        
        BulkOperationResponse<UserResponse> response = userService.bulkCreateUsers(request);
        return ResponseEntity.ok(response);
    }
}
```

## 관련 패턴
- [[DTO and VO Patterns]]
- [[API Design Patterns]]
- [[Enterprise Design Patterns]]
- [[Exception Handling Patterns]]
- [[Spring Web MVC]]
- [[Java 21]]

#request-response #dto #api-design #validation #pagination

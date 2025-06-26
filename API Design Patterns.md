# API Design Patterns

## 개념
- RESTful API 설계의 모범 사례
- [[Enterprise Design Patterns]]과 연동된 API 구조
- 일관성 있고 확장 가능한 API 인터페이스 설계

## 1. RESTful URL 설계 원칙
### 리소스 중심 URL 구조
```java
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class UserController {
    
    // 사용자 목록 조회 (페이징, 정렬, 필터링)
    @GetMapping("/users")
    public ResponseEntity<PageResponse<UserResponse>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id") String sort,
            @RequestParam(defaultValue = "asc") String direction,
            @RequestParam(required = false) String name,
            @RequestParam(required = false) Boolean active) {
        
        UserSearchCriteria criteria = UserSearchCriteria.builder()
            .name(name)
            .active(active)
            .build();
            
        Pageable pageable = PageRequest.of(page, size, 
            Sort.by(Sort.Direction.fromString(direction), sort));
            
        Page<UserResponse> users = userService.findUsers(criteria, pageable);
        return ResponseEntity.ok(PageResponse.of(users));
    }
    
    // 특정 사용자 조회
    @GetMapping("/users/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
    
    // 사용자 생성
    @PostMapping("/users")
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse user = userService.createUser(request);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(user.getId())
            .toUri();
        return ResponseEntity.created(location).body(user);
    }
    
    // 사용자 전체 수정
    @PutMapping("/users/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id, 
            @Valid @RequestBody UpdateUserRequest request) {
        UserResponse user = userService.updateUser(id, request);
        return ResponseEntity.ok(user);
    }
    
    // 사용자 부분 수정
    @PatchMapping("/users/{id}")
    public ResponseEntity<UserResponse> partialUpdateUser(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        UserResponse user = userService.partialUpdateUser(id, updates);
        return ResponseEntity.ok(user);
    }
    
    // 사용자 삭제
    @DeleteMapping("/users/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

## 2. 표준화된 응답 구조
### 일관된 API 응답 형태
```java
// 기본 응답 래퍼
@Data
@Builder
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private String timestamp;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .data(data)
            .timestamp(LocalDateTime.now().toString())
            .build();
    }
    
    public static <T> ApiResponse<T> success(T data, String message) {
        return ApiResponse.<T>builder()
            .success(true)
            .message(message)
            .data(data)
            .timestamp(LocalDateTime.now().toString())
            .build();
    }
    
    public static ApiResponse<Void> error(String message) {
        return ApiResponse.<Void>builder()
            .success(false)
            .message(message)
            .timestamp(LocalDateTime.now().toString())
            .build();
    }
}

// 페이징 응답
@Data
@Builder
public class PageResponse<T> {
    private List<T> content;
    private PageInfo pageInfo;
    
    @Data
    @Builder
    public static class PageInfo {
        private int page;
        private int size;
        private long totalElements;
        private int totalPages;
        private boolean first;
        private boolean last;
        private boolean hasNext;
        private boolean hasPrevious;
    }
    
    public static <T> PageResponse<T> of(Page<T> page) {
        return PageResponse.<T>builder()
            .content(page.getContent())
            .pageInfo(PageInfo.builder()
                .page(page.getNumber())
                .size(page.getSize())
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .first(page.isFirst())
                .last(page.isLast())
                .hasNext(page.hasNext())
                .hasPrevious(page.hasPrevious())
                .build())
            .build();
    }
}
```

## 3. 버전 관리
### API 버전 전략
```java
// URL 기반 버전 관리
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // v1 구현
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // v2 구현 (Breaking Changes)
}

// 헤더 기반 버전 관리
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(headers = "API-Version=1")
    public ResponseEntity<UserResponseV1> getUserV1(@PathVariable Long id) {
        // v1 응답 형식
    }
    
    @GetMapping(headers = "API-Version=2")
    public ResponseEntity<UserResponseV2> getUserV2(@PathVariable Long id) {
        // v2 응답 형식
    }
}

// Accept 헤더 기반
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(produces = "application/vnd.company.user.v1+json")
    public ResponseEntity<UserResponseV1> getUserV1(@PathVariable Long id) {
        // v1 응답
    }
    
    @GetMapping(produces = "application/vnd.company.user.v2+json")
    public ResponseEntity<UserResponseV2> getUserV2(@PathVariable Long id) {
        // v2 응답
    }
}
```

## 4. 검색 및 필터링 API
### 동적 쿼리 파라미터 처리
```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {
    
    private final ProductService productService;
    
    @GetMapping
    public ResponseEntity<PageResponse<ProductResponse>> searchProducts(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(required = false) Boolean inStock,
            @RequestParam(required = false) String brand,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate createdAfter,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate createdBefore,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "name") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDir) {
        
        ProductSearchCriteria criteria = ProductSearchCriteria.builder()
            .name(name)
            .category(category)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .inStock(inStock)
            .brand(brand)
            .createdAfter(createdAfter)
            .createdBefore(createdBefore)
            .build();
        
        Sort.Direction direction = Sort.Direction.fromString(sortDir);
        Pageable pageable = PageRequest.of(page, size, Sort.by(direction, sortBy));
        
        Page<ProductResponse> products = productService.searchProducts(criteria, pageable);
        return ResponseEntity.ok(PageResponse.of(products));
    }
    
    // 고급 검색 (POST 방식)
    @PostMapping("/search")
    public ResponseEntity<PageResponse<ProductResponse>> advancedSearch(
            @RequestBody ProductAdvancedSearchRequest request,
            Pageable pageable) {
        
        Page<ProductResponse> products = productService.advancedSearch(request, pageable);
        return ResponseEntity.ok(PageResponse.of(products));
    }
}
```

## 5. 파일 업로드/다운로드 패턴
### 멀티파트 파일 처리
```java
@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
public class FileController {
    
    private final FileStorageService fileStorageService;
    
    // 단일 파일 업로드
    @PostMapping("/upload")
    public ResponseEntity<FileUploadResponse> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam(required = false) String description) {
        
        validateFile(file);
        
        FileMetadata metadata = fileStorageService.storeFile(file, description);
        
        FileUploadResponse response = FileUploadResponse.builder()
            .fileId(metadata.getId())
            .fileName(metadata.getOriginalName())
            .fileSize(metadata.getSize())
            .contentType(metadata.getContentType())
            .uploadedAt(metadata.getCreatedAt())
            .downloadUrl("/api/v1/files/" + metadata.getId() + "/download")
            .build();
        
        return ResponseEntity.ok(response);
    }
    
    // 파일 다운로드
    @GetMapping("/{fileId}/download")
    public ResponseEntity<Resource> downloadFile(@PathVariable String fileId) {
        FileMetadata metadata = fileStorageService.getFileMetadata(fileId);
        Resource resource = fileStorageService.loadFileAsResource(fileId);
        
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                "attachment; filename=\"" + metadata.getOriginalName() + "\"")
            .contentType(MediaType.parseMediaType(metadata.getContentType()))
            .body(resource);
    }
}
```

## 6. 캐싱 전략
### HTTP 캐싱과 Redis 활용
```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {
    
    private final ProductService productService;
    
    // ETag 기반 조건부 요청
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(
            @PathVariable Long id,
            HttpServletRequest request) {
        
        Product product = productService.getProduct(id);
        String etag = "\"" + product.getVersion() + "\"";
        
        // If-None-Match 헤더 확인
        String ifNoneMatch = request.getHeader("If-None-Match");
        if (etag.equals(ifNoneMatch)) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build();
        }
        
        ProductResponse response = ProductResponse.from(product);
        
        return ResponseEntity.ok()
            .eTag(etag)
            .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)))
            .body(response);
    }
}
```

## 7. HATEOAS (Hypermedia as the Engine of Application State)
### 자기 설명적 API
```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {
    
    private final OrderService orderService;
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderResponseWithLinks> getOrder(@PathVariable Long id) {
        Order order = orderService.getOrder(id);
        OrderResponseWithLinks response = OrderResponseWithLinks.from(order);
        
        // Self link
        response.add(linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel());
        
        // 상태에 따른 가능한 액션 링크들
        if (order.getStatus() == OrderStatus.PENDING) {
            response.add(linkTo(methodOn(OrderController.class).confirmOrder(id)).withRel("confirm"));
            response.add(linkTo(methodOn(OrderController.class).cancelOrder(id)).withRel("cancel"));
        } else if (order.getStatus() == OrderStatus.CONFIRMED) {
            response.add(linkTo(methodOn(OrderController.class).shipOrder(id)).withRel("ship"));
            response.add(linkTo(methodOn(OrderController.class).cancelOrder(id)).withRel("cancel"));
        }
        
        // 관련 리소스 링크
        response.add(linkTo(methodOn(UserController.class).getUser(order.getUserId())).withRel("user"));
        response.add(linkTo(methodOn(ProductController.class).getProduct(order.getProductId())).withRel("product"));
        
        return ResponseEntity.ok(response);
    }
}
```

## 관련 패턴
- [[Enterprise Design Patterns]]
- [[Exception Handling Patterns]]
- [[Validation Patterns]]
- [[Security Patterns]]
- [[Caching Patterns]]

#api-design #restful #hateoas #versioning #rate-limiting
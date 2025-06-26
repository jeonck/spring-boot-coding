# Spring Boot Boilerplate Code

> 실무에서 바로 사용할 수 있는 Spring Boot 보일러플레이트 코드 템플릿

## 📋 목차

- [프로젝트 구조](#프로젝트-구조)
- [기본 설정](#기본-설정)
- [엔티티 클래스](#엔티티-클래스)
- [DTO 클래스](#dto-클래스)
- [Repository 계층](#repository-계층)
- [Service 계층](#service-계층)
- [Controller 계층](#controller-계층)
- [예외 처리](#예외-처리)
- [보안 설정](#보안-설정)
- [유틸리티 클래스](#유틸리티-클래스)
- [테스트 코드](#테스트-코드)

## 🏗 프로젝트 구조

```
src/main/java/com/example/app/
├── Application.java                    # 메인 애플리케이션 클래스
├── config/                            # 설정 클래스들
│   ├── AppConfig.java
│   ├── DatabaseConfig.java
│   ├── SecurityConfig.java
│   ├── WebConfig.java
│   └── SwaggerConfig.java
├── controller/                        # REST API 컨트롤러들
│   ├── AuthController.java
│   ├── UserController.java
│   ├── ProductController.java
│   └── OrderController.java
├── service/                          # 비즈니스 로직 서비스들
│   ├── AuthService.java
│   ├── UserService.java
│   ├── ProductService.java
│   └── OrderService.java
├── repository/                       # 데이터 액세스 레이어
│   ├── UserRepository.java
│   ├── ProductRepository.java
│   └── OrderRepository.java
├── entity/                          # JPA 엔티티들
│   ├── BaseEntity.java
│   ├── User.java
│   ├── Product.java
│   ├── Order.java
│   └── OrderItem.java
├── dto/                            # 데이터 전송 객체들
│   ├── request/
│   │   ├── LoginRequest.java
│   │   ├── RegisterRequest.java
│   │   ├── CreateUserRequest.java
│   │   ├── UpdateUserRequest.java
│   │   ├── CreateProductRequest.java
│   │   ├── UpdateProductRequest.java
│   │   ├── CreateOrderRequest.java
│   │   └── OrderItemRequest.java
│   ├── response/
│   │   ├── AuthResponse.java
│   │   ├── UserResponse.java
│   │   ├── ProductResponse.java
│   │   ├── OrderResponse.java
│   │   ├── OrderItemResponse.java
│   │   ├── PageResponse.java
│   │   └── ApiResponse.java
│   └── common/
│       ├── UserDto.java
│       ├── ProductDto.java
│       └── OrderDto.java
├── exception/                      # 예외 처리 클래스들
│   ├── GlobalExceptionHandler.java
│   ├── BusinessException.java
│   ├── ResourceNotFoundException.java
│   ├── DuplicateResourceException.java
│   └── ValidationException.java
├── security/                       # 보안 관련 클래스들
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java
│   ├── CustomUserDetails.java
│   └── CustomUserDetailsService.java
└── util/                          # 유틸리티 클래스들
    ├── DateUtils.java
    ├── StringUtils.java
    ├── ValidationUtils.java
    └── ResponseUtils.java

src/main/resources/
├── application.yml                # 애플리케이션 설정
├── application-dev.yml           # 개발 환경 설정
├── application-prod.yml          # 운영 환경 설정
├── data.sql                      # 초기 데이터
└── schema.sql                    # 데이터베이스 스키마

src/test/java/com/example/app/
├── ApplicationTests.java
├── controller/
├── service/
└── repository/
```

## ⚙️ 기본 설정

### 메인 애플리케이션 클래스

```java
package com.example.app;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableJpaAuditing
@EnableAsync
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 애플리케이션 설정

```yaml
# application.yml
spring:
  profiles:
    active: dev
  
  application:
    name: spring-boot-app
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        show_sql: false
    open-in-view: false
  
  jackson:
    property-naming-strategy: SNAKE_CASE
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false
  
  cache:
    type: redis
    redis:
      time-to-live: 600000
  
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always

logging:
  level:
    com.example.app: INFO
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

---
# application-dev.yml
spring:
  profiles: dev
  
  datasource:
    url: jdbc:postgresql://localhost:5432/appdb_dev
    username: dev_user
    password: dev_password
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  
  redis:
    host: localhost
    port: 6379
    database: 0
  
  mail:
    host: localhost
    port: 1025

jwt:
  secret: dev-secret-key-change-this-in-production
  expiration: 86400000

app:
  cors:
    allowed-origins: http://localhost:3000,http://localhost:8080
  pagination:
    default-size: 20
    max-size: 100

---
# application-prod.yml
spring:
  profiles: prod
  
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
  
  redis:
    host: ${REDIS_HOST}
    port: ${REDIS_PORT}
    password: ${REDIS_PASSWORD}

jwt:
  secret: ${JWT_SECRET}
  expiration: 86400000

app:
  cors:
    allowed-origins: ${ALLOWED_ORIGINS}
```

### 기본 설정 클래스들

```java
// config/AppConfig.java
package com.example.app.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AppConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
        return mapper;
    }
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("AsyncThread-");
        executor.initialize();
        return executor;
    }
}

// config/WebConfig.java
package com.example.app.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.Arrays;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Value("${app.cors.allowed-origins}")
    private String[] allowedOrigins;
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList(allowedOrigins));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}

// config/SwaggerConfig.java
package com.example.app.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import io.swagger.v3.oas.models.Components;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SwaggerConfig {
    
    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Spring Boot API")
                .description("Spring Boot REST API Documentation")
                .version("1.0.0")
                .contact(new Contact()
                    .name("Developer")
                    .email("developer@example.com")))
            .addSecurityItem(new SecurityRequirement()
                .addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}
```

## 📊 엔티티 클래스

### 기본 엔티티

```java
// entity/BaseEntity.java
package com.example.app.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
public abstract class BaseEntity {
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    @Version
    @Column(name = "version")
    private Long version;
}
```

### 사용자 엔티티

```java
// entity/User.java
package com.example.app.entity;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.time.LocalDateTime;
import java.util.Collection;
import java.util.List;
import java.util.Set;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User extends BaseEntity implements UserDetails {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", unique = true, nullable = false, length = 50)
    private String username;
    
    @Column(name = "email", unique = true, nullable = false, length = 100)
    private String email;
    
    @Column(name = "password", nullable = false)
    private String password;
    
    @Column(name = "first_name", length = 50)
    private String firstName;
    
    @Column(name = "last_name", length = 50)
    private String lastName;
    
    @Column(name = "phone", length = 20)
    private String phone;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "role", nullable = false)
    private UserRole role;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private UserStatus status;
    
    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;
    
    @Column(name = "email_verified", nullable = false)
    private Boolean emailVerified = false;
    
    @Column(name = "email_verification_token")
    private String emailVerificationToken;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Order> orders;
    
    // UserDetails 구현
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }
    
    @Override
    public String getUsername() {
        return username;
    }
    
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }
    
    @Override
    public boolean isAccountNonLocked() {
        return status == UserStatus.ACTIVE;
    }
    
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
    
    @Override
    public boolean isEnabled() {
        return status == UserStatus.ACTIVE && emailVerified;
    }
    
    // 비즈니스 메서드
    public String getFullName() {
        return firstName + " " + lastName;
    }
    
    public void updateLastLoginAt() {
        this.lastLoginAt = LocalDateTime.now();
    }
    
    public void verifyEmail() {
        this.emailVerified = true;
        this.emailVerificationToken = null;
    }
}

// 열거형 클래스들
enum UserRole {
    USER, ADMIN, MANAGER
}

enum UserStatus {
    ACTIVE, INACTIVE, SUSPENDED, DELETED
}
```

### 상품 엔티티

```java
// entity/Product.java
package com.example.app.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.util.Set;

@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, length = 200)
    private String name;
    
    @Column(name = "description", columnDefinition = "TEXT")
    private String description;
    
    @Column(name = "price", nullable = false, precision = 19, scale = 2)
    private BigDecimal price;
    
    @Column(name = "stock_quantity", nullable = false)
    private Integer stockQuantity;
    
    @Column(name = "sku", unique = true, length = 100)
    private String sku;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "category", nullable = false)
    private ProductCategory category;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private ProductStatus status;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    @Column(name = "weight", precision = 8, scale = 2)
    private BigDecimal weight;
    
    @Column(name = "dimensions")
    private String dimensions;
    
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<OrderItem> orderItems;
    
    // 비즈니스 메서드
    public boolean isAvailable() {
        return status == ProductStatus.ACTIVE && stockQuantity > 0;
    }
    
    public void decreaseStock(int quantity) {
        if (stockQuantity < quantity) {
            throw new IllegalArgumentException("재고가 부족합니다.");
        }
        this.stockQuantity -= quantity;
    }
    
    public void increaseStock(int quantity) {
        this.stockQuantity += quantity;
    }
}

enum ProductCategory {
    ELECTRONICS, CLOTHING, BOOKS, HOME_GARDEN, SPORTS, BEAUTY, AUTOMOTIVE
}

enum ProductStatus {
    ACTIVE, INACTIVE, DISCONTINUED, OUT_OF_STOCK
}
```

### 주문 엔티티

```java
// entity/Order.java
package com.example.app.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", unique = true, nullable = false)
    private String orderNumber;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status;
    
    @Column(name = "total_amount", nullable = false, precision = 19, scale = 2)
    private BigDecimal totalAmount;
    
    @Column(name = "shipping_address", nullable = false)
    private String shippingAddress;
    
    @Column(name = "billing_address")
    private String billingAddress;
    
    @Column(name = "payment_method")
    private String paymentMethod;
    
    @Column(name = "shipped_at")
    private LocalDateTime shippedAt;
    
    @Column(name = "delivered_at")
    private LocalDateTime deliveredAt;
    
    @Column(name = "notes", columnDefinition = "TEXT")
    private String notes;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @Builder.Default
    private Set<OrderItem> orderItems = new HashSet<>();
    
    // 비즈니스 메서드
    public void addOrderItem(OrderItem orderItem) {
        orderItems.add(orderItem);
        orderItem.setOrder(this);
        calculateTotalAmount();
    }
    
    public void removeOrderItem(OrderItem orderItem) {
        orderItems.remove(orderItem);
        orderItem.setOrder(null);
        calculateTotalAmount();
    }
    
    public void calculateTotalAmount() {
        this.totalAmount = orderItems.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    public void updateStatus(OrderStatus newStatus) {
        this.status = newStatus;
        
        if (newStatus == OrderStatus.SHIPPED) {
            this.shippedAt = LocalDateTime.now();
        } else if (newStatus == OrderStatus.DELIVERED) {
            this.deliveredAt = LocalDateTime.now();
        }
    }
    
    public boolean canBeCancelled() {
        return status == OrderStatus.PENDING || status == OrderStatus.CONFIRMED;
    }
}

// entity/OrderItem.java
package com.example.app.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderItem extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
    
    @Column(name = "quantity", nullable = false)
    private Integer quantity;
    
    @Column(name = "unit_price", nullable = false, precision = 19, scale = 2)
    private BigDecimal unitPrice;
    
    @Column(name = "subtotal", nullable = false, precision = 19, scale = 2)
    private BigDecimal subtotal;
    
    // 비즈니스 메서드
    @PrePersist
    @PreUpdate
    private void calculateSubtotal() {
        this.subtotal = unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
    
    public BigDecimal getSubtotal() {
        if (subtotal == null) {
            calculateSubtotal();
        }
        return subtotal;
    }
}

enum OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED
}
```

## 📦 DTO 클래스

### 요청 DTO

```java
// dto/request/LoginRequest.java
package com.example.app.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class LoginRequest {
    
    @NotBlank(message = "사용자명은 필수입니다")
    @Size(min = 3, max = 50, message = "사용자명은 3-50자여야 합니다")
    private String username;
    
    @NotBlank(message = "비밀번호는 필수입니다")
    @Size(min = 6, max = 100, message = "비밀번호는 6-100자여야 합니다")
    private String password;
}

// dto/request/RegisterRequest.java
package com.example.app.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

@Data
public class RegisterRequest {
    
    @NotBlank(message = "사용자명은 필수입니다")
    @Size(min = 3, max = 50, message = "사용자명은 3-50자여야 합니다")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "사용자명은 영문, 숫자, 언더스코어만 사용 가능합니다")
    private String username;
    
    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "올바른 이메일 형식이 아닙니다")
    @Size(max = 100, message = "이메일은 100자를 초과할 수 없습니다")
    private String email;
    
    @NotBlank(message = "비밀번호는 필수입니다")
    @Size(min = 8, max = 100, message = "비밀번호는 8-100자여야 합니다")
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$", 
             message = "비밀번호는 대소문자, 숫자, 특수문자를 포함해야 합니다")
    private String password;
    
    @NotBlank(message = "비밀번호 확인은 필수입니다")
    private String confirmPassword;
    
    @NotBlank(message = "이름은 필수입니다")
    @Size(max = 50, message = "이름은 50자를 초과할 수 없습니다")
    private String firstName;
    
    @NotBlank(message = "성은 필수입니다")
    @Size(max = 50, message = "성은 50자를 초과할 수 없습니다")
    private String lastName;
    
    @Pattern(regexp = "^[0-9-+() ]*$", message = "올바른 전화번호 형식이 아닙니다")
    private String phone;
    
    @AssertTrue(message = "비밀번호가 일치하지 않습니다")
    private boolean isPasswordMatching() {
        return password != null && password.equals(confirmPassword);
    }
}

// dto/request/CreateProductRequest.java
package com.example.app.dto.request;

import com.example.app.entity.ProductCategory;
import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class CreateProductRequest {
    
    @NotBlank(message = "상품명은 필수입니다")
    @Size(max = 200, message = "상품명은 200자를 초과할 수 없습니다")
    private String name;
    
    @Size(max = 1000, message = "상품 설명은 1000자를 초과할 수 없습니다")
    private String description;
    
    @NotNull(message = "가격은 필수입니다")
    @DecimalMin(value = "0.01", message = "가격은 0.01 이상이어야 합니다")
    @Digits(integer = 17, fraction = 2, message = "가격 형식이 올바르지 않습니다")
    private BigDecimal price;
    
    @NotNull(message = "재고 수량은 필수입니다")
    @Min(value = 0, message = "재고 수량은 0 이상이어야 합니다")
    private Integer stockQuantity;
    
    @Size(max = 100, message = "SKU는 100자를 초과할 수 없습니다")
    private String sku;
    
    @NotNull(message = "카테고리는 필수입니다")
    private ProductCategory category;
    
    @Pattern(regexp = "^https?://.*", message = "올바른 URL 형식이 아닙니다")
    private String imageUrl;
    
    @DecimalMin(value = "0.01", message = "무게는 0.01 이상이어야 합니다")
    private BigDecimal weight;
    
    private String dimensions;
}

// dto/request/CreateOrderRequest.java
package com.example.app.dto.request;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.Size;
import lombok.Data;

import java.util.List;

@Data
public class CreateOrderRequest {
    
    @NotEmpty(message = "주문 항목은 최소 1개 이상이어야 합니다")
    @Valid
    private List<OrderItemRequest> orderItems;
    
    @NotBlank(message = "배송 주소는 필수입니다")
    @Size(max = 500, message = "배송 주소는 500자를 초과할 수 없습니다")
    private String shippingAddress;
    
    @Size(max = 500, message = "청구 주소는 500자를 초과할 수 없습니다")
    private String billingAddress;
    
    @Size(max = 100, message = "결제 방법은 100자를 초과할 수 없습니다")
    private String paymentMethod;
    
    @Size(max = 1000, message = "주문 메모는 1000자를 초과할 수 없습니다")
    private String notes;
}

// dto/request/OrderItemRequest.java
package com.example.app.dto.request;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class OrderItemRequest {
    
    @NotNull(message = "상품 ID는 필수입니다")
    private Long productId;
    
    @NotNull(message = "수량은 필수입니다")
    @Min(value = 1, message = "수량은 1 이상이어야 합니다")
    private Integer quantity;
}
```

### 응답 DTO

```java
// dto/response/ApiResponse.java
package com.example.app.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    
    private boolean success;
    private String message;
    private T data;
    private String errorCode;
    private LocalDateTime timestamp;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
            .success(true)
            .message(message)
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static ApiResponse<Void> error(String message) {
        return ApiResponse.<Void>builder()
            .success(false)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static ApiResponse<Void> error(String message, String errorCode) {
        return ApiResponse.<Void>builder()
            .success(false)
            .message(message)
            .errorCode(errorCode)
            .timestamp(LocalDateTime.now())
            .build();
    }
}

// dto/response/PageResponse.java
package com.example.app.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.domain.Page;

import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PageResponse<T> {
    
    private List<T> content;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private boolean first;
    private boolean last;
    private boolean hasNext;
    private boolean hasPrevious;
    
    public static <T> PageResponse<T> of(Page<T> page) {
        return PageResponse.<T>builder()
            .content(page.getContent())
            .page(page.getNumber())
            .size(page.getSize())
            .totalElements(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .first(page.isFirst())
            .last(page.isLast())
            .hasNext(page.hasNext())
            .hasPrevious(page.hasPrevious())
            .build();
    }
}

// dto/response/AuthResponse.java
package com.example.app.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuthResponse {
    
    private String accessToken;
    private String tokenType;
    private Long expiresIn;
    private UserResponse user;
    
    public static AuthResponse of(String accessToken, Long expiresIn, UserResponse user) {
        return AuthResponse.builder()
            .accessToken(accessToken)
            .tokenType("Bearer")
            .expiresIn(expiresIn)
            .user(user)
            .build();
    }
}

// dto/response/UserResponse.java
package com.example.app.dto.response;

import com.example.app.entity.UserRole;
import com.example.app.entity.UserStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserResponse {
    
    private Long id;
    private String username;
    private String email;
    private String firstName;
    private String lastName;
    private String phone;
    private UserRole role;
    private UserStatus status;
    private Boolean emailVerified;
    private LocalDateTime lastLoginAt;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    public String getFullName() {
        return firstName + " " + lastName;
    }
}

// dto/response/ProductResponse.java
package com.example.app.dto.response;

import com.example.app.entity.ProductCategory;
import com.example.app.entity.ProductStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductResponse {
    
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Integer stockQuantity;
    private String sku;
    private ProductCategory category;
    private ProductStatus status;
    private String imageUrl;
    private BigDecimal weight;
    private String dimensions;
    private boolean available;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// dto/response/OrderResponse.java
package com.example.app.dto.response;

import com.example.app.entity.OrderStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderResponse {
    
    private Long id;
    private String orderNumber;
    private Long userId;
    private String userFullName;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private String shippingAddress;
    private String billingAddress;
    private String paymentMethod;
    private String notes;
    private LocalDateTime shippedAt;
    private LocalDateTime deliveredAt;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private List<OrderItemResponse> orderItems;
}

// dto/response/OrderItemResponse.java
package com.example.app.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderItemResponse {
    
    private Long id;
    private Long productId;
    private String productName;
    private String productSku;
    private Integer quantity;
    private BigDecimal unitPrice;
    private BigDecimal subtotal;
}
```

## 🗄 Repository 계층

```java
// repository/UserRepository.java
package com.example.app.repository;

import com.example.app.entity.User;
import com.example.app.entity.UserStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    
    Optional<User> findByEmail(String email);
    
    Optional<User> findByEmailVerificationToken(String token);
    
    boolean existsByUsername(String username);
    
    boolean existsByEmail(String email);
    
    Page<User> findByStatus(UserStatus status, Pageable pageable);
    
    @Query("SELECT u FROM User u WHERE u.firstName LIKE %:keyword% OR u.lastName LIKE %:keyword% OR u.email LIKE %:keyword%")
    Page<User> findByKeyword(@Param("keyword") String keyword, Pageable pageable);
    
    @Query("SELECT u FROM User u WHERE u.lastLoginAt < :date")
    List<User> findInactiveUsers(@Param("date") LocalDateTime date);
    
    @Query("SELECT COUNT(u) FROM User u WHERE u.status = :status")
    long countByStatus(@Param("status") UserStatus status);
}

// repository/ProductRepository.java
package com.example.app.repository;

import com.example.app.entity.Product;
import com.example.app.entity.ProductCategory;
import com.example.app.entity.ProductStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    Optional<Product> findBySku(String sku);
    
    Page<Product> findByStatus(ProductStatus status, Pageable pageable);
    
    Page<Product> findByCategory(ProductCategory category, Pageable pageable);
    
    Page<Product> findByCategoryAndStatus(ProductCategory category, ProductStatus status, Pageable pageable);
    
    @Query("SELECT p FROM Product p WHERE p.name LIKE %:keyword% OR p.description LIKE %:keyword%")
    Page<Product> findByKeyword(@Param("keyword") String keyword, Pageable pageable);
    
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :minPrice AND :maxPrice")
    Page<Product> findByPriceRange(@Param("minPrice") BigDecimal minPrice, 
                                  @Param("maxPrice") BigDecimal maxPrice, 
                                  Pageable pageable);
    
    @Query("SELECT p FROM Product p WHERE p.stockQuantity <= :threshold AND p.status = 'ACTIVE'")
    List<Product> findLowStockProducts(@Param("threshold") Integer threshold);
    
    @Query("SELECT p FROM Product p WHERE p.status = 'ACTIVE' ORDER BY p.createdAt DESC")
    List<Product> findLatestProducts(Pageable pageable);
}

// repository/OrderRepository.java
package com.example.app.repository;

import com.example.app.entity.Order;
import com.example.app.entity.OrderStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    Optional<Order> findByOrderNumber(String orderNumber);
    
    Page<Order> findByUserId(Long userId, Pageable pageable);
    
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
    
    Page<Order> findByUserIdAndStatus(Long userId, OrderStatus status, Pageable pageable);
    
    @Query("SELECT o FROM Order o WHERE o.createdAt BETWEEN :startDate AND :endDate")
    List<Order> findByDateRange(@Param("startDate") LocalDateTime startDate, 
                               @Param("endDate") LocalDateTime endDate);
    
    @Query("SELECT COUNT(o) FROM Order o WHERE o.status = :status")
    long countByStatus(@Param("status") OrderStatus status);
    
    @Query("SELECT SUM(o.totalAmount) FROM Order o WHERE o.status = 'DELIVERED' AND o.createdAt BETWEEN :startDate AND :endDate")
    BigDecimal getTotalRevenueByDateRange(@Param("startDate") LocalDateTime startDate, 
                                        @Param("endDate") LocalDateTime endDate);
    
    @Query("SELECT o FROM Order o JOIN FETCH o.orderItems oi JOIN FETCH oi.product WHERE o.id = :orderId")
    Optional<Order> findByIdWithItems(@Param("orderId") Long orderId);
}
```

## 🔧 Service 계층

```java
// service/UserService.java
package com.example.app.service;

import com.example.app.dto.request.CreateUserRequest;
import com.example.app.dto.request.UpdateUserRequest;
import com.example.app.dto.response.UserResponse;
import com.example.app.dto.response.PageResponse;
import com.example.app.entity.User;
import com.example.app.entity.UserStatus;
import com.example.app.exception.DuplicateResourceException;
import com.example.app.exception.ResourceNotFoundException;
import com.example.app.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public PageResponse<UserResponse> getAllUsers(Pageable pageable) {
        Page<User> users = userRepository.findAll(pageable);
        Page<UserResponse> userResponses = users.map(this::convertToResponse);
        return PageResponse.of(userResponses);
    }
    
    public UserResponse getUserById(Long id) {
        User user = findUserById(id);
        return convertToResponse(user);
    }
    
    public UserResponse getUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new ResourceNotFoundException("사용자를 찾을 수 없습니다: " + username));
        return convertToResponse(user);
    }
    
    public PageResponse<UserResponse> searchUsers(String keyword, Pageable pageable) {
        Page<User> users = userRepository.findByKeyword(keyword, pageable);
        Page<UserResponse> userResponses = users.map(this::convertToResponse);
        return PageResponse.of(userResponses);
    }
    
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        validateUniqueConstraints(request.getUsername(), request.getEmail());
        
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .phone(request.getPhone())
            .role(request.getRole())
            .status(UserStatus.ACTIVE)
            .emailVerified(false)
            .emailVerificationToken(UUID.randomUUID().toString())
            .build();
        
        User savedUser = userRepository.save(user);
        log.info("사용자가 생성되었습니다: {}", savedUser.getUsername());
        
        return convertToResponse(savedUser);
    }
    
    @Transactional
    public UserResponse updateUser(Long id, UpdateUserRequest request) {
        User user = findUserById(id);
        
        // 이메일이 변경되는 경우 중복 확인
        if (!user.getEmail().equals(request.getEmail())) {
            if (userRepository.existsByEmail(request.getEmail())) {
                throw new DuplicateResourceException("이미 사용 중인 이메일입니다: " + request.getEmail());
            }
            user.setEmail(request.getEmail());
            user.setEmailVerified(false);
            user.setEmailVerificationToken(UUID.randomUUID().toString());
        }
        
        user.setFirstName(request.getFirstName());
        user.setLastName(request.getLastName());
        user.setPhone(request.getPhone());
        
        User updatedUser = userRepository.save(user);
        log.info("사용자 정보가 업데이트되었습니다: {}", updatedUser.getUsername());
        
        return convertToResponse(updatedUser);
    }
    
    @Transactional
    public void deleteUser(Long id) {
        User user = findUserById(id);
        user.setStatus(UserStatus.DELETED);
        userRepository.save(user);
        log.info("사용자가 삭제되었습니다: {}", user.getUsername());
    }
    
    @Transactional
    public void verifyEmail(String token) {
        User user = userRepository.findByEmailVerificationToken(token)
            .orElseThrow(() -> new ResourceNotFoundException("유효하지 않은 이메일 인증 토큰입니다"));
        
        user.verifyEmail();
        userRepository.save(user);
        log.info("이메일이 인증되었습니다: {}", user.getEmail());
    }
    
    @Transactional
    public void updateLastLoginAt(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new ResourceNotFoundException("사용자를 찾을 수 없습니다: " + username));
        
        user.updateLastLoginAt();
        userRepository.save(user);
    }
    
    private User findUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("사용자를 찾을 수 없습니다: " + id));
    }
    
    private void validateUniqueConstraints(String username, String email) {
        if (userRepository.existsByUsername(username)) {
            throw new DuplicateResourceException("이미 사용 중인 사용자명입니다: " + username);
        }
        
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateResourceException("이미 사용 중인 이메일입니다: " + email);
        }
    }
    
    private UserResponse convertToResponse(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .username(user.getUsername())
            .email(user.getEmail())
            .firstName(user.getFirstName())
            .lastName(user.getLastName())
            .phone(user.getPhone())
            .role(user.getRole())
            .status(user.getStatus())
            .emailVerified(user.getEmailVerified())
            .lastLoginAt(user.getLastLoginAt())
            .createdAt(user.getCreatedAt())
            .updatedAt(user.getUpdatedAt())
            .build();
    }
}
```

## 🎮 Controller 계층

```java
// controller/UserController.java
package com.example.app.controller;

import com.example.app.dto.request.CreateUserRequest;
import com.example.app.dto.request.UpdateUserRequest;
import com.example.app.dto.response.ApiResponse;
import com.example.app.dto.response.PageResponse;
import com.example.app.dto.response.UserResponse;
import com.example.app.service.UserService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Tag(name = "User", description = "사용자 관리 API")
public class UserController {
    
    private final UserService userService;
    
    @GetMapping
    @Operation(summary = "사용자 목록 조회", description = "페이징된 사용자 목록을 조회합니다")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<PageResponse<UserResponse>>> getAllUsers(
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<UserResponse> users = userService.getAllUsers(pageable);
        return ResponseEntity.ok(ApiResponse.success(users));
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "사용자 상세 조회", description = "ID로 사용자 상세 정보를 조회합니다")
    @PreAuthorize("hasRole('ADMIN') or authentication.name == @userService.getUserById(#id).username")
    public ResponseEntity<ApiResponse<UserResponse>> getUserById(
            @Parameter(description = "사용자 ID") @PathVariable Long id) {
        
        UserResponse user = userService.getUserById(id);
        return ResponseEntity.ok(ApiResponse.success(user));
    }
    
    @GetMapping("/search")
    @Operation(summary = "사용자 검색", description = "키워드로 사용자를 검색합니다")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<PageResponse<UserResponse>>> searchUsers(
            @Parameter(description = "검색 키워드") @RequestParam String keyword,
            @PageableDefault(size = 20) Pageable pageable) {
        
        PageResponse<UserResponse> users = userService.searchUsers(keyword, pageable);
        return ResponseEntity.ok(ApiResponse.success(users));
    }
    
    @PostMapping
    @Operation(summary = "사용자 생성", description = "새로운 사용자를 생성합니다")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<UserResponse>> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        
        UserResponse createdUser = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success("사용자가 성공적으로 생성되었습니다", createdUser));
    }
    
    @PutMapping("/{id}")
    @Operation(summary = "사용자 정보 수정", description = "사용자 정보를 수정합니다")
    @PreAuthorize("hasRole('ADMIN') or authentication.name == @userService.getUserById(#id).username")
    public ResponseEntity<ApiResponse<UserResponse>> updateUser(
            @Parameter(description = "사용자 ID") @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        
        UserResponse updatedUser = userService.updateUser(id, request);
        return ResponseEntity.ok(ApiResponse.success("사용자 정보가 성공적으로 수정되었습니다", updatedUser));
    }
    
    @DeleteMapping("/{id}")
    @Operation(summary = "사용자 삭제", description = "사용자를 삭제합니다")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<Void>> deleteUser(
            @Parameter(description = "사용자 ID") @PathVariable Long id) {
        
        userService.deleteUser(id);
        return ResponseEntity.ok(ApiResponse.success("사용자가 성공적으로 삭제되었습니다", null));
    }
    
    @PostMapping("/verify-email")
    @Operation(summary = "이메일 인증", description = "이메일 인증 토큰으로 이메일을 인증합니다")
    public ResponseEntity<ApiResponse<Void>> verifyEmail(
            @Parameter(description = "이메일 인증 토큰") @RequestParam String token) {
        
        userService.verifyEmail(token);
        return ResponseEntity.ok(ApiResponse.success("이메일이 성공적으로 인증되었습니다", null));
    }
}
```

## ⚠️ 예외 처리

```java
// exception/GlobalExceptionHandler.java
package com.example.app.exception;

import com.example.app.dto.response.ApiResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleResourceNotFoundException(
            ResourceNotFoundException ex, WebRequest request) {
        
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ApiResponse.error(ex.getMessage(), "RESOURCE_NOT_FOUND"));
    }
    
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiResponse<Void>> handleDuplicateResourceException(
            DuplicateResourceException ex, WebRequest request) {
        
        log.warn("Duplicate resource: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(ApiResponse.error(ex.getMessage(), "DUPLICATE_RESOURCE"));
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusinessException(
            BusinessException ex, WebRequest request) {
        
        log.warn("Business exception: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ApiResponse.error(ex.getMessage(), "BUSINESS_ERROR"));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleValidationException(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        log.warn("Validation failed: {}", errors);
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ApiResponse.<Map<String, String>>builder()
                .success(false)
                .message("입력값 검증에 실패했습니다")
                .data(errors)
                .errorCode("VALIDATION_ERROR")
                .timestamp(java.time.LocalDateTime.now())
                .build());
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGenericException(
            Exception ex, WebRequest request) {
        
        log.error("Unexpected error occurred", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("서버 내부 오류가 발생했습니다", "INTERNAL_SERVER_ERROR"));
    }
}

// 커스텀 예외 클래스들
package com.example.app.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}

public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}

public class ValidationException extends RuntimeException {
    public ValidationException(String message) {
        super(message);
    }
}
```

## 📚 관련 노트

- [[Spring Boot Project Setup]] - 프로젝트 초기 설정
- [[Spring Data JPA]] - 데이터 액세스 계층
- [[Spring Security 6]] - 보안 설정
- [[API Design Patterns]] - REST API 설계 패턴
- [[Testing]] - 테스트 코드 작성

## 🔗 사용 방법

1. **프로젝트 생성**: Spring Initializr에서 필요한 의존성과 함께 프로젝트 생성
2. **패키지 구조 생성**: 위의 구조대로 패키지와 클래스 파일들 생성
3. **설정 파일 작성**: application.yml과 환경별 설정 파일 작성
4. **데이터베이스 설정**: 사용할 데이터베이스에 맞게 설정 수정
5. **보안 설정**: JWT나 OAuth2 등 필요한 보안 설정 추가
6. **테스트 코드 작성**: 각 계층별 테스트 코드 작성


## 🔐 보안 설정

### JWT 토큰 프로바이더

```java
// security/JwtTokenProvider.java
package com.example.app.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;

@Component
@Slf4j
public class JwtTokenProvider {
    
    private final SecretKey key;
    private final long jwtExpirationMs;
    
    public JwtTokenProvider(@Value("${jwt.secret}") String secret,
                           @Value("${jwt.expiration}") long jwtExpirationMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes());
        this.jwtExpirationMs = jwtExpirationMs;
    }
    
    public String generateToken(Authentication authentication) {
        UserDetails userPrincipal = (UserDetails) authentication.getPrincipal();
        Date expiryDate = new Date(System.currentTimeMillis() + jwtExpirationMs);
        
        return Jwts.builder()
            .setSubject(userPrincipal.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(expiryDate)
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
    }
    
    public String generateToken(String username) {
        Date expiryDate = new Date(System.currentTimeMillis() + jwtExpirationMs);
        
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(expiryDate)
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
    }
    
    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        return claims.getSubject();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (MalformedJwtException ex) {
            log.error("Invalid JWT token: {}", ex.getMessage());
        } catch (ExpiredJwtException ex) {
            log.error("JWT token is expired: {}", ex.getMessage());
        } catch (UnsupportedJwtException ex) {
            log.error("JWT token is unsupported: {}", ex.getMessage());
        } catch (IllegalArgumentException ex) {
            log.error("JWT claims string is empty: {}", ex.getMessage());
        }
        return false;
    }
    
    public long getExpirationTime() {
        return jwtExpirationMs;
    }
}

// security/JwtAuthenticationFilter.java
package com.example.app.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenProvider tokenProvider;
    private final CustomUserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        try {
            String jwt = getJwtFromRequest(request);
            
            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromToken(jwt);
                
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            log.error("Could not set user authentication in security context", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}

// security/CustomUserDetailsService.java
package com.example.app.security;

import com.example.app.entity.User;
import com.example.app.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {
    
    private final UserRepository userRepository;
    
    @Override
    @Transactional
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        
        return CustomUserDetails.create(user);
    }
    
    @Transactional
    public UserDetails loadUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + id));
        
        return CustomUserDetails.create(user);
    }
}

// security/CustomUserDetails.java
package com.example.app.security;

import com.example.app.entity.User;
import lombok.AllArgsConstructor;
import lombok.Data;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.Collections;

@Data
@AllArgsConstructor
public class CustomUserDetails implements UserDetails {
    
    private Long id;
    private String username;
    private String email;
    private String password;
    private Collection<? extends GrantedAuthority> authorities;
    private boolean enabled;
    private boolean accountNonLocked;
    
    public static CustomUserDetails create(User user) {
        Collection<GrantedAuthority> authorities = Collections.singletonList(
            new SimpleGrantedAuthority("ROLE_" + user.getRole().name())
        );
        
        return new CustomUserDetails(
            user.getId(),
            user.getUsername(),
            user.getEmail(),
            user.getPassword(),
            authorities,
            user.isEnabled(),
            user.isAccountNonLocked()
        );
    }
    
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }
    
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
}

// security/JwtAuthenticationEntryPoint.java
package com.example.app.security;

import com.example.app.dto.response.ApiResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Slf4j
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    
    @Override
    public void commence(HttpServletRequest request, 
                        HttpServletResponse response,
                        AuthenticationException authException) throws IOException, ServletException {
        
        log.error("Unauthorized error: {}", authException.getMessage());
        
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        
        ApiResponse<Void> apiResponse = ApiResponse.error("인증이 필요합니다", "UNAUTHORIZED");
        
        ObjectMapper mapper = new ObjectMapper();
        mapper.writeValue(response.getOutputStream(), apiResponse);
    }
}

// config/SecurityConfig.java
package com.example.app.config;

import com.example.app.security.CustomUserDetailsService;
import com.example.app.security.JwtAuthenticationEntryPoint;
import com.example.app.security.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfigurationSource;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final CustomUserDetailsService userDetailsService;
    private final JwtAuthenticationEntryPoint unauthorizedHandler;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final CorsConfigurationSource corsConfigurationSource;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(AbstractHttpConfigurer::disable)
            .cors(cors -> cors.configurationSource(corsConfigurationSource))
            .exceptionHandling(ex -> ex.authenticationEntryPoint(unauthorizedHandler))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/users/verify-email").permitAll()
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        
        http.authenticationProvider(authenticationProvider());
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

### 인증 서비스 및 컨트롤러

```java
// service/AuthService.java
package com.example.app.service;

import com.example.app.dto.request.LoginRequest;
import com.example.app.dto.request.RegisterRequest;
import com.example.app.dto.response.AuthResponse;
import com.example.app.dto.response.UserResponse;
import com.example.app.entity.User;
import com.example.app.entity.UserRole;
import com.example.app.entity.UserStatus;
import com.example.app.exception.BusinessException;
import com.example.app.exception.DuplicateResourceException;
import com.example.app.repository.UserRepository;
import com.example.app.security.JwtTokenProvider;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class AuthService {
    
    private final AuthenticationManager authenticationManager;
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider tokenProvider;
    private final UserService userService;
    
    @Transactional
    public AuthResponse login(LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword())
        );
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        String jwt = tokenProvider.generateToken(authentication);
        
        // 마지막 로그인 시간 업데이트
        userService.updateLastLoginAt(request.getUsername());
        
        UserResponse userResponse = userService.getUserByUsername(request.getUsername());
        
        log.info("User logged in: {}", request.getUsername());
        
        return AuthResponse.of(jwt, tokenProvider.getExpirationTime(), userResponse);
    }
    
    @Transactional
    public UserResponse register(RegisterRequest request) {
        validateRegistrationRequest(request);
        
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .phone(request.getPhone())
            .role(UserRole.USER)
            .status(UserStatus.ACTIVE)
            .emailVerified(false)
            .emailVerificationToken(UUID.randomUUID().toString())
            .build();
        
        User savedUser = userRepository.save(user);
        
        log.info("New user registered: {}", savedUser.getUsername());
        
        // TODO: 이메일 인증 메일 발송
        
        return convertToUserResponse(savedUser);
    }
    
    private void validateRegistrationRequest(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateResourceException("이미 사용 중인 사용자명입니다: " + request.getUsername());
        }
        
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException("이미 사용 중인 이메일입니다: " + request.getEmail());
        }
        
        if (!request.getPassword().equals(request.getConfirmPassword())) {
            throw new BusinessException("비밀번호가 일치하지 않습니다");
        }
    }
    
    private UserResponse convertToUserResponse(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .username(user.getUsername())
            .email(user.getEmail())
            .firstName(user.getFirstName())
            .lastName(user.getLastName())
            .phone(user.getPhone())
            .role(user.getRole())
            .status(user.getStatus())
            .emailVerified(user.getEmailVerified())
            .createdAt(user.getCreatedAt())
            .build();
    }
}

// controller/AuthController.java
package com.example.app.controller;

import com.example.app.dto.request.LoginRequest;
import com.example.app.dto.request.RegisterRequest;
import com.example.app.dto.response.ApiResponse;
import com.example.app.dto.response.AuthResponse;
import com.example.app.dto.response.UserResponse;
import com.example.app.service.AuthService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Tag(name = "Authentication", description = "인증 관련 API")
public class AuthController {
    
    private final AuthService authService;
    
    @PostMapping("/login")
    @Operation(summary = "로그인", description = "사용자 로그인을 처리합니다")
    public ResponseEntity<ApiResponse<AuthResponse>> login(@Valid @RequestBody LoginRequest request) {
        AuthResponse authResponse = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success("로그인이 성공했습니다", authResponse));
    }
    
    @PostMapping("/register")
    @Operation(summary = "회원가입", description = "새로운 사용자를 등록합니다")
    public ResponseEntity<ApiResponse<UserResponse>> register(@Valid @RequestBody RegisterRequest request) {
        UserResponse userResponse = authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success("회원가입이 성공했습니다. 이메일 인증을 완료해주세요.", userResponse));
    }
}
```

## 🛠 유틸리티 클래스

```java
// util/DateUtils.java
package com.example.app.util;

import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.Date;

public class DateUtils {
    
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    public static final String DEFAULT_DATETIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    public static final String ISO_DATETIME_FORMAT = "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'";
    
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT);
    private static final DateTimeFormatter DATETIME_FORMATTER = DateTimeFormatter.ofPattern(DEFAULT_DATETIME_FORMAT);
    private static final DateTimeFormatter ISO_FORMATTER = DateTimeFormatter.ofPattern(ISO_DATETIME_FORMAT);
    
    public static String formatDate(LocalDate date) {
        return date != null ? date.format(DATE_FORMATTER) : null;
    }
    
    public static String formatDateTime(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.format(DATETIME_FORMATTER) : null;
    }
    
    public static String formatToIso(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.atZone(ZoneId.systemDefault()).format(ISO_FORMATTER) : null;
    }
    
    public static LocalDate parseDate(String dateString) {
        return dateString != null ? LocalDate.parse(dateString, DATE_FORMATTER) : null;
    }
    
    public static LocalDateTime parseDateTime(String dateTimeString) {
        return dateTimeString != null ? LocalDateTime.parse(dateTimeString, DATETIME_FORMATTER) : null;
    }
    
    public static LocalDateTime toLocalDateTime(Date date) {
        return date != null ? date.toInstant().atZone(ZoneId.systemDefault()).toLocalDateTime() : null;
    }
    
    public static Date toDate(LocalDateTime localDateTime) {
        return localDateTime != null ? Date.from(localDateTime.atZone(ZoneId.systemDefault()).toInstant()) : null;
    }
    
    public static boolean isToday(LocalDateTime dateTime) {
        return dateTime != null && dateTime.toLocalDate().equals(LocalDate.now());
    }
    
    public static boolean isPast(LocalDateTime dateTime) {
        return dateTime != null && dateTime.isBefore(LocalDateTime.now());
    }
    
    public static boolean isFuture(LocalDateTime dateTime) {
        return dateTime != null && dateTime.isAfter(LocalDateTime.now());
    }
    
    public static long daysBetween(LocalDate start, LocalDate end) {
        return start != null && end != null ? Duration.between(start.atStartOfDay(), end.atStartOfDay()).toDays() : 0;
    }
    
    public static LocalDateTime startOfDay(LocalDate date) {
        return date != null ? date.atStartOfDay() : null;
    }
    
    public static LocalDateTime endOfDay(LocalDate date) {
        return date != null ? date.atTime(23, 59, 59, 999_999_999) : null;
    }
}

// util/StringUtils.java
package com.example.app.util;

import java.util.UUID;
import java.util.regex.Pattern;

public class StringUtils {
    
    private static final Pattern EMAIL_PATTERN = Pattern.compile(
        "^[A-Za-z0-9+_.-]+@([A-Za-z0-9.-]+\\.[A-Za-z]{2,})$"
    );
    
    private static final Pattern PHONE_PATTERN = Pattern.compile(
        "^[\\+]?[0-9\\-\\(\\)\\s]+$"
    );
    
    public static boolean isEmpty(String str) {
        return str == null || str.trim().isEmpty();
    }
    
    public static boolean isNotEmpty(String str) {
        return !isEmpty(str);
    }
    
    public static boolean hasText(String str) {
        return str != null && !str.trim().isEmpty();
    }
    
    public static String defaultIfEmpty(String str, String defaultValue) {
        return isEmpty(str) ? defaultValue : str;
    }
    
    public static String trim(String str) {
        return str != null ? str.trim() : null;
    }
    
    public static String truncate(String str, int maxLength) {
        if (str == null || str.length() <= maxLength) {
            return str;
        }
        return str.substring(0, maxLength);
    }
    
    public static boolean isValidEmail(String email) {
        return hasText(email) && EMAIL_PATTERN.matcher(email).matches();
    }
    
    public static boolean isValidPhone(String phone) {
        return hasText(phone) && PHONE_PATTERN.matcher(phone).matches();
    }
    
    public static String generateRandomString(int length) {
        return UUID.randomUUID().toString().replace("-", "").substring(0, Math.min(length, 32));
    }
    
    public static String maskEmail(String email) {
        if (!hasText(email) || !email.contains("@")) {
            return email;
        }
        
        String[] parts = email.split("@");
        String localPart = parts[0];
        String domainPart = parts[1];
        
        if (localPart.length() <= 2) {
            return email;
        }
        
        String masked = localPart.charAt(0) + "***" + localPart.charAt(localPart.length() - 1);
        return masked + "@" + domainPart;
    }
    
    public static String maskPhone(String phone) {
        if (!hasText(phone) || phone.length() < 8) {
            return phone;
        }
        
        String cleaned = phone.replaceAll("[^0-9]", "");
        if (cleaned.length() < 8) {
            return phone;
        }
        
        return cleaned.substring(0, 3) + "****" + cleaned.substring(cleaned.length() - 4);
    }
    
    public static String camelToSnakeCase(String camelCase) {
        if (isEmpty(camelCase)) {
            return camelCase;
        }
        
        return camelCase.replaceAll("([a-z])([A-Z]+)", "$1_$2").toLowerCase();
    }
    
    public static String snakeToCamelCase(String snakeCase) {
        if (isEmpty(snakeCase)) {
            return snakeCase;
        }
        
        StringBuilder result = new StringBuilder();
        boolean nextUpperCase = false;
        
        for (char c : snakeCase.toCharArray()) {
            if (c == '_') {
                nextUpperCase = true;
            } else {
                result.append(nextUpperCase ? Character.toUpperCase(c) : c);
                nextUpperCase = false;
            }
        }
        
        return result.toString();
    }
}

// util/ValidationUtils.java
package com.example.app.util;

import java.util.regex.Pattern;

public class ValidationUtils {
    
    private static final Pattern PASSWORD_PATTERN = Pattern.compile(
        "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$"
    );
    
    private static final Pattern USERNAME_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9_]+$"
    );
    
    public static boolean isValidPassword(String password) {
        return StringUtils.hasText(password) && 
               password.length() >= 8 && 
               password.length() <= 100 &&
               PASSWORD_PATTERN.matcher(password).matches();
    }
    
    public static boolean isValidUsername(String username) {
        return StringUtils.hasText(username) && 
               username.length() >= 3 && 
               username.length() <= 50 &&
               USERNAME_PATTERN.matcher(username).matches();
    }
    
    public static boolean isValidId(Long id) {
        return id != null && id > 0;
    }
    
    public static boolean isValidPageNumber(int page) {
        return page >= 0;
    }
    
    public static boolean isValidPageSize(int size) {
        return size > 0 && size <= 100;
    }
    
    public static String getPasswordValidationMessage() {
        return "비밀번호는 8-100자이며, 대소문자, 숫자, 특수문자를 포함해야 합니다";
    }
    
    public static String getUsernameValidationMessage() {
        return "사용자명은 3-50자이며, 영문, 숫자, 언더스코어만 사용 가능합니다";
    }
}

// util/ResponseUtils.java
package com.example.app.util;

import com.example.app.dto.response.ApiResponse;
import com.example.app.dto.response.PageResponse;
import org.springframework.data.domain.Page;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

public class ResponseUtils {
    
    public static <T> ResponseEntity<ApiResponse<T>> success(T data) {
        return ResponseEntity.ok(ApiResponse.success(data));
    }
    
    public static <T> ResponseEntity<ApiResponse<T>> success(String message, T data) {
        return ResponseEntity.ok(ApiResponse.success(message, data));
    }
    
    public static <T> ResponseEntity<ApiResponse<T>> created(T data) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(data));
    }
    
    public static <T> ResponseEntity<ApiResponse<T>> created(String message, T data) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(message, data));
    }
    
    public static ResponseEntity<ApiResponse<Void>> noContent() {
        return ResponseEntity.noContent().build();
    }
    
    public static ResponseEntity<ApiResponse<Void>> badRequest(String message) {
        return ResponseEntity.badRequest()
            .body(ApiResponse.error(message));
    }
    
    public static ResponseEntity<ApiResponse<Void>> notFound(String message) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiResponse.error(message));
    }
    
    public static ResponseEntity<ApiResponse<Void>> conflict(String message) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ApiResponse.error(message));
    }
    
    public static ResponseEntity<ApiResponse<Void>> internalServerError(String message) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error(message));
    }
    
    public static <T> ResponseEntity<ApiResponse<PageResponse<T>>> pagedSuccess(Page<T> page) {
        PageResponse<T> pageResponse = PageResponse.of(page);
        return ResponseEntity.ok(ApiResponse.success(pageResponse));
    }
    
    public static <T> ResponseEntity<ApiResponse<PageResponse<T>>> pagedSuccess(String message, Page<T> page) {
        PageResponse<T> pageResponse = PageResponse.of(page);
        return ResponseEntity.ok(ApiResponse.success(message, pageResponse));
    }
}
```

## 📋 빌드 파일

### Gradle 빌드 파일

```gradle
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'org.flywaydb.flyway' version '9.22.3'
    id 'jacoco'
}

group = 'com.example'
version = '1.0.0'
java {
    sourceCompatibility = '17'
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot Starters
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-cache'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.boot:spring-boot-starter-mail'
    
    // Database
    runtimeOnly 'org.postgresql:postgresql'
    runtimeOnly 'com.h2database:h2'
    implementation 'org.flywaydb:flyway-core'
    
    // JWT
    implementation 'io.jsonwebtoken:jjwt-api:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.3'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.3'
    
    // Documentation
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0'
    
    // Utilities
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
    
    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'com.github.tomakehurst:wiremock-jre8:2.35.0'
}

dependencyManagement {
    imports {
        mavenBom "org.testcontainers:testcontainers-bom:1.19.3"
    }
}

tasks.named('test') {
    useJUnitPlatform()
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true
        html.required = true
    }
    
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/config/**',
                '**/dto/**',
                '**/entity/**',
                '**/*Application*',
                '**/exception/**'
            ])
        }))
    }
}

jar {
    enabled = false
    archiveClassifier = ''
}

bootJar {
    enabled = true
    archiveClassifier = ''
    archiveFileName = "${project.name}.jar"
}

// Docker 빌드
task buildDocker(type: Exec) {
    dependsOn bootJar
    commandLine 'docker', 'build', '-t', "${project.name}:${project.version}", '.'
}
```

### Maven 빌드 파일

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>spring-boot-app</artifactId>
    <version>1.0.0</version>
    <name>spring-boot-app</name>
    <description>Spring Boot Boilerplate Application</description>
    
    <properties>
        <java.version>17</java.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <testcontainers.version>1.19.3</testcontainers.version>
        <springdoc.version>2.2.0</springdoc.version>
        <jjwt.version>0.12.3</jjwt.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Documentation -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>${springdoc.version}</version>
        </dependency>
        
        <!-- Utilities -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.8</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <excludes>
                        <exclude>**/config/**</exclude>
                        <exclude>**/dto/**</exclude>
                        <exclude>**/entity/**</exclude>
                        <exclude>**/*Application*</exclude>
                        <exclude>**/exception/**</exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## 🧪 테스트 코드

### 단위 테스트

```java
// test/java/com/example/app/service/UserServiceTest.java
package com.example.app.service;

import com.example.app.dto.request.CreateUserRequest;
import com.example.app.dto.response.UserResponse;
import com.example.app.entity.User;
import com.example.app.entity.UserRole;
import com.example.app.entity.UserStatus;
import com.example.app.exception.DuplicateResourceException;
import com.example.app.exception.ResourceNotFoundException;
import com.example.app.repository.UserRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.time.LocalDateTime;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("사용자 서비스 테스트")
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @InjectMocks
    private UserService userService;
    
    private User testUser;
    private CreateUserRequest createUserRequest;
    
    @BeforeEach
    void setUp() {
        testUser = User.builder()
            .id(1L)
            .username("testuser")
            .email("test@example.com")
            .password("encodedPassword")
            .firstName("Test")
            .lastName("User")
            .phone("010-1234-5678")
            .role(UserRole.USER)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .createdAt(LocalDateTime.now())
            .updatedAt(LocalDateTime.now())
            .build();
        
        createUserRequest = new CreateUserRequest();
        createUserRequest.setUsername("newuser");
        createUserRequest.setEmail("newuser@example.com");
        createUserRequest.setPassword("password123");
        createUserRequest.setFirstName("New");
        createUserRequest.setLastName("User");
        createUserRequest.setPhone("010-9876-5432");
        createUserRequest.setRole(UserRole.USER);
    }
    
    @Test
    @DisplayName("사용자 ID로 조회 성공")
    void getUserById_Success() {
        // given
        given(userRepository.findById(1L)).willReturn(Optional.of(testUser));
        
        // when
        UserResponse result = userService.getUserById(1L);
        
        // then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getUsername()).isEqualTo("testuser");
        assertThat(result.getEmail()).isEqualTo("test@example.com");
    }
    
    @Test
    @DisplayName("사용자 ID로 조회 실패 - 사용자 없음")
    void getUserById_NotFound() {
        // given
        given(userRepository.findById(999L)).willReturn(Optional.empty());
        
        // when & then
        assertThatThrownBy(() -> userService.getUserById(999L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("사용자를 찾을 수 없습니다");
    }
    
    @Test
    @DisplayName("사용자 생성 성공")
    void createUser_Success() {
        // given
        given(userRepository.existsByUsername("newuser")).willReturn(false);
        given(userRepository.existsByEmail("newuser@example.com")).willReturn(false);
        given(passwordEncoder.encode("password123")).willReturn("encodedPassword");
        given(userRepository.save(any(User.class))).willReturn(testUser);
        
        // when
        UserResponse result = userService.createUser(createUserRequest);
        
        // then
        assertThat(result).isNotNull();
        assertThat(result.getUsername()).isEqualTo("testuser");
        verify(userRepository).save(any(User.class));
    }
    
    @Test
    @DisplayName("사용자 생성 실패 - 중복 사용자명")
    void createUser_DuplicateUsername() {
        // given
        given(userRepository.existsByUsername("newuser")).willReturn(true);
        
        // when & then
        assertThatThrownBy(() -> userService.createUser(createUserRequest))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("이미 사용 중인 사용자명");
    }
    
    @Test
    @DisplayName("사용자 생성 실패 - 중복 이메일")
    void createUser_DuplicateEmail() {
        // given
        given(userRepository.existsByUsername("newuser")).willReturn(false);
        given(userRepository.existsByEmail("newuser@example.com")).willReturn(true);
        
        // when & then
        assertThatThrownBy(() -> userService.createUser(createUserRequest))
            .isInstanceOf(DuplicateResourceException.class)
            .hasMessageContaining("이미 사용 중인 이메일");
    }
}

// test/java/com/example/app/repository/UserRepositoryTest.java
package com.example.app.repository;

import com.example.app.entity.User;
import com.example.app.entity.UserRole;
import com.example.app.entity.UserStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.ActiveProfiles;

import java.time.LocalDateTime;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@ActiveProfiles("test")
@DisplayName("사용자 리포지토리 테스트")
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    private User testUser;
    
    @BeforeEach
    void setUp() {
        testUser = User.builder()
            .username("testuser")
            .email("test@example.com")
            .password("password")
            .firstName("Test")
            .lastName("User")
            .phone("010-1234-5678")
            .role(UserRole.USER)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .build();
    }
    
    @Test
    @DisplayName("사용자명으로 사용자 조회")
    void findByUsername_Success() {
        // given
        entityManager.persistAndFlush(testUser);
        
        // when
        Optional<User> found = userRepository.findByUsername("testuser");
        
        // then
        assertThat(found).isPresent();
        assertThat(found.get().getEmail()).isEqualTo("test@example.com");
    }
    
    @Test
    @DisplayName("이메일로 사용자 조회")
    void findByEmail_Success() {
        // given
        entityManager.persistAndFlush(testUser);
        
        // when
        Optional<User> found = userRepository.findByEmail("test@example.com");
        
        // then
        assertThat(found).isPresent();
        assertThat(found.get().getUsername()).isEqualTo("testuser");
    }
    
    @Test
    @DisplayName("사용자명 존재 여부 확인")
    void existsByUsername() {
        // given
        entityManager.persistAndFlush(testUser);
        
        // when & then
        assertThat(userRepository.existsByUsername("testuser")).isTrue();
        assertThat(userRepository.existsByUsername("nonexistent")).isFalse();
    }
    
    @Test
    @DisplayName("이메일 존재 여부 확인")
    void existsByEmail() {
        // given
        entityManager.persistAndFlush(testUser);
        
        // when & then
        assertThat(userRepository.existsByEmail("test@example.com")).isTrue();
        assertThat(userRepository.existsByEmail("nonexistent@example.com")).isFalse();
    }
    
    @Test
    @DisplayName("키워드로 사용자 검색")
    void findByKeyword() {
        // given
        entityManager.persistAndFlush(testUser);
        
        User anotherUser = User.builder()
            .username("john")
            .email("john@example.com")
            .password("password")
            .firstName("John")
            .lastName("Doe")
            .role(UserRole.USER)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .build();
        entityManager.persistAndFlush(anotherUser);
        
        // when
        Page<User> result = userRepository.findByKeyword("test", PageRequest.of(0, 10));
        
        // then
        assertThat(result.getContent()).hasSize(1);
        assertThat(result.getContent().get(0).getUsername()).isEqualTo("testuser");
    }
}
```

### 통합 테스트

```java
// test/java/com/example/app/controller/UserControllerIntegrationTest.java
package com.example.app.controller;

import com.example.app.dto.request.CreateUserRequest;
import com.example.app.dto.response.ApiResponse;
import com.example.app.dto.response.UserResponse;
import com.example.app.entity.User;
import com.example.app.entity.UserRole;
import com.example.app.entity.UserStatus;
import com.example.app.repository.UserRepository;
import com.example.app.security.JwtTokenProvider;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureWebMvcSecurity;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.context.WebApplicationContext;

import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureWebMvcSecurity
@ActiveProfiles("test")
@Transactional
@DisplayName("사용자 컨트롤러 통합 테스트")
class UserControllerIntegrationTest {
    
    @Autowired
    private WebApplicationContext context;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    private MockMvc mockMvc;
    private String adminToken;
    private String userToken;
    
    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders
            .webAppContextSetup(context)
            .apply(springSecurity())
            .build();
        
        // 관리자 사용자 생성
        User adminUser = User.builder()
            .username("admin")
            .email("admin@example.com")
            .password(passwordEncoder.encode("password"))
            .firstName("Admin")
            .lastName("User")
            .role(UserRole.ADMIN)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .build();
        userRepository.save(adminUser);
        adminToken = jwtTokenProvider.generateToken("admin");
        
        // 일반 사용자 생성
        User normalUser = User.builder()
            .username("user")
            .email("user@example.com")
            .password(passwordEncoder.encode("password"))
            .firstName("Normal")
            .lastName("User")
            .role(UserRole.USER)
            .status(UserStatus.ACTIVE)
            .emailVerified(true)
            .build();
        userRepository.save(normalUser);
        userToken = jwtTokenProvider.generateToken("user");
    }
    
    @Test
    @DisplayName("관리자로 사용자 목록 조회 성공")
    void getAllUsers_AsAdmin_Success() throws Exception {
        mockMvc.perform(get("/api/users")
                .header("Authorization", "Bearer " + adminToken)
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.content").isArray());
    }
    
    @Test
    @DisplayName("일반 사용자로 사용자 목록 조회 실패 - 권한 없음")
    void getAllUsers_AsUser_Forbidden() throws Exception {
        mockMvc.perform(get("/api/users")
                .header("Authorization", "Bearer " + userToken)
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isForbidden());
    }
    
    @Test
    @DisplayName("관리자로 사용자 생성 성공")
    void createUser_AsAdmin_Success() throws Exception {
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("newuser");
        request.setEmail("newuser@example.com");
        request.setPassword("password123!");
        request.setConfirmPassword("password123!");
        request.setFirstName("New");
        request.setLastName("User");
        request.setPhone("010-1234-5678");
        
        mockMvc.perform(post("/api/users")
                .header("Authorization", "Bearer " + adminToken)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpected(status().isCreated())
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data.username").value("newuser"));
    }
    
    @Test
    @DisplayName("유효성 검증 실패 - 비밀번호 형식 오류")
    void createUser_ValidationError_InvalidPassword() throws Exception {
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("newuser");
        request.setEmail("newuser@example.com");
        request.setPassword("weak");  // 약한 비밀번호
        request.setConfirmPassword("weak");
        request.setFirstName("New");
        request.setLastName("User");
        
        mockMvc.perform(post("/api/users")
                .header("Authorization", "Bearer " + adminToken)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.success").value(false))
            .andExpect(jsonPath("$.errorCode").value("VALIDATION_ERROR"));
    }
}

// test/java/com/example/app/security/JwtTokenProviderTest.java
package com.example.app.security;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

import java.util.Collections;

import static org.assertj.core.api.Assertions.*;

@DisplayName("JWT 토큰 프로바이더 테스트")
class JwtTokenProviderTest {
    
    private JwtTokenProvider jwtTokenProvider;
    private final String secret = "mySecretKey1234567890123456789012345678901234567890";
    private final long expiration = 86400000L; // 24시간
    
    @BeforeEach
    void setUp() {
        jwtTokenProvider = new JwtTokenProvider(secret, expiration);
    }
    
    @Test
    @DisplayName("인증 객체로 JWT 토큰 생성")
    void generateToken_WithAuthentication_Success() {
        // given
        CustomUserDetails userDetails = new CustomUserDetails(
            1L, "testuser", "test@example.com", "password",
            Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER")),
            true, true
        );
        Authentication authentication = new UsernamePasswordAuthenticationToken(
            userDetails, null, userDetails.getAuthorities());
        
        // when
        String token = jwtTokenProvider.generateToken(authentication);
        
        // then
        assertThat(token).isNotNull();
        assertThat(token).isNotEmpty();
        assertThat(jwtTokenProvider.validateToken(token)).isTrue();
        assertThat(jwtTokenProvider.getUsernameFromToken(token)).isEqualTo("testuser");
    }
    
    @Test
    @DisplayName("사용자명으로 JWT 토큰 생성")
    void generateToken_WithUsername_Success() {
        // given
        String username = "testuser";
        
        // when
        String token = jwtTokenProvider.generateToken(username);
        
        // then
        assertThat(token).isNotNull();
        assertThat(token).isNotEmpty();
        assertThat(jwtTokenProvider.validateToken(token)).isTrue();
        assertThat(jwtTokenProvider.getUsernameFromToken(token)).isEqualTo(username);
    }
    
    @Test
    @DisplayName("잘못된 토큰 검증 실패")
    void validateToken_InvalidToken_ReturnsFalse() {
        // given
        String invalidToken = "invalid.jwt.token";
        
        // when & then
        assertThat(jwtTokenProvider.validateToken(invalidToken)).isFalse();
    }
    
    @Test
    @DisplayName("빈 토큰 검증 실패")
    void validateToken_EmptyToken_ReturnsFalse() {
        // when & then
        assertThat(jwtTokenProvider.validateToken("")).isFalse();
        assertThat(jwtTokenProvider.validateToken(null)).isFalse();
    }
}
```

### TestContainers를 사용한 통합 테스트

```java
// test/java/com/example/app/IntegrationTestBase.java
package com.example.app;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
public abstract class IntegrationTestBase {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}

// test/java/com/example/app/ApplicationIntegrationTest.java
package com.example.app;

import com.example.app.repository.UserRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.*;

@DisplayName("애플리케이션 통합 테스트")
class ApplicationIntegrationTest extends IntegrationTestBase {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("애플리케이션 컨텍스트 로드 테스트")
    void contextLoads() {
        assertThat(userRepository).isNotNull();
    }
    
    @Test
    @Transactional
    @DisplayName("데이터베이스 연결 테스트")
    void databaseConnectionTest() {
        long count = userRepository.count();
        assertThat(count).isGreaterThanOrEqualTo(0);
    }
}
```

## 🐳 Docker 설정

### Dockerfile

```dockerfile
# Dockerfile
FROM eclipse-temurin:17-jre-alpine

# 작업 디렉토리 설정
WORKDIR /app

# 비root 사용자 생성
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# 애플리케이션 JAR 파일 복사
COPY build/libs/spring-boot-app.jar app.jar

# 파일 소유자 변경
RUN chown appuser:appgroup app.jar

# 비root 사용자로 전환
USER appuser

# 포트 노출
EXPOSE 8080

# 헬스체크
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# JVM 옵션 설정
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:+UseStringDeduplication"

# 애플리케이션 실행
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    container_name: spring-boot-app
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=jdbc:postgresql://postgres:5432/appdb
      - DATABASE_USERNAME=appuser
      - DATABASE_PASSWORD=apppassword
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - JWT_SECRET=mySecretKey1234567890123456789012345678901234567890
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    command: redis-server --appendonly yes

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### Docker 환경 설정

```yaml
# application-docker.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  redis:
    host: ${REDIS_HOST}
    port: ${REDIS_PORT}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0

  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: false
        show_sql: false

jwt:
  secret: ${JWT_SECRET}
  expiration: 86400000

logging:
  level:
    com.example.app: INFO
    org.springframework.security: WARN
    org.hibernate: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized
```

## 🗃 데이터베이스 마이그레이션

### Flyway 마이그레이션 파일

```sql
-- src/main/resources/db/migration/V1__Initial_schema.sql
-- 사용자 테이블
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    phone VARCHAR(20),
    role VARCHAR(20) NOT NULL DEFAULT 'USER',
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    email_verification_token VARCHAR(255),
    last_login_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);

-- 상품 테이블
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(19,2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    sku VARCHAR(100) UNIQUE,
    category VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    image_url VARCHAR(500),
    weight DECIMAL(8,2),
    dimensions VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);

-- 주문 테이블
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    total_amount DECIMAL(19,2) NOT NULL,
    shipping_address TEXT NOT NULL,
    billing_address TEXT,
    payment_method VARCHAR(50),
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);

-- 주문 아이템 테이블
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(19,2) NOT NULL,
    subtotal DECIMAL(19,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0
);

-- 인덱스 생성
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created_at ON users(created_at);

CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_name ON products(name);
CREATE INDEX idx_products_created_at ON products(created_at);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_order_number ON orders(order_number);
CREATE INDEX idx_orders_created_at ON orders(created_at);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- 트리거 생성 (updated_at 자동 업데이트)
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_products_updated_at BEFORE UPDATE ON products FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_orders_updated_at BEFORE UPDATE ON orders FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_order_items_updated_at BEFORE UPDATE ON order_items FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- src/main/resources/db/migration/V2__Insert_sample_data.sql
-- 관리자 사용자 생성
INSERT INTO users (username, email, password, first_name, last_name, role, status, email_verified)
VALUES ('admin', 'admin@example.com', '$2a$10$N.zmdr9k7uOCQb376NoUnuTJ8iYqiSfFw6l2x8L5z8hD6J3vY7mxy', 'Admin', 'User', 'ADMIN', 'ACTIVE', true);
-- 비밀번호: admin123!

-- 샘플 상품 데이터
INSERT INTO products (name, description, price, stock_quantity, sku, category, status)
VALUES 
('MacBook Pro 16"', 'Apple MacBook Pro 16-inch with M2 Pro chip', 2499.00, 10, 'MBP-16-M2', 'ELECTRONICS', 'ACTIVE'),
('iPhone 15 Pro', 'Latest iPhone with titanium design', 999.00, 25, 'IPH-15-PRO', 'ELECTRONICS', 'ACTIVE'),
('AirPods Pro', 'Wireless earbuds with active noise cancellation', 249.00, 50, 'APP-PRO-2', 'ELECTRONICS', 'ACTIVE'),
('Spring Boot in Action', 'Comprehensive guide to Spring Boot development', 49.99, 100, 'BOOK-SB-001', 'BOOKS', 'ACTIVE'),
('Java Concurrency in Practice', 'Essential book for Java developers', 59.99, 75, 'BOOK-JCP-001', 'BOOKS', 'ACTIVE');
```

### 데이터베이스 초기화 스크립트

```sql
-- scripts/init-db.sql
-- Docker 컴포즈에서 사용되는 DB 초기화 스크립트

-- 데이터베이스와 사용자가 이미 만들어져 있으므로 추가 설정만 수행

-- 확장 기능 활성화
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- 전체 텍스트 검색을 위한 인덱스 (향후 사용)
-- CREATE INDEX IF NOT EXISTS idx_products_search ON products USING gin(to_tsvector('english', name || ' ' || description));

GRANT ALL PRIVILEGES ON DATABASE appdb TO appuser;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO appuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO appuser;
```
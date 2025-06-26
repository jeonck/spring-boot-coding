# Spring Cloud Native Development

> Spring Cloud를 활용한 클라우드 네이티브 마이크로서비스 개발 완전 가이드

## 📋 목차

- [Spring Cloud 개요](#spring-cloud-개요)
- [서비스 디스커버리](#서비스-디스커버리)
- [구성 관리](#구성-관리)
- [API 게이트웨이](#api-게이트웨이)
- [로드 밸런싱](#로드-밸런싱)
- [회로 차단기](#회로-차단기)
- [분산 추적](#분산-추적)
- [메시지 스트리밍](#메시지-스트리밍)
- [보안 통합](#보안-통합)

## ☁️ Spring Cloud 개요

### Spring Cloud 생태계

```yaml
Spring Cloud 핵심 프로젝트:
  Service Discovery:
    - Spring Cloud Netflix Eureka
    - Spring Cloud Consul
    - Spring Cloud Zookeeper
  
  Configuration:
    - Spring Cloud Config
    - Spring Cloud Vault
    - Spring Cloud Kubernetes
  
  Gateway:
    - Spring Cloud Gateway
    - Spring Cloud OpenFeign
  
  Circuit Breaker:
    - Spring Cloud CircuitBreaker
    - Resilience4j
  
  Observability:
    - Spring Cloud Sleuth
    - Micrometer Tracing
  
  Messaging:
    - Spring Cloud Stream
    - Spring Cloud Function
  
  Security:
    - Spring Cloud Security
    - Spring Cloud OAuth2
```

### 의존성 설정

```gradle
// BOM (Bill of Materials) 설정
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.0"
    }
}

dependencies {
    // Spring Boot 기본
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    // Service Discovery
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    implementation 'org.springframework.cloud:spring-cloud-starter-consul-discovery'
    
    // Configuration
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.cloud:spring-cloud-starter-vault-config'
    
    // API Gateway
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    
    // Load Balancing
    implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
    
    // Circuit Breaker
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    
    // Tracing
    implementation 'org.springframework.cloud:spring-cloud-starter-sleuth'
    implementation 'org.springframework.cloud:spring-cloud-sleuth-zipkin'
    
    // Messaging
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-kafka'
    
    // Security
    implementation 'org.springframework.cloud:spring-cloud-starter-security'
    implementation 'org.springframework.cloud:spring-cloud-starter-oauth2'
    
    // Testing
    testImplementation 'org.springframework.cloud:spring-cloud-starter-contract-verifier'
}
```

## 🔍 서비스 디스커버리

### Eureka 서버 구성

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# eureka-server/application.yml
spring:
  application:
    name: eureka-server
  profiles:
    active: standalone

server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 10000

---
spring:
  profiles: peer1
  application:
    name: eureka-server

server:
  port: 8761

eureka:
  instance:
    hostname: peer1
  client:
    service-url:
      defaultZone: http://peer2:8762/eureka/

---
spring:
  profiles: peer2
  application:
    name: eureka-server

server:
  port: 8762

eureka:
  instance:
    hostname: peer2
  client:
    service-url:
      defaultZone: http://peer1:8761/eureka/
```

### Eureka 클라이언트 구성

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class ProductServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```yaml
# product-service/application.yml
spring:
  application:
    name: product-service
  profiles:
    active: development

server:
  port: 0  # 랜덤 포트

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    healthcheck:
      enabled: true
    fetch-registry-with-delta: true
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
    metadata-map:
      zone: zone1
      version: 1.0.0
      environment: ${spring.profiles.active}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
  info:
    env:
      enabled: true
```

### 서비스 디스커버리 활용

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class ProductController {
    
    private final DiscoveryClient discoveryClient;
    private final RestTemplate restTemplate;
    private final LoadBalancerClient loadBalancerClient;
    
    @GetMapping("/api/products")
    public ResponseEntity<List<Product>> getProducts() {
        // 서비스 인스턴스 조회
        List<ServiceInstance> instances = discoveryClient.getInstances("inventory-service");
        
        if (instances.isEmpty()) {
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).build();
        }
        
        // 로드 밸런싱을 통한 서비스 호출
        String inventoryUrl = "http://inventory-service/api/inventory/check";
        boolean inventoryAvailable = restTemplate.getForObject(inventoryUrl, Boolean.class);
        
        List<Product> products = getProductsFromDatabase();
        
        if (inventoryAvailable) {
            products = enrichWithInventoryInfo(products);
        }
        
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/api/products/discovery-info")
    public ResponseEntity<Map<String, Object>> getDiscoveryInfo() {
        Map<String, Object> info = new HashMap<>();
        
        // 모든 서비스 조회
        List<String> services = discoveryClient.getServices();
        info.put("services", services);
        
        // 각 서비스의 인스턴스 정보
        Map<String, List<ServiceInstance>> serviceInstances = new HashMap<>();
        for (String service : services) {
            List<ServiceInstance> instances = discoveryClient.getInstances(service);
            serviceInstances.put(service, instances);
        }
        info.put("serviceInstances", serviceInstances);
        
        return ResponseEntity.ok(info);
    }
    
    @GetMapping("/api/products/call-with-loadbalancer")
    public ResponseEntity<String> callWithLoadBalancer() {
        // 수동 로드 밸런싱
        ServiceInstance instance = loadBalancerClient.choose("user-service");
        
        if (instance == null) {
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body("user-service not available");
        }
        
        String url = String.format("http://%s:%s/api/users/count", 
            instance.getHost(), instance.getPort());
        
        String userCount = restTemplate.getForObject(url, String.class);
        
        return ResponseEntity.ok("User count from " + instance.getInstanceId() + ": " + userCount);
    }
    
    private List<Product> getProductsFromDatabase() {
        // 데이터베이스에서 상품 조회
        return Arrays.asList(
            new Product("1", "Product 1"),
            new Product("2", "Product 2")
        );
    }
    
    private List<Product> enrichWithInventoryInfo(List<Product> products) {
        // 재고 정보로 상품 정보 보강
        return products.stream()
            .map(product -> {
                String inventoryUrl = "http://inventory-service/api/inventory/" + product.getId();
                Integer stock = restTemplate.getForObject(inventoryUrl, Integer.class);
                product.setStockQuantity(stock);
                return product;
            })
            .collect(Collectors.toList());
    }
}
```

### Consul 서비스 디스커버리

```java
@Configuration
@EnableConsulServiceRegistry
public class ConsulConfig {
    
    @Bean
    public ConsulServiceRegistryAutoConfiguration consulServiceRegistryAutoConfiguration() {
        return new ConsulServiceRegistryAutoConfiguration();
    }
    
    @Bean
    public NewRelicMeterRegistry meterRegistry() {
        return NewRelicMeterRegistry.builder(NewRelicConfig.DEFAULT).build();
    }
}
```

```yaml
# consul 설정
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        enabled: true
        prefer-ip-address: true
        health-check-path: /actuator/health
        health-check-interval: 10s
        health-check-timeout: 10s
        health-check-critical-timeout: 30s
        instance-id: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
        service-name: ${spring.application.name}
        metadata:
          version: 1.0.0
          environment: ${spring.profiles.active}
        tags:
          - version=1.0.0
          - environment=${spring.profiles.active}
      config:
        enabled: true
        prefix: config
        default-context: application
        profile-separator: ","
        data-key: data
        format: yaml
        watch:
          enabled: true
          delay: 1000
```

## ⚙️ 구성 관리

### Spring Cloud Config 서버

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config-server/application.yml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          clone-on-start: true
          force-pull: true
          search-paths:
            - '{application}'
            - '{application}/{profile}'
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
        default-label: main
        health:
          enabled: true
          repositories:
            - name: product-service
              profiles: development,production
        encrypt:
          enabled: true

server:
  port: 8888

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always

encrypt:
  key: ${ENCRYPT_KEY:my-secret-key}
```

### Config 클라이언트 설정

```java
@Configuration
@EnableConfigurationProperties
public class ConfigClientConfig {
    
    @Bean
    @RefreshScope
    public DatabaseConfig databaseConfig() {
        return new DatabaseConfig();
    }
    
    @ConfigurationProperties(prefix = "app.database")
    @RefreshScope
    @Data
    public static class DatabaseConfig {
        private String url;
        private String username;
        private String password;
        private int maxPoolSize;
        private int minPoolSize;
        private long connectionTimeout;
    }
}

@RestController
@RefreshScope
@RequiredArgsConstructor
public class ConfigTestController {
    
    private final ConfigClientConfig.DatabaseConfig databaseConfig;
    
    @Value("${app.message:Default Message}")
    private String message;
    
    @Value("${app.feature.enabled:false}")
    private boolean featureEnabled;
    
    @GetMapping("/api/config/test")
    public ResponseEntity<Map<String, Object>> getConfig() {
        Map<String, Object> config = new HashMap<>();
        config.put("message", message);
        config.put("featureEnabled", featureEnabled);
        config.put("databaseConfig", databaseConfig);
        
        return ResponseEntity.ok(config);
    }
    
    @PostMapping("/api/config/refresh")
    public ResponseEntity<String> refresh() {
        // 설정 새로고침은 Spring Cloud Bus나 Actuator를 통해 수행
        return ResponseEntity.ok("Configuration refresh triggered");
    }
}
```

```yaml
# product-service/bootstrap.yml
spring:
  application:
    name: product-service
  profiles:
    active: development
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
        max-interval: 2000
        multiplier: 1.1
      discovery:
        enabled: true
        service-id: config-server
      username: ${CONFIG_SERVER_USERNAME:}
      password: ${CONFIG_SERVER_PASSWORD:}

management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
```

### 암호화된 설정 관리

```bash
# 설정 암호화
curl -X POST http://localhost:8888/encrypt \
  -d "mysecretpassword" \
  -H "Content-Type: text/plain"

# 결과: AQA4JlKj9X8VgzX...
```

```yaml
# config-repo/product-service-development.yml
app:
  message: "Development Environment"
  feature:
    enabled: true
    new-ui: false
  
  database:
    url: jdbc:postgresql://localhost:5432/product_dev
    username: dev_user
    password: '{cipher}AQA4JlKj9X8VgzX...'  # 암호화된 비밀번호
    max-pool-size: 10
    min-pool-size: 2
    connection-timeout: 30000

  external-api:
    url: https://api-dev.example.com
    api-key: '{cipher}BBX7YmPr4Z9WfzN...'  # 암호화된 API 키
    timeout: 5000

  security:
    jwt:
      secret: '{cipher}CCA2NqKm2L5XthQ...'  # 암호화된 JWT 시크릿
      expiration: 86400000

logging:
  level:
    com.example.product: DEBUG
    org.springframework.cloud: DEBUG
```

### Vault 통합

```java
@Configuration
@EnableVaultRepositories
public class VaultConfig {
    
    @Bean
    public VaultTemplate vaultTemplate() {
        return new VaultTemplate(vaultEndpoint(), clientAuthentication());
    }
    
    @Bean
    public VaultEndpoint vaultEndpoint() {
        return VaultEndpoint.create("localhost", 8200);
    }
    
    @Bean
    public ClientAuthentication clientAuthentication() {
        return new TokenAuthentication("your-vault-token");
    }
}

@Service
@RequiredArgsConstructor
public class VaultSecretService {
    
    private final VaultTemplate vaultTemplate;
    
    public String getSecret(String path, String key) {
        VaultResponse response = vaultTemplate.read(path);
        
        if (response != null && response.getData() != null) {
            return (String) response.getData().get(key);
        }
        
        throw new RuntimeException("Secret not found: " + path + "/" + key);
    }
    
    public void writeSecret(String path, Map<String, Object> data) {
        vaultTemplate.write(path, data);
    }
    
    public Map<String, Object> getAllSecrets(String path) {
        VaultResponse response = vaultTemplate.read(path);
        return response != null ? response.getData() : Collections.emptyMap();
    }
}
```

## 🚪 API 게이트웨이

### Spring Cloud Gateway 설정

```java
@SpringBootApplication
@EnableEurekaClient
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}

@Configuration
@RequiredArgsConstructor
public class GatewayConfig {
    
    private final AuthenticationFilter authenticationFilter;
    private final RateLimitFilter rateLimitFilter;
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // 상품 서비스 라우팅
            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .filter(authenticationFilter)
                    .filter(rateLimitFilter)
                    .addRequestHeader("X-Gateway", "spring-cloud-gateway")
                    .addResponseHeader("X-Response-Time", String.valueOf(System.currentTimeMillis()))
                    .circuitBreaker(c -> c
                        .setName("product-cb")
                        .setFallbackUri("forward:/fallback/products")))
                .uri("lb://product-service"))
            
            // 사용자 서비스 라우팅
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .filter(authenticationFilter)
                    .requestRateLimiter(rl -> rl
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://user-service"))
            
            // WebSocket 라우팅
            .route("websocket-service", r -> r
                .path("/ws/**")
                .uri("lb:ws://notification-service"))
            
            // 정적 리소스 라우팅
            .route("static-resources", r -> r
                .path("/static/**")
                .filters(f -> f
                    .setPath("/resources")
                    .addResponseHeader("Cache-Control", "max-age=3600"))
                .uri("lb://static-service"))
            
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(10, 20, 1); // replenishRate, burstCapacity, requestedTokens
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> exchange.getPrincipal()
            .map(principal -> principal.getName())
            .switchIfEmpty(Mono.just("anonymous"));
    }
    
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
    }
}
```

### 커스텀 필터 구현

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AuthenticationFilter implements GatewayFilter, Ordered {
    
    private final JwtTokenProvider jwtTokenProvider;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 인증이 필요 없는 경로
        if (isPublicPath(request.getPath().value())) {
            return chain.filter(exchange);
        }
        
        String token = extractToken(request);
        
        if (token == null || !jwtTokenProvider.validateToken(token)) {
            return unauthorized(exchange);
        }
        
        // 토큰에서 사용자 정보 추출
        String userId = jwtTokenProvider.getUserId(token);
        List<String> roles = jwtTokenProvider.getRoles(token);
        
        // 요청에 사용자 정보 추가
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-User-ID", userId)
            .header("X-User-Roles", String.join(",", roles))
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    private boolean isPublicPath(String path) {
        List<String> publicPaths = Arrays.asList(
            "/api/auth/login",
            "/api/auth/register",
            "/api/health",
            "/actuator/health"
        );
        
        return publicPaths.stream().anyMatch(path::startsWith);
    }
    
    private String extractToken(ServerHttpRequest request) {
        String authorization = request.getHeaders().getFirst("Authorization");
        if (authorization != null && authorization.startsWith("Bearer ")) {
            return authorization.substring(7);
        }
        return null;
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        
        String body = "{\"error\":\"Unauthorized\",\"message\":\"Authentication required\"}";
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes());
        
        return response.writeWith(Mono.just(buffer));
    }
    
    @Override
    public int getOrder() {
        return -100; // 높은 우선순위
    }
}

@Component
@Slf4j
public class LoggingFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String path = exchange.getRequest().getPath().value();
        String method = exchange.getRequest().getMethodValue();
        
        log.info("Incoming request: {} {}", method, path);
        
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                long duration = System.currentTimeMillis() - startTime;
                HttpStatus statusCode = exchange.getResponse().getStatusCode();
                
                log.info("Completed request: {} {} - {} ({}ms)", 
                    method, path, statusCode, duration);
            }));
    }
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}

@Component
@RequiredArgsConstructor
public class CorrelationIdFilter implements GlobalFilter, Ordered {
    
    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String correlationId = request.getHeaders().getFirst(CORRELATION_ID_HEADER);
        
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        
        // 요청에 상관관계 ID 추가
        ServerHttpRequest modifiedRequest = request.mutate()
            .header(CORRELATION_ID_HEADER, correlationId)
            .build();
        
        // 응답에도 상관관계 ID 추가
        ServerHttpResponse response = exchange.getResponse();
        response.getHeaders().add(CORRELATION_ID_HEADER, correlationId);
        
        // MDC에 상관관계 ID 설정
        return chain.filter(exchange.mutate().request(modifiedRequest).build())
            .contextWrite(Context.of("correlationId", correlationId));
    }
    
    @Override
    public int getOrder() {
        return -200; // 가장 높은 우선순위
    }
}
```

### 폴백 컨트롤러

```java
@RestController
@RequestMapping("/fallback")
@Slf4j
public class FallbackController {
    
    @GetMapping("/products")
    public ResponseEntity<Map<String, Object>> productFallback(ServerHttpRequest request) {
        log.warn("Product service fallback triggered for: {}", request.getPath());
        
        Map<String, Object> fallbackResponse = Map.of(
            "message", "Product service is temporarily unavailable",
            "status", "fallback",
            "timestamp", Instant.now(),
            "data", Collections.emptyList()
        );
        
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(fallbackResponse);
    }
    
    @GetMapping("/users")
    public ResponseEntity<Map<String, Object>> userFallback(ServerHttpRequest request) {
        log.warn("User service fallback triggered for: {}", request.getPath());
        
        Map<String, Object> fallbackResponse = Map.of(
            "message", "User service is temporarily unavailable",
            "status", "fallback",
            "timestamp", Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(fallbackResponse);
    }
    
    @GetMapping("/generic")
    public ResponseEntity<Map<String, Object>> genericFallback(ServerHttpRequest request) {
        log.warn("Generic fallback triggered for: {}", request.getPath());
        
        Map<String, Object> fallbackResponse = Map.of(
            "message", "Service is temporarily unavailable",
            "status", "fallback",
            "timestamp", Instant.now(),
            "retryAfter", 30
        );
        
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(fallbackResponse);
    }
}
```

## ⚖️ 로드 밸런싱

### LoadBalancer 설정

```java
@Configuration
public class LoadBalancerConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
    
    // 커스텀 로드 밸런서 설정
    @Bean
    public ReactorLoadBalancer<ServiceInstance> userServiceLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
    
    // 건강하지 않은 인스턴스 필터링
    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withHealthChecks()
            .withCaching()
            .build(context);
    }
}

// 커스텀 로드 밸런싱 규칙
@Component
public class WeightedLoadBalancer implements ReactorServiceInstanceLoadBalancer {
    
    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
    private final String serviceId;
    private final Random random = new Random();
    
    public WeightedLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
                               String serviceId) {
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
        this.serviceId = serviceId;
    }
    
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
            .getIfAvailable(NoopServiceInstanceListSupplier::new);
        
        return supplier.get(request)
            .next()
            .map(this::chooseInstanceByWeight);
    }
    
    private Response<ServiceInstance> chooseInstanceByWeight(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }
        
        // 가중치 기반 선택
        int totalWeight = instances.stream()
            .mapToInt(instance -> {
                String weight = instance.getMetadata().get("weight");
                return weight != null ? Integer.parseInt(weight) : 1;
            })
            .sum();
        
        int randomWeight = random.nextInt(totalWeight);
        int currentWeight = 0;
        
        for (ServiceInstance instance : instances) {
            String weight = instance.getMetadata().get("weight");
            currentWeight += weight != null ? Integer.parseInt(weight) : 1;
            
            if (randomWeight < currentWeight) {
                return new DefaultResponse(instance);
            }
        }
        
        return new DefaultResponse(instances.get(0));
    }
}
```

### OpenFeign 클라이언트

```java
@FeignClient(
    name = "user-service",
    fallbackFactory = UserServiceFallbackFactory.class,
    configuration = FeignConfiguration.class
)
public interface UserServiceClient {
    
    @GetMapping("/api/users/{id}")
    UserDto getUser(@PathVariable("id") String id);
    
    @GetMapping("/api/users")
    List<UserDto> getAllUsers(@RequestParam("page") int page, 
                             @RequestParam("size") int size);
    
    @PostMapping("/api/users")
    UserDto createUser(@RequestBody CreateUserRequest request);
    
    @PutMapping("/api/users/{id}")
    UserDto updateUser(@PathVariable("id") String id, 
                      @RequestBody UpdateUserRequest request);
    
    @DeleteMapping("/api/users/{id}")
    void deleteUser(@PathVariable("id") String id);
    
    @GetMapping("/api/users/search")
    List<UserDto> searchUsers(@RequestParam("name") String name);
}

@Component
public class UserServiceFallbackFactory implements FallbackFactory<UserServiceClient> {
    
    @Override
    public UserServiceClient create(Throwable cause) {
        return new UserServiceFallback(cause);
    }
}

@Slf4j
public class UserServiceFallback implements UserServiceClient {
    
    private final Throwable cause;
    
    public UserServiceFallback(Throwable cause) {
        this.cause = cause;
    }
    
    @Override
    public UserDto getUser(String id) {
        log.error("Fallback for getUser with id: {}", id, cause);
        return UserDto.builder()
            .id(id)
            .name("Unavailable")
            .email("unavailable@example.com")
            .build();
    }
    
    @Override
    public List<UserDto> getAllUsers(int page, int size) {
        log.error("Fallback for getAllUsers", cause);
        return Collections.emptyList();
    }
    
    @Override
    public UserDto createUser(CreateUserRequest request) {
        log.error("Fallback for createUser", cause);
        throw new ServiceUnavailableException("User service is temporarily unavailable");
    }
    
    @Override
    public UserDto updateUser(String id, UpdateUserRequest request) {
        log.error("Fallback for updateUser with id: {}", id, cause);
        throw new ServiceUnavailableException("User service is temporarily unavailable");
    }
    
    @Override
    public void deleteUser(String id) {
        log.error("Fallback for deleteUser with id: {}", id, cause);
        throw new ServiceUnavailableException("User service is temporarily unavailable");
    }
    
    @Override
    public List<UserDto> searchUsers(String name) {
        log.error("Fallback for searchUsers with name: {}", name, cause);
        return Collections.emptyList();
    }
}

@Configuration
public class FeignConfiguration {
    
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    
    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(1000, 2000, 3);
    }
    
    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            // JWT 토큰 전파
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
            if (authentication != null && authentication.getCredentials() instanceof String) {
                String token = (String) authentication.getCredentials();
                requestTemplate.header("Authorization", "Bearer " + token);
            }
            
            // 상관관계 ID 전파
            String correlationId = MDC.get("correlationId");
            if (correlationId != null) {
                requestTemplate.header("X-Correlation-ID", correlationId);
            }
            
            // 요청 ID 추가
            requestTemplate.header("X-Request-ID", UUID.randomUUID().toString());
        };
    }
    
    @Bean
    public Contract feignContract() {
        return new SpringMvcContract();
    }
    
    @Bean
    public Decoder feignDecoder() {
        return new JacksonDecoder();
    }
    
    @Bean
    public Encoder feignEncoder() {
        return new JacksonEncoder();
    }
}
```

## ⚡ 회로 차단기

### Resilience4j 설정

```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreaker userServiceCircuitBreaker() {
        return CircuitBreaker.ofDefaults("user-service");
    }
    
    @Bean
    public Customizer<Resilience4JConfigBuilder> globalCustomConfiguration() {
        return builder -> builder
            .circuitBreakerAspectOrder(398)
            .retryAspectOrder(399)
            .bulkheadAspectOrder(400)
            .timeLimiterAspectOrder(401);
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {
    
    private final UserServiceClient userServiceClient;
    
    @CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
    @Retry(name = "user-service")
    @TimeLimiter(name = "user-service")
    @Bulkhead(name = "user-service")
    public CompletableFuture<UserDto> getUser(String id) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("Calling user service for id: {}", id);
            return userServiceClient.getUser(id);
        });
    }
    
    public CompletableFuture<UserDto> getUserFallback(String id, Exception ex) {
        log.error("Fallback executed for user id: {}", id, ex);
        
        UserDto fallbackUser = UserDto.builder()
            .id(id)
            .name("Fallback User")
            .email("fallback@example.com")
            .build();
        
        return CompletableFuture.completedFuture(fallbackUser);
    }
    
    @CircuitBreaker(name = "user-service")
    @Retry(name = "user-service", fallbackMethod = "createUserFallback")
    public UserDto createUser(CreateUserRequest request) {
        log.info("Creating user: {}", request.getEmail());
        return userServiceClient.createUser(request);
    }
    
    public UserDto createUserFallback(CreateUserRequest request, Exception ex) {
        log.error("Create user fallback executed for: {}", request.getEmail(), ex);
        throw new ServiceUnavailableException("User creation is temporarily unavailable");
    }
}
```

```yaml
# application.yml - Resilience4j 설정
resilience4j:
  circuitbreaker:
    instances:
      user-service:
        register-health-indicator: true
        sliding-window-size: 10
        permitted-number-of-calls-in-half-open-state: 3
        sliding-window-type: TIME_BASED
        minimum-number-of-calls: 5
        wait-duration-in-open-state: 30s
        failure-rate-threshold: 50
        event-consumer-buffer-size: 10
        record-exceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.io.IOException
        ignore-exceptions:
          - java.lang.IllegalArgumentException
  
  retry:
    instances:
      user-service:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - java.lang.IllegalArgumentException
  
  timelimiter:
    instances:
      user-service:
        timeout-duration: 2s
        cancel-running-future: true
  
  bulkhead:
    instances:
      user-service:
        max-concurrent-calls: 20
        max-wait-duration: 1s

management:
  health:
    circuitbreakers:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        resilience4j.circuitbreaker.calls: true
```

## 🔍 분산 추적

### Spring Cloud Sleuth 설정

```java
@Configuration
public class TracingConfig {
    
    @Bean
    public Sampler alwaysSampler() {
        return Sampler.create(1.0f); // 100% 샘플링
    }
    
    @Bean
    public SpanCustomizer spanCustomizer() {
        return span -> span.tag("service.version", "1.0.0");
    }
    
    @NewSpan("order-processing")
    @GetMapping("/api/orders/{id}/process")
    public ResponseEntity<OrderDto> processOrder(@PathVariable String id) {
        return orderService.processOrder(id);
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {
    
    private final UserServiceClient userServiceClient;
    private final PaymentServiceClient paymentServiceClient;
    private final TraceContext.Injector<Map<String, String>> injector;
    
    @NewSpan("get-order")
    public OrderDto getOrder(@SpanTag("order.id") String orderId) {
        Span span = tracer.nextSpan().name("database-query")
            .tag("db.type", "postgresql")
            .tag("db.statement", "SELECT * FROM orders WHERE id = ?")
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // 데이터베이스 조회
            OrderDto order = findOrderById(orderId);
            
            span.tag("order.status", order.getStatus());
            span.tag("order.amount", order.getAmount().toString());
            
            return order;
        } finally {
            span.end();
        }
    }
    
    @NewSpan("process-order")
    public ResponseEntity<OrderDto> processOrder(@SpanTag("order.id") String orderId) {
        // 1. 주문 조회
        OrderDto order = getOrder(orderId);
        
        // 2. 사용자 정보 조회 (분산 추적)
        UserDto user = userServiceClient.getUser(order.getUserId());
        
        // 3. 결제 처리 (분산 추적)
        PaymentResult paymentResult = paymentServiceClient.processPayment(
            order.getId(), order.getAmount());
        
        // 4. 주문 상태 업데이트
        if (paymentResult.isSuccess()) {
            order.setStatus("COMPLETED");
            order = updateOrder(order);
        }
        
        return ResponseEntity.ok(order);
    }
    
    @ContinueSpan
    public OrderDto updateOrder(OrderDto order) {
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            currentSpan.tag("order.previous_status", order.getStatus());
            currentSpan.tag("order.new_status", "COMPLETED");
            currentSpan.annotate("order.status.updated");
        }
        
        // 데이터베이스 업데이트
        return saveOrder(order);
    }
    
    private OrderDto findOrderById(String orderId) {
        // 데이터베이스 조회 시뮬레이션
        return OrderDto.builder()
            .id(orderId)
            .userId("user-123")
            .amount(BigDecimal.valueOf(100.00))
            .status("PENDING")
            .build();
    }
    
    private OrderDto saveOrder(OrderDto order) {
        // 데이터베이스 저장 시뮬레이션
        return order;
    }
}
```

### 커스텀 추적 정보

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CustomTraceEnhancer {
    
    private final Tracer tracer;
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        Span span = tracer.nextSpan()
            .name("order-created-event")
            .tag("event.type", "OrderCreated")
            .tag("order.id", event.getOrderId())
            .tag("customer.id", event.getCustomerId())
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // 이벤트 처리 로직
            processOrderCreatedEvent(event);
            
            span.annotate("event.processed");
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
    
    @Async
    @NewSpan("async-notification")
    public void sendNotificationAsync(@SpanTag("user.id") String userId, 
                                    @SpanTag("notification.type") String type) {
        
        // 비동기 알림 발송
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            currentSpan.tag("notification.channel", "email");
            currentSpan.tag("notification.template", "order-confirmation");
        }
        
        // 실제 알림 발송 로직
        sendNotification(userId, type);
    }
    
    private void processOrderCreatedEvent(OrderCreatedEvent event) {
        // 이벤트 처리 로직
        log.info("Processing order created event: {}", event.getOrderId());
    }
    
    private void sendNotification(String userId, String type) {
        // 알림 발송 로직
        log.info("Sending {} notification to user: {}", type, userId);
    }
}
```

```yaml
# tracing 설정
spring:
  sleuth:
    zipkin:
      base-url: http://zipkin-server:9411
    sampler:
      probability: 1.0
    trace-id128: true
    web:
      client:
        enabled: true
      servlet:
        enabled: true
    async:
      enabled: true
    scheduled:
      enabled: true
    jdbc:
      enabled: true
      includes:
        - connection
        - query
    redis:
      enabled: true
    kafka:
      enabled: true

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level [%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n"
```

## 📚 관련 노트

- [[Microservices Architecture Patterns]] - 마이크로서비스 아키텍처
- [[Spring Boot Docker Kubernetes Advanced]] - 컨테이너 배포
- [[API Design Patterns]] - API 게이트웨이 설계
- [[Observability]] - 분산 시스템 모니터링
- [[Security Patterns]] - 분산 시스템 보안

## 🔗 외부 리소스

- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Netflix OSS](https://netflix.github.io/)
- [Consul by HashiCorp](https://www.consul.io/)
- [Resilience4j Documentation](https://resilience4j.readme.io/)

## 📨 메시지 스트리밍

### Spring Cloud Stream 설정

```java
@SpringBootApplication
@EnableBinding({
    Source.class,
    Sink.class,
    OrderProcessor.class
})
public class StreamApplication {
    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }
}

// 커스텀 채널 정의
public interface OrderProcessor {
    
    String ORDER_INPUT = "orderInput";
    String ORDER_OUTPUT = "orderOutput";
    String PAYMENT_OUTPUT = "paymentOutput";
    
    @Input(ORDER_INPUT)
    SubscribableChannel orderInput();
    
    @Output(ORDER_OUTPUT)
    MessageChannel orderOutput();
    
    @Output(PAYMENT_OUTPUT)
    MessageChannel paymentOutput();
}
```

```yaml
# application.yml - Spring Cloud Stream 설정
spring:
  cloud:
    stream:
      bindings:
        # 주문 입력 채널
        orderInput:
          destination: orders
          group: order-service
          consumer:
            max-attempts: 3
            back-off-initial-interval: 1000
            back-off-max-interval: 10000
            back-off-multiplier: 2.0
        
        # 주문 출력 채널
        orderOutput:
          destination: orders
          producer:
            partition-key-expression: payload.customerId
            partition-count: 3
        
        # 결제 출력 채널
        paymentOutput:
          destination: payments
          producer:
            required-groups: payment-service,audit-service
      
      # RabbitMQ 설정
      rabbit:
        bindings:
          orderInput:
            consumer:
              acknowledge-mode: manual
              durability: true
              ttl: 300000
              max-length: 1000
          orderOutput:
            producer:
              delivery-mode: PERSISTENT
              mandatory: true
      
      # Kafka 설정
      kafka:
        binder:
          brokers: localhost:9092
          zkNodes: localhost:2181
          auto-create-topics: false
          auto-add-partitions: false
        bindings:
          orderInput:
            consumer:
              auto-offset-reset: earliest
              enable-dlq: true
              dlq-name: orders-dlq
          orderOutput:
            producer:
              sync: true
              batch-timeout: 100
```

### 메시지 프로듀서

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventPublisher {
    
    private final OrderProcessor orderProcessor;
    private final MessageChannel orderOutput;
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        Message<OrderCreatedEvent> message = MessageBuilder
            .withPayload(event)
            .setHeader("eventType", "ORDER_CREATED")
            .setHeader("version", "1.0")
            .setHeader("timestamp", Instant.now())
            .setHeader("correlationId", event.getCorrelationId())
            .setHeader("partitionKey", event.getCustomerId())
            .build();
        
        boolean sent = orderOutput.send(message);
        
        if (sent) {
            log.info("Order created event published: {}", event.getOrderId());
        } else {
            log.error("Failed to publish order created event: {}", event.getOrderId());
            throw new EventPublishException("Failed to publish order created event");
        }
    }
    
    public void publishOrderStatusChanged(OrderStatusChangedEvent event) {
        Message<OrderStatusChangedEvent> message = MessageBuilder
            .withPayload(event)
            .setHeader("eventType", "ORDER_STATUS_CHANGED")
            .setHeader("version", "1.0")
            .setHeader("timestamp", Instant.now())
            .setHeader("correlationId", event.getCorrelationId())
            .build();
        
        orderProcessor.orderOutput().send(message);
        log.info("Order status changed event published: {} -> {}", 
                event.getOrderId(), event.getNewStatus());
    }
    
    @Async
    public CompletableFuture<Void> publishPaymentRequested(PaymentRequestedEvent event) {
        return CompletableFuture.runAsync(() -> {
            Message<PaymentRequestedEvent> message = MessageBuilder
                .withPayload(event)
                .setHeader("eventType", "PAYMENT_REQUESTED")
                .setHeader("priority", "HIGH")
                .setHeader("timeout", 30000) // 30초 타임아웃
                .build();
            
            orderProcessor.paymentOutput().send(message);
            log.info("Payment requested event published: {}", event.getOrderId());
        });
    }
}

// 커스텀 메시지 변환기
@Component
public class OrderEventMessageConverter implements MessageConverter {
    
    private final ObjectMapper objectMapper;
    
    public OrderEventMessageConverter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }
    
    @Override
    public Object fromMessage(Message<?> message, Class<?> targetClass) {
        try {
            if (message.getPayload() instanceof byte[]) {
                byte[] payload = (byte[]) message.getPayload();
                return objectMapper.readValue(payload, targetClass);
            }
            return message.getPayload();
        } catch (Exception e) {
            throw new MessageConversionException("Failed to convert message", e);
        }
    }
    
    @Override
    public Message<?> toMessage(Object payload, MessageHeaders headers) {
        try {
            byte[] bytes = objectMapper.writeValueAsBytes(payload);
            return MessageBuilder.withPayload(bytes)
                .copyHeaders(headers)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build();
        } catch (Exception e) {
            throw new MessageConversionException("Failed to convert payload", e);
        }
    }
}
```

### 메시지 컨슈머

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventConsumer {
    
    private final OrderService orderService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    
    @StreamListener(OrderProcessor.ORDER_INPUT)
    public void handleOrderCreated(@Payload OrderCreatedEvent event,
                                 @Header Map<String, Object> headers) {
        
        String correlationId = (String) headers.get("correlationId");
        String eventType = (String) headers.get("eventType");
        
        log.info("Received order event - Type: {}, OrderId: {}, CorrelationId: {}", 
                eventType, event.getOrderId(), correlationId);
        
        try {
            // MDC에 상관관계 ID 설정
            MDC.put("correlationId", correlationId);
            
            switch (eventType) {
                case "ORDER_CREATED":
                    processOrderCreated(event);
                    break;
                case "ORDER_STATUS_CHANGED":
                    processOrderStatusChanged((OrderStatusChangedEvent) event);
                    break;
                default:
                    log.warn("Unknown event type: {}", eventType);
            }
            
        } catch (Exception e) {
            log.error("Error processing order event: {}", event.getOrderId(), e);
            throw e; // 재시도를 위해 예외 다시 던지기
        } finally {
            MDC.clear();
        }
    }
    
    @StreamListener(value = OrderProcessor.ORDER_INPUT, condition = "headers['eventType']=='ORDER_PAYMENT_FAILED'")
    public void handlePaymentFailed(@Payload PaymentFailedEvent event) {
        log.warn("Payment failed for order: {}, reason: {}", 
                event.getOrderId(), event.getFailureReason());
        
        // 주문 상태를 결제 실패로 변경
        orderService.markPaymentFailed(event.getOrderId(), event.getFailureReason());
        
        // 고객에게 알림 발송
        notificationService.notifyPaymentFailed(event.getCustomerId(), event.getOrderId());
    }
    
    // 조건부 메시지 처리
    @StreamListener(value = OrderProcessor.ORDER_INPUT, 
                   condition = "headers['priority']=='HIGH'")
    public void handleHighPriorityOrder(@Payload OrderCreatedEvent event) {
        log.info("Processing high priority order: {}", event.getOrderId());
        
        // 우선순위가 높은 주문 처리
        orderService.processHighPriorityOrder(event);
    }
    
    // 에러 핸들링
    @ServiceActivator(inputChannel = "orders.order-service.errors")
    public void handleError(ErrorMessage errorMessage) {
        Throwable throwable = errorMessage.getPayload();
        Message<?> failedMessage = (Message<?>) errorMessage.getHeaders().get("originalMessage");
        
        log.error("Error processing message: {}", failedMessage, throwable);
        
        // 데드 레터 큐로 이동 또는 재시도 로직
        if (shouldRetry(throwable)) {
            // 재시도 로직
            retryMessage(failedMessage);
        } else {
            // 데드 레터 큐로 이동
            sendToDeadLetterQueue(failedMessage, throwable);
        }
    }
    
    private void processOrderCreated(OrderCreatedEvent event) {
        // 주문 생성 처리 로직
        orderService.confirmOrder(event.getOrderId());
        
        // 알림 발송
        notificationService.notifyOrderCreated(event.getCustomerId(), event.getOrderId());
        
        // 감사 로그
        auditService.logOrderEvent("ORDER_CREATED", event.getOrderId());
    }
    
    private void processOrderStatusChanged(OrderStatusChangedEvent event) {
        // 주문 상태 변경 처리
        notificationService.notifyOrderStatusChanged(
            event.getCustomerId(), event.getOrderId(), event.getNewStatus());
        
        auditService.logOrderEvent("ORDER_STATUS_CHANGED", event.getOrderId());
    }
    
    private boolean shouldRetry(Throwable throwable) {
        // 재시도 가능한 예외인지 판단
        return throwable instanceof TransientDataAccessException ||
               throwable instanceof ConnectException;
    }
    
    private void retryMessage(Message<?> message) {
        // 재시도 로직 구현
        log.info("Retrying message: {}", message);
    }
    
    private void sendToDeadLetterQueue(Message<?> message, Throwable throwable) {
        // 데드 레터 큐로 메시지 이동
        log.error("Sending message to DLQ: {}", message, throwable);
    }
}
```

### 메시지 추적 및 모니터링

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class MessageTracker {
    
    private final MeterRegistry meterRegistry;
    private final Counter messagesSent;
    private final Counter messagesReceived;
    private final Timer messageProcessingTime;
    
    public MessageTracker(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.messagesSent = Counter.builder("messages.sent")
            .description("Total messages sent")
            .register(meterRegistry);
        this.messagesReceived = Counter.builder("messages.received")
            .description("Total messages received")
            .register(meterRegistry);
        this.messageProcessingTime = Timer.builder("message.processing.time")
            .description("Message processing time")
            .register(meterRegistry);
    }
    
    @EventListener
    public void handleMessageSent(MessageSentEvent event) {
        messagesSent.increment(
            Tags.of(
                "destination", event.getDestination(),
                "eventType", event.getEventType()
            )
        );
        
        log.debug("Message sent - Destination: {}, EventType: {}, MessageId: {}",
                event.getDestination(), event.getEventType(), event.getMessageId());
    }
    
    @EventListener
    public void handleMessageReceived(MessageReceivedEvent event) {
        messagesReceived.increment(
            Tags.of(
                "source", event.getSource(),
                "eventType", event.getEventType()
            )
        );
        
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(messageProcessingTime.tag("eventType", event.getEventType()));
    }
    
    @Scheduled(fixedRate = 60000) // 1분마다
    public void logMessageStatistics() {
        double sentCount = messagesSent.count();
        double receivedCount = messagesReceived.count();
        
        log.info("Message Statistics - Sent: {}, Received: {}", sentCount, receivedCount);
    }
}

// 메시지 헬스 인디케이터
@Component
public class MessageHealthIndicator implements HealthIndicator {
    
    private final OrderProcessor orderProcessor;
    
    public MessageHealthIndicator(OrderProcessor orderProcessor) {
        this.orderProcessor = orderProcessor;
    }
    
    @Override
    public Health health() {
        try {
            // 메시지 채널 상태 확인
            if (isChannelHealthy(orderProcessor.orderInput()) &&
                isChannelHealthy(orderProcessor.orderOutput())) {
                return Health.up()
                    .withDetail("orderInput", "UP")
                    .withDetail("orderOutput", "UP")
                    .build();
            } else {
                return Health.down()
                    .withDetail("message", "One or more message channels are unhealthy")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
    
    private boolean isChannelHealthy(MessageChannel channel) {
        // 채널 상태 확인 로직
        return channel != null;
    }
}
```

## 🔒 보안 통합

### OAuth2 리소스 서버 설정

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").hasAuthority("SCOPE_read")
                .requestMatchers(HttpMethod.POST, "/api/products/**").hasAuthority("SCOPE_write")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                    .jwtDecoder(jwtDecoder())
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(csrf -> csrf.disable());
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        // JWT 검증을 위한 디코더 설정
        return NimbusJwtDecoder.withJwkSetUri("http://auth-server:8080/oauth2/jwks")
            .cache(Duration.ofMinutes(5))
            .build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("SCOPE_");
        authoritiesConverter.setAuthoritiesClaimName("scope");
        
        JwtAuthenticationConverter authenticationConverter = new JwtAuthenticationConverter();
        authenticationConverter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        authenticationConverter.setPrincipalClaimName("sub");
        
        return authenticationConverter;
    }
}

// JWT 토큰 추출 및 전파
@Component
@RequiredArgsConstructor
public class JwtTokenExtractor {
    
    public String extractToken(HttpServletRequest request) {
        String authorization = request.getHeader("Authorization");
        if (authorization != null && authorization.startsWith("Bearer ")) {
            return authorization.substring(7);
        }
        return null;
    }
    
    public String getCurrentUserId() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication instanceof JwtAuthenticationToken) {
            JwtAuthenticationToken jwtToken = (JwtAuthenticationToken) authentication;
            return jwtToken.getToken().getClaimAsString("sub");
        }
        return null;
    }
    
    public List<String> getCurrentUserRoles() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());
    }
    
    public boolean hasScope(String scope) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return authentication.getAuthorities().stream()
            .anyMatch(authority -> authority.getAuthority().equals("SCOPE_" + scope));
    }
}
```

### 서비스 간 보안 통신

```java
@Configuration
public class SecureServiceCommunicationConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate secureRestTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add(new JwtTokenPropagationInterceptor());
        return restTemplate;
    }
    
    @Bean
    @LoadBalanced
    public WebClient.Builder secureWebClientBuilder() {
        return WebClient.builder()
            .filter(jwtTokenPropagationFilter());
    }
}

// JWT 토큰 전파 인터셉터
@Component
@RequiredArgsConstructor
public class JwtTokenPropagationInterceptor implements ClientHttpRequestInterceptor {
    
    private final JwtTokenExtractor jwtTokenExtractor;
    
    @Override
    public ClientHttpResponse intercept(
            HttpRequest request, 
            byte[] body, 
            ClientHttpRequestExecution execution) throws IOException {
        
        // 현재 인증 정보에서 JWT 토큰 추출
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication instanceof JwtAuthenticationToken) {
            JwtAuthenticationToken jwtToken = (JwtAuthenticationToken) authentication;
            String tokenValue = jwtToken.getToken().getTokenValue();
            
            // Authorization 헤더에 토큰 추가
            request.getHeaders().add("Authorization", "Bearer " + tokenValue);
        }
        
        return execution.execute(request, body);
    }
}

// WebClient용 필터
@Component
public class JwtTokenPropagationFilter implements ExchangeFilterFunction {
    
    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        return Mono.deferContextual(contextView -> {
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
            
            if (authentication instanceof JwtAuthenticationToken) {
                JwtAuthenticationToken jwtToken = (JwtAuthenticationToken) authentication;
                String tokenValue = jwtToken.getToken().getTokenValue();
                
                ClientRequest modifiedRequest = ClientRequest.from(request)
                    .header("Authorization", "Bearer " + tokenValue)
                    .build();
                
                return next.exchange(modifiedRequest);
            }
            
            return next.exchange(request);
        });
    }
}
```

### 메서드 레벨 보안

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class SecureOrderService {
    
    private final OrderRepository orderRepository;
    private final JwtTokenExtractor jwtTokenExtractor;
    
    @PreAuthorize("hasAuthority('SCOPE_read')")
    public List<OrderDto> getAllOrders() {
        return orderRepository.findAll().stream()
            .map(this::convertToDto)
            .collect(Collectors.toList());
    }
    
    @PreAuthorize("hasAuthority('SCOPE_read') and #userId == authentication.name")
    public List<OrderDto> getUserOrders(String userId) {
        return orderRepository.findByUserId(userId).stream()
            .map(this::convertToDto)
            .collect(Collectors.toList());
    }
    
    @PreAuthorize("hasAuthority('SCOPE_write')")
    @PostAuthorize("returnObject.userId == authentication.name or hasRole('ADMIN')")
    public OrderDto createOrder(CreateOrderRequest request) {
        String currentUserId = jwtTokenExtractor.getCurrentUserId();
        
        Order order = Order.builder()
            .userId(currentUserId)
            .amount(request.getAmount())
            .items(request.getItems())
            .status(OrderStatus.PENDING)
            .createdAt(LocalDateTime.now())
            .build();
        
        Order savedOrder = orderRepository.save(order);
        
        log.info("Order created by user: {}, orderId: {}", currentUserId, savedOrder.getId());
        
        return convertToDto(savedOrder);
    }
    
    @PreAuthorize("hasAuthority('SCOPE_write') and (@orderService.isOrderOwner(#orderId, authentication.name) or hasRole('ADMIN'))")
    public OrderDto updateOrder(String orderId, UpdateOrderRequest request) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));
        
        order.update(request);
        Order updatedOrder = orderRepository.save(order);
        
        return convertToDto(updatedOrder);
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(String orderId) {
        orderRepository.deleteById(orderId);
        log.warn("Order deleted by admin: {}", orderId);
    }
    
    // 커스텀 보안 검증 메서드
    public boolean isOrderOwner(String orderId, String userId) {
        return orderRepository.findById(orderId)
            .map(order -> order.getUserId().equals(userId))
            .orElse(false);
    }
    
    private OrderDto convertToDto(Order order) {
        return OrderDto.builder()
            .id(order.getId())
            .userId(order.getUserId())
            .amount(order.getAmount())
            .items(order.getItems())
            .status(order.getStatus())
            .createdAt(order.getCreatedAt())
            .build();
    }
}
```

### 보안 감사 및 로깅

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SecurityAuditLogger {
    
    private final AuditEventRepository auditEventRepository;
    private final JwtTokenExtractor jwtTokenExtractor;
    
    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        String userId = event.getAuthentication().getName();
        
        AuditEvent auditEvent = AuditEvent.builder()
            .eventType("AUTHENTICATION_SUCCESS")
            .userId(userId)
            .timestamp(LocalDateTime.now())
            .details(Map.of(
                "authenticationMethod", event.getAuthentication().getClass().getSimpleName(),
                "authorities", event.getAuthentication().getAuthorities().toString()
            ))
            .build();
        
        auditEventRepository.save(auditEvent);
        
        log.info("Authentication successful for user: {}", userId);
    }
    
    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        String failureReason = event.getException().getMessage();
        
        AuditEvent auditEvent = AuditEvent.builder()
            .eventType("AUTHENTICATION_FAILURE")
            .userId(username)
            .timestamp(LocalDateTime.now())
            .details(Map.of(
                "failureReason", failureReason,
                "remoteAddress", getCurrentRemoteAddress()
            ))
            .build();
        
        auditEventRepository.save(auditEvent);
        
        log.warn("Authentication failed for user: {}, reason: {}", username, failureReason);
    }
    
    @EventListener
    public void handleAccessDenied(AccessDeniedEvent event) {
        String userId = jwtTokenExtractor.getCurrentUserId();
        
        AuditEvent auditEvent = AuditEvent.builder()
            .eventType("ACCESS_DENIED")
            .userId(userId)
            .timestamp(LocalDateTime.now())
            .details(Map.of(
                "requestedResource", event.getSource().toString(),
                "userAuthorities", jwtTokenExtractor.getCurrentUserRoles().toString()
            ))
            .build();
        
        auditEventRepository.save(auditEvent);
        
        log.warn("Access denied for user: {} to resource: {}", userId, event.getSource());
    }
    
    private String getCurrentRemoteAddress() {
        // 현재 요청의 원격 주소 추출
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        if (requestAttributes instanceof ServletRequestAttributes) {
            HttpServletRequest request = ((ServletRequestAttributes) requestAttributes).getRequest();
            return request.getRemoteAddr();
        }
        return "unknown";
    }
}

// 보안 메트릭
@Component
@RequiredArgsConstructor
public class SecurityMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter authenticationAttempts;
    private final Counter authenticationFailures;
    private final Counter accessDeniedEvents;
    
    public SecurityMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.authenticationAttempts = Counter.builder("security.authentication.attempts")
            .description("Total authentication attempts")
            .register(meterRegistry);
        this.authenticationFailures = Counter.builder("security.authentication.failures")
            .description("Total authentication failures")
            .register(meterRegistry);
        this.accessDeniedEvents = Counter.builder("security.access.denied")
            .description("Total access denied events")
            .register(meterRegistry);
    }
    
    @EventListener
    public void onAuthenticationAttempt(AbstractAuthenticationEvent event) {
        authenticationAttempts.increment();
    }
    
    @EventListener
    public void onAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        authenticationFailures.increment(
            Tags.of("reason", event.getException().getClass().getSimpleName())
        );
    }
    
    @EventListener
    public void onAccessDenied(AccessDeniedEvent event) {
        accessDeniedEvents.increment();
    }
}
```

---

*Spring Cloud는 마이크로서비스 개발을 위한 포괄적인 솔루션을 제공합니다. 점진적으로 도입하여 시스템의 복잡성을 관리하세요.*
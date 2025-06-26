# Security Patterns

## 개념
- Spring Security를 활용한 실무 보안 패턴
- [[Spring Security 6]]와 [[Enterprise Design Patterns]]의 결합
- 실제 상용 서비스에서 사용되는 보안 구현 사례

## 1. JWT 인증 패턴
### 토큰 기반 인증 시스템
```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    
    private final AuthService authService;
    private final JwtTokenProvider jwtTokenProvider;
    
    // 로그인
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest request) {
        UserDetails userDetails = authService.authenticate(request.getEmail(), request.getPassword());
        
        String accessToken = jwtTokenProvider.generateAccessToken(userDetails);
        String refreshToken = jwtTokenProvider.generateRefreshToken(userDetails);
        
        // Refresh Token을 쿠키에 저장 (HttpOnly, Secure)
        ResponseCookie refreshCookie = ResponseCookie.from("refreshToken", refreshToken)
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .maxAge(Duration.ofDays(7))
            .path("/api/auth")
            .build();
        
        LoginResponse response = LoginResponse.builder()
            .accessToken(accessToken)
            .tokenType("Bearer")
            .expiresIn(jwtTokenProvider.getAccessTokenExpiration())
            .user(UserResponse.from((User) userDetails))
            .build();
        
        return ResponseEntity.ok()
            .header(HttpHeaders.SET_COOKIE, refreshCookie.toString())
            .body(response);
    }
    
    // 토큰 갱신
    @PostMapping("/refresh")
    public ResponseEntity<TokenRefreshResponse> refreshToken(
            @CookieValue("refreshToken") String refreshToken) {
        
        if (!jwtTokenProvider.validateToken(refreshToken)) {
            throw new InvalidTokenException("Invalid refresh token");
        }
        
        UserDetails userDetails = jwtTokenProvider.getUserDetailsFromToken(refreshToken);
        String newAccessToken = jwtTokenProvider.generateAccessToken(userDetails);
        
        TokenRefreshResponse response = TokenRefreshResponse.builder()
            .accessToken(newAccessToken)
            .tokenType("Bearer")
            .expiresIn(jwtTokenProvider.getAccessTokenExpiration())
            .build();
        
        return ResponseEntity.ok(response);
    }
}
```

## 2. OAuth2 소셜 로그인
### Google, GitHub 등 소셜 로그인 통합
```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class OAuth2SecurityConfig {
    
    private final CustomOAuth2UserService customOAuth2UserService;
    private final OAuth2AuthenticationSuccessHandler successHandler;
    
    @Bean
    public SecurityFilterChain oauth2FilterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)
                )
                .successHandler(successHandler)
            )
            .build();
    }
}

@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {
    
    private final UserRepository userRepository;
    
    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oauth2User = new DefaultOAuth2UserService().loadUser(userRequest);
        
        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        OAuth2UserInfo userInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(registrationId, oauth2User.getAttributes());
        
        User user = userRepository.findByEmail(userInfo.getEmail())
            .orElseGet(() -> createNewUser(userInfo, registrationId));
        
        return UserPrincipal.create(user, oauth2User.getAttributes());
    }
    
    private User createNewUser(OAuth2UserInfo userInfo, String provider) {
        User user = User.builder()
            .email(userInfo.getEmail())
            .name(userInfo.getName())
            .profileImageUrl(userInfo.getImageUrl())
            .provider(AuthProvider.valueOf(provider.toUpperCase()))
            .providerId(userInfo.getId())
            .role(Role.USER)
            .emailVerified(true)
            .build();
        
        return userRepository.save(user);
    }
}
```

## 3. 역할 기반 접근 제어 (RBAC)
### 세밀한 권한 관리
```java
// 권한 체크 어노테이션
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasRole('ADMIN') or (hasRole('USER') and @securityService.isOwner(authentication.name, #userId))")
public @interface OwnerOrAdmin {
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
public @interface AdminOrManager {
}

// 보안 서비스
@Service
@RequiredArgsConstructor
public class SecurityService {
    
    private final UserRepository userRepository;
    
    public boolean isOwner(String email, Long userId) {
        return userRepository.findByEmail(email)
            .map(user -> user.getId().equals(userId))
            .orElse(false);
    }
    
    public boolean canAccessProject(String email, Long projectId) {
        User user = userRepository.findByEmail(email).orElse(null);
        if (user == null) return false;
        
        if (user.getRole() == Role.ADMIN) return true;
        
        return user.getProjectMemberships().stream()
            .anyMatch(membership -> membership.getProject().getId().equals(projectId));
    }
}
```

## 4. API 키 인증
### 외부 API 호출을 위한 키 기반 인증
```java
@RestController
@RequestMapping("/api/external")
@RequiredArgsConstructor
public class ExternalApiController {
    
    @PostMapping("/webhook")
    @ApiKeyAuthentication
    public ResponseEntity<WebhookResponse> handleWebhook(
            @RequestHeader("X-API-Key") String apiKey,
            @RequestBody WebhookRequest request) {
        
        WebhookResponse response = externalApiService.processWebhook(request);
        return ResponseEntity.ok(response);
    }
}

// API 키 인증 필터
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
    
    private final ApiKeyService apiKeyService;
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        
        String apiKey = request.getHeader("X-API-Key");
        
        if (apiKey != null && apiKeyService.isValidApiKey(apiKey)) {
            ApiKeyDetails apiKeyDetails = apiKeyService.getApiKeyDetails(apiKey);
            
            ApiKeyAuthenticationToken authToken = new ApiKeyAuthenticationToken(
                apiKeyDetails.getClientId(),
                apiKey,
                apiKeyDetails.getAuthorities()
            );
            
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }
        
        filterChain.doFilter(request, response);
    }
    
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return !request.getRequestURI().startsWith("/api/external");
    }
}
```

## 5. 보안 헤더 설정
### 웹 보안 강화
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .frameOptions().deny()
                .contentTypeOptions().and()
                .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true)
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/api/auth/**", "/api/external/**")
            )
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList("https://*.example.com", "http://localhost:*"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

## 6. 비밀번호 정책 및 암호화
### 강력한 비밀번호 관리
```java
@Service
@RequiredArgsConstructor
public class PasswordService {
    
    private final PasswordEncoder passwordEncoder;
    private final PasswordHistoryRepository passwordHistoryRepository;
    
    public void validatePassword(String password, String email) {
        // 비밀번호 강도 검증
        if (!isStrongPassword(password)) {
            throw new WeakPasswordException("비밀번호는 최소 8자 이상이며, 대소문자, 숫자, 특수문자를 포함해야 합니다");
        }
        
        // 이전 비밀번호와 중복 확인
        if (isPasswordReused(email, password)) {
            throw new PasswordReusedException("최근 5개의 비밀번호는 재사용할 수 없습니다");
        }
        
        // 일반적인 비밀번호 확인
        if (isCommonPassword(password)) {
            throw new CommonPasswordException("너무 일반적인 비밀번호입니다. 다른 비밀번호를 선택해주세요");
        }
    }
    
    private boolean isStrongPassword(String password) {
        if (password.length() < 8) return false;
        
        boolean hasUppercase = password.chars().anyMatch(Character::isUpperCase);
        boolean hasLowercase = password.chars().anyMatch(Character::isLowerCase);
        boolean hasDigit = password.chars().anyMatch(Character::isDigit);
        boolean hasSpecialChar = password.chars().anyMatch(ch -> "!@#$%^&*()_+-=[]{}|;:,.<>?".indexOf(ch) >= 0);
        
        return hasUppercase && hasLowercase && hasDigit && hasSpecialChar;
    }
    
    private boolean isPasswordReused(String email, String rawPassword) {
        List<PasswordHistory> histories = passwordHistoryRepository.findTop5ByEmailOrderByCreatedAtDesc(email);
        
        return histories.stream()
            .anyMatch(history -> passwordEncoder.matches(rawPassword, history.getPassword()));
    }
}
```

## 7. 감사 로깅 (Audit Logging)
### 보안 이벤트 추적
```java
@Aspect
@Component
@RequiredArgsConstructor
public class SecurityAuditAspect {
    
    private final AuditLogService auditLogService;
    
    @AfterReturning("@annotation(AuditLog)")
    public void logSuccessfulOperation(JoinPoint joinPoint) {
        AuditLog auditAnnotation = getAuditAnnotation(joinPoint);
        
        SecurityAuditEvent event = SecurityAuditEvent.builder()
            .action(auditAnnotation.action())
            .resource(auditAnnotation.resource())
            .userId(getCurrentUserId())
            .ipAddress(getCurrentIpAddress())
            .userAgent(getCurrentUserAgent())
            .timestamp(LocalDateTime.now())
            .status("SUCCESS")
            .build();
        
        auditLogService.logEvent(event);
    }
    
    @AfterThrowing(pointcut = "@annotation(AuditLog)", throwing = "ex")
    public void logFailedOperation(JoinPoint joinPoint, Exception ex) {
        AuditLog auditAnnotation = getAuditAnnotation(joinPoint);
        
        SecurityAuditEvent event = SecurityAuditEvent.builder()
            .action(auditAnnotation.action())
            .resource(auditAnnotation.resource())
            .userId(getCurrentUserId())
            .ipAddress(getCurrentIpAddress())
            .userAgent(getCurrentUserAgent())
            .timestamp(LocalDateTime.now())
            .status("FAILED")
            .errorMessage(ex.getMessage())
            .build();
        
        auditLogService.logEvent(event);
    }
}

// 사용 예시
@RestController
@RequestMapping("/api/admin")
@RequiredArgsConstructor
public class AdminController {
    
    @DeleteMapping("/users/{id}")
    @AuditLog(action = "DELETE_USER", resource = "USER")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
    
    @PostMapping("/roles/{userId}")
    @AuditLog(action = "ASSIGN_ROLE", resource = "USER_ROLE")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> assignRole(
            @PathVariable Long userId,
            @RequestBody AssignRoleRequest request) {
        userService.assignRole(userId, request.getRole());
        return ResponseEntity.ok().build();
    }
}
```

## 관련 패턴
- [[Spring Security 6]]
- [[Enterprise Design Patterns]]
- [[Exception Handling Patterns]]
- [[API Design Patterns]]
- [[Logging Patterns]]

#security-patterns #jwt #oauth2 #rbac #audit-logging
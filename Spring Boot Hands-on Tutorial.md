# Spring Boot Hands-on Tutorial

## 개념
- 완전한 Spring Boot 애플리케이션을 처음부터 끝까지 구축하는 실습 가이드
- [[Spring Boot Project Setup]]부터 [[Spring Boot CI CD with GitLab and ArgoCD]]까지 전체 과정
- 모든 [[DTO and VO Patterns]] 및 [[Enterprise Design Patterns]] 실제 적용
- 실무에서 바로 사용할 수 있는 완전한 예제

## 🎯 학습 목표
이 튜토리얼을 완료하면 다음을 할 수 있습니다:
- 완전한 REST API 서버 구축
- 데이터베이스 연동 및 CRUD 작업
- 보안 인증/인가 구현
- 테스트 코드 작성
- 운영 환경 배포

## 📋 프로젝트 개요
**"TaskMaster Pro"** - 팀 협업을 위한 작업 관리 시스템 구축

### 핵심 기능
- 사용자 관리 (회원가입, 로그인, 권한 관리)
- 프로젝트 관리 (생성, 수정, 팀원 초대)
- 작업 관리 (할일 생성, 상태 변경, 할당)
- 알림 시스템 (이메일, 실시간 알림)
- 대시보드 (통계, 차트)

### 기술 스택
- **Backend**: Spring Boot 3.5, Java 21
- **Database**: PostgreSQL, Redis
- **Security**: Spring Security 6, JWT
- **Testing**: JUnit 5, Testcontainers
- **Docs**: OpenAPI 3
- **Monitoring**: Actuator, Micrometer

## 🚀 Phase 1: 프로젝트 초기 설정

### Step 1: 프로젝트 생성
```bash
# Spring Initializr를 통한 프로젝트 생성
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,security,validation,actuator,devtools \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.5.0 \
  -d baseDir=taskmaster-pro \
  -d groupId=com.taskmaster \
  -d artifactId=taskmaster-pro \
  -d name=TaskMaster-Pro \
  -d description="Team Collaboration Task Management System" \
  -d packageName=com.taskmaster.pro \
  -d packaging=jar \
  -d javaVersion=21 \
  -o taskmaster-pro.zip

unzip taskmaster-pro.zip
cd taskmaster-pro
```

### Step 2: 디렉토리 구조 설정
```
taskmaster-pro/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── taskmaster/
│   │   │           └── pro/
│   │   │               ├── TaskMasterProApplication.java
│   │   │               ├── config/
│   │   │               ├── controller/
│   │   │               │   ├── api/v1/
│   │   │               │   └── admin/
│   │   │               ├── service/
│   │   │               │   ├── impl/
│   │   │               │   └── interfaces/
│   │   │               ├── repository/
│   │   │               ├── domain/
│   │   │               │   ├── entity/
│   │   │               │   ├── enums/
│   │   │               │   └── vo/
│   │   │               ├── dto/
│   │   │               │   ├── request/
│   │   │               │   ├── response/
│   │   │               │   └── internal/
│   │   │               ├── mapper/
│   │   │               ├── security/
│   │   │               ├── exception/
│   │   │               ├── util/
│   │   │               └── aspect/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/migration/
│   │       └── static/
│   └── test/
├── docker/
├── docs/
└── scripts/
```

### Step 3: 기본 설정 파일
```yaml
# application.yml
spring:
  application:
    name: taskmaster-pro
  
  profiles:
    active: dev
  
  datasource:
    url: jdbc:postgresql://localhost:5432/taskmaster_pro
    username: ${DB_USERNAME:taskmaster}
    password: ${DB_PASSWORD:taskmaster123}
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
  
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
  
  security:
    jwt:
      secret: ${JWT_SECRET:mySecretKey}
      expiration: 86400000  # 24 hours

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
    com.taskmaster.pro: DEBUG
    org.springframework.security: DEBUG
```

## 🏗 Phase 2: 도메인 모델 설계

### Step 4: 엔티티 정의
```java
// User 엔티티
@Entity
@Table(name = "users")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class User extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private String password;
    
    @Column(nullable = false)
    private String firstName;
    
    @Column(nullable = false)
    private String lastName;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserRole role;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private UserStatus status;
    
    private String profileImageUrl;
    
    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;
    
    @OneToMany(mappedBy = "owner", cascade = CascadeType.ALL)
    private List<Project> ownedProjects = new ArrayList<>();
    
    @ManyToMany(mappedBy = "members")
    private Set<Project> memberProjects = new HashSet<>();
    
    @OneToMany(mappedBy = "assignee")
    private List<Task> assignedTasks = new ArrayList<>();
    
    // 비즈니스 메서드
    public String getFullName() {
        return firstName + " " + lastName;
    }
    
    public boolean isActive() {
        return status == UserStatus.ACTIVE;
    }
    
    public void updateLastLogin() {
        this.lastLoginAt = LocalDateTime.now();
    }
}

// Project 엔티티
@Entity
@Table(name = "projects")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class Project extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(length = 1000)
    private String description;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProjectStatus status;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProjectPriority priority;
    
    @Column(name = "start_date")
    private LocalDate startDate;
    
    @Column(name = "end_date")
    private LocalDate endDate;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id", nullable = false)
    private User owner;
    
    @ManyToMany
    @JoinTable(
        name = "project_members",
        joinColumns = @JoinColumn(name = "project_id"),
        inverseJoinColumns = @JoinColumn(name = "user_id")
    )
    private Set<User> members = new HashSet<>();
    
    @OneToMany(mappedBy = "project", cascade = CascadeType.ALL)
    private List<Task> tasks = new ArrayList<>();
    
    // 비즈니스 메서드
    public boolean isOwner(User user) {
        return owner != null && owner.getId().equals(user.getId());
    }
    
    public boolean isMember(User user) {
        return members.stream()
            .anyMatch(member -> member.getId().equals(user.getId()));
    }
    
    public void addMember(User user) {
        members.add(user);
    }
    
    public void removeMember(User user) {
        members.remove(user);
    }
}

// Task 엔티티
@Entity
@Table(name = "tasks")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class Task extends BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String title;
    
    @Column(length = 2000)
    private String description;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TaskStatus status;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TaskPriority priority;
    
    @Column(name = "due_date")
    private LocalDateTime dueDate;
    
    @Column(name = "estimated_hours")
    private Integer estimatedHours;
    
    @Column(name = "actual_hours")
    private Integer actualHours;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "project_id", nullable = false)
    private Project project;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "assignee_id")
    private User assignee;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reporter_id", nullable = false)
    private User reporter;
    
    @OneToMany(mappedBy = "task", cascade = CascadeType.ALL)
    private List<Comment> comments = new ArrayList<>();
    
    // 비즈니스 메서드
    public boolean isOverdue() {
        return dueDate != null && dueDate.isBefore(LocalDateTime.now()) 
               && status != TaskStatus.COMPLETED;
    }
    
    public boolean canBeEditedBy(User user) {
        return assignee != null && assignee.getId().equals(user.getId()) ||
               reporter.getId().equals(user.getId()) ||
               project.isOwner(user);
    }
    
    public void assignTo(User user) {
        this.assignee = user;
    }
    
    public void complete() {
        this.status = TaskStatus.COMPLETED;
        this.updatedAt = LocalDateTime.now();
    }
}

// BaseEntity
@MappedSuperclass
@Data
@NoArgsConstructor
@AllArgsConstructor
public abstract class BaseEntity {
    
    @Column(name = "created_at", nullable = false, updatable = false)
    @CreationTimestamp
    protected LocalDateTime createdAt;
    
    @Column(name = "updated_at", nullable = false)
    @UpdateTimestamp
    protected LocalDateTime updatedAt;
    
    @Version
    protected Long version;
}
```

### Step 5: Enum 정의
```java
// UserRole
public enum UserRole {
    ADMIN("관리자", Set.of(
        Permission.USER_READ, Permission.USER_WRITE, Permission.USER_DELETE,
        Permission.PROJECT_READ, Permission.PROJECT_WRITE, Permission.PROJECT_DELETE,
        Permission.TASK_READ, Permission.TASK_WRITE, Permission.TASK_DELETE
    )),
    
    PROJECT_MANAGER("프로젝트 매니저", Set.of(
        Permission.USER_READ,
        Permission.PROJECT_READ, Permission.PROJECT_WRITE,
        Permission.TASK_READ, Permission.TASK_WRITE, Permission.TASK_DELETE
    )),
    
    DEVELOPER("개발자", Set.of(
        Permission.USER_READ,
        Permission.PROJECT_READ,
        Permission.TASK_READ, Permission.TASK_WRITE
    )),
    
    VIEWER("조회자", Set.of(
        Permission.USER_READ,
        Permission.PROJECT_READ,
        Permission.TASK_READ
    ));
    
    private final String displayName;
    private final Set<Permission> permissions;
    
    UserRole(String displayName, Set<Permission> permissions) {
        this.displayName = displayName;
        this.permissions = permissions;
    }
    
    public boolean hasPermission(Permission permission) {
        return permissions.contains(permission);
    }
    
    public Set<Permission> getPermissions() {
        return Collections.unmodifiableSet(permissions);
    }
}

// TaskStatus
public enum TaskStatus {
    TODO("할 일", "secondary"),
    IN_PROGRESS("진행 중", "primary"),
    IN_REVIEW("검토 중", "warning"),
    COMPLETED("완료", "success"),
    CANCELLED("취소", "danger");
    
    private final String displayName;
    private final String badgeColor;
    
    TaskStatus(String displayName, String badgeColor) {
        this.displayName = displayName;
        this.badgeColor = badgeColor;
    }
    
    public boolean isCompleted() {
        return this == COMPLETED;
    }
    
    public boolean canTransitionTo(TaskStatus newStatus) {
        return switch (this) {
            case TODO -> newStatus == IN_PROGRESS || newStatus == CANCELLED;
            case IN_PROGRESS -> newStatus == IN_REVIEW || newStatus == COMPLETED || newStatus == CANCELLED;
            case IN_REVIEW -> newStatus == IN_PROGRESS || newStatus == COMPLETED;
            case COMPLETED, CANCELLED -> false;
        };
    }
}
```

## 📊 Phase 3: DTO 설계 및 구현

### Step 6: Request/Response DTO 구현
```java
// 사용자 생성 요청 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateUserRequest {
    
    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "올바른 이메일 형식이어야 합니다")
    @JsonProperty("email")
    private String email;
    
    @NotBlank(message = "비밀번호는 필수입니다")
    @Size(min = 8, message = "비밀번호는 최소 8자 이상이어야 합니다")
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
             message = "비밀번호는 대소문자, 숫자, 특수문자를 포함해야 합니다")
    @JsonProperty("password")
    private String password;
    
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 1, max = 50, message = "이름은 1-50자 사이여야 합니다")
    @JsonProperty("first_name")
    private String firstName;
    
    @NotBlank(message = "성은 필수입니다")
    @Size(min = 1, max = 50, message = "성은 1-50자 사이여야 합니다")
    @JsonProperty("last_name")
    private String lastName;
    
    @JsonProperty("profile_image_url")
    private String profileImageUrl;
}

// 사용자 응답 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResponse {
    
    @JsonProperty("id")
    private Long id;
    
    @JsonProperty("email")
    private String email;
    
    @JsonProperty("first_name")
    private String firstName;
    
    @JsonProperty("last_name")
    private String lastName;
    
    @JsonProperty("full_name")
    private String fullName;
    
    @JsonProperty("role")
    private UserRole role;
    
    @JsonProperty("status")
    private UserStatus status;
    
    @JsonProperty("profile_image_url")
    private String profileImageUrl;
    
    @JsonProperty("last_login_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime lastLoginAt;
    
    @JsonProperty("created_at")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;
    
    @JsonProperty("project_count")
    private Integer projectCount;
    
    @JsonProperty("task_count")
    private Integer taskCount;
    
    // 팩토리 메서드
    public static UserResponse from(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .email(user.getEmail())
            .firstName(user.getFirstName())
            .lastName(user.getLastName())
            .fullName(user.getFullName())
            .role(user.getRole())
            .status(user.getStatus())
            .profileImageUrl(user.getProfileImageUrl())
            .lastLoginAt(user.getLastLoginAt())
            .createdAt(user.getCreatedAt())
            .projectCount(user.getOwnedProjects().size() + user.getMemberProjects().size())
            .taskCount(user.getAssignedTasks().size())
            .build();
    }
}

// 프로젝트 생성 요청 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateProjectRequest {
    
    @NotBlank(message = "프로젝트 이름은 필수입니다")
    @Size(min = 3, max = 100, message = "프로젝트 이름은 3-100자 사이여야 합니다")
    @JsonProperty("name")
    private String name;
    
    @Size(max = 1000, message = "설명은 1000자를 초과할 수 없습니다")
    @JsonProperty("description")
    private String description;
    
    @NotNull(message = "우선순위는 필수입니다")
    @JsonProperty("priority")
    private ProjectPriority priority;
    
    @JsonProperty("start_date")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate startDate;
    
    @JsonProperty("end_date")
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate endDate;
    
    @JsonProperty("member_emails")
    private List<@Email String> memberEmails = new ArrayList<>();
    
    @AssertTrue(message = "종료일은 시작일보다 이후여야 합니다")
    private boolean isDateRangeValid() {
        if (startDate != null && endDate != null) {
            return !endDate.isBefore(startDate);
        }
        return true;
    }
}

// 작업 생성 요청 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateTaskRequest {
    
    @NotBlank(message = "작업 제목은 필수입니다")
    @Size(min = 3, max = 200, message = "작업 제목은 3-200자 사이여야 합니다")
    @JsonProperty("title")
    private String title;
    
    @Size(max = 2000, message = "설명은 2000자를 초과할 수 없습니다")
    @JsonProperty("description")
    private String description;
    
    @NotNull(message = "우선순위는 필수입니다")
    @JsonProperty("priority")
    private TaskPriority priority;
    
    @JsonProperty("due_date")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @Future(message = "마감일은 미래 날짜여야 합니다")
    private LocalDateTime dueDate;
    
    @Min(value = 1, message = "예상 시간은 최소 1시간이어야 합니다")
    @Max(value = 1000, message = "예상 시간은 최대 1000시간이어야 합니다")
    @JsonProperty("estimated_hours")
    private Integer estimatedHours;
    
    @JsonProperty("assignee_email")
    @Email(message = "올바른 이메일 형식이어야 합니다")
    private String assigneeEmail;
}
```

### Step 7: MapStruct 매퍼 구현
```java
@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE,
    uses = {UserMapper.class}
)
public interface ProjectMapper {
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", constant = "PLANNING")
    @Mapping(target = "owner", ignore = true)
    @Mapping(target = "members", ignore = true)
    @Mapping(target = "tasks", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "version", ignore = true)
    Project toEntity(CreateProjectRequest request);
    
    @Mapping(source = "owner", target = "owner")
    @Mapping(source = "tasks", target = "taskCount", qualifiedByName = "countTasks")
    @Mapping(source = "members", target = "memberCount", qualifiedByName = "countMembers")
    ProjectResponse toResponse(Project project);
    
    List<ProjectResponse> toResponseList(List<Project> projects);
    
    @Named("countTasks")
    default Integer countTasks(List<Task> tasks) {
        return tasks != null ? tasks.size() : 0;
    }
    
    @Named("countMembers")
    default Integer countMembers(Set<User> members) {
        return members != null ? members.size() : 0;
    }
    
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateProjectFromRequest(UpdateProjectRequest request, @MappingTarget Project project);
}
```

## 🔒 Phase 4: 보안 구현

### Step 8: JWT 인증 구현
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtTokenProvider {
    
    @Value("${spring.security.jwt.secret}")
    private String jwtSecret;
    
    @Value("${spring.security.jwt.expiration}")
    private int jwtExpirationMs;
    
    private final UserDetailsService userDetailsService;
    
    @PostConstruct
    protected void init() {
        jwtSecret = Base64.getEncoder().encodeToString(jwtSecret.getBytes());
    }
    
    public String generateToken(UserPrincipal userPrincipal) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationMs);
        
        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .setIssuedAt(now)
                .setExpirationDate(expiryDate)
                .claim("userId", userPrincipal.getId())
                .claim("role", userPrincipal.getRole().name())
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }
    
    public String generateRefreshToken(UserPrincipal userPrincipal) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpirationMs * 7); // 7 days
        
        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .setIssuedAt(now)
                .setExpirationDate(expiryDate)
                .claim("type", "refresh")
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }
    
    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(token)
                .getBody();
        return claims.getSubject();
    }
    
    public boolean validateToken(String authToken) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(authToken);
            return true;
        } catch (SignatureException ex) {
            log.error("Invalid JWT signature");
        } catch (MalformedJwtException ex) {
            log.error("Invalid JWT token");
        } catch (ExpiredJwtException ex) {
            log.error("Expired JWT token");
        } catch (UnsupportedJwtException ex) {
            log.error("Unsupported JWT token");
        } catch (IllegalArgumentException ex) {
            log.error("JWT claims string is empty");
        }
        return false;
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain filterChain) throws ServletException, IOException {
        
        String jwt = getJwtFromRequest(request);
        
        if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
            String username = tokenProvider.getUsernameFromToken(jwt);
            
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken authentication = 
                new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            
            SecurityContextHolder.getContext().setAuthentication(authentication);
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
```

### Step 9: 권한 기반 접근 제어
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final PasswordEncoder passwordEncoder;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.cors().and().csrf().disable()
            .exceptionHandling().authenticationEntryPoint(jwtAuthenticationEntryPoint)
            .and()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/users/me").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authConfig) 
            throws Exception {
        return authConfig.getAuthenticationManager();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

## 🛠 Phase 5: 서비스 레이어 구현

### Step 10: 비즈니스 로직 구현
```java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class ProjectService {
    
    private final ProjectRepository projectRepository;
    private final UserRepository userRepository;
    private final ProjectMapper projectMapper;
    private final EmailService emailService;
    private final ApplicationEventPublisher eventPublisher;
    
    public ProjectResponse createProject(CreateProjectRequest request, String ownerEmail) {
        log.info("Creating project: {} for owner: {}", request.getName(), ownerEmail);
        
        // 1. 프로젝트 소유자 조회
        User owner = userRepository.findByEmail(ownerEmail)
                .orElseThrow(() -> new UserNotFoundException("Owner not found: " + ownerEmail));
        
        // 2. DTO → Entity 변환
        Project project = projectMapper.toEntity(request);
        project.setOwner(owner);
        project.setStatus(ProjectStatus.PLANNING);
        
        // 3. 프로젝트 멤버 추가
        if (request.getMemberEmails() != null && !request.getMemberEmails().isEmpty()) {
            Set<User> members = request.getMemberEmails().stream()
                    .map(email -> userRepository.findByEmail(email)
                            .orElseThrow(() -> new UserNotFoundException("Member not found: " + email)))
                    .collect(Collectors.toSet());
            project.setMembers(members);
        }
        
        // 4. 프로젝트 저장
        Project savedProject = projectRepository.save(project);
        
        // 5. 이벤트 발행 (이메일 알림 등)
        eventPublisher.publishEvent(new ProjectCreatedEvent(savedProject.getId(), ownerEmail));
        
        // 6. 프로젝트 멤버들에게 초대 이메일 발송
        sendInvitationEmails(savedProject);
        
        log.info("Successfully created project with ID: {}", savedProject.getId());
        return projectMapper.toResponse(savedProject);
    }
    
    @Transactional(readOnly = true)
    public Page<ProjectResponse> getProjectsForUser(String userEmail, Pageable pageable) {
        User user = userRepository.findByEmail(userEmail)
                .orElseThrow(() -> new UserNotFoundException("User not found: " + userEmail));
        
        Page<Project> projectPage = projectRepository.findProjectsForUser(user.getId(), pageable);
        return projectPage.map(projectMapper::toResponse);
    }
    
    @Transactional(readOnly = true)
    @PreAuthorize("@projectSecurityService.canAccessProject(#projectId, authentication.name)")
    public ProjectDetailResponse getProjectDetail(Long projectId) {
        Project project = getProjectEntity(projectId);
        return projectMapper.toDetailResponse(project);
    }
    
    @PreAuthorize("@projectSecurityService.canEditProject(#projectId, authentication.name)")
    public ProjectResponse updateProject(Long projectId, UpdateProjectRequest request) {
        Project project = getProjectEntity(projectId);
        
        // 부분 업데이트 적용
        projectMapper.updateProjectFromRequest(request, project);
        
        Project updatedProject = projectRepository.save(project);
        
        // 프로젝트 업데이트 이벤트 발행
        eventPublisher.publishEvent(new ProjectUpdatedEvent(projectId, getCurrentUserEmail()));
        
        return projectMapper.toResponse(updatedProject);
    }
    
    @PreAuthorize("@projectSecurityService.canEditProject(#projectId, authentication.name)")
    public void addMemberToProject(Long projectId, String memberEmail) {
        Project project = getProjectEntity(projectId);
        User member = userRepository.findByEmail(memberEmail)
                .orElseThrow(() -> new UserNotFoundException("Member not found: " + memberEmail));
        
        if (project.isMember(member)) {
            throw new BusinessException("User is already a member of this project");
        }
        
        project.addMember(member);
        projectRepository.save(project);
        
        // 멤버 추가 알림
        emailService.sendProjectInvitation(member.getEmail(), project.getName());
        
        log.info("Added member {} to project {}", memberEmail, projectId);
    }
    
    @PreAuthorize("@projectSecurityService.canDeleteProject(#projectId, authentication.name)")
    public void deleteProject(Long projectId) {
        Project project = getProjectEntity(projectId);
        
        // 프로젝트에 진행 중인 작업이 있는지 확인
        boolean hasActiveTasks = project.getTasks().stream()
                .anyMatch(task -> task.getStatus() == TaskStatus.IN_PROGRESS);
        
        if (hasActiveTasks) {
            throw new BusinessException("Cannot delete project with active tasks");
        }
        
        projectRepository.delete(project);
        
        // 프로젝트 삭제 이벤트 발행
        eventPublisher.publishEvent(new ProjectDeletedEvent(projectId, getCurrentUserEmail()));
        
        log.info("Successfully deleted project: {}", projectId);
    }
    
    // 헬퍼 메서드들
    private Project getProjectEntity(Long projectId) {
        return projectRepository.findById(projectId)
                .orElseThrow(() -> new ProjectNotFoundException("Project not found: " + projectId));
    }
    
    private void sendInvitationEmails(Project project) {
        project.getMembers().forEach(member -> 
            emailService.sendProjectInvitation(member.getEmail(), project.getName())
        );
    }
    
    private String getCurrentUserEmail() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return authentication.getName();
    }
}
```

## 🧪 Phase 6: 테스트 구현

### Step 11: 단위 테스트 작성
```java
@ExtendWith(MockitoExtension.class)
class ProjectServiceTest {
    
    @Mock
    private ProjectRepository projectRepository;
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private ProjectMapper projectMapper;
    
    @Mock
    private EmailService emailService;
    
    @Mock
    private ApplicationEventPublisher eventPublisher;
    
    @InjectMocks
    private ProjectService projectService;
    
    @Test
    @DisplayName("프로젝트 생성 - 성공")
    void createProject_Success() {
        // given
        String ownerEmail = "owner@example.com";
        CreateProjectRequest request = CreateProjectRequest.builder()
                .name("Test Project")
                .description("Test Description")
                .priority(ProjectPriority.HIGH)
                .startDate(LocalDate.now())
                .endDate(LocalDate.now().plusDays(30))
                .memberEmails(List.of("member1@example.com", "member2@example.com"))
                .build();
        
        User owner = createTestUser(1L, ownerEmail, "Owner", "User");
        User member1 = createTestUser(2L, "member1@example.com", "Member1", "User");
        User member2 = createTestUser(3L, "member2@example.com", "Member2", "User");
        
        Project project = createTestProject(1L, "Test Project", owner);
        ProjectResponse expectedResponse = createTestProjectResponse(1L, "Test Project");
        
        when(userRepository.findByEmail(ownerEmail)).thenReturn(Optional.of(owner));
        when(userRepository.findByEmail("member1@example.com")).thenReturn(Optional.of(member1));
        when(userRepository.findByEmail("member2@example.com")).thenReturn(Optional.of(member2));
        when(projectMapper.toEntity(request)).thenReturn(project);
        when(projectRepository.save(any(Project.class))).thenReturn(project);
        when(projectMapper.toResponse(project)).thenReturn(expectedResponse);
        
        // when
        ProjectResponse result = projectService.createProject(request, ownerEmail);
        
        // then
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("Test Project");
        
        verify(userRepository).findByEmail(ownerEmail);
        verify(userRepository).findByEmail("member1@example.com");
        verify(userRepository).findByEmail("member2@example.com");
        verify(projectRepository).save(any(Project.class));
        verify(eventPublisher).publishEvent(any(ProjectCreatedEvent.class));
        verify(emailService, times(2)).sendProjectInvitation(anyString(), anyString());
    }
    
    @Test
    @DisplayName("프로젝트 생성 - 소유자를 찾을 수 없음")
    void createProject_OwnerNotFound() {
        // given
        String ownerEmail = "nonexistent@example.com";
        CreateProjectRequest request = CreateProjectRequest.builder()
                .name("Test Project")
                .build();
        
        when(userRepository.findByEmail(ownerEmail)).thenReturn(Optional.empty());
        
        // when & then
        assertThatThrownBy(() -> projectService.createProject(request, ownerEmail))
                .isInstanceOf(UserNotFoundException.class)
                .hasMessageContaining("Owner not found: " + ownerEmail);
    }
    
    @Test
    @DisplayName("프로젝트 조회 - 사용자별 프로젝트 목록")
    void getProjectsForUser_Success() {
        // given
        String userEmail = "user@example.com";
        Pageable pageable = PageRequest.of(0, 10);
        
        User user = createTestUser(1L, userEmail, "Test", "User");
        List<Project> projects = List.of(
                createTestProject(1L, "Project 1", user),
                createTestProject(2L, "Project 2", user)
        );
        Page<Project> projectPage = new PageImpl<>(projects, pageable, projects.size());
        
        when(userRepository.findByEmail(userEmail)).thenReturn(Optional.of(user));
        when(projectRepository.findProjectsForUser(user.getId(), pageable)).thenReturn(projectPage);
        when(projectMapper.toResponse(any(Project.class)))
                .thenReturn(createTestProjectResponse(1L, "Project 1"))
                .thenReturn(createTestProjectResponse(2L, "Project 2"));
        
        // when
        Page<ProjectResponse> result = projectService.getProjectsForUser(userEmail, pageable);
        
        // then
        assertThat(result).isNotNull();
        assertThat(result.getContent()).hasSize(2);
        assertThat(result.getTotalElements()).isEqualTo(2);
    }
    
    // 테스트 헬퍼 메서드들
    private User createTestUser(Long id, String email, String firstName, String lastName) {
        return User.builder()
                .id(id)
                .email(email)
                .firstName(firstName)
                .lastName(lastName)
                .role(UserRole.DEVELOPER)
                .status(UserStatus.ACTIVE)
                .build();
    }
    
    private Project createTestProject(Long id, String name, User owner) {
        return Project.builder()
                .id(id)
                .name(name)
                .description("Test Description")
                .status(ProjectStatus.PLANNING)
                .priority(ProjectPriority.MEDIUM)
                .owner(owner)
                .members(new HashSet<>())
                .tasks(new ArrayList<>())
                .build();
    }
    
    private ProjectResponse createTestProjectResponse(Long id, String name) {
        return ProjectResponse.builder()
                .id(id)
                .name(name)
                .description("Test Description")
                .status(ProjectStatus.PLANNING)
                .priority(ProjectPriority.MEDIUM)
                .taskCount(0)
                .memberCount(0)
                .build();
    }
}
```

### Step 12: 통합 테스트 작성
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@Transactional
class ProjectControllerIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private ProjectRepository projectRepository;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("taskmaster_test")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    private User testUser;
    private String jwtToken;
    
    @BeforeEach
    void setUp() {
        // 테스트 사용자 생성
        testUser = User.builder()
                .email("test@example.com")
                .password("encodedPassword")
                .firstName("Test")
                .lastName("User")
                .role(UserRole.PROJECT_MANAGER)
                .status(UserStatus.ACTIVE)
                .build();
        testUser = userRepository.save(testUser);
        
        // JWT 토큰 생성
        UserPrincipal userPrincipal = UserPrincipal.create(testUser);
        jwtToken = tokenProvider.generateToken(userPrincipal);
    }
    
    @Test
    @DisplayName("프로젝트 생성 API 테스트")
    void createProject_Integration() {
        // given
        CreateProjectRequest request = CreateProjectRequest.builder()
                .name("Integration Test Project")
                .description("Test Description")
                .priority(ProjectPriority.HIGH)
                .startDate(LocalDate.now())
                .endDate(LocalDate.now().plusDays(30))
                .build();
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(jwtToken);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        HttpEntity<CreateProjectRequest> entity = new HttpEntity<>(request, headers);
        
        // when
        ResponseEntity<ProjectResponse> response = restTemplate.postForEntity(
                "/api/v1/projects", entity, ProjectResponse.class);
        
        // then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getName()).isEqualTo("Integration Test Project");
        
        // DB 검증
        Optional<Project> savedProject = projectRepository.findById(response.getBody().getId());
        assertThat(savedProject).isPresent();
        assertThat(savedProject.get().getName()).isEqualTo("Integration Test Project");
        assertThat(savedProject.get().getOwner().getId()).isEqualTo(testUser.getId());
    }
    
    @Test
    @DisplayName("프로젝트 목록 조회 API 테스트")
    void getProjects_Integration() {
        // given
        Project project1 = Project.builder()
                .name("Project 1")
                .description("Description 1")
                .status(ProjectStatus.ACTIVE)
                .priority(ProjectPriority.HIGH)
                .owner(testUser)
                .build();
        
        Project project2 = Project.builder()
                .name("Project 2")
                .description("Description 2")
                .status(ProjectStatus.PLANNING)
                .priority(ProjectPriority.MEDIUM)
                .owner(testUser)
                .build();
        
        projectRepository.saveAll(List.of(project1, project2));
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(jwtToken);
        
        HttpEntity<Void> entity = new HttpEntity<>(headers);
        
        // when
        ResponseEntity<String> response = restTemplate.exchange(
                "/api/v1/projects?page=0&size=10", 
                HttpMethod.GET, 
                entity, 
                String.class);
        
        // then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).contains("Project 1");
        assertThat(response.getBody()).contains("Project 2");
    }
    
    @Test
    @DisplayName("권한이 없는 사용자의 프로젝트 접근 테스트")
    void accessProject_Unauthorized() {
        // given
        User anotherUser = User.builder()
                .email("another@example.com")
                .password("encodedPassword")
                .firstName("Another")
                .lastName("User")
                .role(UserRole.DEVELOPER)
                .status(UserStatus.ACTIVE)
                .build();
        anotherUser = userRepository.save(anotherUser);
        
        Project project = Project.builder()
                .name("Private Project")
                .owner(anotherUser)
                .status(ProjectStatus.ACTIVE)
                .priority(ProjectPriority.HIGH)
                .build();
        project = projectRepository.save(project);
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(jwtToken);
        
        HttpEntity<Void> entity = new HttpEntity<>(headers);
        
        // when
        ResponseEntity<String> response = restTemplate.exchange(
                "/api/v1/projects/" + project.getId(), 
                HttpMethod.GET, 
                entity, 
                String.class);
        
        // then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
    }
}
```

## 🚀 Phase 7: 운영 환경 구성

### Step 13: Docker 컨테이너화
```dockerfile
# Dockerfile
FROM openjdk:21-jdk-slim as builder

WORKDIR /app
COPY pom.xml .
COPY src ./src

RUN apt-get update && apt-get install -y maven
RUN mvn clean package -DskipTests

FROM openjdk:21-jdk-slim

RUN addgroup --system spring && adduser --system spring --ingroup spring

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

RUN chown spring:spring app.jar

USER spring:spring

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: taskmaster-postgres
    environment:
      POSTGRES_DB: taskmaster_pro
      POSTGRES_USER: taskmaster
      POSTGRES_PASSWORD: taskmaster123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - taskmaster-network

  redis:
    image: redis:7-alpine
    container_name: taskmaster-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - taskmaster-network

  app:
    build: .
    container_name: taskmaster-app
    environment:
      SPRING_PROFILES_ACTIVE: docker
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: taskmaster_pro
      DB_USERNAME: taskmaster
      DB_PASSWORD: taskmaster123
      REDIS_HOST: redis
      REDIS_PORT: 6379
      JWT_SECRET: myVerySecretKeyForJWTTokenGeneration
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
    networks:
      - taskmaster-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  taskmaster-network:
    driver: bridge
```

### Step 14: 모니터링 및 헬스체크
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final DataSource dataSource;
    
    public DatabaseHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Health health() {
        try (Connection connection = dataSource.getConnection()) {
            if (connection.isValid(1)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected")
                    .build();
            } else {
                return Health.down()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connection invalid")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("database", "PostgreSQL")
                .withDetail("status", "Connection failed")
                .build();
        }
    }
}

@Component
public class CustomMetrics {
    
    private final Counter projectCreationCounter;
    private final Counter taskCompletionCounter;
    private final Timer projectCreationTimer;
    private final Gauge activeUsersGauge;
    
    public CustomMetrics(MeterRegistry meterRegistry, UserRepository userRepository) {
        this.projectCreationCounter = Counter.builder("projects.created")
            .description("Number of projects created")
            .register(meterRegistry);
            
        this.taskCompletionCounter = Counter.builder("tasks.completed")
            .description("Number of tasks completed")
            .register(meterRegistry);
            
        this.projectCreationTimer = Timer.builder("projects.creation.time")
            .description("Time taken to create a project")
            .register(meterRegistry);
            
        this.activeUsersGauge = Gauge.builder("users.active")
            .description("Number of active users")
            .register(meterRegistry, this, metrics -> userRepository.countByStatus(UserStatus.ACTIVE));
    }
    
    public void incrementProjectCreation() {
        projectCreationCounter.increment();
    }
    
    public void incrementTaskCompletion() {
        taskCompletionCounter.increment();
    }
    
    public Timer.Sample startProjectCreationTimer() {
        return Timer.start();
    }
    
    public void recordProjectCreationTime(Timer.Sample sample) {
        sample.stop(projectCreationTimer);
    }
}
```

## 📚 Phase 8: 문서화 및 배포

### Step 15: API 문서화
```java
@Configuration
@EnableOpenApi
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("TaskMaster Pro API")
                .version("1.0.0")
                .description("Team Collaboration Task Management System API")
                .contact(new Contact()
                    .name("TaskMaster Team")
                    .email("support@taskmaster.com")
                    .url("https://taskmaster.com"))
                .license(new License()
                    .name("MIT License")
                    .url("https://opensource.org/licenses/MIT")))
            .addSecurityItem(new SecurityRequirement()
                .addList("Bearer Authentication"))
            .components(new Components()
                .addSecuritySchemes("Bearer Authentication", createAPIKeyScheme()));
    }
    
    private SecurityScheme createAPIKeyScheme() {
        return new SecurityScheme()
            .type(SecurityScheme.Type.HTTP)
            .bearerFormat("JWT")
            .scheme("bearer");
    }
}

// 컨트롤러에 API 문서 어노테이션 추가
@RestController
@RequestMapping("/api/v1/projects")
@RequiredArgsConstructor
@Validated
@Tag(name = "Project Management", description = "프로젝트 관리 API")
public class ProjectController {
    
    @Operation(
        summary = "프로젝트 생성",
        description = "새로운 프로젝트를 생성합니다. 프로젝트 소유자는 현재 로그인한 사용자가 됩니다."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "프로젝트 생성 성공",
                content = @Content(schema = @Schema(implementation = ProjectResponse.class))),
        @ApiResponse(responseCode = "400", description = "잘못된 요청",
                content = @Content(schema = @Schema(implementation = ErrorResponse.class))),
        @ApiResponse(responseCode = "401", description = "인증 실패"),
        @ApiResponse(responseCode = "403", description = "권한 없음")
    })
    @PostMapping
    public ResponseEntity<ProjectResponse> createProject(
            @Parameter(description = "프로젝트 생성 요청 데이터", required = true)
            @Valid @RequestBody CreateProjectRequest request,
            Authentication authentication) {
        
        ProjectResponse response = projectService.createProject(request, authentication.getName());
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

### Step 16: 배포 스크립트
```bash
#!/bin/bash
# deploy.sh

set -e

echo "🚀 TaskMaster Pro 배포 시작..."

# 환경 변수 확인
if [ -z "$ENVIRONMENT" ]; then
    echo "❌ ENVIRONMENT 환경 변수가 설정되지 않았습니다."
    exit 1
fi

echo "📦 환경: $ENVIRONMENT"

# 애플리케이션 빌드
echo "🔨 애플리케이션 빌드 중..."
./mvnw clean package -DskipTests

# Docker 이미지 빌드
echo "🐳 Docker 이미지 빌드 중..."
docker build -t taskmaster-pro:latest .

# 환경별 배포
case $ENVIRONMENT in
    "dev")
        echo "🚀 개발 환경 배포 중..."
        docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
        ;;
    "staging")
        echo "🚀 스테이징 환경 배포 중..."
        docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d
        ;;
    "prod")
        echo "🚀 운영 환경 배포 중..."
        docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
        ;;
    *)
        echo "❌ 지원하지 않는 환경: $ENVIRONMENT"
        exit 1
        ;;
esac

# 헬스체크
echo "🏥 헬스체크 중..."
for i in {1..30}; do
    if curl -f http://localhost:8080/actuator/health > /dev/null 2>&1; then
        echo "✅ 애플리케이션이 정상적으로 시작되었습니다!"
        break
    fi
    echo "⏳ 애플리케이션 시작 대기 중... ($i/30)"
    sleep 5
done

echo "🎉 배포 완료!"
```

## 🎯 학습 완료 체크리스트

### ✅ 완료해야 할 실습 항목들

#### Phase 1-3: 기초 설정
- [ ] Spring Boot 프로젝트 생성 및 구조 설정
- [ ] 데이터베이스 연결 및 엔티티 설계
- [ ] DTO 패턴 구현 및 MapStruct 매핑

#### Phase 4-5: 핵심 기능
- [ ] JWT 인증/인가 구현
- [ ] RESTful API 개발
- [ ] 비즈니스 로직 구현

#### Phase 6: 테스트
- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성
- [ ] Testcontainers 활용

#### Phase 7-8: 운영
- [ ] Docker 컨테이너화
- [ ] 모니터링 구성
- [ ] API 문서화
- [ ] 배포 자동화

## 🔗 연관 노트 활용
이 튜토리얼은 다음 노트들과 연결되어 있습니다:
- [[Spring Boot Project Setup]] - 프로젝트 초기 설정
- [[Spring Boot Project Structure]] - 디렉토리 구조
- [[DTO and VO Patterns]] - 데이터 구조 설계
- [[Request Response DTO Patterns]] - API DTO 패턴
- [[DTO Validation and Mapping]] - 검증과 매핑
- [[Security Patterns]] - 보안 구현
- [[Testing]] - 테스트 전략
- [[Spring Boot CI CD with GitLab and ArgoCD]] - 배포 자동화

## 📖 다음 단계
이 튜토리얼을 완료한 후에는:
1. [[Complete Project Example]]에서 전체 소스 코드 확인
2. [[Spring Boot Code Templates]]에서 재사용 가능한 템플릿 활용
3. [[Microservices Architecture Patterns]]으로 확장 아키텍처 학습

#hands-on #tutorial #실습 #프로젝트 #step-by-step

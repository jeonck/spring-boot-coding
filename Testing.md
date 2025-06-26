# Testing

## 개념
- Spring Boot의 통합 테스트 지원
- [[Java 21]] 기능 활용
- [[Spring Boot 3.5 개요]]와 완전 통합

## 주요 어노테이션
- `@SpringBootTest` - 통합 테스트
- `@WebMvcTest` - [[Spring Web MVC]] 테스트
- `@DataJpaTest` - [[Spring Data JPA]] 테스트
- `@JsonTest` - JSON 직렬화 테스트

## 통합 테스트 예제
```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class ApplicationIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void contextLoads() {
        String response = restTemplate.getForObject("/actuator/health", String.class);
        assertThat(response).contains("UP");
    }
}
```

## Mock 테스트
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void shouldReturnUser() throws Exception {
        // given
        User user = new User(1L, "john");
        when(userService.findById(1L)).thenReturn(user);
        
        // when & then
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("john"));
    }
}
```

## Testcontainers 활용
```java
@SpringBootTest
@Testcontainers
class DatabaseIntegrationTest {
    
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
```

## 관련 기술
- [[JUnit 5]]
- [[Mockito]]
- [[Testcontainers]]
- [[Spring Boot Actuator]] 테스트
- [[Dependency Injection Patterns]] Mock 활용
- [[Enterprise Design Patterns]]
- [[Exception Handling Patterns]]

## 학습 리소스
- [GeeksforGeeks Spring Boot Testing](https://www.geeksforgeeks.org/spring-boot-testing/)
- [Udemy Testing 강의](https://www.udemy.com/topic/spring-boot/?persist_locale=&locale=ko_KR&srsltid=AfmBOooB5-udDY-EPxcpbeZh3WDBZU5s62ekyzBodFEEaTcm8bQ3lyJ0)
- [[Spring Boot 학습 자료]] 테스트 섹션

#testing #junit #mockito #testcontainers
# Spring Security 6

## 개념
- Spring Boot 3.x의 보안 프레임워크
- [[Java 21]] 기능 활용
- [[Spring Boot 3.5 개요]]와 완전 통합

## 주요 특징
- Method Security 개선
- OAuth2 지원 강화
- CSRF 보호 향상
- JWT 토큰 처리

## 설정 예제
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(OAuth2ResourceServerConfigurer::jwt)
            .build();
    }
}
```

## 주요 기능
- Authentication
- Authorization
- CSRF Protection
- Session Management

## 관련 개념
- [[Spring Boot Starter]] (spring-boot-starter-security)
- [[JWT Authentication]]
- [[OAuth2 Integration]]
- [[Method Security]]

## 연동 기술
- [[Spring Web MVC]]
- [[Spring Data JPA]]
- [[Testing Security]]
- [[Enterprise Design Patterns]]
- [[API Design Patterns]]

## 학습 리소스
- [GeeksforGeeks Spring Security](https://www.geeksforgeeks.org/spring-security/)
- [Udemy Spring Security 강의](https://www.udemy.com/topic/spring-boot/?persist_locale=&locale=ko_KR&srsltid=AfmBOooB5-udDY-EPxcpbeZh3WDBZU5s62ekyzBodFEEaTcm8bQ3lyJ0)
- [[Spring Boot 학습 자료]] 보안 섹션

#spring-security #authentication #authorization
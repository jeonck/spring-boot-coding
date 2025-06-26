# Auto Configuration

## 개념
- Spring Boot의 핵심 기능
- 조건부 Bean 자동 설정
- [[Spring Boot Starter]]와 연동
- [[Spring IoC와 DI]]의 구현 기술

## 작동 원리
1. `@EnableAutoConfiguration` 어노테이션
2. `spring.factories` 파일 스캔
3. 조건부 Configuration 로딩

## 주요 어노테이션
```java
@ConditionalOnClass
@ConditionalOnMissingBean
@ConditionalOnProperty
@ConditionalOnWebApplication
```

## 사용 예제
```java
@Configuration
@ConditionalOnClass(DataSource.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    // 자동 설정 로직
}
```

## 커스터마이징
- [[Configuration Properties]]
- [[Profile-based Configuration]]
- [[Conditional Configuration]]

## 학습 자료
- [GeeksforGeeks Auto Configuration](https://www.geeksforgeeks.org/spring-boot-autoconfiguration/)
- [[Spring Boot 학습 자료]]에서 더 많은 리소스 확인

## 관련 개념
- [[Spring Boot 3.5 개요]]
- [[Spring Framework 6.x]]
- [[Spring IoC와 DI]]
- [[Spring Bean Management]]
- [[Configuration Classes]]

#auto-configuration #spring-boot #conditional
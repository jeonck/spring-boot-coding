# Configuration Classes

## 개념
- `@Configuration` 어노테이션으로 설정 클래스 정의
- 메서드 레벨에서 Bean 등록
- [[Spring IoC와 DI]]의 설정 중앙화
- [[Component Scanning]]과 함께 Bean 등록 방식

## 기본 사용법
```java
@Configuration  // 설정 클래스임을 선언
public class OpenApiConfig {

    @Bean  // 메서드 레벨에서 Bean 등록
    public OpenAPI customOpenAPI() {  // 메서드명이 Bean 이름
        return new OpenAPI()  // 반환 객체가 Spring 컨테이너에 등록
                .info(new Info()
                        .title("Spring Boot Boilerplate API")
                        .version("1.0.0"));
    }
}
```

## 의존성 주입이 있는 Bean
```java
@Configuration
public class DataInitializer {

    @Bean
    public CommandLineRunner initData(ItemRepository itemRepository) {  // 파라미터로 DI
        return args -> {
            if (itemRepository.count() == 0) {
                itemRepository.save(Item.builder()
                        .name("Sample Item 1")
                        .description("This is the first sample item")
                        .active(true)
                        .build());
            }
        };
    }
}
```

## 조건부 Bean 등록
```java
@Configuration
public class DatabaseConfig {

    @Bean
    @ConditionalOnProperty(name = "app.database.type", havingValue = "h2")
    public DataSource h2DataSource() {
        // H2 데이터소스 설정
    }

    @Bean
    @ConditionalOnProperty(name = "app.database.type", havingValue = "mysql")
    public DataSource mysqlDataSource() {
        // MySQL 데이터소스 설정
    }
}
```

## Profile 기반 설정
```java
@Configuration
@Profile("dev")
public class DevConfig {
    
    @Bean
    public CommandLineRunner devDataLoader() {
        return args -> {
            // 개발 환경 전용 데이터 로딩
        };
    }
}
```

## 외부 설정 값 주입
```java
@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig {
    
    @Bean
    public SomeService someService(AppProperties properties) {
        return new SomeService(properties.getApiKey());
    }
}
```

## vs Component Scanning
### Configuration 방식
- 써드파티 라이브러리 Bean 등록
- 복잡한 초기화 로직
- 조건부 Bean 생성

### Component Scanning 방식
- 자체 개발 클래스들
- 간단한 Bean 등록
- 스테레오타입 역할 명시

## 관련 개념
- [[Spring IoC와 DI]]
- [[Auto Configuration]]
- [[Configuration Properties]]
- [[Bean Lifecycle Management]]

#configuration #bean-definition #java-config
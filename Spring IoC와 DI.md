# Spring IoC와 DI

## 개념
- **IoC (Inversion of Control)**: 제어의 역전
- **DI (Dependency Injection)**: 의존성 주입
- [[Spring Boot 3.5 개요]]의 핵심 원리
- [[Spring Framework 6.x]]의 기반 개념

## IoC 컨테이너
Spring Container가 객체의 생명주기와 의존성을 관리

```java
@SpringBootApplication  // IoC 컨테이너 활성화
public class BoilerplateApplication {
    public static void main(String[] args) {
        SpringApplication.run(BoilerplateApplication.class, args);  // Spring 컨테이너 생성 및 시작
    }
}
```

## DI 패턴 구현
### 생성자 주입 (권장 방식)
```java
@RestController
@RequestMapping("/api/items")
public class ItemController {
    private final ItemService itemService;  // final로 불변성 보장

    @Autowired  // 생성자가 하나면 생략 가능
    public ItemController(ItemService itemService) {
        this.itemService = itemService;
    }
}
```

## Bean 등록 방식
- [[Component Scanning]] - 스테레오타입 어노테이션
- [[Configuration Classes]] - @Configuration과 @Bean
- [[Bean Lifecycle Management]]

## 의존성 주입 체인
```
ItemController → ItemService → ItemRepository
```

## 장점
- **느슨한 결합**: 인터페이스 기반 의존성
- **테스트 용이성**: Mock 객체 주입 가능
- **설정의 외부화**: [[Configuration Properties]]
- **생명주기 관리**: Spring이 싱글톤 관리

## 관련 개념
- [[Auto Configuration]]
- [[Spring Bean Management]]
- [[Dependency Injection Patterns]]
- [[Testing]]에서 Mock 활용

#ioc #dependency-injection #spring-core #design-patterns
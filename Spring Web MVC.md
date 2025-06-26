# Spring Web MVC

## 개념
- Spring의 전통적인 웹 프레임워크
- 서블릿 기반 동기 처리
- [[Spring Boot 3.5 개요]]에서 [[Virtual Threads]] 지원

## 주요 구성요소
- DispatcherServlet
- Controller
- ViewResolver
- HandlerMapping

## Java 21과의 통합
```java
@RestController
public class UserController {
    
    @Async
    @GetMapping("/users/{id}")
    public CompletableFuture<User> getUser(@PathVariable Long id) {
        // Virtual Thread에서 실행
        return CompletableFuture.supplyAsync(() -> userService.findById(id));
    }
}
```

## 주요 어노테이션
- `@Controller` / `@RestController`
- `@RequestMapping` / `@GetMapping` 등
- `@RequestBody` / `@ResponseBody`
- `@PathVariable` / `@RequestParam`

## 관련 기술
- [[Spring Boot Starter]] (spring-boot-starter-web)
- [[Spring Security 6]]
- [[Spring Data JPA]]
- [[Validation]]
- [[Dependency Injection Patterns]]
- [[Enterprise Design Patterns]]
- [[API Design Patterns]]

## vs WebFlux
- [[Spring WebFlux]] 비교
- 동기 vs 비동기 처리
- 서블릿 vs 리액티브 스택

## 학습 리소스
- [GeeksforGeeks Spring MVC](https://www.geeksforgeeks.org/spring-mvc/)
- [Udemy Spring MVC 강의](https://www.udemy.com/topic/spring-boot/?persist_locale=&locale=ko_KR&srsltid=AfmBOooB5-udDY-EPxcpbeZh3WDBZU5s62ekyzBodFEEaTcm8bQ3lyJ0)
- [[Spring Boot 학습 자료]] 참고

#spring-mvc #web #controller
# Virtual Threads

## 개념
- [[Java 21]]의 핵심 기능
- Project Loom의 결과물
- 경량 스레드 구현

## 특징
- OS 스레드보다 훨씬 가벼움
- 수백만 개의 스레드 생성 가능
- 블로킹 I/O 작업 최적화

## Spring Boot에서 활용
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public TaskExecutor taskExecutor() {
        return task -> Thread.ofVirtual().start(task);
    }
}
```

## 사용 사례
- [[Spring Web MVC]] 요청 처리
- [[Spring WebFlux]] 대안
- [[Database Connection]] 관리
- [[HTTP Client]] 호출

## 관련 개념
- [[Spring Boot 3.5 개요]]
- [[Reactive Programming]]
- [[Concurrency Patterns]]

#virtual-threads #java21 #concurrency
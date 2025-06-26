# Java 21

## 주요 특징
- [[Virtual Threads]] (Project Loom)
- [[Pattern Matching]] 향상
- [[Record Patterns]]
- [[String Templates]] (Preview)
- [[Sequenced Collections]]

## Virtual Threads
```java
// Virtual Thread 생성 예제
Thread.ofVirtual()
    .name("virtual-thread")
    .start(() -> {
        // 비동기 작업
    });
```

## Spring Boot와의 통합
- [[Spring Boot 3.5 개요]]에서 완전 지원
- [[Spring WebFlux]]와 Virtual Threads 연동
- [[Reactive Programming]] 성능 향상

## 성능 개선
- GC 최적화
- 메모리 사용량 감소
- 처리량 증가

## 관련 기술
- [[GraalVM Native Image]]
- [[Project Loom]]
- [[JVM Performance Tuning]]

#java21 #virtual-threads #performance
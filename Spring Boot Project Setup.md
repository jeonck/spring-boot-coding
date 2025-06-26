# Spring Boot Project Setup

## 프로젝트 생성
- Spring Initializr 사용
- [[Java 21]] 선택
- [[Spring Boot 3.5 개요]] 버전 선택

## 필수 의존성
```xml
<!-- Maven -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.0</version>
    <relativePath/>
</parent>

<properties>
    <java.version>21</java.version>
</properties>
```

## 기본 구조
```
src/
├── main/
│   ├── java/
│   │   └── com.example.demo/
│   │       └── DemoApplication.java
│   └── resources/
│       ├── application.yml
│       └── static/
└── test/
    └── java/
```

## 메인 클래스
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

## 관련 설정
- [[Maven Configuration]]
- [[Gradle Configuration]]
- [[Configuration Properties]]
- [[Profile-based Configuration]]
- [[Spring Boot Project Structure]]

## 연동 모듈
- [[Spring Boot Starter]]
- [[Auto Configuration]]
- [[Spring Boot DevTools]]

#project-setup #initialization #structure
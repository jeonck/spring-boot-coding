# Spring Boot Starter

## 개념
- Spring Boot의 의존성 관리 메커니즘
- 관련 라이브러리들을 한 번에 포함
- [[Auto Configuration]]과 연동
- [[Component Scanning]]을 통한 Bean 등록

## 주요 Starter들
- `spring-boot-starter-web` - [[Spring Web MVC]]
- `spring-boot-starter-webflux` - [[Spring WebFlux]]
- `spring-boot-starter-data-jpa` - [[Spring Data JPA]]
- `spring-boot-starter-security` - [[Spring Security 6]]
- `spring-boot-starter-actuator` - [[Spring Boot Actuator]]

## 사용 예제
```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```gradle
// Gradle
implementation 'org.springframework.boot:spring-boot-starter-web'
```

## 커스텀 Starter 생성
- [[Custom Starter Development]]
- [[Auto Configuration]] 설정
- 메타데이터 정의

## 관련 개념
- [[Spring Boot 3.5 개요]]
- [[Maven Configuration]]
- [[Gradle Configuration]]

#spring-boot-starter #dependencies #auto-configuration
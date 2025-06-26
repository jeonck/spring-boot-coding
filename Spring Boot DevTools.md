# Spring Boot DevTools

## 개념
- 개발 생산성 향상 도구
- [[Spring Boot 3.5 개요]]의 개발 지원 기능
- 핫 리로드 및 자동 재시작

## 주요 기능
- 자동 재시작 (Automatic Restart)
- 라이브 리로드 (Live Reload)
- 글로벌 설정
- 원격 애플리케이션 지원

## 설정
```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

## 자동 재시작 설정
```yaml
spring:
  devtools:
    restart:
      enabled: true
      exclude: static/**,public/**
    livereload:
      enabled: true
```

## IDE 설정
- IntelliJ IDEA: Build project automatically 활성화
- Eclipse: 자동 빌드 활성화
- VS Code: Java Extension Pack 사용

## 원격 개발
```java
@Configuration
public class DevToolsConfig {
    
    @Bean
    @Profile("dev")
    public RemoteSpringApplication remoteApplication() {
        return new RemoteSpringApplication("secret");
    }
}
```

## 관련 기술
- [[Spring Boot Project Setup]]
- [[Profile-based Configuration]]
- [[Hot Swapping]]

#devtools #development #hot-reload #productivity
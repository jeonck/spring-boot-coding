# Spring Boot Actuator

## 개념
- Spring Boot 애플리케이션 모니터링 및 관리
- [[Observability]] 기능 제공
- [[Spring Boot 3.5 개요]]의 핵심 구성요소

## 주요 엔드포인트
- `/actuator/health` - 애플리케이션 상태
- `/actuator/metrics` - 메트릭 정보
- `/actuator/info` - 애플리케이션 정보
- `/actuator/env` - 환경 변수
- `/actuator/loggers` - 로깅 설정

## 설정
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

## 커스텀 Health Indicator
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // 커스텀 health check 로직
        return Health.up()
            .withDetail("custom", "All systems operational")
            .build();
    }
}
```

## 보안 설정
- [[Spring Security 6]]와 연동
- 엔드포인트 접근 제어
- 민감한 정보 보호

## 관련 기술
- [[Spring Boot Starter]] (spring-boot-starter-actuator)
- [[Micrometer Metrics]]
- [[Prometheus Integration]]

#actuator #monitoring #observability
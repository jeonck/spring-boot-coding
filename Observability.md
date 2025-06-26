# Observability

## 개념
- 애플리케이션의 상태와 동작을 관찰
- [[Spring Boot 3.5 개요]]의 확장된 기능
- [[Spring Boot Actuator]]와 연동

## Three Pillars of Observability
1. **Metrics** - 수치적 측정값
2. **Logging** - 이벤트 기록
3. **Tracing** - 요청 추적

## Micrometer 통합
```java
@RestController
public class UserController {
    
    private final MeterRegistry meterRegistry;
    private final Counter userCreationCounter;
    
    public UserController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.userCreationCounter = Counter.builder("user.creation")
            .description("Number of users created")
            .register(meterRegistry);
    }
    
    @PostMapping("/users")
    public User createUser(@RequestBody User user) {
        userCreationCounter.increment();
        return userService.save(user);
    }
}
```

## Distributed Tracing
```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

## Custom Metrics
```java
@Component
public class BusinessMetrics {
    
    private final Gauge businessValue;
    
    public BusinessMetrics(MeterRegistry meterRegistry) {
        this.businessValue = Gauge.builder("business.value")
            .description("Current business value")
            .register(meterRegistry, this, BusinessMetrics::calculateValue);
    }
    
    private double calculateValue() {
        // 비즈니스 로직
        return 42.0;
    }
}
```

## 관련 기술
- [[Micrometer Metrics]]
- [[Prometheus Integration]]
- [[Grafana Dashboard]]
- [[Zipkin Tracing]]
- [[Spring Boot CI CD with GitLab and ArgoCD]]

#observability #metrics #tracing #monitoring
# Configuration Properties

## 개념
- 외부 설정 값을 Java 객체로 바인딩
- [[Spring Boot 3.5 개요]]의 핵심 기능
- [[Auto Configuration]]과 연동
- [[Spring IoC와 DI]]의 설정 관리

## 기본 사용법
```java
@ConfigurationProperties(prefix = "app")
@Component
public class AppProperties {
    private String name;
    private String version;
    private Security security = new Security();
    
    // getters, setters...
    
    public static class Security {
        private boolean enabled = true;
        private String algorithm = "SHA-256";
        // getters, setters...
    }
}
```

## application.yml 설정
```yaml
app:
  name: My Application
  version: 1.0.0
  security:
    enabled: true
    algorithm: SHA-256
```

## Java 21 Record 활용
```java
@ConfigurationProperties(prefix = "database")
public record DatabaseProperties(
    String url,
    String username,
    String password,
    Pool pool
) {
    public record Pool(int maxSize, int minSize) {}
}
```

## 검증
```java
@ConfigurationProperties(prefix = "app")
@Validated
public class ValidatedProperties {
    
    @NotBlank
    private String name;
    
    @Min(1)
    @Max(65535)
    private int port;
    
    // getters, setters...
}
```

## 관련 개념
- [[Profile-based Configuration]]
- [[Environment Variables]]
- [[Spring Boot Project Setup]]

#configuration-properties #external-config #validation
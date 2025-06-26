# Component Scanning

## 개념
- Spring이 자동으로 클래스들을 스캔하여 Bean으로 등록
- [[Spring IoC와 DI]]의 핵심 메커니즘
- [[Auto Configuration]]과 함께 동작

## 스테레오타입 어노테이션
### @Component 계열
```java
// 1. Controller Bean 등록
@RestController  // = @Controller + @ResponseBody
public class ItemController { ... }

// 2. Service Bean 등록  
@Service  // = @Component + 비즈니스 로직 계층
public class ItemService { ... }

// 3. Repository Bean 등록
@Repository  // = @Component + 데이터 접근 계층
public interface ItemRepository extends JpaRepository<Item, Long> { ... }
```

## 컴포넌트 스캔 동작 과정
1. `@SpringBootApplication`이 스캔 시작점 설정
2. 패키지와 하위 패키지 스캔
3. 스테레오타입 어노테이션 발견
4. Bean 등록 및 의존성 분석
5. [[Dependency Injection Patterns]] 적용

## 스캔 범위 설정
```java
@SpringBootApplication
@ComponentScan(basePackages = "com.example")  // 명시적 스캔 범위
public class Application { ... }
```

## 필터링
```java
@ComponentScan(
    includeFilters = @Filter(Service.class),
    excludeFilters = @Filter(Repository.class)
)
```

## 관련 기능
- [[Configuration Classes]]와 조합
- [[Spring Bean Management]]
- [[Profile-based Configuration]]

## 실제 프로젝트 예시
```java
// Spring Container가 빈들을 생성하는 순서:
// ItemRepository (가장 하위 의존성) → ItemService → ItemController
```

#component-scan #auto-detection #stereotype-annotations
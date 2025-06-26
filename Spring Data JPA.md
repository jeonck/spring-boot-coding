# Spring Data JPA

## 개념
- JPA 기반 데이터 액세스 계층
- [[Spring Boot 3.5 개요]]와 완전 통합
- [[Java 21]] Record와 연동

## 주요 특징
- Repository 패턴
- 쿼리 메서드 자동 생성
- Pagination & Sorting
- Auditing 지원

## Entity 정의
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String username;
    
    // getters, setters...
}
```

## Repository 인터페이스
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<User> findByUsernameContaining(String username);
    
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmail(@Param("email") String email);
}
```

## 관련 기능
- [[Database Configuration]]
- [[Transaction Management]]
- [[Hibernate Configuration]]
- [[Query Optimization]]
- [[Spring Bean Management]]

## 연동 기술
- [[Spring Boot Starter]] (spring-boot-starter-data-jpa)
- [[Database Connection]]
- [[Liquibase Migration]]

## 학습 리소스
- [GeeksforGeeks Spring Data JPA](https://www.geeksforgeeks.org/spring-data-jpa/)
- [Udemy JPA 강의](https://www.udemy.com/topic/spring-boot/?persist_locale=&locale=ko_KR&srsltid=AfmBOooB5-udDY-EPxcpbeZh3WDBZU5s62ekyzBodFEEaTcm8bQ3lyJ0)
- [[Spring Boot 학습 자료]] 데이터베이스 섹션

#spring-data-jpa #database #repository
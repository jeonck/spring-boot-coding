# GraalVM Native Image

## 개념
- AOT(Ahead-of-Time) 컴파일
- [[Java 21]]과 [[Spring Boot 3.5 개요]] 완전 지원
- 빠른 시작 시간과 낮은 메모리 사용량

## 주요 특징
- 네이티브 실행 파일 생성
- JVM 없이 실행 가능
- 클라우드 환경 최적화
- Cold Start 성능 향상

## Spring Boot에서 Native Build
```bash
# Maven
./mvnw -Pnative native:compile

# Gradle
./gradlew nativeCompile
```

## 설정 예제
```yaml
# application.yml
spring:
  aot:
    enabled: true
```

## 제약사항
- 리플렉션 제한
- 동적 프록시 제한
- 클래스 로딩 제한

## 최적화 기법
- [[AOT Processing]]
- [[Native Hints]]
- [[Build Time Configuration]]

## 관련 기술
- [[Spring Framework 6.x]]
- [[Cloud Native Deployment]]
- [[Container Optimization]]

#graalvm #native-image #aot #performance
# Spring Boot Docker Kubernetes Advanced

> Spring Boot 애플리케이션의 Docker 컨테이너화와 Kubernetes 환경에서의 고급 배포 및 운영

## 📋 목차

- [Docker 심화](#docker-심화)
- [Kubernetes 기초](#kubernetes-기초)
- [Spring Boot Kubernetes 통합](#spring-boot-kubernetes-통합)
- [고급 배포 전략](#고급-배포-전략)
- [서비스 메시](#서비스-메시)
- [모니터링과 로깅](#모니터링과-로깅)
- [보안 강화](#보안-강화)
- [운영 최적화](#운영-최적화)

## 🐳 Docker 심화

### 멀티 스테이지 Dockerfile 최적화

```dockerfile
# === 빌드 스테이지 ===
FROM openjdk:21-jdk-slim as builder

# 작업 디렉토리 설정
WORKDIR /app

# Gradle 파일들 먼저 복사 (캐시 최적화)
COPY gradle/ gradle/
COPY gradlew build.gradle settings.gradle ./

# 의존성 다운로드 (레이어 캐싱)
RUN ./gradlew dependencies --no-daemon

# 소스 코드 복사
COPY src/ src/

# 애플리케이션 빌드
RUN ./gradlew build -x test --no-daemon && \
    java -Djarmode=layertools -jar build/libs/*.jar extract

# === 런타임 스테이지 ===
FROM openjdk:21-jre-slim

# 보안 사용자 생성
RUN groupadd -r spring && useradd -r -g spring spring

# 필요한 패키지 설치
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        jq \
        netcat-traditional && \
    rm -rf /var/lib/apt/lists/*

# 작업 디렉토리 설정
WORKDIR /app

# 레이어별 복사 (효율적인 캐싱)
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# 파일 권한 설정
RUN chown -R spring:spring /app

# 비루트 사용자로 전환
USER spring:spring

# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 포트 노출
EXPOSE 8080

# JVM 옵션 설정
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+ExitOnOutOfMemoryError \
               -XX:+UnlockExperimentalVMOptions \
               -XX:+UseZGC \
               -Djava.security.egd=file:/dev/./urandom"

# 애플리케이션 실행
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

### Docker Compose 운영 환경

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myapp:${VERSION:-latest}
    container_name: myapp
    restart: unless-stopped
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/myapp
      - SPRING_DATASOURCE_USERNAME=${DB_USERNAME}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - SPRING_REDIS_HOST=redis
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - MANAGEMENT_OTLP_TRACING_ENDPOINT=http://jaeger:14268/api/traces
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    networks:
      - app-network
    volumes:
      - app-logs:/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    resources:
      limits:
        memory: 1G
        cpus: '0.5'
      reservations:
        memory: 512M
        cpus: '0.25'
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"

  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=${DB_USERNAME}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME} -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: kafka
    restart: unless-stopped
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 30s
      timeout: 10s
      retries: 3

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: zookeeper
    restart: unless-stopped
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
    networks:
      - app-network

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - app-network

  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    restart: unless-stopped
    ports:
      - "16686:16686"
      - "14268:14268"
    networks:
      - app-network

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  zookeeper-data:
  elasticsearch-data:
  prometheus-data:
  grafana-data:
  app-logs:
```

### Docker 이미지 최적화

```dockerfile
# === 최적화된 프로덕션 Dockerfile ===
FROM openjdk:21-jdk-slim as builder

# 캐시 최적화를 위한 레이어 분리
WORKDIR /workspace/app

# Gradle Wrapper 및 설정 파일 복사
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle ./

# 의존성 해결 (별도 레이어)
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew dependencies --no-daemon

# 소스 코드 복사 및 빌드
COPY src src
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew build -x test --no-daemon

# 레이어 추출
RUN java -Djarmode=layertools -jar build/libs/*.jar extract

# === 런타임 이미지 ===
FROM gcr.io/distroless/java21-debian12:nonroot

# 메타데이터
LABEL maintainer="team@example.com"
LABEL version="1.0.0"
LABEL description="Spring Boot Application"

# 작업 디렉토리
WORKDIR /app

# 레이어별 복사 (캐시 최적화)
COPY --from=builder /workspace/app/dependencies/ ./
COPY --from=builder /workspace/app/spring-boot-loader/ ./
COPY --from=builder /workspace/app/snapshot-dependencies/ ./
COPY --from=builder /workspace/app/application/ ./

# 포트 설정
EXPOSE 8080

# JVM 설정
ENV JVM_OPTS="-XX:+UseContainerSupport \
              -XX:InitialRAMPercentage=50.0 \
              -XX:MaxRAMPercentage=80.0 \
              -XX:+TieredCompilation \
              -XX:TieredStopAtLevel=1 \
              -Djava.awt.headless=true \
              -Djava.security.egd=file:/dev/./urandom"

# 애플리케이션 실행
ENTRYPOINT ["java", "-cp", "/app", "org.springframework.boot.loader.JarLauncher"]
```

## ☸️ Kubernetes 기초

### 네임스페이스와 리소스 쿼터

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-production
  labels:
    name: myapp-production
    environment: production

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp-production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
    services: "5"
    persistentvolumeclaims: "4"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: myapp-limits
  namespace: myapp-production
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

### ConfigMap과 Secret 관리

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp-production
data:
  application.yml: |
    spring:
      profiles:
        active: production
      
      datasource:
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
          connection-timeout: 30000
          idle-timeout: 600000
          max-lifetime: 1800000
      
      jpa:
        hibernate:
          ddl-auto: validate
        properties:
          hibernate:
            dialect: org.hibernate.dialect.PostgreSQLDialect
            jdbc:
              batch_size: 20
              order_inserts: true
              order_updates: true
      
      kafka:
        consumer:
          group-id: myapp-prod
          max-poll-records: 500
        producer:
          batch-size: 16384
          linger-ms: 5
          compression-type: snappy
      
      cache:
        type: redis
        redis:
          time-to-live: 600000
      
    server:
      port: 8080
      shutdown: graceful
    
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
      endpoint:
        health:
          show-details: when-authorized
      metrics:
        export:
          prometheus:
            enabled: true
    
    logging:
      level:
        com.example: INFO
        org.springframework.security: WARN
      pattern:
        console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"

---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp-production
type: Opaque
stringData:
  database-username: myapp_user
  database-password: secure_password_123
  redis-password: redis_password_456
  jwt-secret: jwt_secret_key_789
  kafka-username: kafka_user
  kafka-password: kafka_password_abc
```

### Deployment 설정

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-production
  labels:
    app: myapp
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: myapp
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: myapp
        image: myapp:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: SPRING_CONFIG_LOCATION
          value: "classpath:/application.yml,/config/application.yml"
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-password
        - name: SPRING_REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: redis-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: jwt-secret
        volumeMounts:
        - name: config-volume
          mountPath: /config
        - name: logs-volume
          mountPath: /app/logs
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
      volumes:
      - name: config-volume
        configMap:
          name: myapp-config
      - name: logs-volume
        emptyDir: {}
      terminationGracePeriodSeconds: 30
      nodeSelector:
        node-type: application
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - myapp
              topologyKey: kubernetes.io/hostname
```

### Service와 Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp-production
  labels:
    app: myapp
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: myapp

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp-production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /api/.*
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## 🔗 Spring Boot Kubernetes 통합

### Kubernetes Discovery Client

```java
// Kubernetes Service Discovery 설정
@Configuration
@EnableDiscoveryClient
public class KubernetesConfig {
    
    @Bean
    @ConditionalOnProperty(value = "spring.cloud.kubernetes.discovery.enabled", havingValue = "true")
    public KubernetesDiscoveryProperties kubernetesDiscoveryProperties() {
        KubernetesDiscoveryProperties properties = new KubernetesDiscoveryProperties();
        properties.setAllNamespaces(false);
        properties.setWaitCacheReady(true);
        properties.setCacheLoadingTimeoutSeconds(60);
        return properties;
    }
}

// Service Registry 설정
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        all-namespaces: false
        wait-cache-ready: true
        cache-loading-timeout-seconds: 60
        service-labels:
          app: myapp
        metadata:
          add-labels: true
          add-annotations: true
      config:
        enabled: true
        name: myapp-config
        namespace: myapp-production
      secrets:
        enabled: true
        name: myapp-secrets
        namespace: myapp-production
```

### Graceful Shutdown 구현

```java
// Graceful Shutdown 설정
@Component
@RequiredArgsConstructor
@Slf4j
public class GracefulShutdownHandler {
    
    private final ApplicationContext applicationContext;
    private final Executor taskExecutor;
    
    @EventListener
    public void handleContextClosedEvent(ContextClosedEvent event) {
        log.info("애플리케이션 종료 시작");
        
        // 1. 헬스체크 실패로 변경
        setHealthCheckToDown();
        
        // 2. 새로운 요청 수락 중지를 위한 대기
        try {
            Thread.sleep(15000); // 15초 대기
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // 3. 진행 중인 작업 완료 대기
        shutdownTaskExecutor();
        
        log.info("애플리케이션 종료 완료");
    }
    
    private void setHealthCheckToDown() {
        // 헬스체크 상태를 DOWN으로 변경
        HealthEndpoint healthEndpoint = applicationContext.getBean(HealthEndpoint.class);
        // 커스텀 헬스 인디케이터를 통해 상태 변경
    }
    
    private void shutdownTaskExecutor() {
        if (taskExecutor instanceof ThreadPoolTaskExecutor) {
            ThreadPoolTaskExecutor executor = (ThreadPoolTaskExecutor) taskExecutor;
            executor.shutdown();
            
            try {
                if (!executor.getThreadPoolExecutor().awaitTermination(30, TimeUnit.SECONDS)) {
                    log.warn("작업 완료 대기 시간 초과, 강제 종료");
                    executor.getThreadPoolExecutor().shutdownNow();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                executor.getThreadPoolExecutor().shutdownNow();
            }
        }
    }
}

// 커스텀 헬스 인디케이터
@Component
public class ShutdownHealthIndicator implements HealthIndicator {
    
    private volatile boolean shutdown = false;
    
    @Override
    public Health health() {
        if (shutdown) {
            return Health.down()
                .withDetail("status", "shutting down")
                .build();
        }
        
        return Health.up()
            .withDetail("status", "running")
            .build();
    }
    
    public void initiateShutdown() {
        this.shutdown = true;
    }
}
```

### Kubernetes Config 동적 갱신

```java
// 설정 동적 갱신
@Component
@RequiredArgsConstructor
@Slf4j
public class ConfigReloadHandler {
    
    private final Environment environment;
    private final RefreshScope refreshScope;
    
    @EventListener
    public void handleConfigMapChanged(ConfigMapChangedEvent event) {
        log.info("ConfigMap 변경 감지: {}", event.getConfigMapName());
        
        // 설정 새로고침
        refreshScope.refreshAll();
        
        log.info("설정 새로고침 완료");
    }
    
    @EventListener
    public void handleSecretChanged(SecretChangedEvent event) {
        log.info("Secret 변경 감지: {}", event.getSecretName());
        
        // 민감한 설정 새로고침
        refreshSensitiveConfigurations();
        
        log.info("Secret 새로고침 완료");
    }
    
    private void refreshSensitiveConfigurations() {
        // 데이터베이스 연결 풀 재시작
        // 캐시 설정 갱신
        // JWT 키 갱신 등
    }
}

// 설정 프로퍼티 클래스
@ConfigurationProperties(prefix = "app")
@Component
@RefreshScope
@Getter
@Setter
public class AppProperties {
    
    private Database database = new Database();
    private Cache cache = new Cache();
    private Security security = new Security();
    
    @Getter
    @Setter
    public static class Database {
        private int maxPoolSize = 20;
        private int minIdle = 5;
        private long connectionTimeout = 30000;
    }
    
    @Getter
    @Setter
    public static class Cache {
        private long ttl = 600000;
        private int maxSize = 1000;
    }
    
    @Getter
    @Setter
    public static class Security {
        private String jwtSecret;
        private long jwtExpiration = 86400000;
    }
}
```

## 🚀 고급 배포 전략

### Blue-Green 배포

```yaml
# blue-green-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
  namespace: myapp-production
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp-preview.myapp-production.svc.cluster.local
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp-active.myapp-production.svc.cluster.local
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:{{.Values.image.tag}}
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-active
  namespace: myapp-production
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: myapp

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-preview
  namespace: myapp-production
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: myapp

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: myapp-production
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.95
    interval: 60s
    count: 5
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m])) /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

### Canary 배포

```yaml
# canary-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-canary
  namespace: myapp-production
spec:
  replicas: 10
  strategy:
    canary:
      maxSurge: "25%"
      maxUnavailable: 0
      analysis:
        templates:
        - templateName: success-rate
        - templateName: latency
        startingStep: 2
        args:
        - name: service-name
          value: myapp-canary
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 20
      - pause: {duration: 2m}
      - setWeight: 40
      - pause: {duration: 2m}
      - setWeight: 60
      - pause: {duration: 2m}
      - setWeight: 80
      - pause: {duration: 2m}
      trafficRouting:
        nginx:
          stableService: myapp-stable
          canaryService: myapp-canary
          additionalIngressAnnotations:
            canary-by-header: X-Canary
            canary-by-header-value: "true"
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:{{.Values.image.tag}}
        # ... container spec

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
  namespace: myapp-production
spec:
  args:
  - name: service-name
  metrics:
  - name: latency
    successCondition: result[0] <= 0.5
    interval: 60s
    count: 5
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[5m])) by (le)
          )
```

### GitOps with ArgoCD

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/myapp-k8s-config
    targetRevision: main
    path: overlays/production
    helm:
      valueFiles:
      - values-prod.yaml
      parameters:
      - name: image.tag
        value: "1.0.0"
      - name: replicaCount
        value: "3"
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10

---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: myapp-project
  namespace: argocd
spec:
  description: MyApp Application Project
  sourceRepos:
  - 'https://github.com/mycompany/myapp-k8s-config'
  destinations:
  - namespace: myapp-*
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: ''
    kind: Service
  - group: 'networking.k8s.io'
    kind: Ingress
  roles:
  - name: developer
    description: Developer role
    policies:
    - p, proj:myapp-project:developer, applications, get, myapp-project/*, allow
    - p, proj:myapp-project:developer, applications, sync, myapp-project/*, allow
    groups:
    - mycompany:developers
```

## 🕸 서비스 메시

### Istio 통합

```yaml
# istio-configuration.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: control-plane
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
        hpaSpec:
          maxReplicas: 5
          minReplicas: 1
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: istiod
          targetCPUUtilizationPercentage: 80

---
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-production
  labels:
    istio-injection: enabled

---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: myapp-production
spec:
  mtls:
    mode: STRICT

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
  namespace: myapp-production
spec:
  hosts:
  - api.myapp.com
  gateways:
  - myapp-gateway
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: myapp-service
        subset: canary
      weight: 100
  - route:
    - destination:
        host: myapp-service
        subset: stable
      weight: 90
    - destination:
        host: myapp-service
        subset: canary
      weight: 10
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
    timeout: 10s

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-dr
  namespace: myapp-production
spec:
  host: myapp-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 64
        http2MaxRequests: 100
        maxRequestsPerConnection: 2
        maxRetries: 3
        consecutiveGatewayErrors: 5
        interval: 30s
        baseEjectionTime: 30s
        maxEjectionPercent: 50
        minHealthPercent: 50
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary

---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: myapp-production
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: myapp-tls
    hosts:
    - api.myapp.com
```

### 서비스 메시 정책

```yaml
# security-policies.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: myapp-authz
  namespace: myapp-production
spec:
  selector:
    matchLabels:
      app: myapp
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/myapp-production/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/v1/*"]
  - from:
    - source:
        principals: ["cluster.local/ns/myapp-production/sa/admin"]
    to:
    - operation:
        methods: ["*"]
        paths: ["/api/admin/*"]

---
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: myapp-jwt
  namespace: myapp-production
spec:
  selector:
    matchLabels:
      app: myapp
  jwtRules:
  - issuer: "https://auth.myapp.com"
    jwksUri: "https://auth.myapp.com/.well-known/jwks.json"
    audiences:
    - "myapp-api"

---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: myapp-metrics
  namespace: myapp-production
spec:
  metrics:
  - providers:
    - name: prometheus
  - overrides:
    - match:
        metric: ALL_METRICS
      tagOverrides:
        request_id:
          operation: UPSERT
          value: "%{REQUEST_ID}"
```

## 📊 모니터링과 로깅

### Prometheus 모니터링

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
    - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
    
    - job_name: 'myapp'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - myapp-production
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: myapp
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  myapp.yml: |
    groups:
    - name: myapp
      rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{app="myapp",status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} for app myapp"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{app="myapp"}[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "95th percentile latency is {{ $value }}s for app myapp"
      
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total{namespace="myapp-production"}[5m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod is crash looping"
          description: "Pod {{ $labels.pod }} is restarting frequently"
```

### 구조화된 로깅

```java
// Structured Logging 설정
@Configuration
public class LoggingConfig {
    
    @Bean
    public Logger structuredLogger() {
        LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
        
        // JSON Encoder 설정
        JsonEncoder jsonEncoder = new JsonEncoder();
        jsonEncoder.setContext(context);
        jsonEncoder.start();
        
        // Console Appender
        ConsoleAppender<ILoggingEvent> consoleAppender = new ConsoleAppender<>();
        consoleAppender.setContext(context);
        consoleAppender.setEncoder(jsonEncoder);
        consoleAppender.start();
        
        // Root Logger 설정
        ch.qos.logback.classic.Logger rootLogger = context.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.addAppender(consoleAppender);
        rootLogger.setLevel(Level.INFO);
        
        return rootLogger;
    }
}

// Structured Logging 서비스
@Service
@RequiredArgsConstructor
@Slf4j
public class AuditLogService {
    
    private final ObjectMapper objectMapper;
    
    public void logUserAction(String userId, String action, Map<String, Object> details) {
        try {
            Map<String, Object> logEntry = Map.of(
                "timestamp", Instant.now(),
                "level", "INFO",
                "logger", "audit",
                "thread", Thread.currentThread().getName(),
                "event_type", "user_action",
                "user_id", userId,
                "action", action,
                "details", details,
                "trace_id", getTraceId(),
                "span_id", getSpanId()
            );
            
            log.info("{}", objectMapper.writeValueAsString(logEntry));
            
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize audit log", e);
        }
    }
    
    public void logBusinessEvent(String eventType, Map<String, Object> eventData) {
        try {
            Map<String, Object> logEntry = Map.of(
                "timestamp", Instant.now(),
                "level", "INFO",
                "logger", "business",
                "event_type", eventType,
                "event_data", eventData,
                "trace_id", getTraceId(),
                "span_id", getSpanId()
            );
            
            log.info("{}", objectMapper.writeValueAsString(logEntry));
            
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize business event log", e);
        }
    }
    
    private String getTraceId() {
        return TraceContext.current() != null ? 
            TraceContext.current().traceId() : "unknown";
    }
    
    private String getSpanId() {
        return SpanContext.current() != null ? 
            SpanContext.current().spanId() : "unknown";
    }
}
```

### ELK Stack 통합

```yaml
# elasticsearch.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: cluster.name
          value: "kubernetes-logging"
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        ports:
        - containerPort: 9200
          name: rest
        - containerPort: 9300
          name: inter-node
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 1000m
            memory: 2Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: logging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.11.0
        ports:
        - containerPort: 5044
        env:
        - name: LS_JAVA_OPTS
          value: "-Xmx1g -Xms1g"
        volumeMounts:
        - name: pipeline
          mountPath: /usr/share/logstash/pipeline
        - name: config
          mountPath: /usr/share/logstash/config
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
      volumes:
      - name: pipeline
        configMap:
          name: logstash-pipeline
      - name: config
        configMap:
          name: logstash-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipeline
  namespace: logging
data:
  logstash.conf: |
    input {
      beats {
        port => 5044
      }
    }
    
    filter {
      if [kubernetes][labels][app] == "myapp" {
        json {
          source => "message"
        }
        
        date {
          match => [ "timestamp", "ISO8601" ]
        }
        
        mutate {
          add_field => { "app" => "myapp" }
          add_field => { "environment" => "production" }
        }
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch:9200"]
        index => "myapp-logs-%{+YYYY.MM.dd}"
      }
    }
```

## 🔐 보안 강화

### Pod Security Standards

```yaml
# pod-security-policy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: myapp-production
automountServiceAccountToken: false

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: myapp-production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: myapp-production
subjects:
- kind: ServiceAccount
  name: myapp
  namespace: myapp-production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: myapp-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

### Secret 관리

```yaml
# sealed-secrets.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: myapp-secrets
  namespace: myapp-production
spec:
  encryptedData:
    database-password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEQAx...
    redis-password: AgAKAoiQm84/F8l4J1P8q7N9x2Ln3HzKr...
    jwt-secret: AgAr7I9Hx6K+lM8dF9E2vB4tC7/kW3pJ...
  template:
    metadata:
      name: myapp-secrets
      namespace: myapp-production
    type: Opaque

---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: myapp-production
spec:
  provider:
    vault:
      server: "https://vault.company.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-vault-secret
  namespace: myapp-production
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-vault-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: myapp/production
      property: database-password
  - secretKey: redis-password
    remoteRef:
      key: myapp/production
      property: redis-password
```

## 📚 관련 노트

- [[Spring Boot Redis Kafka Elasticsearch Integration]] - 기술 스택 통합
- [[Microservices Architecture Patterns]] - 마이크로서비스 아키텍처
- [[Spring Boot CI CD with GitLab and ArgoCD]] - CI/CD 파이프라인
- [[Observability]] - 모니터링과 관찰성
- [[Security Patterns]] - 보안 패턴

## 🔗 외부 리소스

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Spring Boot on Kubernetes](https://spring.io/guides/gs/spring-boot-kubernetes/)
- [Istio Service Mesh](https://istio.io/latest/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)

---

*컨테이너화와 오케스트레이션은 현대 애플리케이션 배포의 핵심입니다. 점진적으로 도입하여 안정성을 확보하세요.*
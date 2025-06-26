# Spring Boot Performance Tuning

> Spring Boot 애플리케이션의 JVM, 메모리, 쿼리 최적화를 통한 성능 튜닝 완전 가이드

## 📋 목차

- [성능 튜닝 개요](#성능-튜닝-개요)
- [JVM 튜닝](#jvm-튜닝)
- [메모리 최적화](#메모리-최적화)
- [데이터베이스 최적화](#데이터베이스-최적화)
- [캐싱 전략](#캐싱-전략)
- [네트워크 최적화](#네트워크-최적화)
- [모니터링과 프로파일링](#모니터링과-프로파일링)
- [부하 테스트](#부하-테스트)

## ⚡ 성능 튜닝 개요

### 성능 지표와 목표

```yaml
핵심 성능 지표:
  응답 시간 (Response Time):
    - P50: 50% 요청이 완료되는 시간
    - P95: 95% 요청이 완료되는 시간
    - P99: 99% 요청이 완료되는 시간
  
  처리량 (Throughput):
    - RPS: 초당 요청 수
    - TPS: 초당 트랜잭션 수
  
  자원 사용률:
    - CPU 사용률: < 80%
    - 메모리 사용률: < 85%
    - 디스크 I/O 대기시간
    - 네트워크 대역폭
  
  가용성:
    - 업타임: 99.9% 이상
    - 에러율: < 0.1%

성능 최적화 단계:
  1. 측정 (Measure): 현재 성능 상태 파악
  2. 분석 (Analyze): 병목 지점 식별
  3. 최적화 (Optimize): 개선 작업 수행
  4. 검증 (Verify): 개선 효과 확인
  5. 반복 (Iterate): 지속적 개선
```

### 성능 테스트 환경 설정

```java
@Configuration
@Profile("performance-test")
public class PerformanceTestConfig {
    
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }
    
    @Bean
    public PerformanceMonitor performanceMonitor() {
        return new PerformanceMonitor();
    }
    
    @Bean
    @Primary
    public DataSource performanceDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/performance_db");
        config.setUsername("perf_user");
        config.setPassword("perf_password");
        config.setMaximumPoolSize(50);
        config.setMinimumIdle(10);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setLeakDetectionThreshold(60000);
        
        return new HikariDataSource(config);
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class PerformanceMonitor {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void handleApplicationReady(ApplicationReadyEvent event) {
        log.info("=== Performance Monitoring Started ===");
        
        // JVM 메트릭 등록
        new JvmMetrics().bindTo(meterRegistry);
        new ProcessorMetrics().bindTo(meterRegistry);
        new JvmThreadMetrics().bindTo(meterRegistry);
        new JvmGcMetrics().bindTo(meterRegistry);
        new JvmMemoryMetrics().bindTo(meterRegistry);
        
        // 커스텀 메트릭 등록
        registerCustomMetrics();
    }
    
    private void registerCustomMetrics() {
        // HTTP 요청 메트릭
        Timer.builder("http.request.duration")
            .description("HTTP request duration")
            .register(meterRegistry);
        
        // 데이터베이스 연결 메트릭
        Gauge.builder("database.connections.active")
            .description("Active database connections")
            .register(meterRegistry, this, PerformanceMonitor::getActiveConnections);
        
        // 비즈니스 메트릭
        Counter.builder("business.orders.created")
            .description("Total orders created")
            .register(meterRegistry);
    }
    
    private double getActiveConnections(PerformanceMonitor monitor) {
        // HikariCP에서 활성 연결 수 조회
        return 0.0; // 실제 구현 필요
    }
}
```

## 🚀 JVM 튜닝

### JVM 파라미터 최적화

```bash
#!/bin/bash
# production-jvm-options.sh

# 기본 JVM 설정
JAVA_OPTS="-server"

# 힙 메모리 설정
JAVA_OPTS="$JAVA_OPTS -Xms2g"           # 초기 힙 크기
JAVA_OPTS="$JAVA_OPTS -Xmx4g"           # 최대 힙 크기
JAVA_OPTS="$JAVA_OPTS -XX:NewRatio=1"   # Young:Old = 1:1
JAVA_OPTS="$JAVA_OPTS -XX:SurvivorRatio=8" # Eden:Survivor = 8:1

# 가비지 컬렉터 설정 (G1GC)
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"
JAVA_OPTS="$JAVA_OPTS -XX:MaxGCPauseMillis=200"
JAVA_OPTS="$JAVA_OPTS -XX:G1HeapRegionSize=16m"
JAVA_OPTS="$JAVA_OPTS -XX:G1NewSizePercent=30"
JAVA_OPTS="$JAVA_OPTS -XX:G1MaxNewSizePercent=40"
JAVA_OPTS="$JAVA_OPTS -XX:G1MixedGCCountTarget=8"
JAVA_OPTS="$JAVA_OPTS -XX:InitiatingHeapOccupancyPercent=15"

# 메타스페이스 설정
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=256m"
JAVA_OPTS="$JAVA_OPTS -XX:MaxMetaspaceSize=512m"

# 직접 메모리 설정
JAVA_OPTS="$JAVA_OPTS -XX:MaxDirectMemorySize=1g"

# JIT 컴파일러 최적화
JAVA_OPTS="$JAVA_OPTS -XX:+TieredCompilation"
JAVA_OPTS="$JAVA_OPTS -XX:TieredStopAtLevel=4"
JAVA_OPTS="$JAVA_OPTS -XX:CompileThreshold=10000"

# 스레드 최적화
JAVA_OPTS="$JAVA_OPTS -XX:+UseBiasedLocking"
JAVA_OPTS="$JAVA_OPTS -XX:BiasedLockingStartupDelay=0"

# 성능 모니터링
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGC"
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails"
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCTimeStamps"
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCApplicationStoppedTime"
JAVA_OPTS="$JAVA_OPTS -Xloggc:/var/log/app/gc.log"
JAVA_OPTS="$JAVA_OPTS -XX:+UseGCLogFileRotation"
JAVA_OPTS="$JAVA_OPTS -XX:NumberOfGCLogFiles=5"
JAVA_OPTS="$JAVA_OPTS -XX:GCLogFileSize=100M"

# 디버깅 및 진단
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPTS="$JAVA_OPTS -XX:HeapDumpPath=/var/log/app/heapdumps/"
JAVA_OPTS="$JAVA_OPTS -XX:+PrintCommandLineFlags"

# 애플리케이션별 최적화
JAVA_OPTS="$JAVA_OPTS -Djava.security.egd=file:/dev/./urandom"
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true"
JAVA_OPTS="$JAVA_OPTS -Dfile.encoding=UTF-8"
JAVA_OPTS="$JAVA_OPTS -Duser.timezone=Asia/Seoul"

# Spring Boot 최적화
JAVA_OPTS="$JAVA_OPTS -Dspring.jmx.enabled=false"
JAVA_OPTS="$JAVA_OPTS -Dspring.main.lazy-initialization=true"

export JAVA_OPTS

echo "JVM Options: $JAVA_OPTS"
```

### GC 알고리즘 선택과 튜닝

```java
@Configuration
public class GCTuningConfig {
    
    @Value("${app.gc.algorithm:G1}")
    private String gcAlgorithm;
    
    @PostConstruct
    public void logGCConfiguration() {
        log.info("=== GC Configuration ===");
        log.info("GC Algorithm: {}", gcAlgorithm);
        log.info("Max Heap Size: {} MB", Runtime.getRuntime().maxMemory() / 1024 / 1024);
        log.info("Available Processors: {}", Runtime.getRuntime().availableProcessors());
        
        // GC 알고리즘별 추천 설정 로깅
        logGCRecommendations();
    }
    
    private void logGCRecommendations() {
        switch (gcAlgorithm.toUpperCase()) {
            case "G1":
                log.info("G1GC Recommendations:");
                log.info("- Target pause time: 100-200ms");
                log.info("- Heap region size: 1-32MB (power of 2)");
                log.info("- Young generation: 30-40% of heap");
                break;
                
            case "PARALLEL":
                log.info("Parallel GC Recommendations:");
                log.info("- Good for throughput-oriented applications");
                log.info("- Use when pause time is not critical");
                break;
                
            case "ZGC":
                log.info("ZGC Recommendations:");
                log.info("- Ultra-low latency applications");
                log.info("- Large heap sizes (> 8GB)");
                log.info("- Consistent sub-10ms pause times");
                break;
                
            case "SHENANDOAH":
                log.info("Shenandoah GC Recommendations:");
                log.info("- Low latency applications");
                log.info("- Good for heap sizes 1GB-100GB+");
                break;
        }
    }
}

// GC 모니터링 컴포넌트
@Component
@RequiredArgsConstructor
@Slf4j
public class GCMonitor {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void startGCMonitoring(ApplicationReadyEvent event) {
        // GC 알림 리스너 등록
        for (GarbageCollectorMXBean gcBean : ManagementFactory.getGarbageCollectorMXBeans()) {
            if (gcBean instanceof NotificationEmitter) {
                NotificationEmitter emitter = (NotificationEmitter) gcBean;
                emitter.addNotificationListener(this::handleGCNotification, null, null);
            }
        }
        
        // GC 메트릭 등록
        registerGCMetrics();
    }
    
    private void handleGCNotification(Notification notification, Object handback) {
        if (notification.getType().equals(GarbageCollectionNotificationInfo.GARBAGE_COLLECTION_NOTIFICATION)) {
            GarbageCollectionNotificationInfo info = GarbageCollectionNotificationInfo.from(
                (CompositeData) notification.getUserData());
            
            String gcName = info.getGcName();
            String gcAction = info.getGcAction();
            long duration = info.getGcInfo().getDuration();
            
            log.info("GC Event: {} - {} ({}ms)", gcName, gcAction, duration);
            
            // 긴 GC 시간 경고
            if (duration > 1000) {
                log.warn("Long GC pause detected: {}ms", duration);
            }
            
            // 메트릭 기록
            meterRegistry.timer("gc.duration", "gc.name", gcName, "gc.action", gcAction)
                .record(duration, TimeUnit.MILLISECONDS);
        }
    }
    
    private void registerGCMetrics() {
        // 힙 메모리 사용률
        Gauge.builder("jvm.memory.heap.utilization")
            .description("Heap memory utilization percentage")
            .register(meterRegistry, this, monitor -> {
                MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
                MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
                return (double) heapUsage.getUsed() / heapUsage.getMax() * 100;
            });
        
        // GC 빈도
        for (GarbageCollectorMXBean gcBean : ManagementFactory.getGarbageCollectorMXBeans()) {
            Gauge.builder("jvm.gc.collections")
                .description("GC collection count")
                .tag("gc.name", gcBean.getName())
                .register(meterRegistry, gcBean, GarbageCollectorMXBean::getCollectionCount);
            
            Gauge.builder("jvm.gc.time")
                .description("GC collection time")
                .tag("gc.name", gcBean.getName())
                .register(meterRegistry, gcBean, GarbageCollectorMXBean::getCollectionTime);
        }
    }
}
```

### JIT 컴파일러 최적화

```java
@Configuration
public class JITOptimizationConfig {
    
    @PostConstruct
    public void optimizeJITCompilation() {
        log.info("=== JIT Compilation Optimization ===");
        
        // 컴파일 임계값 모니터링
        CompilationMXBean compilationBean = ManagementFactory.getCompilationMXBean();
        if (compilationBean.isCompilationTimeMonitoringSupported()) {
            log.info("JIT compilation time: {} ms", compilationBean.getTotalCompilationTime());
        }
        
        // 핫스팟 메서드 예열
        warmUpCriticalPaths();
    }
    
    private void warmUpCriticalPaths() {
        log.info("Warming up critical code paths...");
        
        // 자주 사용되는 메서드들을 미리 실행하여 JIT 컴파일 유도
        for (int i = 0; i < 20000; i++) {
            // JSON 직렬화/역직렬화
            ObjectMapper mapper = new ObjectMapper();
            try {
                String json = mapper.writeValueAsString(Map.of("test", "value"));
                mapper.readValue(json, Map.class);
            } catch (Exception e) {
                // 무시
            }
            
            // 문자열 처리
            String test = "test-string-" + i;
            test.toLowerCase().toUpperCase().substring(0, 4);
            
            // 수치 계산
            Math.sqrt(i * 2.5);
            
            // 컬렉션 연산
            List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
            list.stream().filter(n -> n > 2).map(n -> n * 2).collect(Collectors.toList());
        }
        
        log.info("Warm-up completed");
    }
}
```

## 💾 메모리 최적화

### 메모리 사용량 분석

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MemoryAnalysisService {
    
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedRate = 60000) // 1분마다
    public void analyzeMemoryUsage() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        // 힙 메모리 분석
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        long heapUsed = heapUsage.getUsed();
        long heapMax = heapUsage.getMax();
        double heapUtilization = (double) heapUsed / heapMax * 100;
        
        log.debug("Heap Memory - Used: {} MB, Max: {} MB, Utilization: {:.2f}%",
            heapUsed / 1024 / 1024, heapMax / 1024 / 1024, heapUtilization);
        
        // 논힙 메모리 분석
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        long nonHeapUsed = nonHeapUsage.getUsed();
        long nonHeapMax = nonHeapUsage.getMax();
        
        log.debug("Non-Heap Memory - Used: {} MB, Max: {} MB",
            nonHeapUsed / 1024 / 1024, nonHeapMax > 0 ? nonHeapMax / 1024 / 1024 : "Unlimited");
        
        // 메모리 풀별 분석
        analyzeMemoryPools();
        
        // 경고 조건 체크
        if (heapUtilization > 85) {
            log.warn("High heap memory utilization: {:.2f}%", heapUtilization);
        }
        
        // 메트릭 기록
        meterRegistry.gauge("memory.heap.utilization", heapUtilization);
        meterRegistry.gauge("memory.nonheap.used", nonHeapUsed);
    }
    
    private void analyzeMemoryPools() {
        for (MemoryPoolMXBean poolBean : ManagementFactory.getMemoryPoolMXBeans()) {
            MemoryUsage usage = poolBean.getUsage();
            String poolName = poolBean.getName();
            
            if (usage != null) {
                long used = usage.getUsed();
                long max = usage.getMax();
                
                log.debug("Memory Pool [{}] - Used: {} MB, Max: {} MB",
                    poolName, used / 1024 / 1024, max > 0 ? max / 1024 / 1024 : "Unlimited");
                
                meterRegistry.gauge("memory.pool.used", Tags.of("pool", poolName), used);
            }
        }
    }
    
    @EventListener
    public void handleMemoryWarning(MemoryWarningEvent event) {
        log.warn("Memory warning received: {}", event.getMessage());
        
        // 메모리 부족 시 대응
        System.gc(); // 강제 GC (운영에서는 권장하지 않음)
        
        // 캐시 정리
        clearNonEssentialCaches();
        
        // 메모리 덤프 생성 (필요시)
        if (event.isCritical()) {
            generateHeapDump();
        }
    }
    
    private void clearNonEssentialCaches() {
        // 캐시 매니저를 통한 캐시 정리
        log.info("Clearing non-essential caches due to memory pressure");
        // 구현 필요
    }
    
    private void generateHeapDump() {
        try {
            MBeanServer server = ManagementFactory.getPlatformMBeanServer();
            HotSpotDiagnosticMXBean mxBean = ManagementFactory.newPlatformMXBeanProxy(
                server, "com.sun.management:type=HotSpotDiagnostic", HotSpotDiagnosticMXBean.class);
            
            String filename = "heapdump-" + System.currentTimeMillis() + ".hprof";
            String filepath = "/tmp/" + filename;
            
            mxBean.dumpHeap(filepath, true);
            log.info("Heap dump generated: {}", filepath);
            
        } catch (Exception e) {
            log.error("Failed to generate heap dump", e);
        }
    }
}

@Component
public class MemoryWarningEvent {
    private final String message;
    private final boolean critical;
    
    public MemoryWarningEvent(String message, boolean critical) {
        this.message = message;
        this.critical = critical;
    }
    
    public String getMessage() { return message; }
    public boolean isCritical() { return critical; }
}
```

### 객체 생성 최적화

```java
@Service
@Slf4j
public class ObjectOptimizationService {
    
    // 객체 풀링
    private final ObjectPool<StringBuilder> stringBuilderPool = 
        new GenericObjectPool<>(new StringBuilderPooledObjectFactory());
    
    // 스레드 로컬 재사용
    private final ThreadLocal<SimpleDateFormat> dateFormatThreadLocal = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
    
    // 캐시된 객체 재사용
    private static final Map<String, Pattern> PATTERN_CACHE = new ConcurrentHashMap<>();
    
    // 불변 객체 재사용
    private static final List<String> IMMUTABLE_LIST = Collections.unmodifiableList(
        Arrays.asList("value1", "value2", "value3"));
    
    public String formatMessage(String template, Object... args) {
        // StringBuilder 풀 사용
        StringBuilder sb = null;
        try {
            sb = stringBuilderPool.borrowObject();
            sb.setLength(0); // 초기화
            
            return MessageFormat.format(template, args);
            
        } catch (Exception e) {
            log.error("Error formatting message", e);
            return template;
        } finally {
            if (sb != null) {
                try {
                    stringBuilderPool.returnObject(sb);
                } catch (Exception e) {
                    log.warn("Error returning StringBuilder to pool", e);
                }
            }
        }
    }
    
    public String formatDate(Date date) {
        // ThreadLocal 사용으로 객체 생성 최소화
        return dateFormatThreadLocal.get().format(date);
    }
    
    public boolean matchesPattern(String text, String patternString) {
        // 패턴 캐싱으로 중복 생성 방지
        Pattern pattern = PATTERN_CACHE.computeIfAbsent(patternString, Pattern::compile);
        return pattern.matcher(text).matches();
    }
    
    public List<String> getStaticList() {
        // 불변 객체 재사용
        return IMMUTABLE_LIST;
    }
    
    // 배치 처리로 객체 생성 최소화
    public List<ProcessedData> processBatch(List<RawData> rawDataList) {
        List<ProcessedData> results = new ArrayList<>(rawDataList.size()); // 크기 미리 지정
        
        for (RawData rawData : rawDataList) {
            ProcessedData processed = processData(rawData);
            results.add(processed);
        }
        
        return results;
    }
    
    private ProcessedData processData(RawData rawData) {
        // 빌더 패턴으로 객체 생성 최적화
        return ProcessedData.builder()
            .id(rawData.getId())
            .processedValue(rawData.getValue().toUpperCase())
            .processedAt(LocalDateTime.now())
            .build();
    }
    
    // StringBuilder 팩토리 클래스
    private static class StringBuilderPooledObjectFactory extends BasePooledObjectFactory<StringBuilder> {
        
        @Override
        public StringBuilder create() {
            return new StringBuilder(256); // 적절한 초기 크기 설정
        }
        
        @Override
        public PooledObject<StringBuilder> wrap(StringBuilder obj) {
            return new DefaultPooledObject<>(obj);
        }
        
        @Override
        public void passivateObject(PooledObject<StringBuilder> p) {
            p.getObject().setLength(0); // 재사용 전 초기화
        }
    }
}

// 메모리 효율적인 데이터 구조 예제
@Component
public class EfficientDataStructures {
    
    // 원시 타입 컬렉션 사용 (TroveCollections)
    private final TIntObjectHashMap<String> primitiveMap = new TIntObjectHashMap<>();
    
    // 메모리 효율적인 캐시
    private final Cache<String, Object> efficientCache = Caffeine.newBuilder()
        .maximumSize(10000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .softValues() // 소프트 참조 사용
        .build();
    
    // 압축된 문자열 저장
    private final Map<String, byte[]> compressedStringMap = new ConcurrentHashMap<>();
    
    public void storeCompressedString(String key, String value) {
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            try (GZIPOutputStream gzipOut = new GZIPOutputStream(baos)) {
                gzipOut.write(value.getBytes(StandardCharsets.UTF_8));
            }
            compressedStringMap.put(key, baos.toByteArray());
        } catch (IOException e) {
            log.error("Failed to compress string", e);
        }
    }
    
    public String getCompressedString(String key) {
        byte[] compressed = compressedStringMap.get(key);
        if (compressed == null) {
            return null;
        }
        
        try {
            ByteArrayInputStream bais = new ByteArrayInputStream(compressed);
            try (GZIPInputStream gzipIn = new GZIPInputStream(bais)) {
                return new String(gzipIn.readAllBytes(), StandardCharsets.UTF_8);
            }
        } catch (IOException e) {
            log.error("Failed to decompress string", e);
            return null;
        }
    }
    
    // 비트셋을 이용한 메모리 효율적인 플래그 관리
    private final BitSet flags = new BitSet(10000);
    
    public void setFlag(int index, boolean value) {
        flags.set(index, value);
    }
    
    public boolean getFlag(int index) {
        return flags.get(index);
    }
}
```

## 🗄 데이터베이스 최적화

### 연결 풀 최적화

```java
@Configuration
public class DatabaseOptimizationConfig {
    
    @Bean
    @Primary
    public DataSource optimizedDataSource() {
        HikariConfig config = new HikariConfig();
        
        // 기본 연결 설정
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        config.setDriverClassName("org.postgresql.Driver");
        
        // 연결 풀 최적화
        config.setMaximumPoolSize(getOptimalPoolSize());
        config.setMinimumIdle(Math.max(2, getOptimalPoolSize() / 4));
        config.setConnectionTimeout(30000); // 30초
        config.setIdleTimeout(600000); // 10분
        config.setMaxLifetime(1800000); // 30분
        config.setLeakDetectionThreshold(60000); // 1분
        
        // 성능 최적화 설정
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("useLocalSessionState", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");
        config.addDataSourceProperty("cacheResultSetMetadata", "true");
        config.addDataSourceProperty("cacheServerConfiguration", "true");
        config.addDataSourceProperty("elideSetAutoCommits", "true");
        config.addDataSourceProperty("maintainTimeStats", "false");
        
        // PostgreSQL 최적화
        config.addDataSourceProperty("applicationName", "MySpringBootApp");
        config.addDataSourceProperty("tcpKeepAlive", "true");
        config.addDataSourceProperty("socketTimeout", "30");
        
        return new HikariDataSource(config);
    }
    
    private int getOptimalPoolSize() {
        // CPU 코어 수 기반 최적 풀 크기 계산
        int cores = Runtime.getRuntime().availableProcessors();
        return Math.min(cores * 2 + 1, 20); // 최대 20개로 제한
    }
    
    @Bean
    public ConnectionPoolMetrics connectionPoolMetrics(DataSource dataSource) {
        return new ConnectionPoolMetrics((HikariDataSource) dataSource);
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class ConnectionPoolMetrics {
    
    private final HikariDataSource dataSource;
    private final MeterRegistry meterRegistry;
    
    @PostConstruct
    public void registerMetrics() {
        // 활성 연결 수
        Gauge.builder("hikaricp.connections.active")
            .description("Active connections")
            .register(meterRegistry, dataSource, ds -> ds.getHikariPoolMXBean().getActiveConnections());
        
        // 유휴 연결 수
        Gauge.builder("hikaricp.connections.idle")
            .description("Idle connections")
            .register(meterRegistry, dataSource, ds -> ds.getHikariPoolMXBean().getIdleConnections());
        
        // 대기 중인 스레드 수
        Gauge.builder("hikaricp.connections.pending")
            .description("Pending threads")
            .register(meterRegistry, dataSource, ds -> ds.getHikariPoolMXBean().getThreadsAwaitingConnection());
        
        // 총 연결 수
        Gauge.builder("hikaricp.connections.total")
            .description("Total connections")
            .register(meterRegistry, dataSource, ds -> ds.getHikariPoolMXBean().getTotalConnections());
    }
    
    @Scheduled(fixedRate = 30000) // 30초마다
    public void logConnectionPoolStats() {
        HikariPoolMXBean poolBean = dataSource.getHikariPoolMXBean();
        
        log.info("Connection Pool Stats - Active: {}, Idle: {}, Total: {}, Pending: {}",
            poolBean.getActiveConnections(),
            poolBean.getIdleConnections(),
            poolBean.getTotalConnections(),
            poolBean.getThreadsAwaitingConnection());
    }
}
```

### 쿼리 최적화

```java
@Repository
@RequiredArgsConstructor
@Slf4j
public class OptimizedProductRepository {
    
    private final EntityManager entityManager;
    private final JdbcTemplate jdbcTemplate;
    
    // 배치 처리로 N+1 문제 해결
    public List<Product> findProductsWithCategories(List<Long> productIds) {
        return entityManager.createQuery(
            "SELECT p FROM Product p " +
            "LEFT JOIN FETCH p.category " +
            "WHERE p.id IN :ids", Product.class)
            .setParameter("ids", productIds)
            .getResultList();
    }
    
    // 페이징과 정렬 최적화
    @Query(value = "SELECT * FROM products p " +
                   "WHERE p.category_id = :categoryId " +
                   "ORDER BY p.created_at DESC " +
                   "LIMIT :limit OFFSET :offset",
           nativeQuery = true)
    public List<Product> findByCategoryOptimized(@Param("categoryId") Long categoryId,
                                               @Param("limit") int limit,
                                               @Param("offset") int offset) {
        // 네이티브 쿼리로 성능 최적화
        return jdbcTemplate.query(
            "SELECT p.*, c.name as category_name " +
            "FROM products p " +
            "LEFT JOIN categories c ON p.category_id = c.id " +
            "WHERE p.category_id = ? " +
            "ORDER BY p.created_at DESC " +
            "LIMIT ? OFFSET ?",
            new Object[]{categoryId, limit, offset},
            this::mapRowToProduct
        );
    }
    
    // 집계 쿼리 최적화
    public ProductStatistics getProductStatistics(Long categoryId) {
        String sql = """
            SELECT 
                COUNT(*) as total_count,
                AVG(price) as avg_price,
                MIN(price) as min_price,
                MAX(price) as max_price,
                SUM(stock_quantity) as total_stock
            FROM products 
            WHERE category_id = ?
            """;
        
        return jdbcTemplate.queryForObject(sql, new Object[]{categoryId}, (rs, rowNum) ->
            ProductStatistics.builder()
                .totalCount(rs.getLong("total_count"))
                .averagePrice(rs.getBigDecimal("avg_price"))
                .minPrice(rs.getBigDecimal("min_price"))
                .maxPrice(rs.getBigDecimal("max_price"))
                .totalStock(rs.getLong("total_stock"))
                .build()
        );
    }
    
    // 배치 삽입 최적화
    @Transactional
    public void batchInsertProducts(List<Product> products) {
        String sql = """
            INSERT INTO products (name, description, price, category_id, stock_quantity, created_at) 
            VALUES (?, ?, ?, ?, ?, ?)
            """;
        
        jdbcTemplate.batchUpdate(sql, products, 100, (ps, product) -> {
            ps.setString(1, product.getName());
            ps.setString(2, product.getDescription());
            ps.setBigDecimal(3, product.getPrice());
            ps.setLong(4, product.getCategoryId());
            ps.setInt(5, product.getStockQuantity());
            ps.setTimestamp(6, Timestamp.valueOf(product.getCreatedAt()));
        });
    }
    
    // 인덱스 힌트 사용
    @Query(value = "SELECT /*+ INDEX(products, idx_products_category_created) */ * " +
                   "FROM products " +
                   "WHERE category_id = :categoryId " +
                   "AND created_at >= :fromDate",
           nativeQuery = true)
    public List<Product> findRecentByCategoryWithHint(@Param("categoryId") Long categoryId,
                                                    @Param("fromDate") LocalDateTime fromDate);
    
    // 읽기 전용 트랜잭션
    @Transactional(readOnly = true)
    public List<Product> findProductsReadOnly(ProductSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Product> query = cb.createQuery(Product.class);
        Root<Product> root = query.from(Product.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getName() != null) {
            predicates.add(cb.like(cb.lower(root.get("name")), 
                "%" + criteria.getName().toLowerCase() + "%"));
        }
        
        if (criteria.getCategoryId() != null) {
            predicates.add(cb.equal(root.get("categoryId"), criteria.getCategoryId()));
        }
        
        if (criteria.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), criteria.getMinPrice()));
        }
        
        if (criteria.getMaxPrice() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), criteria.getMaxPrice()));
        }
        
        query.where(predicates.toArray(new Predicate[0]));
        query.orderBy(cb.desc(root.get("createdAt")));
        
        return entityManager.createQuery(query)
            .setMaxResults(criteria.getLimit())
            .setFirstResult(criteria.getOffset())
            .setHint("org.hibernate.readOnly", true)
            .getResultList();
    }
    
    private Product mapRowToProduct(ResultSet rs, int rowNum) throws SQLException {
        return Product.builder()
            .id(rs.getLong("id"))
            .name(rs.getString("name"))
            .description(rs.getString("description"))
            .price(rs.getBigDecimal("price"))
            .categoryId(rs.getLong("category_id"))
            .categoryName(rs.getString("category_name"))
            .stockQuantity(rs.getInt("stock_quantity"))
            .createdAt(rs.getTimestamp("created_at").toLocalDateTime())
            .build();
    }
}

// 쿼리 성능 모니터링
@Component
@RequiredArgsConstructor
public class QueryPerformanceMonitor {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void handleSlowQuery(SlowQueryEvent event) {
        log.warn("Slow query detected - SQL: {}, Duration: {}ms", 
            event.getSql(), event.getDuration());
        
        meterRegistry.timer("database.query.slow")
            .record(event.getDuration(), TimeUnit.MILLISECONDS);
        
        // 임계값 초과 시 알림
        if (event.getDuration() > 5000) { // 5초 초과
            sendSlowQueryAlert(event);
        }
    }
    
    private void sendSlowQueryAlert(SlowQueryEvent event) {
        // 슬로우 쿼리 알림 발송 로직
        log.error("Critical slow query alert - Duration: {}ms, SQL: {}", 
            event.getDuration(), event.getSql());
    }
}
```

## 🚀 캐싱 전략

### 멀티 레벨 캐싱

```java
@Configuration
@EnableCaching
public class CachingConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(caffeineCacheBuilder());
        cacheManager.setCacheNames(Arrays.asList(
            "products", "categories", "users", "shortLived", "longLived"));
        return cacheManager;
    }
    
    private Caffeine<Object, Object> caffeineCacheBuilder() {
        return Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(Duration.ofMinutes(30))
            .expireAfterAccess(Duration.ofMinutes(10))
            .recordStats()
            .softValues(); // 메모리 부족 시 자동 해제
    }
    
    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .transactionAware()
            .build();
    }
    
    @Bean
    @Primary
    public CacheManager multiLevelCacheManager() {
        return new MultiLevelCacheManager(Arrays.asList(
            cacheManager(),           // L1: Local cache (Caffeine)
            redisCacheManager(null)   // L2: Distributed cache (Redis)
        ));
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class OptimizedProductService {
    
    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    
    // L1 캐시 (로컬 메모리)
    @Cacheable(value = "products", key = "#id")
    public ProductDto getProduct(Long id) {
        log.info("Loading product from database: {}", id);
        return productRepository.findById(id)
            .map(this::convertToDto)
            .orElse(null);
    }
    
    // L2 캐시 (Redis) - 분산 캐싱
    public ProductDto getProductFromRedis(Long id) {
        String cacheKey = "product:v2:" + id;
        
        ProductDto cached = (ProductDto) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        ProductDto product = productRepository.findById(id)
            .map(this::convertToDto)
            .orElse(null);
        
        if (product != null) {
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofHours(2));
        }
        
        return product;
    }
    
    // 조건부 캐싱
    @Cacheable(value = "products", key = "#id", condition = "#id > 0", unless = "#result == null")
    public ProductDto getProductConditional(Long id) {
        return productRepository.findById(id)
            .map(this::convertToDto)
            .orElse(null);
    }
    
    // 캐시 갱신
    @CachePut(value = "products", key = "#result.id")
    public ProductDto updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException("Product not found: " + id));
        
        product.update(request);
        Product saved = productRepository.save(product);
        
        // Redis 캐시도 함께 갱신
        String cacheKey = "product:v2:" + id;
        ProductDto dto = convertToDto(saved);
        redisTemplate.opsForValue().set(cacheKey, dto, Duration.ofHours(2));
        
        return dto;
    }
    
    // 캐시 무효화
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
        
        // Redis 캐시도 함께 삭제
        String cacheKey = "product:v2:" + id;
        redisTemplate.delete(cacheKey);
    }
    
    // 대량 캐시 무효화
    @CacheEvict(value = "products", allEntries = true)
    public void invalidateAllProductCache() {
        log.info("Invalidating all product cache");
        
        // Redis 패턴 기반 삭제
        Set<String> keys = redisTemplate.keys("product:v2:*");
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
    
    // 미리 캐시 워밍업
    @PostConstruct
    public void warmUpCache() {
        log.info("Warming up product cache...");
        
        // 인기 상품들을 미리 캐싱
        List<Long> popularProductIds = productRepository.findPopularProductIds(100);
        
        popularProductIds.parallelStream().forEach(id -> {
            try {
                getProduct(id);
            } catch (Exception e) {
                log.warn("Failed to warm up cache for product: {}", id, e);
            }
        });
        
        log.info("Cache warm-up completed for {} products", popularProductIds.size());
    }
    
    private ProductDto convertToDto(Product product) {
        return ProductDto.builder()
            .id(product.getId())
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice())
            .stockQuantity(product.getStockQuantity())
            .createdAt(product.getCreatedAt())
            .build();
    }
}

// 캐시 성능 모니터링
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheMetrics {
    
    private final CacheManager cacheManager;
    private final MeterRegistry meterRegistry;
    
    @PostConstruct
    public void registerCacheMetrics() {
        cacheManager.getCacheNames().forEach(this::registerCacheMetrics);
    }
    
    private void registerCacheMetrics(String cacheName) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache instanceof CaffeineCache) {
            com.github.benmanes.caffeine.cache.Cache<Object, Object> nativeCache = 
                ((CaffeineCache) cache).getNativeCache();
            
            CacheMetrics.monitor(meterRegistry, nativeCache, cacheName);
        }
    }
    
    @Scheduled(fixedRate = 60000) // 1분마다
    public void logCacheStatistics() {
        cacheManager.getCacheNames().forEach(cacheName -> {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache instanceof CaffeineCache) {
                com.github.benmanes.caffeine.cache.Cache<Object, Object> nativeCache = 
                    ((CaffeineCache) cache).getNativeCache();
                
                CacheStats stats = nativeCache.stats();
                
                log.info("Cache [{}] - Hit Rate: {:.2f}%, Evictions: {}, Load Time: {:.2f}ms",
                    cacheName,
                    stats.hitRate() * 100,
                    stats.evictionCount(),
                    stats.averageLoadPenalty() / 1_000_000.0);
            }
        });
    }
}
```

## 📚 관련 노트

- [[Spring Boot Docker Kubernetes Advanced]] - 컨테이너 환경 최적화
- [[Spring Data JPA]] - 데이터 액세스 최적화
- [[Spring WebFlux Reactive Programming]] - 리액티브 성능 최적화
- [[Observability]] - 성능 모니터링
- [[Spring Boot Redis Kafka Elasticsearch Integration]] - 인프라 최적화

## 🔗 외부 리소스

- [JVM Performance Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)
- [G1GC Tuning Guide](https://www.oracle.com/technical-resources/articles/java/g1gc.html)

---

*성능 튜닝은 측정과 분석을 기반으로 한 점진적 개선 과정입니다. 프로파일링을 통해 병목 지점을 정확히 파악하고 최적화하세요.*
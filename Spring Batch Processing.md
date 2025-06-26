# Spring Batch Processing

> Spring Boot에서 대용량 데이터 배치 처리를 위한 Spring Batch 완전 가이드

## 📋 목차

- [Spring Batch 개요](#spring-batch-개요)
- [기본 구성 요소](#기본-구성-요소)
- [Job과 Step 설계](#job과-step-설계)
- [ItemReader 구현](#itemreader-구현)
- [ItemProcessor 구현](#itemprocessor-구현)
- [ItemWriter 구현](#itemwriter-구현)
- [배치 모니터링](#배치-모니터링)
- [성능 최적화](#성능-최적화)
- [에러 처리와 재시작](#에러-처리와-재시작)

## 🔄 Spring Batch 개요

### Spring Batch 핵심 개념

```yaml
Spring Batch 특징:
  - 대용량 데이터 처리에 최적화
  - 트랜잭션 관리
  - 청크 기반 처리
  - 재시작 및 복구 기능
  - 확장성과 병렬 처리

주요 사용 사례:
  - 대용량 파일 처리
  - 데이터 마이그레이션
  - 정기적인 데이터 동기화
  - 리포트 생성
  - 데이터 정제 및 변환
```

### 의존성 설정

```gradle
dependencies {
    // Spring Boot Batch
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    
    // 데이터베이스
    implementation 'org.postgresql:postgresql'
    implementation 'com.h2database:h2' // 테스트용
    
    // 파일 처리
    implementation 'org.apache.commons:commons-csv:1.10.0'
    implementation 'org.apache.poi:poi:5.2.4'
    implementation 'org.apache.poi:poi-ooxml:5.2.4'
    
    // 모니터링
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    
    // 테스트
    testImplementation 'org.springframework.batch:spring-batch-test'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

## 🏗 기본 구성 요소

### Batch 설정 클래스

```java
@Configuration
@EnableBatchProcessing
@RequiredArgsConstructor
@Slf4j
public class BatchConfig {
    
    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final DataSource dataSource;
    
    @Bean
    public JobLauncher jobLauncher() throws Exception {
        TaskExecutorJobLauncher jobLauncher = new TaskExecutorJobLauncher();
        jobLauncher.setJobRepository(jobRepository);
        jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor());
        jobLauncher.afterPropertiesSet();
        return jobLauncher;
    }
    
    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("batch-");
        executor.initialize();
        return executor;
    }
    
    @Bean
    public JobExecutionListener jobExecutionListener() {
        return new JobExecutionListener() {
            @Override
            public void beforeJob(JobExecution jobExecution) {
                log.info("배치 작업 시작: {}", jobExecution.getJobInstance().getJobName());
            }
            
            @Override
            public void afterJob(JobExecution jobExecution) {
                log.info("배치 작업 완료: {}, 상태: {}, 처리 시간: {}ms", 
                    jobExecution.getJobInstance().getJobName(),
                    jobExecution.getStatus(),
                    jobExecution.getEndTime().getTime() - jobExecution.getStartTime().getTime());
            }
        };
    }
}
```

### 배치 데이터 모델

```java
// 고객 데이터 모델
@Entity
@Table(name = "customers")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Customer {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String firstName;
    
    @Column(nullable = false)
    private String lastName;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column
    private String phone;
    
    @Enumerated(EnumType.STRING)
    private CustomerStatus status;
    
    @Column(precision = 19, scale = 2)
    private BigDecimal totalPurchaseAmount;
    
    @Column
    private Integer loyaltyPoints;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    // 비즈니스 메서드
    public String getFullName() {
        return firstName + " " + lastName;
    }
    
    public void addLoyaltyPoints(Integer points) {
        this.loyaltyPoints = (this.loyaltyPoints != null ? this.loyaltyPoints : 0) + points;
    }
}

// 주문 데이터 모델
@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
    
    @Column(nullable = false)
    private String orderNumber;
    
    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @CreatedDate
    private LocalDateTime orderDate;
}

// CSV 입력 데이터 모델
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class CustomerCSVData {
    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String status;
    private String totalPurchaseAmount;
    private String loyaltyPoints;
}
```

## 💼 Job과 Step 설계

### 단순 Job 구성

```java
@Configuration
@RequiredArgsConstructor
@Slf4j
public class CustomerImportJobConfig {
    
    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final CustomerRepository customerRepository;
    
    @Bean
    public Job customerImportJob() {
        return new JobBuilder("customerImportJob", jobRepository)
            .listener(jobExecutionListener())
            .start(csvImportStep())
            .next(dataValidationStep())
            .next(loyaltyPointsCalculationStep())
            .build();
    }
    
    @Bean
    public Step csvImportStep() {
        return new StepBuilder("csvImportStep", jobRepository)
            .<CustomerCSVData, Customer>chunk(1000, transactionManager)
            .reader(csvItemReader())
            .processor(customerItemProcessor())
            .writer(customerItemWriter())
            .faultTolerant()
            .skipLimit(10)
            .skip(Exception.class)
            .listener(stepExecutionListener())
            .build();
    }
    
    @Bean
    public Step dataValidationStep() {
        return new StepBuilder("dataValidationStep", jobRepository)
            .<Customer, Customer>chunk(500, transactionManager)
            .reader(customerValidationReader())
            .processor(customerValidationProcessor())
            .writer(customerValidationWriter())
            .build();
    }
    
    @Bean
    public Step loyaltyPointsCalculationStep() {
        return new StepBuilder("loyaltyPointsCalculationStep", jobRepository)
            .tasklet(loyaltyPointsTasklet(), transactionManager)
            .build();
    }
    
    @Bean
    public StepExecutionListener stepExecutionListener() {
        return new StepExecutionListener() {
            @Override
            public void beforeStep(StepExecution stepExecution) {
                log.info("스텝 시작: {}", stepExecution.getStepName());
            }
            
            @Override
            public ExitStatus afterStep(StepExecution stepExecution) {
                log.info("스텝 완료: {}, 읽은 수: {}, 처리한 수: {}, 작성한 수: {}",
                    stepExecution.getStepName(),
                    stepExecution.getReadCount(),
                    stepExecution.getProcessCount(),
                    stepExecution.getWriteCount());
                return stepExecution.getExitStatus();
            }
        };
    }
}
```

### 병렬 처리 Job

```java
@Configuration
@RequiredArgsConstructor
public class ParallelProcessingJobConfig {
    
    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final TaskExecutor taskExecutor;
    
    @Bean
    public Job parallelCustomerProcessingJob() {
        return new JobBuilder("parallelCustomerProcessingJob", jobRepository)
            .start(partitionedStep())
            .build();
    }
    
    @Bean
    public Step partitionedStep() {
        return new StepBuilder("partitionedStep", jobRepository)
            .partitioner("slaveStep", partitioner())
            .step(slaveStep())
            .gridSize(4) // 4개의 파티션
            .taskExecutor(taskExecutor)
            .build();
    }
    
    @Bean
    public Step slaveStep() {
        return new StepBuilder("slaveStep", jobRepository)
            .<Customer, Customer>chunk(100, transactionManager)
            .reader(customerPartitionReader(null, null))
            .processor(customerProcessor())
            .writer(customerWriter())
            .build();
    }
    
    @Bean
    public Partitioner partitioner() {
        return new ColumnRangePartitioner();
    }
    
    @Bean
    @StepScope
    public ItemReader<Customer> customerPartitionReader(
            @Value("#{stepExecutionContext[minValue]}") Long minValue,
            @Value("#{stepExecutionContext[maxValue]}") Long maxValue) {
        
        JdbcCursorItemReader<Customer> reader = new JdbcCursorItemReader<>();
        reader.setDataSource(dataSource);
        reader.setSql("SELECT * FROM customers WHERE id >= ? AND id <= ? ORDER BY id");
        reader.setPreparedStatementSetter(ps -> {
            ps.setLong(1, minValue != null ? minValue : 0L);
            ps.setLong(2, maxValue != null ? maxValue : Long.MAX_VALUE);
        });
        reader.setRowMapper(new BeanPropertyRowMapper<>(Customer.class));
        
        return reader;
    }
}

// 파티셔너 구현
public class ColumnRangePartitioner implements Partitioner {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Map<String, ExecutionContext> result = new HashMap<>();
        
        // 테이블의 최소/최대 ID 조회
        long minId = getMinId();
        long maxId = getMaxId();
        
        long targetSize = (maxId - minId) / gridSize + 1;
        
        long number = 0;
        long start = minId;
        long end = start + targetSize - 1;
        
        while (start <= maxId) {
            ExecutionContext value = new ExecutionContext();
            result.put("partition" + number, value);
            
            if (end >= maxId) {
                end = maxId;
            }
            
            value.putLong("minValue", start);
            value.putLong("maxValue", end);
            
            start += targetSize;
            end += targetSize;
            number++;
        }
        
        return result;
    }
    
    private long getMinId() {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        return jdbcTemplate.queryForObject("SELECT MIN(id) FROM customers", Long.class);
    }
    
    private long getMaxId() {
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        return jdbcTemplate.queryForObject("SELECT MAX(id) FROM customers", Long.class);
    }
}
```

## 📖 ItemReader 구현

### CSV 파일 Reader

```java
@Bean
@StepScope
public FlatFileItemReader<CustomerCSVData> csvItemReader() {
    return new FlatFileItemReaderBuilder<CustomerCSVData>()
        .name("csvItemReader")
        .resource(new ClassPathResource("customers.csv"))
        .delimited()
        .delimiter(",")
        .names("firstName", "lastName", "email", "phone", "status", "totalPurchaseAmount", "loyaltyPoints")
        .linesToSkip(1) // 헤더 스킵
        .fieldSetMapper(new BeanWrapperFieldSetMapper<CustomerCSVData>() {{
            setTargetType(CustomerCSVData.class);
        }})
        .build();
}

// 커스텀 CSV Reader
@Component
public class CustomCSVItemReader implements ItemReader<CustomerCSVData> {
    
    private CSVParser csvParser;
    private Iterator<CSVRecord> recordIterator;
    private boolean initialized = false;
    
    @Value("${batch.input.file.path}")
    private String inputFilePath;
    
    @PostConstruct
    public void init() throws IOException {
        Resource resource = new FileSystemResource(inputFilePath);
        
        CSVFormat format = CSVFormat.DEFAULT
            .withFirstRecordAsHeader()
            .withIgnoreHeaderCase()
            .withTrim();
        
        Reader reader = new InputStreamReader(resource.getInputStream(), StandardCharsets.UTF_8);
        csvParser = format.parse(reader);
        recordIterator = csvParser.iterator();
        initialized = true;
    }
    
    @Override
    public CustomerCSVData read() throws Exception {
        if (!initialized) {
            init();
        }
        
        if (recordIterator.hasNext()) {
            CSVRecord record = recordIterator.next();
            
            return CustomerCSVData.builder()
                .firstName(record.get("firstName"))
                .lastName(record.get("lastName"))
                .email(record.get("email"))
                .phone(record.get("phone"))
                .status(record.get("status"))
                .totalPurchaseAmount(record.get("totalPurchaseAmount"))
                .loyaltyPoints(record.get("loyaltyPoints"))
                .build();
        }
        
        return null; // 읽을 데이터가 없음을 의미
    }
}
```

### Database Reader

```java
@Bean
@StepScope
public JdbcCursorItemReader<Customer> customerDatabaseReader() {
    return new JdbcCursorItemReaderBuilder<Customer>()
        .name("customerDatabaseReader")
        .dataSource(dataSource)
        .sql("SELECT id, first_name, last_name, email, phone, status, total_purchase_amount, loyalty_points " +
             "FROM customers WHERE status = ? ORDER BY id")
        .preparedStatementSetter(ps -> ps.setString(1, CustomerStatus.ACTIVE.name()))
        .rowMapper(new BeanPropertyRowMapper<>(Customer.class))
        .fetchSize(1000)
        .build();
}

// 페이징 기반 Reader
@Bean
@StepScope
public JdbcPagingItemReader<Customer> customerPagingReader() {
    Map<String, Object> parameterValues = new HashMap<>();
    parameterValues.put("status", CustomerStatus.ACTIVE.name());
    
    return new JdbcPagingItemReaderBuilder<Customer>()
        .name("customerPagingReader")
        .dataSource(dataSource)
        .selectClause("SELECT id, first_name, last_name, email, phone, status")
        .fromClause("FROM customers")
        .whereClause("WHERE status = :status")
        .sortKeys(Map.of("id", Order.ASCENDING))
        .parameterValues(parameterValues)
        .rowMapper(new BeanPropertyRowMapper<>(Customer.class))
        .pageSize(1000)
        .build();
}

// JPA Reader
@Bean
@StepScope
public JpaPagingItemReader<Customer> customerJpaReader() {
    return new JpaPagingItemReaderBuilder<Customer>()
        .name("customerJpaReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT c FROM Customer c WHERE c.status = :status ORDER BY c.id")
        .parameterValues(Map.of("status", CustomerStatus.ACTIVE))
        .pageSize(1000)
        .build();
}
```

### Excel 파일 Reader

```java
@Component
public class ExcelItemReader implements ItemReader<CustomerCSVData> {
    
    private Workbook workbook;
    private Sheet sheet;
    private Iterator<Row> rowIterator;
    private boolean initialized = false;
    
    @Value("${batch.input.excel.path}")
    private String excelFilePath;
    
    @PostConstruct
    public void init() throws IOException {
        Resource resource = new FileSystemResource(excelFilePath);
        
        try (InputStream inputStream = resource.getInputStream()) {
            workbook = WorkbookFactory.create(inputStream);
            sheet = workbook.getSheetAt(0);
            rowIterator = sheet.iterator();
            
            // 헤더 행 스킵
            if (rowIterator.hasNext()) {
                rowIterator.next();
            }
            
            initialized = true;
        }
    }
    
    @Override
    public CustomerCSVData read() throws Exception {
        if (!initialized) {
            init();
        }
        
        if (rowIterator.hasNext()) {
            Row row = rowIterator.next();
            
            return CustomerCSVData.builder()
                .firstName(getCellValue(row, 0))
                .lastName(getCellValue(row, 1))
                .email(getCellValue(row, 2))
                .phone(getCellValue(row, 3))
                .status(getCellValue(row, 4))
                .totalPurchaseAmount(getCellValue(row, 5))
                .loyaltyPoints(getCellValue(row, 6))
                .build();
        }
        
        return null;
    }
    
    private String getCellValue(Row row, int cellIndex) {
        Cell cell = row.getCell(cellIndex);
        if (cell == null) {
            return "";
        }
        
        return switch (cell.getCellType()) {
            case STRING -> cell.getStringCellValue();
            case NUMERIC -> String.valueOf((long) cell.getNumericCellValue());
            case BOOLEAN -> String.valueOf(cell.getBooleanCellValue());
            default -> "";
        };
    }
}
```

## ⚙️ ItemProcessor 구현

### 기본 Processor

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CustomerItemProcessor implements ItemProcessor<CustomerCSVData, Customer> {
    
    private final CustomerRepository customerRepository;
    
    @Override
    public Customer process(CustomerCSVData item) throws Exception {
        // 1. 데이터 검증
        if (!isValidCustomerData(item)) {
            log.warn("유효하지 않은 고객 데이터: {}", item.getEmail());
            return null; // null 반환 시 해당 아이템은 Writer로 전달되지 않음
        }
        
        // 2. 중복 체크
        if (customerRepository.existsByEmail(item.getEmail())) {
            log.info("이미 존재하는 고객: {}", item.getEmail());
            return null;
        }
        
        // 3. 데이터 변환
        Customer customer = convertToCustomer(item);
        
        // 4. 비즈니스 로직 적용
        applyBusinessLogic(customer);
        
        log.debug("고객 데이터 처리 완료: {}", customer.getEmail());
        return customer;
    }
    
    private boolean isValidCustomerData(CustomerCSVData data) {
        return data.getEmail() != null && 
               data.getEmail().contains("@") &&
               data.getFirstName() != null && 
               !data.getFirstName().trim().isEmpty() &&
               data.getLastName() != null && 
               !data.getLastName().trim().isEmpty();
    }
    
    private Customer convertToCustomer(CustomerCSVData data) {
        Customer customer = new Customer();
        customer.setFirstName(data.getFirstName().trim());
        customer.setLastName(data.getLastName().trim());
        customer.setEmail(data.getEmail().toLowerCase().trim());
        customer.setPhone(normalizePhoneNumber(data.getPhone()));
        customer.setStatus(parseCustomerStatus(data.getStatus()));
        customer.setTotalPurchaseAmount(parseBigDecimal(data.getTotalPurchaseAmount()));
        customer.setLoyaltyPoints(parseInteger(data.getLoyaltyPoints()));
        
        return customer;
    }
    
    private void applyBusinessLogic(Customer customer) {
        // 구매 금액에 따른 등급 설정
        if (customer.getTotalPurchaseAmount() != null) {
            if (customer.getTotalPurchaseAmount().compareTo(BigDecimal.valueOf(10000)) >= 0) {
                customer.setStatus(CustomerStatus.VIP);
            } else if (customer.getTotalPurchaseAmount().compareTo(BigDecimal.valueOf(1000)) >= 0) {
                customer.setStatus(CustomerStatus.PREMIUM);
            }
        }
        
        // 로열티 포인트 초기화
        if (customer.getLoyaltyPoints() == null) {
            customer.setLoyaltyPoints(100); // 신규 고객 웰컴 포인트
        }
    }
    
    private String normalizePhoneNumber(String phone) {
        if (phone == null) return null;
        return phone.replaceAll("[^0-9]", "");
    }
    
    private CustomerStatus parseCustomerStatus(String status) {
        try {
            return status != null ? CustomerStatus.valueOf(status.toUpperCase()) : CustomerStatus.ACTIVE;
        } catch (IllegalArgumentException e) {
            return CustomerStatus.ACTIVE;
        }
    }
    
    private BigDecimal parseBigDecimal(String value) {
        try {
            return value != null && !value.trim().isEmpty() ? new BigDecimal(value) : BigDecimal.ZERO;
        } catch (NumberFormatException e) {
            return BigDecimal.ZERO;
        }
    }
    
    private Integer parseInteger(String value) {
        try {
            return value != null && !value.trim().isEmpty() ? Integer.valueOf(value) : 0;
        } catch (NumberFormatException e) {
            return 0;
        }
    }
}
```

### 복합 Processor

```java
@Component
@RequiredArgsConstructor
public class CompositeCustomerProcessor implements ItemProcessor<CustomerCSVData, Customer> {
    
    private final List<ItemProcessor<?, ?>> processors;
    
    @PostConstruct
    public void initProcessors() {
        processors.add(new ValidationProcessor());
        processors.add(new TransformationProcessor());
        processors.add(new EnrichmentProcessor());
    }
    
    @Override
    @SuppressWarnings("unchecked")
    public Customer process(CustomerCSVData item) throws Exception {
        Object result = item;
        
        for (ItemProcessor processor : processors) {
            result = processor.process(result);
            if (result == null) {
                return null; // 중간에 null이 반환되면 처리 중단
            }
        }
        
        return (Customer) result;
    }
    
    // 검증 프로세서
    private static class ValidationProcessor implements ItemProcessor<CustomerCSVData, CustomerCSVData> {
        @Override
        public CustomerCSVData process(CustomerCSVData item) throws Exception {
            if (item.getEmail() == null || !item.getEmail().contains("@")) {
                throw new ValidationException("유효하지 않은 이메일: " + item.getEmail());
            }
            return item;
        }
    }
    
    // 변환 프로세서
    private static class TransformationProcessor implements ItemProcessor<CustomerCSVData, Customer> {
        @Override
        public Customer process(CustomerCSVData item) throws Exception {
            return Customer.builder()
                .firstName(item.getFirstName())
                .lastName(item.getLastName())
                .email(item.getEmail().toLowerCase())
                .phone(item.getPhone())
                .build();
        }
    }
    
    // 데이터 보강 프로세서
    private static class EnrichmentProcessor implements ItemProcessor<Customer, Customer> {
        @Override
        public Customer process(Customer item) throws Exception {
            // 외부 API 호출, 추가 데이터 조회 등
            item.setStatus(CustomerStatus.ACTIVE);
            item.setLoyaltyPoints(100);
            return item;
        }
    }
}
```

## ✍️ ItemWriter 구현

### Database Writer

```java
@Component
@RequiredArgsConstructor
public class CustomerJpaItemWriter implements ItemWriter<Customer> {
    
    private final CustomerRepository customerRepository;
    
    @Override
    public void write(Chunk<? extends Customer> chunk) throws Exception {
        List<? extends Customer> customers = chunk.getItems();
        
        try {
            customerRepository.saveAll(customers);
            log.info("고객 데이터 저장 완료: {} 건", customers.size());
            
        } catch (Exception e) {
            log.error("고객 데이터 저장 실패", e);
            throw e;
        }
    }
}

// JDBC Batch Writer
@Bean
public JdbcBatchItemWriter<Customer> customerJdbcWriter() {
    return new JdbcBatchItemWriterBuilder<Customer>()
        .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
        .sql("INSERT INTO customers (first_name, last_name, email, phone, status, total_purchase_amount, loyalty_points) " +
             "VALUES (:firstName, :lastName, :email, :phone, :status, :totalPurchaseAmount, :loyaltyPoints)")
        .dataSource(dataSource)
        .build();
}
```

### 파일 Writer

```java
@Bean
@StepScope
public FlatFileItemWriter<Customer> customerCsvWriter() {
    return new FlatFileItemWriterBuilder<Customer>()
        .name("customerCsvWriter")
        .resource(new FileSystemResource("output/processed-customers.csv"))
        .delimited()
        .delimiter(",")
        .names("id", "firstName", "lastName", "email", "phone", "status", "totalPurchaseAmount", "loyaltyPoints")
        .headerCallback(writer -> writer.write("ID,First Name,Last Name,Email,Phone,Status,Total Amount,Loyalty Points"))
        .build();
}

// Excel Writer
@Component
public class ExcelItemWriter implements ItemWriter<Customer> {
    
    private Workbook workbook;
    private Sheet sheet;
    private int currentRow = 0;
    
    @Value("${batch.output.excel.path}")
    private String outputPath;
    
    @PostConstruct
    public void init() {
        workbook = new XSSFWorkbook();
        sheet = workbook.createSheet("Customers");
        
        // 헤더 작성
        Row headerRow = sheet.createRow(currentRow++);
        String[] headers = {"ID", "First Name", "Last Name", "Email", "Phone", "Status", "Total Amount", "Loyalty Points"};
        for (int i = 0; i < headers.length; i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(headers[i]);
        }
    }
    
    @Override
    public void write(Chunk<? extends Customer> chunk) throws Exception {
        for (Customer customer : chunk.getItems()) {
            Row row = sheet.createRow(currentRow++);
            
            row.createCell(0).setCellValue(customer.getId() != null ? customer.getId() : 0);
            row.createCell(1).setCellValue(customer.getFirstName());
            row.createCell(2).setCellValue(customer.getLastName());
            row.createCell(3).setCellValue(customer.getEmail());
            row.createCell(4).setCellValue(customer.getPhone());
            row.createCell(5).setCellValue(customer.getStatus().name());
            row.createCell(6).setCellValue(customer.getTotalPurchaseAmount().doubleValue());
            row.createCell(7).setCellValue(customer.getLoyaltyPoints());
        }
    }
    
    @PreDestroy
    public void close() throws IOException {
        try (FileOutputStream fileOut = new FileOutputStream(outputPath)) {
            workbook.write(fileOut);
        }
        workbook.close();
    }
}
```

### 복합 Writer

```java
@Component
@RequiredArgsConstructor
public class CompositeCustomerWriter implements ItemWriter<Customer> {
    
    private final CustomerJpaItemWriter databaseWriter;
    private final ItemWriter<Customer> csvWriter;
    private final ItemWriter<Customer> auditWriter;
    
    @Override
    public void write(Chunk<? extends Customer> chunk) throws Exception {
        // 1. 데이터베이스에 저장
        databaseWriter.write(chunk);
        
        // 2. CSV 파일로 백업
        csvWriter.write(chunk);
        
        // 3. 감사 로그 작성
        auditWriter.write(chunk);
    }
}
```

## 📊 배치 모니터링

### Job Execution Listener

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class BatchJobListener implements JobExecutionListener {
    
    private final MeterRegistry meterRegistry;
    private final NotificationService notificationService;
    
    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("=== 배치 작업 시작 ===");
        log.info("Job Name: {}", jobExecution.getJobInstance().getJobName());
        log.info("Job Parameters: {}", jobExecution.getJobParameters());
        log.info("시작 시간: {}", jobExecution.getStartTime());
        
        // 메트릭 기록
        meterRegistry.counter("batch.job.started", "job", jobExecution.getJobInstance().getJobName())
            .increment();
    }
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        log.info("=== 배치 작업 완료 ===");
        log.info("Job Name: {}", jobExecution.getJobInstance().getJobName());
        log.info("종료 시간: {}", jobExecution.getEndTime());
        log.info("실행 상태: {}", jobExecution.getStatus());
        log.info("종료 상태: {}", jobExecution.getExitStatus());
        
        long duration = jobExecution.getEndTime().getTime() - jobExecution.getStartTime().getTime();
        log.info("실행 시간: {}ms", duration);
        
        // 메트릭 기록
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(Timer.builder("batch.job.duration")
            .tag("job", jobExecution.getJobInstance().getJobName())
            .tag("status", jobExecution.getStatus().name())
            .register(meterRegistry));
        
        // 실패한 경우 알림
        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            notificationService.sendBatchFailureNotification(jobExecution);
        }
        
        // Step별 상세 정보
        jobExecution.getStepExecutions().forEach(stepExecution -> {
            log.info("Step: {}, 읽기: {}, 처리: {}, 쓰기: {}, 스킵: {}",
                stepExecution.getStepName(),
                stepExecution.getReadCount(),
                stepExecution.getProcessCount(),
                stepExecution.getWriteCount(),
                stepExecution.getSkipCount());
        });
    }
}
```

### 배치 모니터링 서비스

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BatchMonitoringService {
    
    private final JobExplorer jobExplorer;
    private final JobOperator jobOperator;
    
    public List<JobExecutionSummary> getRecentJobExecutions(int count) {
        return jobExplorer.getJobNames().stream()
            .flatMap(jobName -> jobExplorer.getJobInstances(jobName, 0, count).stream())
            .flatMap(jobInstance -> jobExplorer.getJobExecutions(jobInstance).stream())
            .sorted((a, b) -> b.getStartTime().compareTo(a.getStartTime()))
            .limit(count)
            .map(this::createJobExecutionSummary)
            .collect(Collectors.toList());
    }
    
    public JobExecutionDetails getJobExecutionDetails(Long executionId) {
        JobExecution jobExecution = jobExplorer.getJobExecution(executionId);
        if (jobExecution == null) {
            throw new IllegalArgumentException("Job execution not found: " + executionId);
        }
        
        return JobExecutionDetails.builder()
            .executionId(jobExecution.getId())
            .jobName(jobExecution.getJobInstance().getJobName())
            .status(jobExecution.getStatus())
            .startTime(jobExecution.getStartTime())
            .endTime(jobExecution.getEndTime())
            .parameters(jobExecution.getJobParameters().getParameters())
            .stepExecutions(jobExecution.getStepExecutions().stream()
                .map(this::createStepExecutionSummary)
                .collect(Collectors.toList()))
            .build();
    }
    
    public void stopJob(Long executionId) throws Exception {
        jobOperator.stop(executionId);
        log.info("Job 중지 요청: {}", executionId);
    }
    
    public Long restartJob(Long executionId) throws Exception {
        return jobOperator.restart(executionId);
    }
    
    private JobExecutionSummary createJobExecutionSummary(JobExecution jobExecution) {
        return JobExecutionSummary.builder()
            .executionId(jobExecution.getId())
            .jobName(jobExecution.getJobInstance().getJobName())
            .status(jobExecution.getStatus())
            .startTime(jobExecution.getStartTime())
            .endTime(jobExecution.getEndTime())
            .duration(calculateDuration(jobExecution))
            .build();
    }
    
    private StepExecutionSummary createStepExecutionSummary(StepExecution stepExecution) {
        return StepExecutionSummary.builder()
            .stepName(stepExecution.getStepName())
            .status(stepExecution.getStatus())
            .readCount(stepExecution.getReadCount())
            .writeCount(stepExecution.getWriteCount())
            .skipCount(stepExecution.getSkipCount())
            .duration(calculateDuration(stepExecution))
            .build();
    }
    
    private long calculateDuration(JobExecution jobExecution) {
        if (jobExecution.getStartTime() != null && jobExecution.getEndTime() != null) {
            return jobExecution.getEndTime().getTime() - jobExecution.getStartTime().getTime();
        }
        return 0;
    }
}
```

### REST API 컨트롤러

```java
@RestController
@RequestMapping("/api/batch")
@RequiredArgsConstructor
public class BatchMonitoringController {
    
    private final BatchMonitoringService monitoringService;
    private final JobLauncher jobLauncher;
    private final Job customerImportJob;
    
    @GetMapping("/executions")
    public ResponseEntity<List<JobExecutionSummary>> getRecentJobExecutions(
            @RequestParam(defaultValue = "20") int count) {
        
        List<JobExecutionSummary> executions = monitoringService.getRecentJobExecutions(count);
        return ResponseEntity.ok(executions);
    }
    
    @GetMapping("/executions/{executionId}")
    public ResponseEntity<JobExecutionDetails> getJobExecutionDetails(
            @PathVariable Long executionId) {
        
        JobExecutionDetails details = monitoringService.getJobExecutionDetails(executionId);
        return ResponseEntity.ok(details);
    }
    
    @PostMapping("/jobs/{jobName}/start")
    public ResponseEntity<Long> startJob(
            @PathVariable String jobName,
            @RequestBody Map<String, String> parameters) throws Exception {
        
        JobParameters jobParameters = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .addString("inputFile", parameters.get("inputFile"))
            .toJobParameters();
        
        JobExecution jobExecution = jobLauncher.run(customerImportJob, jobParameters);
        
        return ResponseEntity.ok(jobExecution.getId());
    }
    
    @PostMapping("/executions/{executionId}/stop")
    public ResponseEntity<Void> stopJob(@PathVariable Long executionId) throws Exception {
        monitoringService.stopJob(executionId);
        return ResponseEntity.ok().build();
    }
    
    @PostMapping("/executions/{executionId}/restart")
    public ResponseEntity<Long> restartJob(@PathVariable Long executionId) throws Exception {
        Long newExecutionId = monitoringService.restartJob(executionId);
        return ResponseEntity.ok(newExecutionId);
    }
}
```

## ⚡ 성능 최적화

### 청크 크기 최적화

```java
@Configuration
public class PerformanceOptimizedBatchConfig {
    
    @Bean
    public Step optimizedStep() {
        return new StepBuilder("optimizedStep", jobRepository)
            .<Customer, Customer>chunk(calculateOptimalChunkSize(), transactionManager)
            .reader(customerReader())
            .processor(customerProcessor())
            .writer(customerWriter())
            .taskExecutor(taskExecutor()) // 병렬 처리
            .throttleLimit(4) // 동시 실행 스레드 수
            .build();
    }
    
    private int calculateOptimalChunkSize() {
        // 메모리 사용량과 성능을 고려한 청크 크기 계산
        long availableMemory = Runtime.getRuntime().maxMemory() - Runtime.getRuntime().totalMemory();
        int estimatedItemSize = 1024; // 예상 아이템 크기 (bytes)
        int maxChunkSize = (int) (availableMemory / estimatedItemSize / 10); // 여유분 고려
        
        return Math.min(Math.max(maxChunkSize, 100), 5000); // 최소 100, 최대 5000
    }
}
```

### 메모리 사용량 최적화

```java
@Component
public class MemoryOptimizedItemReader implements ItemReader<Customer> {
    
    private final DataSource dataSource;
    private Connection connection;
    private PreparedStatement statement;
    private ResultSet resultSet;
    private boolean initialized = false;
    
    public MemoryOptimizedItemReader(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Customer read() throws Exception {
        if (!initialized) {
            initialize();
        }
        
        if (resultSet.next()) {
            return mapRowToCustomer(resultSet);
        } else {
            cleanup();
            return null;
        }
    }
    
    private void initialize() throws SQLException {
        connection = dataSource.getConnection();
        
        // 스트리밍 결과셋 설정 (메모리 효율적)
        statement = connection.prepareStatement(
            "SELECT * FROM customers ORDER BY id",
            ResultSet.TYPE_FORWARD_ONLY,
            ResultSet.CONCUR_READ_ONLY
        );
        statement.setFetchSize(Integer.MIN_VALUE); // MySQL streaming
        
        resultSet = statement.executeQuery();
        initialized = true;
    }
    
    private Customer mapRowToCustomer(ResultSet rs) throws SQLException {
        return Customer.builder()
            .id(rs.getLong("id"))
            .firstName(rs.getString("first_name"))
            .lastName(rs.getString("last_name"))
            .email(rs.getString("email"))
            .phone(rs.getString("phone"))
            .status(CustomerStatus.valueOf(rs.getString("status")))
            .totalPurchaseAmount(rs.getBigDecimal("total_purchase_amount"))
            .loyaltyPoints(rs.getInt("loyalty_points"))
            .build();
    }
    
    private void cleanup() throws SQLException {
        if (resultSet != null) resultSet.close();
        if (statement != null) statement.close();
        if (connection != null) connection.close();
    }
}
```

## 🔧 에러 처리와 재시작

### 에러 처리 전략

```java
@Bean
public Step faultTolerantStep() {
    return new StepBuilder("faultTolerantStep", jobRepository)
        .<CustomerCSVData, Customer>chunk(1000, transactionManager)
        .reader(csvItemReader())
        .processor(customerItemProcessor())
        .writer(customerItemWriter())
        .faultTolerant()
        .skipLimit(100) // 최대 100개 스킵 허용
        .skip(ValidationException.class)
        .skip(DataIntegrityViolationException.class)
        .noSkip(SystemException.class) // 시스템 예외는 스킵하지 않음
        .retryLimit(3) // 최대 3회 재시도
        .retry(DeadlockLoserDataAccessException.class)
        .retry(TransientDataAccessException.class)
        .listener(skipListener())
        .listener(retryListener())
        .build();
}

@Bean
public SkipListener<CustomerCSVData, Customer> skipListener() {
    return new SkipListener<CustomerCSVData, Customer>() {
        @Override
        public void onSkipInRead(Throwable t) {
            log.warn("읽기 중 스킵 발생: {}", t.getMessage());
        }
        
        @Override
        public void onSkipInProcess(CustomerCSVData item, Throwable t) {
            log.warn("처리 중 스킵 발생: item={}, error={}", item.getEmail(), t.getMessage());
        }
        
        @Override
        public void onSkipInWrite(Customer item, Throwable t) {
            log.warn("쓰기 중 스킵 발생: item={}, error={}", item.getEmail(), t.getMessage());
        }
    };
}

@Bean
public RetryListener retryListener() {
    return new RetryListener() {
        @Override
        public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
            log.warn("재시도 중 에러 발생: 시도={}, 에러={}", context.getRetryCount(), throwable.getMessage());
        }
    };
}
```

### 재시작 가능한 Job 설계

```java
@Bean
public Job restartableJob() {
    return new JobBuilder("restartableJob", jobRepository)
        .start(restartableStep1())
        .next(restartableStep2())
        .incrementer(new RunIdIncrementer()) // 재시작 시 고유 실행 ID 생성
        .build();
}

@Bean
public Step restartableStep1() {
    return new StepBuilder("restartableStep1", jobRepository)
        .<CustomerCSVData, Customer>chunk(1000, transactionManager)
        .reader(restartableReader())
        .processor(customerItemProcessor())
        .writer(customerItemWriter())
        .build();
}

@Bean
@StepScope
public ItemReader<CustomerCSVData> restartableReader() {
    FlatFileItemReader<CustomerCSVData> reader = new FlatFileItemReader<>();
    reader.setResource(new ClassPathResource("customers.csv"));
    reader.setLineMapper(createLineMapper());
    reader.setLinesToSkip(1);
    
    // 재시작 시 중단된 지점부터 계속 읽기
    reader.setSaveState(true);
    
    return reader;
}
```

## 📚 관련 노트

- [[Spring Data JPA]] - 데이터 접근 계층
- [[Spring Boot Docker Kubernetes Advanced]] - 배치 배포
- [[Testing]] - 배치 테스트 전략
- [[Observability]] - 배치 모니터링
- [[API Design Patterns]] - 배치 API 설계

## 🔗 외부 리소스

- [Spring Batch Reference Documentation](https://docs.spring.io/spring-batch/docs/current/reference/html/)
- [Spring Batch Samples](https://github.com/spring-projects/spring-batch/tree/main/spring-batch-samples)
- [Spring Batch Best Practices](https://spring.io/blog/2023/05/19/spring-batch-best-practices)

---

*Spring Batch는 대용량 데이터 처리의 핵심 기술입니다. 점진적으로 복잡한 요구사항을 적용해 나가세요.*
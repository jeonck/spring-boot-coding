# DTO and VO Patterns

## 개념
- 데이터 전송 객체(DTO)와 값 객체(VO) 설계 패턴
- [[Enterprise Design Patterns]]의 핵심 구성요소
- [[API Design Patterns]]와 연동된 데이터 변환 전략
- [[Java 21]] Record를 활용한 현대적 구현

## DTO vs VO 차이점
- **DTO**: 계층 간 데이터 전송을 위한 객체 (가변)
- **VO**: 값 자체를 나타내는 불변 객체 (불변)
- **Entity**: 데이터베이스 테이블과 매핑되는 영속성 객체

## 1. Java 21 Record를 활용한 DTO
### 현대적 DTO 구현 (Pydantic BaseModel과 유사)
```java
// Java 21 Record 기반 DTO
public record PersonaInfoDto(
    @JsonProperty("id")
    Long id,
    
    @NotBlank(message = "이름은 필수입니다")
    @JsonProperty("name")
    String name,
    
    @JsonProperty("description")
    String description,
    
    @JsonProperty("last_version")
    Integer lastVersion,
    
    @JsonProperty("create_time")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    LocalDateTime createTime,
    
    @JsonProperty("update_time")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    LocalDateTime updateTime
) {
    // 기본값을 제공하는 생성자 (Pydantic 기본값과 유사)
    public PersonaInfoDto {
        if (createTime == null) {
            createTime = LocalDateTime.now();
        }
        if (updateTime == null) {
            updateTime = LocalDateTime.now();
        }
    }
    
    // 부분 생성자 (Optional 필드 처리)
    public PersonaInfoDto(String name, String description) {
        this(null, name, description, null, LocalDateTime.now(), LocalDateTime.now());
    }
    
    // Entity로부터 생성하는 팩토리 메서드 (from_attributes와 유사)
    public static PersonaInfoDto from(PersonaInfo entity) {
        return new PersonaInfoDto(
            entity.getId(),
            entity.getName(),
            entity.getDescription(),
            entity.getLastVersion(),
            entity.getCreateTime(),
            entity.getUpdateTime()
        );
    }
    
    // Map으로 변환 (dict()와 유사)
    public Map<String, Object> toMap() {
        Map<String, Object> map = new HashMap<>();
        map.put("id", id);
        map.put("name", name);
        map.put("description", description);
        map.put("last_version", lastVersion);
        map.put("create_time", createTime);
        map.put("update_time", updateTime);
        return map;
    }
    
    // JSON 문자열로 변환
    public String toJson() {
        try {
            ObjectMapper objectMapper = new ObjectMapper();
            objectMapper.registerModule(new JavaTimeModule());
            return objectMapper.writeValueAsString(this);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON 변환 실패", e);
        }
    }
}
```

## 2. 전통적인 Class 기반 DTO
### Builder 패턴과 Validation 적용
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class PersonaInfoDto {
    
    @JsonProperty("id")
    private Long id;
    
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 100, message = "이름은 2-100자 사이여야 합니다")
    @JsonProperty("name")
    private String name;
    
    @Size(max = 500, message = "설명은 500자를 초과할 수 없습니다")
    @JsonProperty("description")
    private String description;
    
    @Min(value = 1, message = "버전은 1 이상이어야 합니다")
    @JsonProperty("last_version")
    private Integer lastVersion;
    
    @JsonProperty("create_time")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;
    
    @JsonProperty("update_time")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime updateTime;
    
    // 기본값 설정 (Pydantic 기본값과 유사)
    @Builder.Default
    private LocalDateTime createTime = LocalDateTime.now();
    
    @Builder.Default
    private LocalDateTime updateTime = LocalDateTime.now();
    
    // Entity 변환 메서드
    public static PersonaInfoDto from(PersonaInfo entity) {
        if (entity == null) {
            return null;
        }
        
        return PersonaInfoDto.builder()
            .id(entity.getId())
            .name(entity.getName())
            .description(entity.getDescription())
            .lastVersion(entity.getLastVersion())
            .createTime(entity.getCreateTime())
            .updateTime(entity.getUpdateTime())
            .build();
    }
    
    // Entity 목록 변환
    public static List<PersonaInfoDto> fromList(List<PersonaInfo> entities) {
        return entities.stream()
            .map(PersonaInfoDto::from)
            .collect(Collectors.toList());
    }
    
    // Page 변환
    public static Page<PersonaInfoDto> fromPage(Page<PersonaInfo> entityPage) {
        return entityPage.map(PersonaInfoDto::from);
    }
}
```

## 3. 커스텀 JSON 직렬화 (field_serializer와 유사)
### Jackson Serializer를 활용한 커스텀 변환
```java
// 커스텀 DateTime Serializer (timestamp 형식으로 출력)
public class TimestampSerializer extends JsonSerializer<LocalDateTime> {
    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider serializers) 
            throws IOException {
        if (value != null) {
            long timestamp = value.toEpochSecond(ZoneOffset.UTC);
            gen.writeNumber(timestamp);
        }
    }
}

// 커스텀 DateTime Deserializer
public class TimestampDeserializer extends JsonDeserializer<LocalDateTime> {
    @Override
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException {
        long timestamp = p.getValueAsLong();
        return LocalDateTime.ofEpochSecond(timestamp, 0, ZoneOffset.UTC);
    }
}

// 적용된 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PersonaInfoDto {
    
    private Long id;
    private String name;
    private String description;
    private Integer lastVersion;
    
    // timestamp 형식으로 직렬화/역직렬화
    @JsonSerialize(using = TimestampSerializer.class)
    @JsonDeserialize(using = TimestampDeserializer.class)
    @JsonProperty("create_time")
    private LocalDateTime createTime;
    
    @JsonSerialize(using = TimestampSerializer.class)
    @JsonDeserialize(using = TimestampDeserializer.class)
    @JsonProperty("update_time")
    private LocalDateTime updateTime;
    
    // 또는 어노테이션으로 간단히 처리
    @JsonFormat(shape = JsonFormat.Shape.NUMBER)
    @JsonProperty("created_timestamp")
    private LocalDateTime createdTimestamp;
}
```

## 4. MapStruct를 활용한 자동 매핑
### 자동 변환 및 타입 안전성 보장
```java
@Mapper(componentModel = "spring", 
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface PersonaMapper {
    
    PersonaMapper INSTANCE = Mappers.getMapper(PersonaMapper.class);
    
    // Entity → DTO 변환
    @Mapping(source = "createTime", target = "createTime", 
             dateFormat = "yyyy-MM-dd HH:mm:ss")
    @Mapping(source = "updateTime", target = "updateTime", 
             dateFormat = "yyyy-MM-dd HH:mm:ss")
    PersonaInfoDto toDto(PersonaInfo entity);
    
    // DTO → Entity 변환
    @Mapping(target = "id", ignore = true)
    @Mapping(source = "createTime", target = "createTime", 
             dateFormat = "yyyy-MM-dd HH:mm:ss")
    PersonaInfo toEntity(PersonaInfoDto dto);
    
    // 리스트 변환
    List<PersonaInfoDto> toDtoList(List<PersonaInfo> entities);
    List<PersonaInfo> toEntityList(List<PersonaInfoDto> dtos);
    
    // 부분 업데이트 (null 값은 무시)
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntityFromDto(PersonaInfoDto dto, @MappingTarget PersonaInfo entity);
    
    // 커스텀 변환 메서드
    @Named("timestampToLocalDateTime")
    default LocalDateTime timestampToLocalDateTime(Long timestamp) {
        return timestamp != null ? 
            LocalDateTime.ofEpochSecond(timestamp, 0, ZoneOffset.UTC) : null;
    }
    
    @Named("localDateTimeToTimestamp")
    default Long localDateTimeToTimestamp(LocalDateTime dateTime) {
        return dateTime != null ? dateTime.toEpochSecond(ZoneOffset.UTC) : null;
    }
}
```

## 5. 검증 그룹과 조건부 검증
### 상황별 다른 검증 규칙 적용
```java
// 검증 그룹 인터페이스
public interface CreateValidation {}
public interface UpdateValidation {}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PersonaInfoDto {
    
    @Null(groups = CreateValidation.class, message = "생성 시 ID는 null이어야 합니다")
    @NotNull(groups = UpdateValidation.class, message = "수정 시 ID는 필수입니다")
    private Long id;
    
    @NotBlank(groups = {CreateValidation.class, UpdateValidation.class}, 
              message = "이름은 필수입니다")
    @Size(min = 2, max = 100, message = "이름은 2-100자 사이여야 합니다")
    private String name;
    
    @Size(max = 500, message = "설명은 500자를 초과할 수 없습니다")
    private String description;
    
    @Min(value = 1, message = "버전은 1 이상이어야 합니다")
    private Integer lastVersion;
    
    @Null(groups = CreateValidation.class, message = "생성 시간은 자동 설정됩니다")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;
    
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime updateTime;
    
    // 커스텀 검증 어노테이션
    @ValidPersonaName
    private String name;
    
    // 조건부 검증
    @AssertTrue(message = "설명이 있는 경우 이름은 10자 이상이어야 합니다")
    private boolean isValidNameLength() {
        if (description != null && !description.trim().isEmpty()) {
            return name != null && name.length() >= 10;
        }
        return true;
    }
}

// 컨트롤러에서 그룹별 검증 사용
@RestController
@RequestMapping("/api/personas")
@RequiredArgsConstructor
public class PersonaController {
    
    private final PersonaService personaService;
    
    @PostMapping
    public ResponseEntity<PersonaInfoDto> createPersona(
            @Validated(CreateValidation.class) @RequestBody PersonaInfoDto request) {
        PersonaInfoDto created = personaService.createPersona(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<PersonaInfoDto> updatePersona(
            @PathVariable Long id,
            @Validated(UpdateValidation.class) @RequestBody PersonaInfoDto request) {
        PersonaInfoDto updated = personaService.updatePersona(id, request);
        return ResponseEntity.ok(updated);
    }
}
```

## 6. 중첩 DTO와 복합 객체
### 복잡한 데이터 구조 처리
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class PersonaDetailDto {
    
    private Long id;
    private String name;
    private String description;
    
    // 중첩 DTO
    @Valid
    private PersonaConfigDto config;
    
    // 컬렉션 DTO
    @Valid
    @Size(max = 10, message = "태그는 최대 10개까지 가능합니다")
    private List<PersonaTagDto> tags;
    
    // Map 형태 데이터
    @JsonAnyGetter
    private Map<String, Object> additionalProperties = new HashMap<>();
    
    @JsonAnySetter
    public void setAdditionalProperty(String key, Object value) {
        this.additionalProperties.put(key, value);
    }
    
    // 중첩 DTO 클래스
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class PersonaConfigDto {
        
        @NotNull(message = "설정 타입은 필수입니다")
        private ConfigType type;
        
        @Valid
        private Map<String, String> parameters;
        
        @JsonCreator
        public static PersonaConfigDto fromMap(@JsonProperty("config") Map<String, Object> config) {
            // Map에서 DTO로 변환 로직
            return PersonaConfigDto.builder()
                .type(ConfigType.valueOf((String) config.get("type")))
                .parameters((Map<String, String>) config.get("parameters"))
                .build();
        }
    }
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class PersonaTagDto {
        
        @NotBlank(message = "태그 이름은 필수입니다")
        @Size(max = 50, message = "태그 이름은 50자를 초과할 수 없습니다")
        private String name;
        
        @Pattern(regexp = "^#[A-Fa-f0-9]{6}$", message = "색상은 HEX 형식이어야 합니다")
        private String color;
        
        private Integer order;
    }
}
```

## 7. 동적 DTO와 런타임 검증
### 유연한 데이터 구조 처리
```java
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class DynamicPersonaDto {
    
    // 고정 필드
    private Long id;
    private String name;
    private String type;
    
    // 동적 필드들
    @JsonIgnore
    private Map<String, Object> dynamicFields = new HashMap<>();
    
    @JsonAnyGetter
    public Map<String, Object> getDynamicFields() {
        return dynamicFields;
    }
    
    @JsonAnySetter
    public void setDynamicField(String key, Object value) {
        this.dynamicFields.put(key, value);
    }
    
    // 타입 안전한 동적 필드 접근
    @SuppressWarnings("unchecked")
    public <T> T getDynamicField(String key, Class<T> type) {
        Object value = dynamicFields.get(key);
        if (value == null) {
            return null;
        }
        
        if (type.isInstance(value)) {
            return (T) value;
        }
        
        // 타입 변환 시도
        return convertValue(value, type);
    }
    
    private <T> T convertValue(Object value, Class<T> targetType) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            return mapper.convertValue(value, targetType);
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException(
                String.format("Cannot convert %s to %s", value.getClass(), targetType), e);
        }
    }
    
    // 스키마 기반 검증
    public void validateAgainstSchema(Map<String, FieldSchema> schema) {
        for (Map.Entry<String, FieldSchema> entry : schema.entrySet()) {
            String fieldName = entry.getKey();
            FieldSchema fieldSchema = entry.getValue();
            Object fieldValue = dynamicFields.get(fieldName);
            
            // 필수 필드 검증
            if (fieldSchema.isRequired() && fieldValue == null) {
                throw new ValidationException("Required field missing: " + fieldName);
            }
            
            // 타입 검증
            if (fieldValue != null && !fieldSchema.isValidType(fieldValue)) {
                throw new ValidationException(
                    String.format("Invalid type for field %s: expected %s, got %s", 
                        fieldName, fieldSchema.getExpectedType(), fieldValue.getClass()));
            }
        }
    }
}

// 스키마 정의 클래스
@Data
@AllArgsConstructor
public class FieldSchema {
    private Class<?> expectedType;
    private boolean required;
    private String pattern;
    private Object defaultValue;
    
    public boolean isValidType(Object value) {
        return expectedType.isInstance(value);
    }
}
```

## 8. 성능 최적화된 DTO
### 메모리와 성능을 고려한 설계
```java
// Flyweight 패턴을 활용한 경량 DTO
@Value  // 불변 객체
@Builder
public class LightweightPersonaDto {
    
    Long id;
    @Intern  // 문자열 인터닝으로 메모리 절약
    String name;
    PersonaType type;  // enum 사용으로 메모리 절약
    
    // 지연 로딩을 위한 Supplier
    @JsonIgnore
    Supplier<String> descriptionSupplier;
    
    @JsonProperty("description")
    public String getDescription() {
        return descriptionSupplier != null ? descriptionSupplier.get() : null;
    }
    
    // 캐시된 계산 결과
    @JsonIgnore
    private transient String cachedDisplayName;
    
    public String getDisplayName() {
        if (cachedDisplayName == null) {
            cachedDisplayName = computeDisplayName();
        }
        return cachedDisplayName;
    }
    
    private String computeDisplayName() {
        return String.format("%s (%s)", name, type.getDisplayName());
    }
    
    // 팩토리 메서드로 객체 풀 활용
    private static final Map<String, LightweightPersonaDto> CACHE = new ConcurrentHashMap<>();
    
    public static LightweightPersonaDto cached(Long id, String name, PersonaType type) {
        String key = id + ":" + name + ":" + type;
        return CACHE.computeIfAbsent(key, k -> 
            LightweightPersonaDto.builder()
                .id(id)
                .name(name)
                .type(type)
                .build());
    }
}
```

## 9. 실제 사용 예시
### 서비스에서의 DTO 활용
```java
@Service
@RequiredArgsConstructor
@Transactional
public class PersonaService {
    
    private final PersonaRepository personaRepository;
    private final PersonaMapper personaMapper;
    
    public PersonaInfoDto createPersona(PersonaInfoDto dto) {
        // DTO 검증 (이미 컨트롤러에서 @Valid로 처리됨)
        
        // DTO → Entity 변환
        PersonaInfo entity = personaMapper.toEntity(dto);
        entity.setCreateTime(LocalDateTime.now());
        entity.setUpdateTime(LocalDateTime.now());
        
        // 비즈니스 로직 처리
        entity.generateSlug();
        entity.setStatus(PersonaStatus.ACTIVE);
        
        // 저장
        PersonaInfo savedEntity = personaRepository.save(entity);
        
        // Entity → DTO 변환하여 반환
        return personaMapper.toDto(savedEntity);
    }
    
    @Transactional(readOnly = true)
    public Page<PersonaInfoDto> getPersonas(PersonaSearchCriteria criteria, Pageable pageable) {
        // Entity Page 조회
        Page<PersonaInfo> entityPage = personaRepository.findByCriteria(criteria, pageable);
        
        // Entity Page → DTO Page 변환
        return entityPage.map(personaMapper::toDto);
    }
    
    public PersonaInfoDto updatePersona(Long id, PersonaInfoDto dto) {
        PersonaInfo existingEntity = personaRepository.findById(id)
            .orElseThrow(() -> new PersonaNotFoundException(id));
        
        // 부분 업데이트 (null 값은 기존 값 유지)
        personaMapper.updateEntityFromDto(dto, existingEntity);
        existingEntity.setUpdateTime(LocalDateTime.now());
        
        PersonaInfo updatedEntity = personaRepository.save(existingEntity);
        return personaMapper.toDto(updatedEntity);
    }
}
```

## Python Pydantic vs Java Spring Boot 비교표

| 기능 | Python Pydantic | Java Spring Boot |
|------|-----------------|------------------|
| 기본 클래스 | `BaseModel` | `Record` 또는 `@Data` |
| 기본값 설정 | `field: str = "default"` | `@Builder.Default` 또는 생성자 |
| 검증 | `@validator` | `@Valid`, JSR-303 어노테이션 |
| 직렬화 | `@field_serializer` | `@JsonSerialize`, `@JsonFormat` |
| 역직렬화 | `@field_validator` | `@JsonDeserialize` |
| 딕셔너리 변환 | `.dict()` | `.toMap()` 또는 MapStruct |
| 속성 매핑 | `from_attributes=True` | MapStruct 또는 팩토리 메서드 |
| 타입 힌트 | `typing.Optional` | `@Nullable` 또는 Optional |
| 중첩 객체 | 자동 지원 | `@Valid`로 명시적 검증 |

## 관련 패턴
- [[Enterprise Design Patterns]]
- [[API Design Patterns]]
- [[Exception Handling Patterns]]
- [[Spring Bean Management]]
- [[Java 21]]

#dto #vo #data-transfer #validation #serialization

# Asset Manager Enterprise Coding Standards

## 1. Overview

This document establishes coding standards for the Asset Manager enterprise application, based on proven patterns observed in the existing codebase. These standards ensure consistency, maintainability, and scalability across web and backend development.

**Version**: 1.0  
**Target Audience**: Development teams working on Spring Boot enterprise applications  
**Scope**: Web controllers, backend services, data access, configuration, and testing

---

## 2. Project Structure & Package Organization

### 2.1 Standard Package Structure

Follow the existing Asset Manager package hierarchy:

```
com.microsoft.migration.assets/
├── AssetsManagerApplication.java           // Application entry point
├── config/                                 // Configuration classes
│   ├── AwsS3Config.java                   // External service configurations
│   ├── RabbitConfig.java                  // Message queue configurations
│   └── ValidationConfig.java              // Validation framework setup
├── controller/                            // Web layer controllers
│   ├── HomeController.java                // Navigation and routing
│   └── S3Controller.java                  // REST/MVC endpoints
├── service/                              // Business logic layer
│   ├── StorageService.java               // Service interfaces
│   ├── AwsS3Service.java                 // Service implementations
│   └── BackupMessageProcessor.java       // Message consumers
├── model/                               // Domain models and DTOs
│   ├── ImageMetadata.java               // JPA entities
│   ├── S3StorageItem.java               // View models/DTOs
│   └── ImageProcessingMessage.java      // Message payloads
└── repository/                         // Data access layer
    └── ImageMetadataRepository.java    // JPA repositories
```

### 2.2 Package Naming Conventions

**REQUIRED**:
- Use reverse domain notation: `com.company.project.module`
- Layer-based organization: `controller`, `service`, `model`, `repository`, `config`
- Singular nouns for package names: `controller` not `controllers`

**Example from codebase**:
```java
package com.microsoft.migration.assets.controller;  ✓ Correct
package com.microsoft.migration.assets.controllers; ✗ Incorrect
```

### 2.3 File Naming Standards

**Controllers**: `{Domain}Controller.java` (e.g., `S3Controller`, `CustomerController`)  
**Services**: `{Domain}Service.java` or `{Domain}ServiceImpl.java`  
**Entities**: `{Domain}.java` (e.g., `ImageMetadata`, `Customer`)  
**DTOs**: `{Domain}{Purpose}.java` (e.g., `S3StorageItem`, `CustomerCreateRequest`)  
**Repositories**: `{Domain}Repository.java`  
**Configuration**: `{Purpose}Config.java` (e.g., `AwsS3Config`, `RabbitConfig`)

---

## 3. Spring Boot Application Architecture

### 3.1 Application Entry Point

**Standard**: Use single main class with minimal configuration

```java
// REQUIRED: Based on AssetsManagerApplication.java
@SpringBootApplication
@EnableAzureMessaging  // Add framework-specific annotations as needed
public class AssetsManagerApplication {
    
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(AssetsManagerApplication.class);
        application.addListeners(new ApplicationPidFileWriter()); // For containerization
        application.run(args);
    }
}
```

**REQUIRED**:
- Single `@SpringBootApplication` annotation
- Component scanning covers entire application package
- Add infrastructure listeners (PID writer, health checks)
- No business logic in main class

### 3.2 Dependency Injection Standards

**REQUIRED**: Use constructor injection with Lombok

```java
// PREFERRED: Based on S3Controller pattern
@Controller
@RequiredArgsConstructor  // Lombok generates constructor
public class S3Controller {
    
    private final StorageService storageService;  // Final fields
    
    // No explicit constructor needed - Lombok generates it
}
```

**FORBIDDEN**:
```java
@Autowired
private StorageService storageService;  // ✗ Field injection
```

**REQUIRED**:
- Use `@RequiredArgsConstructor` from Lombok
- All injected dependencies must be `final`
- No `@Autowired` annotations needed with constructor injection

---

## 4. Web Layer Standards

### 4.1 Controller Design Patterns

**Template**: Based on S3Controller and HomeController patterns

```java
@Controller
@RequestMapping("/api/domain")  // RESTful path structure
@RequiredArgsConstructor
@Slf4j                         // Structured logging
public class DomainController {
    
    private final DomainService domainService;
    
    // Web UI endpoints (return template names)
    @GetMapping
    public String listPage(Model model) {
        List<Domain> items = domainService.findAll();
        model.addAttribute("items", items);
        return "domain/list";  // template path
    }
    
    @GetMapping("/upload")
    public String uploadForm() {
        return "domain/upload";
    }
    
    @PostMapping("/upload")
    public String upload(@RequestParam("file") MultipartFile file, 
                        RedirectAttributes redirectAttributes) {
        try {
            domainService.process(file);
            redirectAttributes.addFlashAttribute("success", "Upload successful");
            return "redirect:/api/domain";
        } catch (Exception e) {
            log.error("Upload failed: {}", e.getMessage());
            redirectAttributes.addFlashAttribute("error", "Upload failed: " + e.getMessage());
            return "redirect:/api/domain/upload";
        }
    }
    
    // REST API endpoints (return ResponseEntity)
    @GetMapping(value = "/api", produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseBody
    public ResponseEntity<List<Domain>> listApi() {
        try {
            List<Domain> items = domainService.findAll();
            return ResponseEntity.ok(items);
        } catch (Exception e) {
            log.error("API list failed: {}", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
}
```

**REQUIRED**:
- Separate web UI endpoints from REST API endpoints
- Use `RedirectAttributes` for flash messages in web flows
- Use `ResponseEntity<T>` for REST endpoints
- Use `@ResponseBody` for JSON responses
- Implement proper error handling with try-catch blocks
- Log errors with context but avoid logging sensitive data

### 4.2 HTTP Method Standards

**REQUIRED**: Follow RESTful conventions

- `GET` for read operations (listing, viewing)
- `POST` for create operations and file uploads
- `PUT` for full updates
- `DELETE` for deletion
- `PATCH` for partial updates

**Path Structure**: Based on S3Controller patterns
```java
GET    /domain              // List items (web UI)
GET    /domain/upload       // Upload form (web UI)  
POST   /domain/upload       // Process upload (web UI)
GET    /domain/view/{id}    // View item (web UI)
POST   /domain/delete/{id}  // Delete item (web UI)

GET    /domain/api          // List items (REST API)
POST   /domain/api/batch    // Batch create (REST API)
GET    /domain/api/{id}     // Get item (REST API)
PUT    /domain/api/{id}     // Update item (REST API)
DELETE /domain/api/{id}     // Delete item (REST API)
```

### 4.3 Request/Response Handling

**Validation**: Use Bean Validation

```java
@PostMapping("/api/batch")
@ResponseBody
public ResponseEntity<BatchResponse> createBatch(@Valid @RequestBody BatchRequest request) {
    // Validation handled automatically by @Valid
    // Custom validation in service layer
}
```

**Error Responses**: Return structured error information

```java
// PREFERRED: Business errors in response payload (200 OK with error details)
@PostMapping("/api/batch")
public ResponseEntity<BatchResponse> processBatch(@RequestBody BatchRequest request) {
    try {
        if (request.getItems().size() > MAX_BATCH_SIZE) {
            BatchResponse errorResponse = new BatchResponse(0, 0, 
                List.of(new ValidationError("items", "Batch size exceeds limit")));
            return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE).body(errorResponse);
        }
        
        BatchResponse response = service.processBatch(request);
        return ResponseEntity.ok(response);
        
    } catch (Exception e) {
        log.error("Batch processing failed: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    }
}
```

---

## 5. Service Layer Standards

### 5.1 Service Interface Design

**Template**: Based on StorageService pattern

```java
/**
 * Interface for domain operations following StorageService pattern
 * Provides abstraction for different implementations (cloud, local, etc.)
 */
public interface DomainService {
    
    /**
     * List all domain objects
     * @return list of domain objects
     */
    List<DomainItem> listAll();
    
    /**
     * Process uploaded file
     * @param file uploaded file
     * @throws IOException if processing fails
     */
    void processUpload(MultipartFile file) throws IOException;
    
    /**
     * Get domain object by key
     * @param key unique identifier
     * @return input stream of object data
     * @throws IOException if object not found or access fails
     */
    InputStream getObject(String key) throws IOException;
    
    /**
     * Delete domain object
     * @param key unique identifier
     * @throws IOException if deletion fails
     */
    void deleteObject(String key) throws IOException;
}
```

**REQUIRED**:
- Use interfaces for service abstraction
- Document all public methods with JavaDoc
- Declare checked exceptions in method signatures
- Use domain-specific naming, not generic terms

### 5.2 Service Implementation Standards

**Template**: Based on AwsS3Service pattern

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Profile("!dev")  // Use profiles to control implementation selection
public class CloudDomainServiceImpl implements DomainService {
    
    private final ExternalServiceClient client;
    private final DomainRepository repository;
    private final MessageTemplate messageTemplate;
    
    @Value("${domain.storage.container}")
    private String containerName;
    
    @Override
    @Transactional  // Use for operations that modify data
    public void processUpload(MultipartFile file) throws IOException {
        String key = generateKey(file.getOriginalFilename());
        
        // 1. Upload to external storage
        client.upload(containerName, key, file.getInputStream());
        
        // 2. Save metadata to database
        DomainMetadata metadata = createMetadata(key, file);
        repository.save(metadata);
        
        // 3. Send processing message
        ProcessingMessage message = new ProcessingMessage(key, file.getContentType());
        messageTemplate.send("processing-queue", message);
        
        log.info("Upload completed: key={}, size={}", key, file.getSize());
    }
    
    private String generateKey(String filename) {
        return UUID.randomUUID().toString() + "_" + filename;
    }
    
    private DomainMetadata createMetadata(String key, MultipartFile file) {
        DomainMetadata metadata = new DomainMetadata();
        metadata.setKey(key);
        metadata.setFilename(file.getOriginalFilename());
        metadata.setSize(file.getSize());
        metadata.setContentType(file.getContentType());
        return metadata;
    }
}
```

**REQUIRED**:
- Use `@Service` annotation
- Use `@Profile` for environment-specific implementations
- Use `@Transactional` for database-modifying operations
- Use `@Value` for configuration injection
- Use structured logging with context
- Handle exceptions appropriately
- Break complex methods into smaller private methods

### 5.3 Batch Processing Standards

**Template**: Based on customer batch service patterns

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BatchProcessingServiceImpl implements BatchProcessingService {
    
    private final DomainRepository repository;
    private final BatchProperties batchProperties;
    private final Validator validator;
    
    @Override
    @Transactional
    public BatchResponse processBatch(BatchRequest request) {
        List<BatchItem> items = request.getItems();
        List<BatchResult> results = new ArrayList<>();
        List<Domain> toSave = new ArrayList<>();
        
        int processed = 0;
        int failed = 0;
        
        log.info("Processing batch of {} items", items.size());
        
        // Process each item individually
        for (int i = 0; i < items.size(); i++) {
            BatchItem item = items.get(i);
            int rowNumber = i + 1;
            
            List<ValidationError> errors = validateItem(item);
            
            if (errors.isEmpty()) {
                Domain domain = mapToEntity(item);
                toSave.add(domain);
                results.add(new BatchResult(rowNumber, BatchStatus.SUCCESS, null, null));
                processed++;
            } else {
                results.add(new BatchResult(rowNumber, BatchStatus.FAILED, null, errors));
                failed++;
                log.debug("Row {} validation failed: {} errors", rowNumber, errors.size());
            }
        }
        
        // Batch save valid items
        List<Domain> saved = saveInChunks(toSave);
        updateResultsWithIds(results, saved);
        
        log.info("Batch processing completed: {} processed, {} failed", processed, failed);
        return new BatchResponse(items.size(), processed, failed, results);
    }
    
    private List<Domain> saveInChunks(List<Domain> items) {
        List<Domain> allSaved = new ArrayList<>();
        int chunkSize = batchProperties.getChunkSize();
        
        for (int i = 0; i < items.size(); i += chunkSize) {
            int end = Math.min(i + chunkSize, items.size());
            List<Domain> chunk = items.subList(i, end);
            
            try {
                List<Domain> saved = repository.saveAll(chunk);
                allSaved.addAll(saved);
                log.debug("Saved chunk of {} items", chunk.size());
            } catch (Exception e) {
                log.error("Failed to save chunk: {}", e.getMessage());
                throw new BatchProcessingException("Failed to save items", e);
            }
        }
        
        return allSaved;
    }
}
```

**REQUIRED**:
- Process items individually to isolate failures
- Use chunked saving for large batches
- Log progress and summary statistics
- Validate each item separately
- Return detailed results with row-level success/failure information

---

## 6. Data Access Layer Standards

### 6.1 Repository Design

**Template**: Based on ImageMetadataRepository pattern

```java
@Repository
public interface DomainRepository extends JpaRepository<Domain, String> {
    
    /**
     * Check if domain object exists by business key
     * @param businessKey unique business identifier
     * @return true if exists, false otherwise
     */
    boolean existsByBusinessKey(String businessKey);
    
    /**
     * Find domain object by business key
     * @param businessKey unique business identifier
     * @return optional domain object
     */
    Optional<Domain> findByBusinessKey(String businessKey);
    
    /**
     * Find all domain objects created within date range
     * @param startDate start of date range (inclusive)
     * @param endDate end of date range (inclusive)
     * @return list of domain objects
     */
    List<Domain> findByCreatedAtBetween(LocalDateTime startDate, LocalDateTime endDate);
    
    /**
     * INHERITED from JpaRepository:
     * - List<Domain> findAll()
     * - Page<Domain> findAll(Pageable pageable)
     * - Optional<Domain> findById(String id)
     * - Domain save(Domain entity)
     * - List<Domain> saveAll(Iterable<Domain> entities)
     * - void delete(Domain entity)
     * - long count()
     */
}
```

**REQUIRED**:
- Extend `JpaRepository<Entity, IdType>`
- Use `@Repository` annotation
- Follow Spring Data naming conventions for query methods
- Document custom query methods
- Use `boolean exists*` methods for existence checks
- Use `Optional<T> find*` methods for single-result queries

### 6.2 Entity Design Standards

**Template**: Based on ImageMetadata pattern

```java
@Entity
@Table(name = "domain_objects")  // Explicit table mapping
@Data                           // Lombok for getters/setters
@NoArgsConstructor             // Required for JPA
public class Domain {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)  // Use UUIDs for distributed systems
    private String id;
    
    @Column(name = "business_key", nullable = false, length = 255, unique = true)
    private String businessKey;
    
    @Column(name = "name", nullable = false, length = 100)
    private String name;
    
    @Column(name = "description", length = 500)
    private String description;
    
    @Column(name = "status")
    @Enumerated(EnumType.STRING)  // Use STRING for enum storage
    private DomainStatus status;
    
    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    // JPA lifecycle callbacks for timestamp management
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

**REQUIRED**:
- Use `@Entity` and `@Table` annotations
- Use Lombok `@Data` and `@NoArgsConstructor`
- Use UUID generation strategy for primary keys
- Explicit column mapping with `@Column`
- Use `EnumType.STRING` for enum persistence
- Implement lifecycle callbacks for timestamp management
- Use `LocalDateTime` for temporal fields

### 6.3 Database Schema Standards

**Naming Conventions**:
```sql
-- Table names: lowercase with underscores
CREATE TABLE domain_objects (
    id VARCHAR(255) PRIMARY KEY,              -- UUID primary keys
    business_key VARCHAR(255) NOT NULL,       -- Business identifiers
    name VARCHAR(100) NOT NULL,               -- Required text fields
    description VARCHAR(500),                 -- Optional text fields  
    status VARCHAR(50),                       -- Enum storage as text
    created_at TIMESTAMP NOT NULL,            -- Audit timestamps
    updated_at TIMESTAMP NOT NULL,
    
    CONSTRAINT uk_domain_business_key UNIQUE (business_key)
);

-- Index naming: idx_{table}_{columns}
CREATE INDEX idx_domain_business_key ON domain_objects(business_key);
CREATE INDEX idx_domain_created_at ON domain_objects(created_at);
CREATE INDEX idx_domain_status ON domain_objects(status);
```

**REQUIRED**:
- Snake_case for table and column names
- VARCHAR with explicit lengths for text fields
- NOT NULL constraints for required fields
- UNIQUE constraints for business keys
- Audit timestamp fields on all entities
- Performance indexes on frequently queried columns

---

## 7. Configuration Management

### 7.1 External Service Configuration

**Template**: Based on AwsS3Config and RabbitConfig patterns

```java
@Configuration
@Slf4j
public class ExternalServiceConfig {
    
    @Value("${external.service.endpoint}")
    private String serviceEndpoint;
    
    @Value("${external.service.timeout:30}")  // Default value
    private int timeoutSeconds;
    
    @Bean
    @ConditionalOnProperty(value = "external.service.enabled", havingValue = "true")
    public ExternalServiceClient externalServiceClient(TokenCredential credential) {
        log.info("Initializing external service client: endpoint={}", serviceEndpoint);
        
        return new ExternalServiceClientBuilder()
                .endpoint(serviceEndpoint)
                .credential(credential)
                .timeout(Duration.ofSeconds(timeoutSeconds))
                .buildClient();
    }
    
    @Bean
    @ConditionalOnMissingBean
    public TokenCredential tokenCredential() {
        log.info("Configuring default Azure credential chain");
        return new DefaultAzureCredentialBuilder().build();
    }
}
```

**REQUIRED**:
- Use `@Configuration` for configuration classes
- Use `@Value` with default values where appropriate
- Use conditional annotations (`@ConditionalOnProperty`, `@ConditionalOnMissingBean`)
- Log configuration initialization
- Use builder patterns for complex client configuration

### 7.2 Application Properties Structure

**Template**: Based on existing application.properties

```properties
# Application Information
spring.application.name=asset-manager
app.version=@project.version@

# Server Configuration
server.port=8080
server.servlet.context-path=/

# Database Configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/asset_manager
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql=${JPA_SHOW_SQL:false}

# Azure Services Configuration
azure.storage.account-name=${AZURE_STORAGE_ACCOUNT_NAME}
azure.storage.blob.container-name=${AZURE_STORAGE_CONTAINER:images}
spring.cloud.azure.credential.managed-identity-enabled=true
spring.cloud.azure.credential.client-id=${AZURE_CLIENT_ID}
spring.cloud.azure.servicebus.namespace=${AZURE_SERVICEBUS_NAMESPACE}

# File Upload Configuration
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB

# Business Logic Configuration
domain.batch.max-size=${BATCH_MAX_SIZE:1000}
domain.batch.chunk-size=${BATCH_CHUNK_SIZE:200}
domain.batch.timeout-seconds=${BATCH_TIMEOUT:300}

# Logging Configuration
logging.level.com.microsoft.migration.assets=INFO
logging.level.org.springframework.web=DEBUG
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
```

**REQUIRED**:
- Group related properties with comments
- Use environment variable overrides: `${ENV_VAR:default}`
- Use snake_case for custom property names
- Include default values for non-sensitive configuration
- Separate business configuration from infrastructure configuration

---

## 8. Error Handling & Logging

### 8.1 Exception Handling Standards

**Service Layer**:
```java
@Service
@Slf4j
public class DomainServiceImpl implements DomainService {
    
    @Override
    public void processItem(String itemId) throws ProcessingException {
        try {
            // Business logic here
            log.info("Processing item: {}", itemId);
            
        } catch (ExternalServiceException e) {
            log.error("External service failed for item {}: {}", itemId, e.getMessage());
            throw new ProcessingException("Failed to process item", e);
        } catch (ValidationException e) {
            log.warn("Validation failed for item {}: {}", itemId, e.getMessage());
            throw e;  // Re-throw validation exceptions
        } catch (Exception e) {
            log.error("Unexpected error processing item {}: {}", itemId, e.getMessage(), e);
            throw new ProcessingException("Unexpected processing error", e);
        }
    }
}
```

**Controller Layer**:
```java
@Controller
@Slf4j
public class DomainController {
    
    @PostMapping("/process/{id}")
    public String processItem(@PathVariable String id, RedirectAttributes redirectAttributes) {
        try {
            domainService.processItem(id);
            redirectAttributes.addFlashAttribute("success", "Item processed successfully");
            return "redirect:/domain";
            
        } catch (ValidationException e) {
            log.debug("Validation error for item {}: {}", id, e.getMessage());
            redirectAttributes.addFlashAttribute("error", "Validation error: " + e.getMessage());
            return "redirect:/domain";
            
        } catch (ProcessingException e) {
            log.error("Processing error for item {}: {}", id, e.getMessage());
            redirectAttributes.addFlashAttribute("error", "Processing failed. Please try again.");
            return "redirect:/domain";
            
        } catch (Exception e) {
            log.error("Unexpected error processing item {}: {}", id, e.getMessage(), e);
            redirectAttributes.addFlashAttribute("error", "An unexpected error occurred");
            return "redirect:/domain";
        }
    }
}
```

**REQUIRED**:
- Catch specific exceptions before generic Exception
- Log at appropriate levels (ERROR for failures, WARN for validation, DEBUG for recoverable issues)
- Include context in log messages (item IDs, operation details)
- Don't expose technical details to users
- Re-throw exceptions appropriately in service layer

### 8.2 Logging Standards

**Structure**: Based on existing patterns with @Slf4j

```java
@Service
@Slf4j  // Use Lombok's @Slf4j annotation
public class DomainService {
    
    public void processData(String dataId, int itemCount) {
        // INFO: Business operations and milestones
        log.info("Starting data processing: dataId={}, itemCount={}", dataId, itemCount);
        
        try {
            // DEBUG: Detailed operational information
            log.debug("Validating data structure for dataId={}", dataId);
            
            for (int i = 0; i < itemCount; i++) {
                // Don't log in tight loops - use batch logging
            }
            
            // INFO: Successful completion
            log.info("Data processing completed: dataId={}, processed={}", dataId, itemCount);
            
        } catch (ValidationException e) {
            // WARN: Expected business exceptions
            log.warn("Data validation failed: dataId={}, error={}", dataId, e.getMessage());
            
        } catch (Exception e) {
            // ERROR: Unexpected failures requiring investigation
            log.error("Data processing failed: dataId={}, error={}", dataId, e.getMessage(), e);
            throw new ProcessingException("Processing failed", e);
        }
    }
}
```

**REQUIRED**:
- Use `@Slf4j` annotation from Lombok
- Use parameterized logging: `log.info("Message: {}", variable)`
- Include context: IDs, counts, operation details
- Choose appropriate log levels:
  - `ERROR`: System failures requiring immediate attention
  - `WARN`: Expected business exceptions or recoverable issues
  - `INFO`: Business milestones and operations
  - `DEBUG`: Detailed operational information
- Never log sensitive data (passwords, PII)
- Include stack traces only for ERROR level

---

## 9. Model Design Standards

### 9.1 DTO Design Patterns

**Input DTOs**: Based on CustomerCreateRequest pattern

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainCreateRequest {
    
    @NotBlank(message = "Name is required")
    @Size(min = 1, max = 100, message = "Name must be between 1 and 100 characters")
    private String name;
    
    @NotNull(message = "Type is required")
    private DomainType type;
    
    @Size(max = 500, message = "Description must not exceed 500 characters")
    private String description;
    
    @Valid
    @NotNull(message = "Configuration is required")
    private DomainConfiguration configuration;
}
```

**Response DTOs**: Based on S3StorageItem pattern

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainResponse {
    private String id;
    private String name;
    private DomainType type;
    private String description;
    private DomainStatus status;
    private LocalDateTime createdAt;
    private String createdBy;
    
    // Factory method for entity conversion
    public static DomainResponse from(Domain domain) {
        return new DomainResponse(
            domain.getId(),
            domain.getName(),
            domain.getType(),
            domain.getDescription(),
            domain.getStatus(),
            domain.getCreatedAt(),
            domain.getCreatedBy()
        );
    }
}
```

**Batch DTOs**: Based on customer batch patterns

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainBatchRequest {
    @NotNull(message = "Items list cannot be null")
    @Size(min = 1, max = 1000, message = "Batch size must be between 1 and 1000")
    @Valid
    private List<DomainCreateRequest> items;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainBatchResponse {
    private int total;
    private int processed;
    private int failed;
    private List<DomainBatchResult> results;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class DomainBatchResult {
    private int rowNumber;
    private BatchResultStatus status;
    private String domainId;         // null if failed
    private List<ValidationError> errors;  // null if successful
}

enum BatchResultStatus {
    PROCESSED, FAILED
}
```

**REQUIRED**:
- Use Lombok annotations: `@Data`, `@NoArgsConstructor`, `@AllArgsConstructor`
- Use Bean Validation annotations: `@NotNull`, `@NotBlank`, `@Size`, `@Valid`
- Provide factory methods for entity conversion
- Use meaningful field names and validation messages
- Separate request, response, and internal DTOs

### 9.2 Enum Design Standards

```java
public enum DomainStatus {
    CREATED("Created"),
    PROCESSING("Processing"),
    COMPLETED("Completed"),
    FAILED("Failed"),
    ARCHIVED("Archived");
    
    private final String displayName;
    
    DomainStatus(String displayName) {
        this.displayName = displayName;
    }
    
    public String getDisplayName() {
        return displayName;
    }
    
    // Business logic methods
    public boolean isActive() {
        return this == CREATED || this == PROCESSING;
    }
    
    public boolean canTransitionTo(DomainStatus newStatus) {
        return switch (this) {
            case CREATED -> newStatus == PROCESSING || newStatus == ARCHIVED;
            case PROCESSING -> newStatus == COMPLETED || newStatus == FAILED;
            case COMPLETED, FAILED -> newStatus == ARCHIVED;
            case ARCHIVED -> false;
        };
    }
}
```

**REQUIRED**:
- Provide display names for user-facing text
- Include business logic methods for state transitions
- Use descriptive enum names
- Consider backward compatibility when modifying enums

---

## 10. Testing Standards

### 10.1 Unit Test Structure

**Service Layer Tests**:
```java
@ExtendWith(MockitoExtension.class)
class DomainServiceTest {
    
    @Mock
    private DomainRepository repository;
    
    @Mock
    private ExternalServiceClient externalClient;
    
    @Mock
    private Validator validator;
    
    @InjectMocks
    private DomainServiceImpl domainService;
    
    @Test
    void processItem_ValidItem_ShouldSucceed() {
        // Given
        String itemId = "test-item-id";
        Domain existingDomain = createTestDomain(itemId);
        when(repository.findById(itemId)).thenReturn(Optional.of(existingDomain));
        when(externalClient.process(any())).thenReturn(createSuccessResponse());
        
        // When
        ProcessingResult result = domainService.processItem(itemId);
        
        // Then
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getItemId()).isEqualTo(itemId);
        verify(repository).save(any(Domain.class));
        verify(externalClient).process(any());
    }
    
    @Test
    void processItem_ItemNotFound_ShouldThrowException() {
        // Given
        String itemId = "nonexistent-item";
        when(repository.findById(itemId)).thenReturn(Optional.empty());
        
        // When & Then
        assertThatThrownBy(() -> domainService.processItem(itemId))
            .isInstanceOf(ItemNotFoundException.class)
            .hasMessage("Item not found: " + itemId);
        
        verify(externalClient, never()).process(any());
    }
    
    private Domain createTestDomain(String id) {
        Domain domain = new Domain();
        domain.setId(id);
        domain.setName("Test Domain");
        domain.setStatus(DomainStatus.CREATED);
        return domain;
    }
}
```

**Controller Integration Tests**:
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class DomainControllerIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private DomainRepository repository;
    
    @Test
    void createDomain_ValidRequest_ShouldReturnCreated() {
        // Given
        DomainCreateRequest request = new DomainCreateRequest();
        request.setName("Test Domain");
        request.setType(DomainType.STANDARD);
        
        // When
        ResponseEntity<DomainResponse> response = restTemplate.postForEntity(
            "/domain/api", request, DomainResponse.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getName()).isEqualTo("Test Domain");
        
        // Verify persistence
        Optional<Domain> saved = repository.findById(response.getBody().getId());
        assertThat(saved).isPresent();
    }
}
```

**REQUIRED**:
- Use JUnit 5 and MockitoExtension
- Follow Given-When-Then structure
- Use meaningful test method names: `methodName_condition_expectedResult`
- Verify interactions with mocks
- Use AssertJ for fluent assertions
- Test both happy path and error conditions
- Use test data builders for complex objects

### 10.2 Test Configuration

```java
@TestConfiguration
public class TestConfig {
    
    @Bean
    @Primary
    public ExternalServiceClient mockExternalServiceClient() {
        return Mockito.mock(ExternalServiceClient.class);
    }
    
    @Bean
    @Primary  
    public MessageTemplate mockMessageTemplate() {
        return Mockito.mock(MessageTemplate.class);
    }
}
```

**REQUIRED**:
- Use `@TestConfiguration` for test-specific beans
- Use `@Primary` to override production beans in tests
- Mock external dependencies
- Use in-memory databases for integration tests

---

## 11. Security Standards

### 11.1 Input Validation

**Bean Validation**:
```java
@RestController
@Validated  // Enable method-level validation
public class DomainController {
    
    @PostMapping("/api/domain")
    public ResponseEntity<DomainResponse> createDomain(
            @Valid @RequestBody DomainCreateRequest request) {
        // @Valid triggers automatic validation
        DomainResponse response = domainService.createDomain(request);
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/api/domain/{id}")
    public ResponseEntity<DomainResponse> getDomain(
            @PathVariable @Pattern(regexp = "[a-fA-F0-9-]{36}", 
                          message = "Invalid UUID format") String id) {
        // Method-level validation for path variables
        DomainResponse response = domainService.getDomain(id);
        return ResponseEntity.ok(response);
    }
}
```

**Custom Validation**:
```java
@Service
public class DomainValidationService {
    
    public List<ValidationError> validateDomainRequest(DomainCreateRequest request) {
        List<ValidationError> errors = new ArrayList<>();
        
        // Business rule validation
        if (request.getName() != null && isReservedName(request.getName())) {
            errors.add(new ValidationError("name", "Name is reserved and cannot be used"));
        }
        
        // Cross-field validation
        if (request.getType() == DomainType.SPECIAL && 
            request.getConfiguration() == null) {
            errors.add(new ValidationError("configuration", 
                "Configuration required for special type"));
        }
        
        return errors;
    }
    
    private boolean isReservedName(String name) {
        List<String> reservedNames = Arrays.asList("admin", "system", "root");
        return reservedNames.contains(name.toLowerCase());
    }
}
```

**REQUIRED**:
- Use `@Valid` for automatic bean validation
- Use `@Validated` on controllers for method-level validation
- Implement custom validation for business rules
- Validate all user inputs including path variables and query parameters
- Return structured validation errors

### 11.2 Data Privacy

**Logging Guidelines**:
```java
@Service
@Slf4j
public class DomainService {
    
    public void processUserData(UserDataRequest request) {
        // ✓ Safe logging - no PII
        log.info("Processing user data: userId={}, dataType={}", 
            request.getUserId(), request.getDataType());
        
        // ✗ Unsafe logging - contains PII
        // log.info("Processing data: {}", request); // DON'T LOG ENTIRE REQUEST
        
        // ✓ Safe error logging - generic message
        try {
            // process data
        } catch (Exception e) {
            log.error("Data processing failed for user: userId={}, error={}", 
                request.getUserId(), e.getMessage());
            // Don't log the full exception if it might contain PII
        }
    }
}
```

**Data Sanitization**:
```java
@Component
public class DataSanitizer {
    
    public String sanitizeForLogging(String input) {
        if (input == null || input.length() <= 4) {
            return "***";
        }
        return input.substring(0, 2) + "***" + input.substring(input.length() - 2);
    }
    
    public String sanitizeEmail(String email) {
        if (email == null || !email.contains("@")) {
            return "***";
        }
        String[] parts = email.split("@");
        return sanitizeForLogging(parts[0]) + "@" + parts[1];
    }
}
```

**REQUIRED**:
- Never log sensitive data (passwords, emails, phone numbers, etc.)
- Sanitize data before logging when context is needed
- Use generic error messages for user-facing errors
- Implement data masking for audit trails

---

## 12. Performance Standards

### 12.1 Database Query Optimization

**Repository Methods**:
```java
@Repository
public interface DomainRepository extends JpaRepository<Domain, String> {
    
    // ✓ Use specific queries instead of findAll()
    @Query("SELECT d FROM Domain d WHERE d.status = :status")
    List<Domain> findByStatus(@Param("status") DomainStatus status);
    
    // ✓ Use projection for read-only operations
    @Query("SELECT new com.example.DomainSummary(d.id, d.name, d.status) " +
           "FROM Domain d WHERE d.createdAt >= :since")
    List<DomainSummary> findSummariesSince(@Param("since") LocalDateTime since);
    
    // ✓ Use pagination for large result sets
    Page<Domain> findByStatusOrderByCreatedAtDesc(DomainStatus status, Pageable pageable);
    
    // ✓ Use exists queries for boolean checks
    boolean existsByNameAndStatus(String name, DomainStatus status);
}
```

**Service Layer Optimization**:
```java
@Service
@Transactional(readOnly = true)  // Default to read-only
public class DomainService {
    
    @Transactional  // Override for write operations
    public void updateDomain(String id, DomainUpdateRequest request) {
        Domain domain = repository.findById(id)
            .orElseThrow(() -> new DomainNotFoundException(id));
        
        // Update only changed fields
        if (request.getName() != null) {
            domain.setName(request.getName());
        }
        if (request.getStatus() != null) {
            domain.setStatus(request.getStatus());
        }
        
        repository.save(domain);
    }
    
    public Page<DomainSummary> getDomainSummaries(int page, int size) {
        // Use pagination to limit result size
        Pageable pageable = PageRequest.of(page, Math.min(size, 100));
        return repository.findSummariesByStatus(DomainStatus.ACTIVE, pageable);
    }
}
```

**REQUIRED**:
- Use `@Transactional(readOnly = true)` as default on service classes
- Override with `@Transactional` for write operations
- Use pagination for list operations
- Use projections for read-only data
- Use `exists*` methods instead of `find*` for boolean checks
- Limit query result sizes

### 12.2 Caching Strategy

**Service Layer Caching**:
```java
@Service
@CacheConfig(cacheNames = "domain-cache")
public class DomainService {
    
    @Cacheable(key = "#id")
    public DomainResponse getDomain(String id) {
        Domain domain = repository.findById(id)
            .orElseThrow(() -> new DomainNotFoundException(id));
        return DomainResponse.from(domain);
    }
    
    @CacheEvict(key = "#id")
    public void updateDomain(String id, DomainUpdateRequest request) {
        // Update logic
    }
    
    @CacheEvict(allEntries = true)
    public void clearDomainCache() {
        // Clear entire cache when needed
    }
}
```

**Configuration**:
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("domain-cache");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats());
        return cacheManager;
    }
}
```

**REQUIRED**:
- Use method-level caching for expensive read operations
- Cache read-only data that doesn't change frequently
- Use appropriate cache eviction strategies
- Configure cache size limits and TTL
- Monitor cache hit rates

---

## 13. Documentation Standards

### 13.1 API Documentation

**Controller Documentation**:
```java
@RestController
@RequestMapping("/api/domain")
@Tag(name = "Domain Management", description = "Operations for managing domain objects")
public class DomainController {
    
    @PostMapping
    @Operation(summary = "Create domain object", 
               description = "Creates a new domain object with the provided configuration")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Domain created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid request data"),
        @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<DomainResponse> createDomain(
            @RequestBody 
            @Schema(description = "Domain creation request")
            DomainCreateRequest request) {
        
        DomainResponse response = domainService.createDomain(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

**Service Documentation**:
```java
/**
 * Service for managing domain objects and their lifecycle.
 * 
 * This service provides operations for creating, updating, retrieving, and
 * deleting domain objects. It handles validation, persistence, and business
 * rule enforcement.
 * 
 * @author Development Team
 * @since 1.0
 */
@Service
public class DomainService {
    
    /**
     * Creates a new domain object.
     * 
     * @param request the domain creation request containing name, type, and configuration
     * @return the created domain object with generated ID and timestamps
     * @throws ValidationException if the request data is invalid
     * @throws DomainExistsException if a domain with the same name already exists
     * @throws ProcessingException if domain creation fails due to system error
     */
    public DomainResponse createDomain(DomainCreateRequest request) {
        // Implementation
    }
}
```

**REQUIRED**:
- Use OpenAPI 3 annotations for REST endpoints
- Document all public service methods with JavaDoc
- Include parameter descriptions and exception information
- Provide example requests and responses
- Keep documentation up-to-date with code changes

### 13.2 README Standards

**Project README Structure**:
```markdown
# Asset Manager

Brief description of the application and its purpose.

## Prerequisites

- Java 21+
- PostgreSQL 14+
- Maven 3.9+
- Azure account (for cloud deployment)

## Local Development Setup

1. Clone repository
2. Configure database
3. Set environment variables
4. Run application

## Configuration

### Required Environment Variables
- `DB_USERNAME` - Database username
- `AZURE_STORAGE_ACCOUNT_NAME` - Azure storage account

### Optional Configuration
- `BATCH_MAX_SIZE` - Maximum batch size (default: 1000)

## API Documentation

- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI spec: http://localhost:8080/api-docs

## Testing

Run unit tests: `mvn test`
Run integration tests: `mvn verify`

## Deployment

See [deployment guide](docs/deployment.md)

## Architecture

See [architecture documentation](docs/architecture.md)
```

**REQUIRED**:
- Include prerequisites and setup instructions
- Document all configuration options
- Provide links to additional documentation
- Include testing and deployment instructions
- Keep README concise but complete

---

## 14. Code Quality & Review Standards

### 14.1 Code Quality Metrics

**Complexity Guidelines**:
- Methods should not exceed 20 lines
- Classes should not exceed 300 lines
- Cyclomatic complexity should be < 10
- Test coverage should be > 80%

**Code Review Checklist**:

**Functionality**:
- [ ] Code meets requirements
- [ ] Error handling is appropriate
- [ ] Edge cases are handled
- [ ] Business logic is correct

**Design**:
- [ ] Follows established patterns
- [ ] Proper separation of concerns
- [ ] No code duplication
- [ ] Appropriate abstraction levels

**Security**:
- [ ] Input validation present
- [ ] No sensitive data in logs
- [ ] SQL injection prevention
- [ ] Authentication/authorization appropriate

**Performance**:
- [ ] Database queries optimized
- [ ] Appropriate caching
- [ ] No memory leaks
- [ ] Pagination for large datasets

**Testing**:
- [ ] Unit tests present and meaningful
- [ ] Integration tests for complex flows
- [ ] Test coverage adequate
- [ ] Tests are maintainable

### 14.2 Static Analysis Tools

**Required Tools**:
- **SpotBugs**: Static analysis for bug detection
- **Checkstyle**: Code style enforcement
- **PMD**: Code quality analysis
- **SonarQube**: Comprehensive code quality platform

**Maven Configuration**:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.github.spotbugs</groupId>
            <artifactId>spotbugs-maven-plugin</artifactId>
            <version>4.7.3.0</version>
        </plugin>
        
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>3.3.0</version>
        </plugin>
        
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-pmd-plugin</artifactId>
            <version>3.21.0</version>
        </plugin>
    </plugins>
</build>
```

---

## 15. Conclusion

These coding standards are based on proven patterns from the Asset Manager codebase and enterprise Java development best practices. They should be:

1. **Enforced** through code reviews and automated tools
2. **Updated** as the codebase evolves and new patterns emerge
3. **Taught** to new team members during onboarding
4. **Applied** consistently across all modules and features

**Compliance**: All code must follow these standards before merging to main branch.

**Questions**: Contact the development team for clarification on any standards.

---

*Document Version: 1.0*  
*Last Updated: March 2026*  
*Next Review: June 2026*
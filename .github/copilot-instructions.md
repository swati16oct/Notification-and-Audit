# GitHub Copilot Instructions for TaskBridge (Java Spring Boot)

## Project Context

This document defines standards, architecture patterns, and security rules for TaskBridge development using GitHub Copilot. All AI-assisted code generation must follow these guidelines to ensure consistency, security, and multi-tenant compliance across the Java Spring Boot microservices.

## Technology Stack

- **Language**: Java 17+
- **Framework**: Spring Boot 3.x
- **Web**: Spring MVC (REST APIs)
- **Database**: PostgreSQL 14+ with Spring Data JPA
- **Authentication**: Spring Security 6.x + JWT (jjwt library)
- **Validation**: Jakarta Bean Validation (JSR-380)
- **Testing**: JUnit 5 + Mockito + Spring Boot Test
- **Build**: Maven 3.8+
- **Async**: Spring @Async, @Scheduled
- **ORM**: Hibernate with Spring Data JPA

## Architecture Conventions

### Layered Architecture (Model → Repository → Service → Controller)

All business features must follow strict separation of concerns:

```
HTTP Request (REST API)
    ↓
[Controller/RestController] 
  - Request validation via @Valid
  - HTTP response status codes
  - DTO conversion (Request/Response)
    ↓
[Service @Service]
  - Business logic orchestration
  - Transaction management (@Transactional)
  - Cross-service coordination
    ↓
[Repository extends JpaRepository]
  - Data access abstraction
  - Custom @Query methods
  - Pagination & filtering
    ↓
[Entity @Entity]
  - JPA annotations
  - Database constraints
  - Validation annotations
    ↓
PostgreSQL Database
```

**Rules**:
- Controllers handle HTTP concerns only: status codes, headers, DTO conversion
- Services orchestrate business logic and manage @Transactional boundaries
- Repositories extend Spring Data JPA interfaces; no raw SQL in services/controllers
- Entities are POJOs with JPA annotations; use @Entity, @Table, @Column
- Use DTOs (Data Transfer Objects) for API contracts; never expose entities directly
- No @Repository on entity classes; only on repository interfaces

### Spring Boot Conventions

- Use `@SpringBootApplication` on main application class
- Use `@ComponentScan` to specify base package if needed
- Use `@EnableJpaRepositories` to enable Spring Data JPA
- Use `@EnableTransactionManagement` for explicit transaction management
- Use `@EnableWebMvc` only when customizing web configuration
- Use `application.yml` (not `application.properties`) for configuration
- Use profiles: `dev`, `test`, `prod` (separate `.yml` files)

### Package Structure

```
com.taskbridge
├── config/              # Configuration classes
│   ├── SecurityConfig.java
│   ├── JwtConfig.java
│   └── DatabaseConfig.java
├── projects/            # Project Service Module
│   ├── model/
│   │   ├── Project.java (Entity)
│   │   └── ProjectDTO.java
│   ├── repository/
│   │   └── ProjectRepository.java (extends JpaRepository)
│   ├── service/
│   │   └── ProjectService.java (@Service)
│   └── controller/
│       └── ProjectController.java (@RestController)
├── notifications/       # Notification Service Module
│   ├── model/
│   ├── repository/
│   ├── service/
│   └── controller/
├── audit/               # Audit Service Module
│   ├── model/
│   ├── repository/
│   ├── service/
│   └── controller/
├── security/            # Security & Auth
│   ├── JwtProvider.java
│   ├── JwtAuthenticationFilter.java
│   ├── CustomUserDetailsService.java
│   └── SecurityPrincipal.java
├── middleware/          # Global Handlers & Interceptors
│   ├── GlobalExceptionHandler.java (@ControllerAdvice)
│   ├── ValidationExceptionHandler.java
│   └── RequestLoggingInterceptor.java
├── util/                # Utilities
│   ├── SecurityContextUtil.java
│   └── DateTimeUtil.java
└── TaskBridgeApplication.java (Main)
```

## Multi-Tenant Data Isolation (Critical)

**Every database query must enforce organisation-level isolation.**

### Rules

1. **User Context Required**: Every request must include `organisationId` (extracted from JWT)
   ```java
   // In SecurityPrincipal or Custom Principal
   private UUID organisationId;
   
   // Access in controller
   UUID organisationId = ((SecurityPrincipal) SecurityContextHolder.getContext()
       .getAuthentication().getPrincipal()).getOrganisationId();
   ```

2. **Repository Filtering**: All queries must include `organisationId` WHERE clause
   ```java
   // ✅ Correct
   public interface ProjectRepository extends JpaRepository<Project, UUID> {
       List<Project> findByOrganisationIdAndStatus(
           UUID organisationId, 
           String status
       );
       
       @Query("SELECT p FROM Project p WHERE p.organisationId = :orgId AND p.id = :projectId")
       Optional<Project> findByIdAndOrganisationId(
           @Param("projectId") UUID projectId,
           @Param("orgId") UUID organisationId
       );
   }
   
   // ❌ Wrong - Missing organisation isolation
   List<Project> findAll();  // NEVER use without orgId filter
   Project findById(UUID id); // NEVER use - must include orgId
   ```

3. **Service Layer Enforcement**: Always pass `organisationId` to repository
   ```java
   @Service
   public class ProjectService {
       @Autowired
       private ProjectRepository projectRepository;
       
       public List<Project> getProjectsByOrganisation(UUID organisationId, String status) {
           // ✅ Correct - includes organisationId
           return projectRepository.findByOrganisationIdAndStatus(organisationId, status);
       }
       
       public Project getProjectDetails(UUID projectId, UUID organisationId) {
           return projectRepository.findByIdAndOrganisationId(projectId, organisationId)
               .orElseThrow(() -> new ResourceNotFoundException("Project not found"));
       }
   }
   ```

4. **Controller Enforcement**: Extract organisationId from JWT and pass to service
   ```java
   @RestController
   @RequestMapping("/api/v1/projects")
   public class ProjectController {
       @GetMapping
       public ResponseEntity<List<ProjectDTO>> getProjects() {
           UUID organisationId = SecurityContextUtil.getOrganisationId();
           List<Project> projects = projectService.getProjectsByOrganisation(organisationId, "ACTIVE");
           return ResponseEntity.ok(ProjectDTO.fromEntities(projects));
       }
   }
   ```

5. **Forbidden Patterns**:
   - ❌ `projectRepository.findAll()` (no orgId filter)
   - ❌ `projectRepository.findById(id)` (missing orgId)
   - ❌ Queries without WHERE clause for organisationId
   - ❌ Bypassing repository through direct EntityManager queries
   - ✅ `projectRepository.findByOrganisationIdAndStatus(orgId, status)`
   - ✅ `projectRepository.findByIdAndOrganisationId(projectId, orgId)`

6. **Error Handling**: Return 403 Forbidden if user attempts cross-org access
   ```java
   @GetMapping("/{projectId}")
   public ResponseEntity<ProjectDTO> getProject(@PathVariable UUID projectId) {
       UUID organisationId = SecurityContextUtil.getOrganisationId();
       Project project = projectService.getProjectDetails(projectId, organisationId);
       
       if (project.getOrganisationId().equals(organisationId)) {
           return ResponseEntity.ok(ProjectDTO.fromEntity(project));
       } else {
           return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
       }
   }
   ```

### JWT Payload Structure

```java
@Component
public class JwtProvider {
    // Token claims
    public static final String USER_ID = "userId";
    public static final String ORGANISATION_ID = "organisationId";
    public static final String ROLE = "role";
    
    public String generateToken(UserDetails user, UUID organisationId) {
        Map<String, Object> claims = new HashMap<>();
        claims.put(USER_ID, user.getUsername());
        claims.put(ORGANISATION_ID, organisationId.toString());
        claims.put(ROLE, user.getAuthorities());
        // Build token with claims
    }
}
```

## Security Rules

### Authentication

- All endpoints except `/api/auth/**` require JWT in `Authorization: Bearer <token>` header
- Use `JwtAuthenticationFilter` to verify and decode JWT before request reaches controller
- Store JWT secret in environment variable `jwt.secret` (minimum 32 characters)
- Token expiry: 24 hours (default)
- Use Spring Security's `@PreAuthorize` for role-based access control

```java
@RestController
@RequestMapping("/api/v1/projects")
public class ProjectController {
    @GetMapping
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<List<ProjectDTO>> getProjects() {
        // Protected endpoint
    }
    
    @PostMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public ResponseEntity<ProjectDTO> createProject(@RequestBody @Valid CreateProjectRequest req) {
        // Requires ADMIN or MANAGER role
    }
}
```

### Input Validation

- **All user input must be validated** using Jakarta Bean Validation annotations
- Use `@Valid` on controller method parameters
- Create validation groups for complex rules
- Return 400 Bad Request with detailed error messages for validation failures

```java
// DTO with validation annotations
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CreateProjectRequest {
    @NotBlank(message = "Project name is required")
    @Size(min = 3, max = 255, message = "Name must be 3-255 characters")
    private String projectName;
    
    @NotNull(message = "Status is required")
    @Pattern(regexp = "ACTIVE|INACTIVE|CLOSED", message = "Invalid status")
    private String status;
    
    @NotEmpty(message = "Team members required")
    private List<@NotBlank String> teamMemberIds;
}

// Controller using validation
@PostMapping
public ResponseEntity<ProjectDTO> createProject(
    @RequestBody @Valid CreateProjectRequest request) {
    // Validation happens automatically; invalid requests return 400
    Project project = projectService.createProject(request);
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(ProjectDTO.fromEntity(project));
}

// Global exception handler for validation errors
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
        MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.toList());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("Validation failed", errors));
    }
}
```

### Password & Secrets

- Hash passwords with Spring Security's `BCryptPasswordEncoder` (strength: 10)
- Never log passwords, tokens, or sensitive data
- Never commit `.env` or `application.properties` with secrets
- Use environment variables or Spring Cloud Config for sensitive configuration
- Rotate JWT secrets regularly in production

```java
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }
}

// Usage in user service
@Service
public class UserService {
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    public void createUser(String email, String plainPassword) {
        String hashedPassword = passwordEncoder.encode(plainPassword);
        // Store hashedPassword in database
    }
}
```

### Error Handling

- **Never expose internal database errors** to clients
- Catch exceptions in service layer and rethrow as domain-specific exceptions
- Global error handler in `@ControllerAdvice` converts exceptions to safe HTTP responses
- Log full error stack internally; return generic message to client

```java
// Custom exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class UnauthorisedAccessException extends RuntimeException {
    public UnauthorisedAccessException(String message) {
        super(message);
    }
}

// Global exception handler
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        logger.info("Resource not found: " + ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("Resource not found"));
    }
    
    @ExceptionHandler(UnauthorisedAccessException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorised(UnauthorisedAccessException ex) {
        logger.warn("Unauthorised access attempt: " + ex.getMessage());
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse("Access denied"));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
        logger.error("Unexpected error", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("An error occurred"));
    }
}

// Error response DTO
@Data
@AllArgsConstructor
public class ErrorResponse {
    private String message;
    private List<String> details;
    private LocalDateTime timestamp = LocalDateTime.now();
    
    public ErrorResponse(String message) {
        this.message = message;
    }
}
```

**HTTP Status Codes**:
- 400: Bad Request (validation failure)
- 401: Unauthorized (missing/invalid JWT)
- 403: Forbidden (organisation isolation violation, insufficient role)
- 404: Not Found (resource doesn't exist)
- 409: Conflict (immutability violation, e.g., updating audit log)
- 500: Internal Server Error (unexpected exception)

### Rate Limiting

- Implement via custom interceptor or Spring Security filter
- Rate limit: 100 requests per minute per IP
- Return 429 Too Many Requests when exceeded

```java
@Component
public class RateLimitingInterceptor implements HandlerInterceptor {
    private final RateLimiter rateLimiter = RateLimiter.create(100.0 / 60.0); // 100 per minute
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler) throws Exception {
        if (!rateLimiter.tryAcquire()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            return false;
        }
        return true;
    }
}
```

### CORS

- Configure CORS in `SecurityConfig`
- Allow only whitelisted origins (from `application.yml`)
- Restrict to specific HTTP methods

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.cors(cors -> cors.configurationSource(request -> {
            CorsConfiguration config = new CorsConfiguration();
            config.setAllowedOrigins(Arrays.asList(
                "http://localhost:3000",
                "http://localhost:3001"
            ));
            config.setAllowedMethods(Arrays.asList("GET", "POST", "PATCH", "DELETE"));
            config.setAllowedHeaders(Arrays.asList("*"));
            config.setAllowCredentials(true);
            return config;
        }));
        return http.build();
    }
}
```

## Immutability Rules (Audit & Compliance)

### Audit Log Immutability

Audit entries **must never be updated or deleted**.

```java
// Entity - no setters for immutability
@Entity
@Table(name = "audit_logs")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class AuditLog {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private UUID organisationId;
    
    @Column(nullable = false)
    private String eventType;
    
    @Column(columnDefinition = "jsonb")
    private String beforeState;
    
    @Column(columnDefinition = "jsonb")
    private String afterState;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    // No update/delete methods
}

// Repository - no update/delete methods
public interface AuditRepository extends JpaRepository<AuditLog, UUID> {
    List<AuditLog> findByOrganisationIdAndProjectId(
        UUID organisationId, 
        UUID projectId
    );
    
    // Custom queries only, no delete/update
}

// Service - enforce immutability
@Service
public class AuditService {
    @Autowired
    private AuditRepository auditRepository;
    
    public AuditLog createAuditEntry(UUID organisationId, String eventType,
                                     String beforeState, String afterState) {
        AuditLog entry = new AuditLog();
        entry.setOrganisationId(organisationId);
        entry.setEventType(eventType);
        entry.setBeforeState(beforeState);
        entry.setAfterState(afterState);
        return auditRepository.save(entry);
    }
    
    // Never allow update or delete
    public void deleteAuditEntry(UUID id) {
        throw new UnsupportedOperationException("Audit entries are immutable");
    }
}
```

**Database Constraint** (SQL):
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    before_state JSONB,
    after_state JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    CHECK (created_at IS NOT NULL)
);

-- Prevent updates/deletes
CREATE POLICY audit_immutable ON audit_logs
    FOR UPDATE, DELETE
    USING (false);
```

## Testing Expectations

### Unit Tests (Service Layer)

- Test business logic in isolation
- Mock repositories and external dependencies
- Use Mockito for mocking
- Coverage minimum: 80% for services

```java
@ExtendWith(MockitoExtension.class)
public class ProjectServiceTest {
    @Mock
    private ProjectRepository projectRepository;
    
    @InjectMocks
    private ProjectService projectService;
    
    @Test
    void testCreateProject() {
        // Arrange
        UUID organisationId = UUID.randomUUID();
        CreateProjectRequest request = new CreateProjectRequest("Test Project", "ACTIVE");
        Project expectedProject = new Project();
        expectedProject.setOrganisationId(organisationId);
        
        when(projectRepository.save(any(Project.class))).thenReturn(expectedProject);
        
        // Act
        Project result = projectService.createProject(organisationId, request);
        
        // Assert
        assertNotNull(result);
        assertEquals(organisationId, result.getOrganisationId());
        verify(projectRepository).save(any(Project.class));
    }
}
```

### Integration Tests (API Layer)

- Test complete API endpoints with real database
- Use `@SpringBootTest` + `@Testcontainers`
- Test authentication, authorization, and multi-tenant isolation
- Cover happy path and error cases

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public class ProjectControllerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private ProjectRepository projectRepository;
    
    @Test
    void testCreateProjectWithValidJWT() {
        // Setup
        String token = generateValidJWT("org-123");
        
        // Act
        ResponseEntity<ProjectDTO> response = restTemplate
            .postForEntity("/api/v1/projects",
                new CreateProjectRequest("New Project", "ACTIVE"),
                ProjectDTO.class,
                new HttpHeaders() {{ set("Authorization", "Bearer " + token); }});
        
        // Assert
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody());
    }
    
    @Test
    void testGetProjectDeniedForOtherOrganisation() {
        // Setup: Create project in org-1, request as org-2
        UUID projectId = createProjectInOrganisation(UUID.fromString("org-1"));
        String token = generateValidJWT("org-2");
        
        // Act
        ResponseEntity<ProjectDTO> response = restTemplate
            .getForEntity("/api/v1/projects/" + projectId,
                ProjectDTO.class,
                new HttpHeaders() {{ set("Authorization", "Bearer " + token); }});
        
        // Assert
        assertEquals(HttpStatus.FORBIDDEN, response.getStatusCode());
    }
}
```

### Minimum Test Coverage for Notification & Audit Service

1. **Audit Service Tests**:
   - Create audit entry correctly with organisation isolation
   - Prevent deletion of audit entry (immutability)
   - Query audit history filtered by date range
   - Query audit history filtered by event type
   - Prevent unauthorised access to other organisation's audit logs

2. **Notification Service Tests**:
   - Dispatch notifications to all team members on project state change
   - Mark notification as read (only read status changes)
   - Prevent access to other organisation's notifications

## Code Style & Documentation

### Naming Conventions

- **Classes**: PascalCase (e.g., `ProjectService`, `AuditLog`)
- **Methods**: camelCase (e.g., `createProject()`, `findByOrganisationId()`)
- **Variables**: camelCase (e.g., `projectId`, `organisationId`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `DEFAULT_PAGE_SIZE`, `JWT_SECRET`)
- **Packages**: lowercase.lowercase (e.g., `com.taskbridge.projects`)
- **DTOs**: Suffix with `DTO` or `Request`/`Response` (e.g., `ProjectDTO`, `CreateProjectRequest`)

### Comments & Documentation

- All public classes and methods must have JavaDoc comments
- Document parameters, return types, and thrown exceptions
- Document business rules and multi-tenant considerations

```java
/**
 * Creates an immutable audit log entry for compliance.
 * 
 * @param organisationId The organisation performing the action (required for multi-tenancy)
 * @param eventType Type of event (e.g., MILESTONE_CREATED, MILESTONE_UPDATED)
 * @param beforeState JSON snapshot of entity state before change
 * @param afterState JSON snapshot of entity state after change
 * @return Created immutable AuditLog entity
 * @throws IllegalArgumentException if organisationId is null
 * @implNote Audit entries are write-only and cannot be modified or deleted
 */
public AuditLog createAuditEntry(UUID organisationId, String eventType,
                                  String beforeState, String afterState) {
    // Implementation
}
```

### Code Formatting

- Use 4 spaces for indentation (Java convention)
- Follow Google Java Style Guide
- Line length: 100 characters maximum
- Use `@Slf4j` (Lombok) for logging instead of manual logger initialization

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class ProjectService {
    public void doSomething() {
        log.info("Action completed");
        log.warn("Warning message");
        log.error("Error occurred", exception);
    }
}
```

## Copilot-Specific Guidance

### When to Use Copilot

✅ **Good Uses**:
- Generating boilerplate JPA entities with proper annotations
- Creating CRUD repository interfaces
- Generating DTOs and validation request/response classes
- Writing repetitive service methods (CRUD operations)
- Generating test case structure and mock setup
- Writing database migration SQL scripts
- Creating API documentation/Swagger annotations

❌ **Caution/Override Required**:
- Security-critical code (authentication, JWT, encryption)
- Multi-tenant isolation logic (MUST manually verify organisation filtering)
- Complex business logic (manually review and test)
- Error handling and exception mapping (ensure errors don't leak sensitive info)
- Custom SQL queries (validate for SQL injection and organisation filtering)

### Prompting Strategy

1. **Provide Full Context**: Include tech stack, existing patterns, and constraints
   ```
   "Generate a Spring Boot JPA entity for Project with fields: id (UUID), 
   organisationId (UUID), projectName (String), status (String), createdAt (LocalDateTime).
   Include all Jakarta validation annotations. This is a multi-tenant application 
   where organisationId is required for all queries."
   ```

2. **Be Specific About Architecture**: Name the layer and patterns
   ```
   "Generate a repository interface for Project that extends JpaRepository.
   Include custom query methods: findByOrganisationIdAndStatus() and 
   findByIdAndOrganisationId(). Every query must filter by organisationId 
   for multi-tenant isolation."
   ```

3. **Request Spring Annotations**: Specify the exact annotations needed
   ```
   "Generate a Spring @Service class for ProjectService. Include @Transactional 
   on methods that modify state. Include dependency injection via @Autowired. 
   Add comprehensive JavaDoc for all public methods."
   ```

4. **Specify Constraints & Rules**: Include security and validation requirements
   ```
   "Generate a REST @RestController for audit endpoints. All endpoints must check 
   organisation isolation (throw 403 if user requests another org's data). 
   Include validation with @Valid. Return proper HTTP status codes (201 for created, 
   400 for validation error, 403 for forbidden, 404 for not found)."
   ```

5. **Iterate & Refine**: Generate, review, validate, iterate
   ```
   "Review the audit service code I just pasted. Check for:
   1. Is organisationId included in all queries?
   2. Are there any update/delete operations on AuditLog (must be immutable)?
   3. Is exception handling specific and secure?
   4. Are all public methods documented?"
   ```

### Validation Checklist Before Accepting Copilot Output

- [ ] Uses Spring Data JPA repositories (no raw SQL/JDBC)
- [ ] **Organisation isolation enforced** (`organisationId` in all repository queries)
- [ ] No database access outside repository layer
- [ ] DTOs used for API contracts (entities not exposed directly)
- [ ] Input validation present with `@Valid` and Jakarta Bean Validation
- [ ] Error handling specific (not generic `Exception` catch)
- [ ] No exposed internal errors or stack traces in responses
- [ ] JavaDoc comments on all public methods
- [ ] Follows Java naming conventions (PascalCase for classes, camelCase for methods)
- [ ] Tests included for happy path and error cases
- [ ] `@Transactional` used where state changes occur
- [ ] Proper HTTP status codes returned
- [ ] No hardcoded configuration (uses `application.yml`)
- [ ] Multi-tenant rules documented in comments

## Common Patterns

### Creating a REST Endpoint with Validation

```java
// Request DTO with validation
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CreateProjectRequest {
    @NotBlank(message = "Project name is required")
    @Size(min = 3, max = 255)
    private String projectName;
    
    @NotNull
    @Pattern(regexp = "ACTIVE|INACTIVE")
    private String status;
}

// Controller with endpoint
@RestController
@RequestMapping("/api/v1/projects")
@Slf4j
public class ProjectController {
    @Autowired
    private ProjectService projectService;
    
    @PostMapping
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<ProjectDTO> createProject(
        @RequestBody @Valid CreateProjectRequest request) {
        
        UUID organisationId = SecurityContextUtil.getOrganisationId();
        Project project = projectService.createProject(organisationId, request);
        
        log.info("Project created: {}", project.getId());
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ProjectDTO.fromEntity(project));
    }
}

// Service with business logic
@Service
@Slf4j
@Transactional
public class ProjectService {
    @Autowired
    private ProjectRepository projectRepository;
    
    public Project createProject(UUID organisationId, CreateProjectRequest request) {
        Project project = new Project();
        project.setOrganisationId(organisationId);
        project.setProjectName(request.getProjectName());
        project.setStatus(request.getStatus());
        project.setCreatedAt(LocalDateTime.now());
        
        Project saved = projectRepository.save(project);
        log.info("Project saved with ID: {}", saved.getId());
        return saved;
    }
}
```

### Multi-Tenant Query with Filtering

```java
// Repository with organisation-scoped queries
public interface ProjectRepository extends JpaRepository<Project, UUID> {
    List<Project> findByOrganisationIdAndStatus(
        UUID organisationId,
        String status,
        Pageable pageable
    );
    
    @Query("SELECT p FROM Project p WHERE p.organisationId = :orgId AND p.id = :projectId")
    Optional<Project> findByIdAndOrganisationId(
        @Param("projectId") UUID projectId,
        @Param("orgId") UUID organisationId
    );
    
    @Query("SELECT COUNT(p) FROM Project p WHERE p.organisationId = :orgId")
    long countByOrganisationId(@Param("orgId") UUID organisationId);
}

// Service method using multi-tenant queries
public List<Project> getProjectsByOrganisation(UUID organisationId, String status, int page) {
    Pageable pageable = PageRequest.of(page, 20); // 20 per page
    List<Project> projects = projectRepository
        .findByOrganisationIdAndStatus(organisationId, status, pageable)
        .getContent();
    return projects;
}
```

### Immutable Audit Log Creation

```java
// Entity definition
@Entity
@Table(name = "audit_logs")
@Data
@NoArgsConstructor
public class AuditLog {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private UUID organisationId;
    
    @Column(nullable = false, length = 50)
    private String eventType;
    
    @Column(columnDefinition = "jsonb")
    private String beforeState;
    
    @Column(columnDefinition = "jsonb")
    private String afterState;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    // No update methods - immutable
}

// Service to create audit entries
@Service
@Slf4j
public class AuditService {
    @Autowired
    private AuditRepository auditRepository;
    
    @Transactional
    public AuditLog createAuditEntry(UUID organisationId, String eventType,
                                     Object beforeState, Object afterState) {
        AuditLog entry = new AuditLog();
        entry.setOrganisationId(organisationId);
        entry.setEventType(eventType);
        entry.setBeforeState(jsonStringify(beforeState));
        entry.setAfterState(jsonStringify(afterState));
        
        AuditLog saved = auditRepository.save(entry);
        log.info("Audit entry created: {}", saved.getId());
        return saved;
    }
}
```

## Prohibited Patterns

❌ Direct database access in controllers or services (use repositories)  
❌ Queries without organisation isolation (missing WHERE organisationId)  
❌ Logging passwords, tokens, or sensitive data  
❌ Using raw SQL outside repositories  
❌ Updating or deleting audit log entries  
❌ Exposing database schema in error messages  
❌ Skipping input validation with `@Valid`  
❌ Hardcoding configuration (always use `application.yml`)  
❌ Returning entities directly from controllers (use DTOs)  
❌ Using `@SuppressWarnings` to hide warnings  
❌ Catching generic `Exception` without handling specific types  

## Pull Request Checklist

Before submitting a PR, verify:

- [ ] Code follows all conventions in this document
- [ ] **Organisation isolation enforced in all repository queries**
- [ ] Input validation present with `@Valid` and Jakarta Bean Validation
- [ ] Error handling specific (not generic)
- [ ] JavaDoc comments on all public methods
- [ ] Tests written with minimum 80% coverage
- [ ] `mvn test` and `mvn verify` pass locally
- [ ] No secrets or sensitive data committed
- [ ] Copilot output reviewed for security and multi-tenancy
- [ ] Commit messages follow Conventional Commits
- [ ] DTOs used for API contracts
- [ ] Proper HTTP status codes returned

## References

- Spring Boot Documentation: https://spring.io/projects/spring-boot
- Spring Security: https://spring.io/projects/spring-security
- Spring Data JPA: https://spring.io/projects/spring-data-jpa
- Jakarta Bean Validation: https://jakarta.ee/specifications/bean-validation/
- jjwt (JWT for Java): https://github.com/jpadilla/pyjwt
- OWASP Multi-Tenancy: https://owasp.org/www-community/attacks/

# System Architecture: TaskBridge Notification & Audit Service

## Overview

The TaskBridge Notification & Audit Service is designed as a microservice within a multi-tenant B2B SaaS platform. It orchestrates real-time notifications and immutable compliance audit logging for project milestone changes.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Applications                           │
│  (Web UI, Mobile, Desktop, Integrations)                        │
└──────────┬──────────────────────────────────────────────────────┘
           │ REST API Requests (JWT-authenticated)
           ├─ GET /api/v1/audit/{projectId}
           ├─ GET /api/v1/notifications/{userId}
           └─ PATCH /api/v1/notifications/{id}/read
           │
     ┌─────▼──────────────────────────────────────────────────────┐
     │                   API Gateway                               │
     │  (Rate Limiting, Request Logging, CORS)                    │
     └─────┬──────────────────────────────────────────────────────┘
           │
     ┌─────▼──────────────────────────────────────────────────────┐
     │       Spring Security (JWT Verification Filter)            │
     │  - Extract JWT from Authorization header                   │
     │  - Verify signature and expiry                             │
     │  - Extract organisationId and userId                       │
     │  - Populate SecurityContext                                │
     └─────┬──────────────────────────────────────────────────────┘
           │
     ┌─────▼──────────────────────────────────────────────────────┐
     │              Global Exception Handler                       │
     │  (Catches exceptions, returns safe error responses)        │
     └─────┬──────────────────────────────────────────────────────┘
           │
     ┌─────▼──────────────────────────────────────────────────────┐
     │                    Controllers                              │
     │  - AuditController      (POST /audit)                       │
     │  - NotificationController (GET, PATCH)                     │
     │  (Request validation with @Valid DTOs)                     │
     └──┬────────────────────────────────────────────────────────┘
        │ Input: DTOs, Authentication: JWT
        │ Output: Response DTOs
        │
   ┌────▼────────────────────────────────────────────────────────┐
   │                    Services (Layer)                           │
   │ ┌──────────────────┐  ┌──────────────────────────────────┐  │
   │ │  AuditService    │  │  NotificationService            │  │
   │ │ @Service         │  │ @Service                        │  │
   │ │ @Transactional   │  │ @Transactional                  │  │
   │ │                  │  │                                  │  │
   │ │ - createAudit()  │  │ - dispatchNotifications()       │  │
   │ │ - queryHistory() │  │ - markAsRead()                  │  │
   │ │ - (no delete!)   │  │ - (immutable except isRead)     │  │
   │ └────┬─────────────┘  └────┬───────────────────────────┘  │
   │      │                       │                              │
   │      └───────────┬─────���─────┘                              │
   │                  │ Calls repositories                       │
   └──────────────────┼──────────────────────────────────────────┘
                      │
      ┌───────────────┴────────────────┐
      │                                │
   ┌──▼──────────────────────────┐  ┌─▼──────────────────────────┐
   │  AuditRepository            │  │  NotificationRepository     │
   │  extends JpaRepository      │  │  extends JpaRepository      │
   │                             │  │                             │
   │ All methods include:        │  │ All methods include:        │
   │ - organisationId filtering  │  │ - organisationId filtering  │
   │ - (no delete/update!)       │  │ - (only read flag updatable)│
   └──┬───────────────┬──────────┘  └─┬──────────────┬──────────┘
      │               │               │              │
      │ JPA Queries   │               │ JPA Queries  │
      │               │               │              │
      └───────┬───────┴───────────────┴──────┬───────┘
              │                              │
        ┌─────▼──────────────────────────────▼──────┐
        │         PostgreSQL Database                │
        │                                            │
        │  Tables:                                   │
        │  - audit_logs (immutable, append-only)   │
        │  - notifications (read-only fields)      │
        │  - projects (from Project Service)       │
        │                                            │
        │  Constraints:                             │
        │  - No UPDATE/DELETE on audit_logs        │
        │  - organisationId always in WHERE clause │
        │  - Timestamp immutability                │
        └────────────────────────────────────────────┘
```

## Layered Architecture

### Layer 1: Presentation (Controllers)

**Responsibility**: HTTP request handling, validation, response formatting

```java
@RestController
@RequestMapping("/api/v1")
public class AuditController {
    @PostMapping("/audit")
    public ResponseEntity<AuditLogDTO> createAudit(
        @RequestBody @Valid CreateAuditRequest request) {
        // Validate input
        // Extract organisation from JWT
        // Delegate to service
        // Return response DTO
    }
}
```

**Key Points**:
- Uses DTOs for request/response (not entities)
- `@Valid` validates all input
- Delegates business logic to services
- Returns proper HTTP status codes

---

### Layer 2: Business Logic (Services)

**Responsibility**: Business rules, transaction management, inter-service coordination

```java
@Service
@Transactional
public class AuditService {
    public AuditLog createAuditEntry(
        UUID organisationId,
        String eventType,
        String beforeState,
        String afterState) {
        // Validate business rules
        // Check organisation isolation
        // Delegate to repository
        // Emit events
        // Handle errors
    }
}
```

**Key Points**:
- `@Transactional` ensures atomicity
- Enforces multi-tenant isolation
- Orchestrates multiple repositories
- Throws domain-specific exceptions

---

### Layer 3: Data Access (Repositories)

**Responsibility**: Database queries, ORM abstraction

```java
public interface AuditRepository extends JpaRepository<AuditLog, UUID> {
    List<AuditLog> findByOrganisationIdAndProjectId(
        UUID organisationId,
        UUID projectId
    );
    // All methods include organisationId parameter
}
```

**Key Points**:
- All queries include `organisationId`
- No delete/update methods for audit logs
- Custom `@Query` for complex filtering
- Pagination support via `Pageable`

---

### Layer 4: Data Model (Entities)

**Responsibility**: Database schema definition, ORM mapping

```java
@Entity
@Table(name = "audit_logs")
public class AuditLog {
    // JPA annotations
    // Immutable fields
    // Validation constraints
}
```

**Key Points**:
- JPA `@Entity` annotations
- No setters (immutability)
- `@Column(updatable = false)` on immutable fields

---

## Data Flow: End-to-End Request Example

**Scenario**: Client requests audit history for a project

```
1. Client sends:
   GET /api/v1/audit/project-123
   Headers: Authorization: Bearer eyJhbGc...
   Query: ?eventType=MILESTONE_UPDATED&from=2024-01-01

2. API Gateway:
   - Rate limiting check
   - Request logging

3. JwtAuthenticationFilter:
   - Extract token from header
   - Verify signature: ✅
   - Decode: organisationId = "org-456", userId = "user-789"
   - Populate SecurityContext

4. AuditController.getAuditHistory():
   - Validate @Valid request (none in this case)
   - Extract organisationId from SecurityContext ("org-456")
   - Validate path parameter (project-123 is UUID) ✅
   - Call auditService.getAuditHistory("org-456", "project-123", ...)

5. AuditService.getAuditHistory():
   - Check: Is user part of organisation "org-456"? ✅
   - Call auditRepository.findByOrganisationIdAndProjectId(
       organisationId: "org-456",
       projectId: "project-123"
     )
   - Filter by eventType and date range
   - Return list of AuditLog entities

6. AuditRepository.findByOrganisationIdAndProjectId():
   - Execute SQL:
     SELECT * FROM audit_logs
     WHERE organisation_id = 'org-456'
       AND project_id = 'project-123'
       AND event_type = 'MILESTONE_UPDATED'
       AND created_at >= '2024-01-01'
     ORDER BY created_at DESC
     LIMIT 20;
   - Return results

7. AuditService:
   - Convert AuditLog entities to AuditLogDTO
   - Return paginated response

8. AuditController:
   - Convert service response to HTTP response
   - Set status: 200 OK
   - Set body: { content: [...], totalElements: 42, ... }
   - Return to client

9. GlobalExceptionHandler:
   - (No exception, so not invoked)

10. Client receives:
    200 OK
    {
      "content": [
        {
          "id": "audit-1",
          "eventType": "MILESTONE_UPDATED",
          "beforeState": {...},
          "afterState": {...},
          "createdAt": "2024-09-07T14:30:00Z"
        },
        ...
      ],
      "totalElements": 42,
      "totalPages": 3,
      "currentPage": 0
    }
```

## Multi-Tenant Isolation Strategy

### Principle

**Every query must include organisation filtering.** A user from organisation A should never see data from organisation B.

### Implementation

1. **Authentication Layer**: JWT contains `organisationId`
2. **Repository Layer**: All queries filter by `organisationId`
3. **Service Layer**: Passes `organisationId` to all repository calls
4. **Controller Layer**: Extracts `organisationId` from JWT and validates ownership

### Example

```java
// Controller
UUID organisationId = SecurityContextUtil.getOrganisationId(); // From JWT
auditService.getAuditHistory(organisationId, projectId);

// Service
public List<AuditLog> getAuditHistory(UUID organisationId, UUID projectId) {
    return auditRepository.findByOrganisationIdAndProjectId(organisationId, projectId);
}

// Repository
@Query("SELECT a FROM AuditLog a WHERE a.organisationId = :orgId AND a.projectId = :projId")
List<AuditLog> findByOrganisationIdAndProjectId(
    @Param("orgId") UUID organisationId,
    @Param("projId") UUID projectId
);

// SQL (generated by Hibernate)
SELECT * FROM audit_logs WHERE organisation_id = $1 AND project_id = $2;
```

## Integration with Project Service

### Event-Driven Architecture

The Notification & Audit Service consumes events from the Project Service:

```
Project Service Event
    |
    v
Notification & Audit Service (Internal Listener)
    |
    +---> AuditService: Create immutable audit log
    +---> NotificationService: Dispatch to team members
```

### Event Contract

```json
{
  "eventId": "uuid",
  "eventType": "MILESTONE_UPDATED",
  "organisationId": "org-456",
  "projectId": "project-123",
  "actorUserId": "user-789",
  "beforeState": {...},
  "afterState": {...}
}
```

## Key Design Decisions & Trade-offs

### Decision 1: Write-Only Audit Logs

**Choice**: Audit logs are immutable; no update or delete operations

**Trade-off**:
- ✅ **Pro**: Ensures audit integrity; cannot be tampered with; compliance-ready
- ✅ **Pro**: Simpler schema; no versioning needed
- ❌ **Con**: Cannot correct typos or bad data (must create corrective entry)

**Rationale**: In a B2B SaaS platform, audit integrity is paramount for compliance. Immutability is non-negotiable.

---

### Decision 2: Multi-Service Architecture

**Choice**: Separate services for Projects, Notifications, and Audit

**Trade-off**:
- ✅ **Pro**: Independent scaling; clear separation of concerns
- ✅ **Pro**: Easy to update Project Service without affecting audit
- ❌ **Con**: Distributed system complexity; eventual consistency
- ❌ **Con**: Event ordering challenges

**Rationale**: Multi-tenancy and compliance require strong isolation. Separate services enable independent deployment.

---

### Decision 3: PostgreSQL JSONB for State Snapshots

**Choice**: Store `beforeState` and `afterState` as JSONB

**Trade-off**:
- ✅ **Pro**: Flexible schema; can store any state shape
- ✅ **Pro**: Queryable within PostgreSQL
- ❌ **Con**: Untyped; no schema validation
- ❌ **Con**: Larger storage footprint

**Rationale**: Projects can have varying structures and fields. JSONB allows flexibility while maintaining queryability.

---

### Decision 4: JWT-Based Authentication

**Choice**: JWT tokens for stateless authentication

**Trade-off**:
- ✅ **Pro**: Stateless; scalable horizontally
- ✅ **Pro**: Can encode organisation and user info
- ❌ **Con**: Tokens can be forged if secret is compromised
- ❌ **Con**: Cannot revoke tokens immediately

**Rationale**: Multi-tenant platform needs fast, stateless auth. JWT is the standard for microservices.

---

## Why This Architecture Is Appropriate for Multi-Tenant B2B SaaS

1. **Data Isolation**: Every query is scoped to organisation; impossible to leak data cross-tenant
2. **Compliance Ready**: Immutable audit logs satisfy regulatory requirements (SOC 2, GDPR)
3. **Scalability**: Separate services can scale independently
4. **Security**: JWT-based auth, role-based access control, input validation everywhere
5. **Observability**: Structured logging and audit trails for troubleshooting
6. **Auditability**: Full history of who did what, when, before/after state

## Technology Stack Rationale

- **Spring Boot**: Enterprise-grade framework with excellent Spring Security integration
- **PostgreSQL**: ACID compliance, JSONB support, row-level security
- **JPA/Hibernate**: Abstraction over raw SQL; helps prevent injection attacks
- **JWT (jjwt)**: Stateless auth; industry standard
- **TestContainers**: Real database testing; catches integration issues


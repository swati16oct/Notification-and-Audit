# Technical Specification: Notification & Audit Service

## 1. Overview

The Notification & Audit Service is a microservice component of the TaskBridge B2B SaaS platform. It sits between the Project Service and clients, providing real-time notifications and compliance-ready immutable audit logging for all project milestone state changes.

### Purpose

- **Notifications**: Emit real-time notifications to team members when project milestones change (created, updated, closed)
- **Audit Logging**: Maintain an immutable, compliance-certified audit log of all state changes with full context (who, what, when, before/after state)
- **Query & Filtering**: Enable clients to query audit history filtered by date range and event type
- **Multi-Tenancy**: Enforce strict organisation-level data isolation for security and compliance

### Scope

- Project milestone lifecycle events: `MILESTONE_CREATED`, `MILESTONE_UPDATED`, `MILESTONE_CLOSED`
- Future expansion: `MILESTONE_REOPENED` (scope change planned mid-sprint)
- Integration with Project Service via internal API
- No direct database access from client applications

---

## 2. Data Models

### 2.1 Audit Log Entity

**Table**: `audit_logs`

**Purpose**: Write-only immutable record of all state changes

**Fields**:

| Field | Type | Nullable | Updatable | Constraints | Description |
|-------|------|----------|-----------|-------------|-------------|
| `id` | UUID | No | No | PRIMARY KEY, DEFAULT gen_random_uuid() | Unique identifier |
| `organisationId` | UUID | No | No | FOREIGN KEY (organisations) | Organisation context (multi-tenant) |
| `eventType` | VARCHAR(50) | No | No | CHECK IN ('MILESTONE_CREATED', 'MILESTONE_UPDATED', 'MILESTONE_CLOSED', 'MILESTONE_REOPENED') | Type of event |
| `entityType` | VARCHAR(50) | No | No | Fixed: 'PROJECT' | Entity being changed |
| `entityId` | UUID | No | No | FOREIGN KEY (projects) | Reference to project/milestone |
| `actorUserId` | UUID | No | No | FOREIGN KEY (users) | User who performed action |
| `actorOrganisationId` | UUID | No | No | Same as organisationId for validation | Organisation of actor |
| `beforeState` | JSONB | Yes | No | | Snapshot of entity state before change |
| `afterState` | JSONB | Yes | No | | Snapshot of entity state after change |
| `ipAddress` | VARCHAR(45) | Yes | No | | Actor's IP address (future: MILESTONE_REOPENED) |
| `createdAt` | TIMESTAMP | No | No | DEFAULT NOW() | Immutable timestamp |

**Java Entity**:

```java
@Entity
@Table(name = "audit_logs")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class AuditLog {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private UUID organisationId;
    
    @Column(nullable = false, length = 50)
    @Enumerated(EnumType.STRING)
    private EventType eventType; // MILESTONE_CREATED, MILESTONE_UPDATED, etc.
    
    @Column(nullable = false, length = 50)
    private String entityType; // Always "PROJECT"
    
    @Column(nullable = false)
    private UUID entityId; // Project/Milestone ID
    
    @Column(nullable = false)
    private UUID actorUserId; // Who made the change
    
    @Column(nullable = false)
    private UUID actorOrganisationId; // Actor's organisation
    
    @Column(columnDefinition = "jsonb")
    private String beforeState; // Previous state JSON
    
    @Column(columnDefinition = "jsonb")
    private String afterState; // New state JSON
    
    @Column(length = 45)
    private String ipAddress; // IPv4 or IPv6
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    // No setters for immutability
}
```

**Immutability Enforcement**:
- `@Column(updatable = false)` on all fields
- No update/delete repository methods
- Service layer throws `UnsupportedOperationException` if update/delete attempted
- Database constraint: `CREATE POLICY audit_immutable ON audit_logs FOR UPDATE, DELETE USING (false);`

---

### 2.2 Notification Entity

**Table**: `notifications`

**Purpose**: Track notifications sent to team members

**Fields**:

| Field | Type | Nullable | Updatable | Constraints | Description |
|-------|------|----------|-----------|-------------|-------------|
| `id` | UUID | No | No | PRIMARY KEY | Unique identifier |
| `organisationId` | UUID | No | No | FOREIGN KEY | Organisation context |
| `recipientUserId` | UUID | No | No | FOREIGN KEY (users) | User receiving notification |
| `projectId` | UUID | No | No | FOREIGN KEY (projects) | Related project |
| `eventType` | VARCHAR(50) | No | Yes(read only) | CHECK IN (...) | Type of milestone event |
| `message` | TEXT | No | Yes(read only) | | Human-readable notification text |
| `isRead` | BOOLEAN | No | Yes | DEFAULT false | Read status |
| `readAt` | TIMESTAMP | Yes | No | | When marked as read |
| `createdAt` | TIMESTAMP | No | No | DEFAULT NOW() | Creation timestamp |

**Java Entity**:

```java
@Entity
@Table(name = "notifications")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Notification {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private UUID organisationId;
    
    @Column(nullable = false)
    private UUID recipientUserId;
    
    @Column(nullable = false)
    private UUID projectId;
    
    @Column(nullable = false, length = 50)
    @Enumerated(EnumType.STRING)
    private EventType eventType;
    
    @Column(nullable = false, columnDefinition = "TEXT")
    private String message;
    
    @Column(nullable = false)
    private Boolean isRead = false;
    
    @Column
    private LocalDateTime readAt;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
}
```

**Mutability Constraints**:
- Only `isRead` and `readAt` fields can be updated
- `message`, `eventType`, `projectId` are immutable after creation
- Service layer validates update constraints

---

### 2.3 Enums

```java
public enum EventType {
    MILESTONE_CREATED("Project milestone created"),
    MILESTONE_UPDATED("Project milestone updated"),
    MILESTONE_CLOSED("Project milestone closed"),
    MILESTONE_REOPENED("Project milestone reopened"); // Future
    
    private final String description;
    
    EventType(String description) {
        this.description = description;
    }
}
```

---

## 3. API Contracts

### 3.1 Audit Service Endpoints

#### POST /api/v1/audit

**Purpose**: Internal endpoint to record audit events (called by Project Service)

**Access**: Internal service-to-service only (no external client access)

**Request**:

```json
{
  "organisationId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "MILESTONE_CREATED",
  "entityType": "PROJECT",
  "entityId": "550e8400-e29b-41d4-a716-446655440001",
  "actorUserId": "550e8400-e29b-41d4-a716-446655440002",
  "beforeState": null,
  "afterState": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "name": "Q4 Product Launch",
    "status": "ACTIVE",
    "createdAt": "2024-09-07T10:30:00Z"
  }
}
```

**Request Validation**:
- `organisationId`: Required, valid UUID
- `eventType`: Required, must be valid EventType enum
- `entityId`: Required, valid UUID
- `actorUserId`: Required, valid UUID
- `beforeState`: Optional, valid JSON
- `afterState`: Optional, valid JSON

**Response** (201 Created):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440003",
  "organisationId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "MILESTONE_CREATED",
  "entityType": "PROJECT",
  "entityId": "550e8400-e29b-41d4-a716-446655440001",
  "actorUserId": "550e8400-e29b-41d4-a716-446655440002",
  "beforeState": null,
  "afterState": { ... },
  "createdAt": "2024-09-07T10:30:00.123456Z"
}
```

**Error Responses**:
- 400 Bad Request: Invalid input (validation failure)
- 403 Forbidden: Organisation isolation violation
- 409 Conflict: (Future) Duplicate event detection
- 500 Internal Server Error: Database or unexpected error

---

#### GET /api/v1/audit/{projectId}

**Purpose**: Query audit history for a project (external client access)

**Access**: Authenticated users; scoped to their organisation

**Query Parameters**:
- `eventType` (optional): Filter by event type (MILESTONE_CREATED, MILESTONE_UPDATED, etc.)
- `from` (optional): Start date (ISO 8601 format: `2024-01-01T00:00:00Z`)
- `to` (optional): End date (ISO 8601 format: `2024-12-31T23:59:59Z`)
- `page` (optional): Page number (default: 0)
- `size` (optional): Results per page (default: 20, max: 100)

**Example Request**:
```
GET /api/v1/audit/550e8400-e29b-41d4-a716-446655440001?eventType=MILESTONE_UPDATED&from=2024-01-01T00:00:00Z&to=2024-12-31T23:59:59Z&page=0&size=20
Authorization: Bearer <jwt_token>
```

**Response** (200 OK):

```json
{
  "content": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440003",
      "organisationId": "550e8400-e29b-41d4-a716-446655440000",
      "eventType": "MILESTONE_UPDATED",
      "entityType": "PROJECT",
      "entityId": "550e8400-e29b-41d4-a716-446655440001",
      "actorUserId": "550e8400-e29b-41d4-a716-446655440002",
      "beforeState": { "status": "ACTIVE" },
      "afterState": { "status": "CLOSED" },
      "createdAt": "2024-09-07T14:30:00Z"
    }
  ],
  "totalElements": 42,
  "totalPages": 3,
  "currentPage": 0,
  "size": 20
}
```

**Error Responses**:
- 400 Bad Request: Invalid date format or page parameters
- 401 Unauthorized: Missing or invalid JWT
- 403 Forbidden: User attempting to access another organisation's audit log
- 404 Not Found: Project doesn't exist in user's organisation

---

### 3.2 Notification Service Endpoints

#### GET /api/v1/notifications/{userId}

**Purpose**: Get all unread notifications for a user

**Access**: Authenticated users; can only access their own notifications

**Query Parameters**:
- `isRead` (optional): Filter by read status (true/false)
- `page` (optional): Page number (default: 0)
- `size` (optional): Results per page (default: 20)

**Example Request**:
```
GET /api/v1/notifications/550e8400-e29b-41d4-a716-446655440002?isRead=false
Authorization: Bearer <jwt_token>
```

**Response** (200 OK):

```json
{
  "content": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440004",
      "organisationId": "550e8400-e29b-41d4-a716-446655440000",
      "recipientUserId": "550e8400-e29b-41d4-a716-446655440002",
      "projectId": "550e8400-e29b-41d4-a716-446655440001",
      "eventType": "MILESTONE_CREATED",
      "message": "New milestone 'Q4 Product Launch' created in project",
      "isRead": false,
      "createdAt": "2024-09-07T10:30:00Z"
    }
  ],
  "totalElements": 5,
  "totalPages": 1,
  "currentPage": 0,
  "size": 20
}
```

**Error Responses**:
- 401 Unauthorized: Missing or invalid JWT
- 403 Forbidden: User attempting to access another user's notifications

---

#### PATCH /api/v1/notifications/{id}/read

**Purpose**: Mark a notification as read

**Access**: Authenticated users; can only mark their own notifications

**Request**:

```json
{
  "isRead": true
}
```

**Response** (200 OK):

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440004",
  "isRead": true,
  "readAt": "2024-09-07T10:35:00Z"
}
```

**Error Responses**:
- 400 Bad Request: Invalid request body
- 401 Unauthorized: Missing or invalid JWT
- 403 Forbidden: User attempting to modify another user's notification
- 404 Not Found: Notification doesn't exist

---

## 4. Integration with Project Service

### 4.1 Event Flow

```
┌─────────────────────┐
│  Project Service    │
│  (Project Updated)  │
└──────────┬──────────┘
           │
           ├─ Emit Event: MILESTONE_UPDATED
           │
           ▼
┌─────────────────────────────────────┐
│  Notification & Audit Service       │
│  (Internal Event Handler)           │
└──────┬──────────────────────┬───────┘
       │                      │
       ▼                      ▼
┌─────────────────┐   ┌──────────────────┐
│  Audit Service  │   │ Notification     │
│  Create Audit   │   │ Service          │
│  Log Entry      │   │ Dispatch to Team │
└─────────────────┘   └──────────────────┘
       │                      │
       ├─ Write to audit_logs  ├─ Create notifications
       │  (Immutable)          │  (For each team member)
       │                       │
       └─ Emit Audit Event     └─ Emit Notification Event
```

### 4.2 Service-to-Service Contract

**Project Service → Notification & Audit Service**

When project milestone changes:
1. Project Service emits internal event with full project state (before & after)
2. Audit Service consumes event, creates immutable audit log
3. Notification Service consumes event, identifies team members, creates notifications
4. Both services return success or error to Project Service

**Example Event Message**:

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440010",
  "eventType": "MILESTONE_UPDATED",
  "timestamp": "2024-09-07T14:30:00Z",
  "projectId": "550e8400-e29b-41d4-a716-446655440001",
  "organisationId": "550e8400-e29b-41d4-a716-446655440000",
  "actorUserId": "550e8400-e29b-41d4-a716-446655440002",
  "beforeState": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "name": "Q4 Product Launch",
    "status": "ACTIVE",
    "teamMemberIds": ["user-1", "user-2", "user-3"]
  },
  "afterState": {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "name": "Q4 Product Launch",
    "status": "CLOSED",
    "teamMemberIds": ["user-1", "user-2", "user-3"]
  }
}
```

---

## 5. Security & Authorization

### 5.1 Authentication

- All endpoints require JWT in `Authorization: Bearer <token>` header
- JWT payload contains: `userId`, `organisationId`, `role`, `iat`, `exp`
- Token verified via Spring Security `JwtAuthenticationFilter`

### 5.2 Multi-Tenant Isolation

**Critical Rule**: Every query must include organisation filtering

```java
// ✅ Correct
auditRepository.findByOrganisationIdAndProjectId(organisationId, projectId);

// ❌ Wrong
auditRepository.findByProjectId(projectId); // Missing organisationId!
```

**Enforcement**:
- Repository methods require `organisationId` parameter
- Service layer always passes `organisationId` from JWT
- Controller returns 403 Forbidden if user requests cross-organisation data

### 5.3 Role-Based Access

- `ADMIN`: Full access to organisation's audit logs and settings
- `MANAGER`: View audit logs for assigned projects
- `MEMBER`: View audit logs and notifications for assigned projects

### 5.4 Data Privacy

- Audit log IP addresses must be logged but never exposed to clients
- Error responses must not leak internal system information
- Passwords and tokens never logged

---

## 6. Validation Rules

### 6.1 Input Validation

```java
// Audit Entry Validation
@Data
public class CreateAuditRequest {
    @NotNull(message = "organisationId required")
    private UUID organisationId;
    
    @NotNull(message = "eventType required")
    @Pattern(regexp = "MILESTONE_CREATED|MILESTONE_UPDATED|MILESTONE_CLOSED|MILESTONE_REOPENED")
    private String eventType;
    
    @NotNull(message = "entityId required")
    private UUID entityId;
    
    @NotNull(message = "actorUserId required")
    private UUID actorUserId;
    
    // beforeState and afterState optional but must be valid JSON if provided
}
```

### 6.2 Business Rule Validation

- `organisationId` in audit entry must match JWT's `organisationId`
- `actorUserId` must exist in user service
- `projectId` in audit/notification must exist in project service
- Audit entries cannot be updated or deleted
- Only `isRead` field can be updated on notifications

---

## 7. Immutability & Compliance

### 7.1 Audit Log Immutability

- **Write-only**: Audit entries created once and never modified
- **Database-level**: Constraints prevent UPDATE/DELETE
- **Application-level**: Repository has no update/delete methods
- **Service-level**: Throws `UnsupportedOperationException` if update/delete attempted

### 7.2 Compliance Requirements

- All audit entries timestamped in UTC
- Full before/after state snapshots preserved as JSON
- Actor information (user ID, organisation ID) immutable
- Audit trail cannot be tampered with or deleted
- Supports regulatory requirements (SOC 2, GDPR audit trails)

---

## 8. Constraints & Limitations

### 8.1 Current Constraints (Phase 1)

- Supports only `PROJECT` entity type (future: expand to tasks, milestones)
- Three event types: `MILESTONE_CREATED`, `MILESTONE_UPDATED`, `MILESTONE_CLOSED`
- IP address tracking: Not yet captured (Phase 2: `MILESTONE_REOPENED`)
- Notification retention: 90 days (configurable)
- Audit log retention: Indefinite (compliance requirement)

### 8.2 Performance Constraints

- Audit queries limited to 12-month date range
- Pagination: Maximum 100 results per page
- Notification queries: Maximum 50 unread per user

---

## 9. Future Enhancements (Post-MVP)

- IP address logging for `MILESTONE_REOPENED` event
- Audit log retention policies (configurable per organisation)
- Real-time WebSocket notifications
- Email/SMS notification templates
- Advanced audit analytics and dashboards
- Event-driven cascade (auto-notify on dependent project changes)

---

## Appendix: Copilot-Assisted Decisions

### Where Copilot Helped

1. **DTO Structure**: Copilot generated initial request/response DTO classes with proper validation annotations
2. **Repository Method Names**: Generated Spring Data JPA query method names following naming conventions
3. **Entity Annotations**: Provided boilerplate JPA entity annotations and column definitions
4. **API Documentation**: Generated initial Swagger/OpenAPI annotations for endpoints
5. **Error Response Format**: Suggested consistent error response structure

### Where Manual Judgment Applied

1. **Multi-Tenant Isolation**: Manually verified all repository queries include `organisationId` filter (Copilot tends to omit this)
2. **Immutability Enforcement**: Manually designed the immutability constraints and validated Copilot didn't add update/delete methods
3. **Event Flow Design**: Manually designed the event flow and inter-service contract (Copilot suggested synchronous calls, we chose async)
4. **Security Model**: Manually verified JWT payload structure and role-based access rules
5. **Database Schema**: Manually reviewed Hibernate-generated DDL to ensure constraints are correct

### Key Override Decisions

- **Copilot suggested**: Allow notification updates → **We decided**: Immutable except for `isRead` flag
- **Copilot suggested**: DELETE endpoint for audit logs → **We decided**: No delete operations allowed
- **Copilot suggested**: Separate repositories per service → **We decided**: Shared repository but with strong isolation

---

**Specification Version**: 1.0  
**Last Updated**: 2024-09-07  
**Status**: Approved for Development

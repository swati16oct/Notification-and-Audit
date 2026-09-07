# TaskBridge — Notification & Audit Service

A multi-service B2B SaaS platform built with Spring Boot, real-time notifications, and compliance-ready audit logging.

## Technology Stack

### Backend
- **Language**: Java 17+
- **Framework**: Spring Boot 3.x
- **Web**: Spring MVC (REST APIs)
- **Database**: PostgreSQL 14+ (ACID compliance)
- **ORM**: Spring Data JPA with Hibernate
- **Authentication**: Spring Security 6.x + JWT (jjwt)
- **Async Processing**: Spring Task Scheduling + RabbitMQ

### Validation & Security
- **Validation**: Jakarta Bean Validation (JSR-380)
- **Password**: Spring Security (bcrypt)
- **Rate Limiting**: Spring Cloud Config + Custom Interceptors
- **CORS**: Spring Security CORS configuration
- **Logging**: SLF4J with Logback

### Testing & Quality
- **Test Framework**: JUnit 5 (Jupiter)
- **Mocking**: Mockito
- **Integration Testing**: Spring Boot Test + TestContainers
- **Code Quality**: SonarQube compatible, Checkstyle
- **Build**: Maven 3.8+

### DevOps & Tooling
- **Version Control**: Git (GitHub)
- **CI/CD**: GitHub Actions
- **Containerization**: Docker
- **Configuration**: Spring Cloud Config, application.yml

## Project Structure

```
Notification-and-Audit/
├── .github/
│   └── copilot-instructions.md
├── src/
│   ├── main/
│   │   ├── java/com/taskbridge/
│   │   │   ├── TaskBridgeApplication.java
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── JwtConfig.java
│   │   │   │   └── DatabaseConfig.java
│   │   │   ├── projects/
│   │   │   │   ├── model/
│   │   │   │   │   └── Project.java
│   │   │   │   ├── repository/
│   │   │   │   │   └── ProjectRepository.java
│   │   │   │   ├── service/
│   │   │   │   │   └── ProjectService.java
│   │   │   │   └── controller/
│   │   │   │       └── ProjectController.java
│   │   │   ├── notifications/
│   │   │   │   ├── model/
│   │   │   │   │   └── Notification.java
│   │   │   │   ├── repository/
│   │   │   │   │   └── NotificationRepository.java
│   │   │   │   ├── service/
│   │   │   │   │   └── NotificationService.java
│   │   │   │   └── controller/
│   │   │   │       └── NotificationController.java
│   │   │   ├── audit/
│   │   │   │   ├── model/
│   │   │   │   │   └── AuditLog.java
│   │   │   │   ├── repository/
│   │   │   │   │   └── AuditRepository.java
│   │   │   │   ├── service/
│   │   │   │   │   └── AuditService.java
│   │   │   │   └── controller/
│   │   │   │       └── AuditController.java
│   │   │   ├── security/
│   │   │   │   ├── JwtProvider.java
│   │   │   │   ├── JwtAuthenticationFilter.java
│   │   │   │   └── CustomUserDetailsService.java
│   │   │   ├── middleware/
│   │   │   │   ├── ValidationExceptionHandler.java
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   └── RequestLoggingInterceptor.java
│   │   │   └── util/
│   │   │       └── SecurityContextUtil.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-test.yml
│   │       └── application-prod.yml
│   └── test/
│       ├── java/com/taskbridge/
│       │   ├── audit/
│       │   │   └── AuditServiceIntegrationTest.java
│       │   ├── notifications/
│       │   │   └── NotificationServiceIntegrationTest.java
│       │   └── integration/
│       │       └── MultiTenancyIntegrationTest.java
│       └── resources/
│           └── application-test.yml
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── SPEC.md
├── REVIEW.md
├── ARCHITECTURE.md
├── IMPACT_ANALYSIS.md
├── PROMPTS.md
├── TOOL_STRATEGY.md
└── PR_DESCRIPTION.md
```

## Getting Started

### Prerequisites
- Java 17 or higher
- Maven 3.8+
- PostgreSQL 14+
- Docker (optional)

### Installation

```bash
# Clone the repository
git clone https://github.com/swati16oct/Notification-and-Audit.git
cd Notification-and-Audit

# Copy environment configuration
cp .env.example .env

# Build the project
mvn clean install

# Run database migrations (via Spring Boot startup)
mvn spring-boot:run
```

### Running Tests

```bash
# Run all tests
mvn test

# Run with coverage
mvn test jacoco:report

# Run specific test class
mvn test -Dtest=AuditServiceIntegrationTest
```

## Key Features

### Project Service
- Create, update, and delete projects
- Track project milestone status changes
- Multi-tenant project isolation
- Team member assignment and role-based access

### Notification & Audit Service
- Real-time notifications on project milestone changes
- Immutable audit logging for compliance
- Queryable audit history with filtering (date range, event type)
- User notification management (read/unread status)

### Security & Multi-Tenancy
- JWT-based authentication via Spring Security
- Organization-scoped data access (multi-tenant isolation)
- Input validation with Jakarta Bean Validation
- CORS protection via Spring Security
- Rate limiting via custom interceptors
- Structured error handling

## API Endpoints

### Project Service
- `POST /api/v1/projects` - Create project
- `GET /api/v1/projects` - Get projects (organization-scoped)
- `GET /api/v1/projects/{id}` - Get project by ID
- `PATCH /api/v1/projects/{id}` - Update project status
- `DELETE /api/v1/projects/{id}` - Delete project

### Audit Service (Internal)
- `POST /api/v1/audit` - Record audit event
- `GET /api/v1/audit/{projectId}` - Get audit history (with filters)
- `GET /api/v1/audit/{projectId}?eventType=MILESTONE_CREATED&from=2024-01-01&to=2024-12-31` - Filtered history

### Notification Service
- `GET /api/v1/notifications/{userId}` - Get unread notifications
- `PATCH /api/v1/notifications/{id}/read` - Mark notification as read

## Architecture Highlights

- **Layered Architecture**: Model → Repository → Service → Controller
- **Spring Data JPA**: ORM-based data access with custom queries
- **Spring Security**: Authentication and authorization
- **Event-Driven**: Async notification processing
- **Immutable Audit Logs**: Write-only persistence for compliance
- **Multi-Tenant Isolation**: Organization-level data scoping

## Documentation

- **SPEC.md**: Technical specification for Notification & Audit Service
- **REVIEW.md**: Project Service code review and remediation
- **ARCHITECTURE.md**: System design and integration patterns
- **IMPACT_ANALYSIS.md**: Scope change impact assessment
- **PROMPTS.md**: Copilot prompt engineering documentation
- **TOOL_STRATEGY.md**: AI tool usage patterns and limitations
- **.github/copilot-instructions.md**: Development standards and conventions

## Development Workflow

1. **Branch**: Create feature branch from `main`
2. **Develop**: Follow conventions in `.github/copilot-instructions.md`
3. **Test**: Run `mvn test` ensuring 80%+ coverage
4. **Commit**: Use Conventional Commits format
5. **PR**: Submit with detailed description (see PR_DESCRIPTION.md)
6. **Review**: Peer review before merge

## Building with Docker

```bash
# Build Docker image
docker build -t taskbridge-api:latest .

# Run with docker-compose
docker-compose up -d
```

## Contributing

Refer to `.github/copilot-instructions.md` for code standards, architecture patterns, and development guidelines.

## License

MIT License — See LICENSE file for details

## Contact

**Tech Lead**: TaskBridge Engineering Team  
**Repository**: https://github.com/swati16oct/Notification-and-Audit
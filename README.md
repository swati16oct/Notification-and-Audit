# TaskBridge — Notification & Audit Service

A multi-service B2B SaaS platform built with modern architecture patterns, real-time notifications, and compliance-ready audit logging.

## Technology Stack

### Backend
- **Runtime**: Node.js (v18+)
- **Language**: JavaScript (ES6+) / TypeScript (for type safety)
- **Framework**: Express.js (HTTP API framework)
- **Database**: PostgreSQL 14+ (primary data store, ACID compliance)
- **ORM**: Sequelize (for data abstraction and migrations)
- **Async Processing**: Bull Queue (event processing for notifications)

### Authentication & Security
- **JWT**: jsonwebtoken (Bearer token authentication)
- **Password**: bcryptjs (hashing and salting)
- **Rate Limiting**: express-rate-limit (DDoS/brute-force protection)
- **CORS**: cors middleware (cross-origin request handling)
- **Input Validation**: joi (schema validation)

### Testing & Quality
- **Test Framework**: Jest (unit and integration testing)
- **API Testing**: Supertest (HTTP endpoint testing)
- **Code Coverage**: nyc (coverage reporting)
- **Linting**: ESLint (code quality)
- **Formatting**: Prettier (code style consistency)

### DevOps & Tooling
- **Version Control**: Git (GitHub)
- **CI/CD**: GitHub Actions
- **Environment**: dotenv (environment variable management)
- **Logging**: winston (structured logging)

## Project Structure

```
taskbridge-api/
├── .github/
│   └── copilot-instructions.md
├── src/
│   ├── projects/
│   │   ├── models/
│   │   │   └── Project.js
│   │   ├── repositories/
│   │   │   └── ProjectRepository.js
│   │   ├── services/
│   │   │   └── ProjectService.js
│   │   └── controllers/
│   │       └── ProjectController.js
│   ├── notifications/
│   │   ├── models/
│   │   │   └── Notification.js
│   │   ├── repositories/
│   │   │   └── NotificationRepository.js
│   │   ├── services/
│   │   │   └── NotificationService.js
│   │   └── controllers/
│   │       └── NotificationController.js
│   ├── audit/
│   │   ├── models/
│   │   │   └── AuditLog.js
│   │   ├── repositories/
│   │   │   └── AuditRepository.js
│   │   ├── services/
│   │   │   └── AuditService.js
│   │   └── controllers/
│   │       └── AuditController.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── validation.js
│   │   └── errorHandler.js
│   ├── config/
│   │   └── database.js
│   └── index.js
├── tests/
│   ├── unit/
│   └── integration/
├── package.json
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
- Node.js v18+
- PostgreSQL 14+
- npm v9+

### Installation

```bash
git clone https://github.com/swati16oct/Notification-and-Audit.git
cd Notification-and-Audit
npm install
cp .env.example .env
npm run migrate
npm run dev
```

### Running Tests

```bash
npm test
npm run test:coverage
```

## Key Features

- **Project Service**: Create, update, delete projects with status tracking
- **Notification & Audit Service**: Real-time notifications and immutable audit logging
- **Multi-Tenancy**: Organization-scoped data access
- **Security**: JWT auth, input validation, CORS protection, rate limiting
- **Compliance**: Immutable audit logs for regulatory requirements

## License

MIT License
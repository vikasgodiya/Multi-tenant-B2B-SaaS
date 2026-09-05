# Scalable Multi-Tenant B2B SaaS Backend

A production-oriented **multi-tenant B2B SaaS backend** built to demonstrate secure tenant isolation, authentication, role-based access control, subscription management, caching, rate limiting, background jobs, audit logging, testing, and scalable backend architecture.

## 🚀 Overview

This project simulates the backend of a SaaS platform where multiple organizations (tenants) use the same application infrastructure while keeping their data securely isolated.

Example:

```text
Tenant A (Company A)
 ├── Users
 ├── Projects
 ├── Tasks
 ├── Subscription
 └── Audit Logs

Tenant B (Company B)
 ├── Users
 ├── Projects
 ├── Tasks
 ├── Subscription
 └── Audit Logs
```

The main design goal is to ensure that users belonging to one organization can **never access another organization's resources**.

---

## ✨ Key Features

- Multi-tenant SaaS architecture
- Secure tenant-level data isolation
- JWT-based authentication
- Role-Based Access Control (RBAC)
- Organization/user management
- Project and task management
- Subscription and plan management
- API key authentication
- Redis caching
- Distributed API rate limiting
- Background job processing
- Audit logging
- Pagination, filtering, and sorting
- Input validation
- Centralized error handling
- Health/readiness checks
- Structured logging
- Dockerized development environment
- Automated testing
- OpenAPI/Swagger API documentation
- Architecture designed for horizontal scaling

---

## 🏗️ Architecture

```text
                         ┌───────────────┐
                         │    Clients    │
                         │ Web / Mobile  │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │ Load Balancer │
                         └───────┬───────┘
                                 │
                  ┌──────────────┼──────────────┐
                  ▼              ▼              ▼
              API Server     API Server     API Server
                  │              │              │
                  └──────────────┼──────────────┘
                                 │
             ┌───────────────────┼───────────────────┐
             ▼                   ▼                   ▼
        PostgreSQL             Redis             Job Queue
             │                   │                   │
             │                   │                   ▼
             │                   │               Workers
             │                   │
             ▼                   ▼
       Persistent Data       Cache / Rate Limit
```

The API servers are designed to remain stateless so additional instances can be added when traffic increases.

---

## 🧩 Multi-Tenancy

Every organization is represented by a tenant.

Tenant-owned resources contain a `tenant_id`.

Example:

```text
projects
--------------------------------
id
tenant_id
name
description
created_at
updated_at
```

Every database operation is scoped to the authenticated tenant.

Conceptually:

```sql
SELECT *
FROM projects
WHERE tenant_id = :current_tenant_id;
```

The application derives the tenant from the authenticated user's identity rather than trusting arbitrary tenant IDs supplied by clients.

### Tenant Isolation Example

```text
Tenant A
   │
   └── Project A ✅

Tenant B
   │
   └── Project B

Tenant A requesting Project B
   │
   └── ❌ Access Denied
```

For stronger defense-in-depth, PostgreSQL Row-Level Security can also be used.

---

## 🔐 Authentication

Authentication uses token-based security.

Typical flow:

```text
User
 │
 ▼
POST /auth/login
 │
 ▼
Validate Credentials
 │
 ▼
Generate Access Token
 │
 ▼
Client
 │
 ▼
Authorization: Bearer <token>
```

A token can contain information such as:

```json
{
  "user_id": "user-id",
  "tenant_id": "tenant-id",
  "role": "admin"
}
```

Protected endpoints validate the token before executing business logic.

---

## 👥 Role-Based Access Control

The platform supports organization-level roles.

Example:

```text
OWNER
ADMIN
MEMBER
VIEWER
```

Example permissions:

| Role | Users | Projects | Billing | Settings |
|------|-------|----------|---------|----------|
| Owner | Full | Full | Full | Full |
| Admin | Full | Full | Read | Full |
| Member | Read | Create/Edit | None | None |
| Viewer | Read | Read | None | None |

Authorization flow:

```text
Request
   │
   ▼
Authentication
   │
   ▼
Identify User + Tenant
   │
   ▼
Check Role
   │
   ▼
Check Permission
   │
   ▼
Execute Request
```

---

## 🗄️ Database

The primary database is PostgreSQL.

Core entities include:

```text
Tenant
User
Membership
Role
Permission
Project
Task
Plan
Subscription
APIKey
AuditLog
```

Relationship:

```text
Tenant
 ├── Membership ─── User
 ├── Projects
 │     └── Tasks
 ├── Subscription ─── Plan
 └── Audit Logs
```

### Important Indexes

Commonly queried fields should be indexed:

```text
tenant_id
user_id
project_id
created_at
status
```

The architecture also supports future database optimizations such as read replicas, partitioning, and sharding.

---

## ⚡ Redis

Redis is used for fast, temporary, or frequently accessed data.

Potential use cases:

- Caching
- Rate limiting
- Session/token metadata
- Permission caching
- Subscription caching
- Distributed locks

Example:

```text
Request
  │
  ▼
Redis
  │
  ├── Cache Hit ──► Return Data
  │
  └── Cache Miss
          │
          ▼
      PostgreSQL
          │
          ▼
      Store in Redis
          │
          ▼
      Return Data
```

---

## 🚦 Rate Limiting

The API can enforce tenant/user-level rate limits.

Example:

```text
Free Plan       → 100 requests/minute
Pro Plan        → 1,000 requests/minute
Enterprise      → Custom limits
```

Redis can maintain distributed counters such as:

```text
rate_limit:{tenant_id}:{user_id}
```

This allows rate limiting to work across multiple API instances.

---

## 💳 Subscription & Plans

The SaaS platform supports subscription tiers.

Example:

```text
FREE
PRO
BUSINESS
ENTERPRISE
```

A plan can define:

```text
max_users
max_projects
max_api_requests
storage_limit
feature_flags
```

Example:

```json
{
  "plan": "PRO",
  "max_users": 50,
  "max_projects": 100,
  "api_requests_per_minute": 1000
}
```

Business logic checks subscription limits before performing restricted operations.

---

## 🔑 API Keys

API keys allow external applications and services to communicate with the platform.

Example:

```http
Authorization: ApiKey <API_KEY>
```

Security practices:

- Tenant-scoped keys
- Permission-scoped keys
- Hash keys before storing them
- Support revocation
- Apply rate limits
- Record important API-key actions in audit logs

Plaintext API keys should never be stored in the database.

---

## 📝 Audit Logging

Important organization actions are recorded.

Examples:

```text
USER_CREATED
USER_DELETED
PROJECT_CREATED
PROJECT_UPDATED
ROLE_CHANGED
SUBSCRIPTION_CHANGED
API_KEY_CREATED
API_KEY_REVOKED
```

Example:

```json
{
  "tenant_id": "tenant-id",
  "user_id": "user-id",
  "action": "PROJECT_CREATED",
  "resource": "project",
  "resource_id": "project-id",
  "timestamp": "2026-09-05T10:00:00Z"
}
```

Audit logs are useful for security investigations, compliance, debugging, and administrative visibility.

---

## 🔄 Background Jobs

Long-running tasks should not block API requests.

Examples:

- Email delivery
- Report generation
- File processing
- Data exports
- Notifications
- Analytics aggregation
- Cleanup jobs

Architecture:

```text
API
 │
 ▼
Job Queue
 │
 ├── Worker 1
 ├── Worker 2
 └── Worker 3
```

Workers can be scaled independently from API servers.

---

## 📄 Pagination

Large datasets are paginated.

Example:

```http
GET /api/v1/projects?page=1&limit=20
```

Response:

```json
{
  "items": [],
  "page": 1,
  "limit": 20,
  "total": 150
}
```

For very large datasets, cursor-based pagination can be introduced:

```http
GET /api/v1/projects?cursor=<cursor>
```

---

## 🔍 Filtering & Sorting

Resources can support filtering and sorting.

Examples:

```http
GET /api/v1/tasks?status=completed
```

```http
GET /api/v1/tasks?priority=high
```

```http
GET /api/v1/tasks?sort=-created_at
```

Multiple filters can be combined:

```http
GET /api/v1/tasks?status=pending&priority=high&sort=-created_at
```

---

## 📡 API Endpoints

### Authentication

```http
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
```

### Tenants

```http
POST   /api/v1/tenants
GET    /api/v1/tenants/me
PATCH  /api/v1/tenants/me
DELETE /api/v1/tenants/me
```

### Users

```http
GET    /api/v1/users
GET    /api/v1/users/{id}
POST   /api/v1/users
PATCH  /api/v1/users/{id}
DELETE /api/v1/users/{id}
```

### Projects

```http
GET    /api/v1/projects
POST   /api/v1/projects
GET    /api/v1/projects/{id}
PATCH  /api/v1/projects/{id}
DELETE /api/v1/projects/{id}
```

### Tasks

```http
GET    /api/v1/tasks
POST   /api/v1/tasks
GET    /api/v1/tasks/{id}
PATCH  /api/v1/tasks/{id}
DELETE /api/v1/tasks/{id}
```

### Billing

```http
GET  /api/v1/billing/subscription
POST /api/v1/billing/subscribe
POST /api/v1/billing/cancel
```

### Audit Logs

```http
GET /api/v1/audit-logs
```

---

## 🧱 Clean Architecture

The codebase is organized into separate layers.

```text
app/
│
├── api/
│   ├── routes/
│   └── dependencies/
│
├── core/
│   ├── config.py
│   ├── security.py
│   └── exceptions.py
│
├── models/
│
├── schemas/
│
├── repositories/
│
├── services/
│
├── middleware/
│
├── workers/
│
└── main.py
```

### API Layer

Handles:

- HTTP requests
- Validation
- Responses
- Dependency injection

### Service Layer

Contains business logic:

```text
UserService
TenantService
ProjectService
BillingService
AuditService
```

### Repository Layer

Handles database access:

```text
UserRepository
TenantRepository
ProjectRepository
```

This separation makes the system easier to test, maintain, and extend.

---

## 🔒 Security

Security considerations include:

- Password hashing
- JWT authentication
- RBAC authorization
- Tenant isolation
- Input validation
- SQL injection protection
- Rate limiting
- API key hashing
- CORS configuration
- Secure HTTP headers
- Request size limits
- Audit logging
- Environment-based secrets
- Database constraints
- Optional PostgreSQL Row-Level Security

Never commit secrets to Git.

Use:

```text
.env
```

and keep it in `.gitignore`.

---

## 🩺 Health Checks

Example endpoint:

```http
GET /health
```

Response:

```json
{
  "status": "healthy"
}
```

A readiness endpoint can verify dependencies:

```text
Application
 ├── PostgreSQL
 ├── Redis
 └── Job Queue
```

Example:

```http
GET /health/ready
```

These endpoints are useful with Docker, load balancers, and container orchestration.

---

## 📊 Observability

The architecture supports:

- Structured logging
- Request IDs
- Error tracking
- API latency metrics
- Request counts
- Database performance metrics
- Background job metrics

A request can be associated with:

```text
request_id
tenant_id
user_id
endpoint
latency
status_code
```

This makes production debugging easier.

---

## 🐳 Docker

The development environment can run using Docker Compose.

Typical services:

```text
docker-compose.yml

├── api
├── worker
├── postgres
└── redis
```

Start:

```bash
docker compose up --build
```

Stop:

```bash
docker compose down
```

---

## ⚙️ Local Setup

### 1. Clone

```bash
git clone https://github.com/<username>/<repository>.git
cd <repository>
```

### 2. Create Virtual Environment

Windows:

```bash
python -m venv venv
venv\Scripts\activate
```

Linux/macOS:

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Environment

Copy:

```text
.env.example
```

to:

```text
.env
```

Example:

```env
DATABASE_URL=postgresql://user:password@localhost:5432/saas
REDIS_URL=redis://localhost:6379
JWT_SECRET=change-me
JWT_ALGORITHM=HS256
```

### 5. Start Infrastructure

```bash
docker compose up -d postgres redis
```

### 6. Run Migrations

```bash
alembic upgrade head
```

### 7. Start API

```bash
uvicorn app.main:app --reload
```

The API will be available at:

```text
http://localhost:8000
```

Swagger UI:

```text
http://localhost:8000/docs
```

---

## 🧪 Testing

Run all tests:

```bash
pytest
```

Run coverage:

```bash
pytest --cov=app
```

Example test categories:

```text
tests/
├── unit/
├── integration/
└── api/
```

Important tests include:

### Tenant Isolation

```text
Tenant A → Project A ✅
Tenant A → Project B ❌
```

### Authorization

```text
Viewer → Read Project ✅
Viewer → Delete Project ❌
```

### Authentication

```text
Valid Token → Access API ✅
Expired Token → Reject ❌
```

### Subscription Limits

```text
Within Limit → Allow ✅
Over Limit → Reject ❌
```

---

## 📁 Project Structure

```text
scalable-multi-tenant-saas/
│
├── app/
│   ├── api/
│   ├── core/
│   ├── models/
│   ├── schemas/
│   ├── repositories/
│   ├── services/
│   ├── middleware/
│   ├── workers/
│   └── main.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── api/
│
├── migrations/
├── scripts/
├── docker/
│
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

## 📈 Scalability

### Stage 1 — MVP

```text
FastAPI
   │
PostgreSQL
```

### Stage 2 — Production Backend

```text
FastAPI
 │
 ├── PostgreSQL
 ├── Redis
 └── Background Workers
```

### Stage 3 — Horizontal Scaling

```text
             Load Balancer
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
     API 1      API 2      API 3
       │          │          │
       └──────────┼──────────┘
                  │
            PostgreSQL
                  │
                Redis
```

### Stage 4 — Distributed Architecture

```text
API Gateway
     │
 ┌───┼───────────────┐
 ▼   ▼               ▼
Auth Project       Billing
     │
     ▼
Event Streaming
     │
 ┌───┼─────────────┐
 ▼   ▼             ▼
Workers Analytics Notifications
```

---

## ⚡ Performance Strategy

### Database

- Proper indexes
- Connection pooling
- Query optimization
- Read replicas
- Partitioning for large tables
- Future sharding

### Application

- Stateless API servers
- Horizontal scaling
- Pagination
- Async processing

### Redis

- Cache frequently accessed data
- Distributed rate limiting
- Distributed locks

---

## 🛡️ Threat Model

| Threat | Mitigation |
|--------|------------|
| Cross-tenant data access | Tenant-scoped queries |
| Brute-force attacks | Rate limiting |
| SQL injection | ORM / parameterized queries |
| Stolen API keys | Hashing + revocation |
| Unauthorized operations | RBAC |
| Token abuse | Expiration + refresh |
| Excessive requests | Redis rate limiting |
| Secret leakage | Environment variables |
| Untracked sensitive actions | Audit logs |

---

## 🔮 Future Improvements

- Google/GitHub OAuth
- Two-factor authentication
- SSO/SAML
- Webhooks
- Stripe integration
- Usage-based billing
- Advanced analytics
- Full-text search
- Elasticsearch/OpenSearch
- PostgreSQL read replicas
- Database partitioning
- Tenant sharding
- Kubernetes deployment
- CI/CD pipeline
- Prometheus metrics
- Grafana dashboards
- OpenTelemetry tracing
- Kafka/event-driven architecture
- S3-compatible object storage

---

## 🧠 What This Project Demonstrates

This project goes beyond basic CRUD and demonstrates practical backend engineering concepts:

```text
REST APIs
   ↓
Authentication
   ↓
Authorization
   ↓
Multi-Tenancy
   ↓
Database Design
   ↓
Caching
   ↓
Rate Limiting
   ↓
Background Jobs
   ↓
Audit Logging
   ↓
Docker
   ↓
Testing
   ↓
Observability
   ↓
Horizontal Scaling
   ↓
Distributed Systems
```

---

## 💼 Resume Highlights

Possible resume bullets:

- Designed and developed a **multi-tenant B2B SaaS backend** with tenant-scoped data isolation, JWT authentication, RBAC, subscription controls, and secure REST APIs.
- Implemented **Redis-based caching and distributed rate limiting** to reduce database load and protect APIs from excessive traffic.
- Built asynchronous background processing, audit logging, pagination, centralized error handling, health checks, and automated tests following clean architecture principles.
- Containerized the backend with **Docker** and designed a stateless architecture capable of horizontal scaling behind a load balancer.

---

## 📜 License

This project is intended for educational, portfolio, and demonstration purposes.

Add your preferred license, such as:

```text
MIT License
```

---

## 👨‍💻 Author

**Your Name**

GitHub: `https://github.com/<username>`

LinkedIn: `<your-linkedin-profile>`

---

> **Note:** Keep this README synchronized with the actual implementation. If a feature is planned but not implemented yet, describe it as planned rather than claiming it is currently available.

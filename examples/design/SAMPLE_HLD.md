# TaskFlow Platform — High-Level Design

> **This is a SAMPLE.** TaskFlow is a fictional project. All dates, version strings, owner names, line numbers, and architecture details below are illustrative — they exist to demonstrate the shape of a complete HLD, not to assert facts about a real system. Replace every field with your project's actuals before treating this as authoritative.

**Version:** 1.0.0
**Status:** Approved
**Last Updated:** 2025-03-18
**Owner:** Engineering Team

---

## 1. Executive Summary

TaskFlow is a multi-tenant SaaS task management platform that enables teams to organize, track, and collaborate on work in real time. The platform consists of a web application, REST API, mobile companion app, and supporting cloud infrastructure.

**Key Characteristics:**

| Attribute | Value |
|-----------|-------|
| System Type | Multi-tenant Web SaaS + Mobile |
| Users | Development teams, project managers, freelancers |
| Scale Target | 10K tenants, 100K users, 1M tasks |
| Deployment | AWS (us-east-1, eu-west-1) |

---

## 2. System Purpose & Goals

### 2.1 Goals

| Goal | Description | Measurable Target |
|------|-------------|-------------------|
| Real-time collaboration | Users see changes instantly | WebSocket event delivery < 500ms |
| Multi-tenant isolation | Tenant data never leaks | RLS on all tables, cross-tenant tests fail |
| Mobile companion | Core features available on mobile | Feature parity for task CRUD and notifications |
| Availability | System available during business hours | 99.9% uptime (43min downtime/month max) |

### 2.2 Non-Goals

| Non-Goal | Rationale |
|----------|-----------|
| Gantt charts / timeline views | Deferred to v2; core task management first |
| On-premise deployment | SaaS-only for MVP; reduces operational complexity |
| AI task suggestions | Research phase; not enough training data yet |
| Video/voice calls | Out of scope; integrate with existing tools instead |

---

## 3. Architecture Overview

### 3.1 System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Clients                                     │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐                │
│  │ Web SPA  │  │ Mobile App   │  │ Webhook        │                │
│  │ (React)  │  │ (React Nat.) │  │ Consumers      │                │
│  └────┬─────┘  └──────┬───────┘  └───────┬────────┘                │
│       │               │                  │                          │
└───────┼───────────────┼──────────────────┼──────────────────────────┘
        │ HTTPS         │ HTTPS            │ HTTPS
        ▼               ▼                  ▼
┌───────────────────────────────────────────────────────────────┐
│  CloudFront CDN (static assets) + ALB (API traffic)          │
└───────────────────────────┬───────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│  API Layer (ECS Fargate)                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Express.js API Server                                   │  │
│  │  ├── Auth Middleware (JWT via Keycloak)                   │  │
│  │  ├── Tenant Middleware (RLS context)                      │  │
│  │  ├── Rate Limiter (Redis-backed)                         │  │
│  │  ├── REST Routes (/api/v1/*)                             │  │
│  │  └── WebSocket Server (Socket.io)                        │  │
│  └─────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Background Worker (BullMQ)                              │  │
│  │  ├── Email notifications                                 │  │
│  │  ├── Webhook delivery (with retries)                     │  │
│  │  └── Scheduled cleanup jobs                              │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────┬───────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │ Redis 7.2    │ │ S3           │
│ 16 (RDS)     │ │ (ElastiCache)│ │ (File store) │
│ + RLS        │ │ Cache, Queue │ │ Attachments  │
└──────────────┘ └──────────────┘ └──────────────┘
```

### 3.2 Component Summary

| Component | Responsibility | Technology | Owner |
|-----------|---------------|------------|-------|
| Web Frontend | Task management UI, real-time updates | React 18, Tailwind, Socket.io client | Frontend Team |
| Backend API | Business logic, data access, auth | Node.js 20, Express, Prisma, Socket.io | Backend Team |
| Mobile App | Companion app with push notifications | React Native 0.73, Expo | Mobile Team |
| Database | Persistent storage with tenant isolation | PostgreSQL 16, RLS policies | Backend Team |
| Cache/Queue | Session cache, rate limits, job queue | Redis 7.2, BullMQ | Backend Team |
| File Storage | Task attachments, profile images | S3 with pre-signed URLs | Platform Team |
| Auth Provider | Identity management, SSO | Keycloak 24 | Platform Team |
| Infrastructure | IaC, CI/CD, monitoring | AWS CDK, GitHub Actions | Platform Team |

### 3.3 Component Interaction Matrix

| From \ To | Web | API | Mobile | PostgreSQL | Redis | S3 |
|-----------|-----|-----|--------|------------|-------|-----|
| Web | — | REST + WS | — | — | — | Pre-signed URLs |
| API | WS events | — | Push notifications | Prisma ORM | Cache + Queue | SDK |
| Mobile | — | REST + WS | — | — | — | Pre-signed URLs |

---

## 4. Data Flow

### 4.1 Task Creation Flow

```
User (Web/Mobile)
    │
    │ POST /api/v1/tasks {title, description, assignee_id}
    ▼
Auth Middleware ──► Validate JWT ──► Extract tenant_id
    │
    ▼
Tenant Middleware ──► SET app.current_tenant = tenant_id
    │
    ▼
Task Service ──► Validate input ──► Create task (Prisma)
    │
    ├──► PostgreSQL: INSERT INTO tasks (RLS enforced)
    ├──► Redis: Invalidate task list cache
    ├──► WebSocket: Emit "task:created" to project room
    └──► BullMQ: Queue email notification for assignee
```

### 4.2 Data Flow Table

| Flow | Source | Destination | Protocol | Frequency | Payload |
|------|--------|-------------|----------|-----------|---------|
| Task CRUD | Client | API | REST/HTTPS | ~100 req/s peak | JSON, <10KB |
| Real-time updates | API | Clients | WebSocket | ~500 events/s peak | JSON, <2KB |
| Push notifications | API | Mobile | FCM/APNs | ~50/s peak | JSON, <1KB |
| File uploads | Client | S3 | HTTPS (pre-signed) | ~10/s peak | Binary, <25MB |
| Email notifications | Worker | SES | SMTP | ~20/s peak | HTML, <50KB |

---

## 5. Security Architecture

### 5.1 Authentication & Authorization

- **Identity Provider:** Keycloak 24 with OIDC
- **Token Format:** JWT (RS256, 15-minute expiry, refresh tokens)
- **Authorization Model:** RBAC per tenant (Owner, Admin, Member, Guest)
- **API Auth:** Bearer token in Authorization header
- **WebSocket Auth:** Token passed during connection handshake

### 5.2 Multi-Tenancy

| Layer | Isolation Mechanism |
|-------|-------------------|
| Database | PostgreSQL RLS — every table has `tenant_id`, policy enforced |
| API | Tenant middleware extracts `tenant_id` from JWT, sets DB session |
| File Storage | S3 key prefix: `tenants/{tenant_id}/attachments/` |
| Cache | Redis key prefix: `tenant:{tenant_id}:` |
| Auth | Keycloak realms per tenant (or shared realm with tenant claim) |

### 5.3 Data Protection

- **In Transit:** TLS 1.3 everywhere (ALB termination, RDS SSL, Redis TLS)
- **At Rest:** RDS encryption (AWS KMS), S3 SSE-S3
- **Secrets:** AWS Secrets Manager, rotated automatically
- **PII:** Email and name stored encrypted; logs redact PII fields

---

## 6. Infrastructure & Deployment

### 6.1 Deployment Architecture

```
┌── us-east-1 ──────────────────────────────────────┐
│                                                     │
│  CloudFront ──► ALB ──► ECS Fargate (API + Worker) │
│                                                     │
│  RDS Multi-AZ (PostgreSQL 16)                      │
│  ElastiCache (Redis 7.2, cluster mode)             │
│  S3 (attachments, static assets)                   │
│  Keycloak (ECS Fargate)                            │
│                                                     │
│  CloudWatch (metrics, logs, alarms)                │
│  X-Ray (distributed tracing)                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 6.2 Environment Summary

| Environment | Purpose | Infrastructure |
|-------------|---------|----------------|
| Development | Local dev | Docker Compose (PostgreSQL, Redis, Keycloak) |
| Staging | Pre-production testing | AWS (single-AZ, smaller instances) |
| Production | Live system | AWS (Multi-AZ, auto-scaling) |

---

## 7. Cross-Cutting Concerns

### 7.1 Observability

| Signal | Tool | Retention |
|--------|------|-----------|
| Metrics | CloudWatch + Prometheus | 15 months (CloudWatch) |
| Logs | CloudWatch Logs | 30 days hot, 1 year archived to S3 |
| Traces | AWS X-Ray | 30 days |
| Uptime | CloudWatch Synthetics | Real-time |

### 7.2 Performance Budgets

| Metric | Target | Measurement |
|--------|--------|-------------|
| API response time (p95) | < 200ms | CloudWatch ALB metrics |
| WebSocket event delivery | < 500ms end-to-end | Custom metric (emit → receive) |
| Page load (LCP) | < 2.5s | Lighthouse CI |
| Database query time (p95) | < 50ms | Prisma query logging |
| Background job latency | < 30s for email | BullMQ metrics |

### 7.3 Cost Targets

| Resource | Monthly Cost | At 10K Tenants |
|----------|-------------|----------------|
| ECS Fargate | ~$400 | ~$800 (auto-scale) |
| RDS (db.t4g.medium) | ~$150 | ~$400 (db.r6g.large) |
| ElastiCache | ~$100 | ~$200 (cluster) |
| S3 + CloudFront | ~$50 | ~$500 |
| **Total** | **~$700** | **~$1,900** |

---

## 8. Constraints & Limitations

| Constraint | Impact | Mitigation |
|------------|--------|------------|
| PostgreSQL RLS adds ~5% query overhead | Slightly higher DB latency | Indexing on tenant_id, connection pooling |
| Keycloak single realm for MVP | Token size grows with permissions | Migrate to realm-per-tenant if token > 4KB |
| S3 pre-signed URLs expire | File links break after expiry | 1-hour expiry, client refreshes on 403 |
| WebSocket requires sticky sessions | Limits horizontal scaling | Redis adapter for Socket.io (cross-node events) |

---

## 9. Critical Invariants

These properties must hold at all times. Referenced by LLDs (§9) and enforced by review agents.

1. **Tenant Isolation:** No database query ever returns data from another tenant. RLS is enforced at the database level, not application level.
2. **Auth Required:** No API endpoint (except `/health` and `/auth/*`) is accessible without a valid JWT.
3. **Data Integrity:** Task state transitions follow the defined state machine (§4). Invalid transitions are rejected, not silently corrected.
4. **Audit Trail:** All write operations (create, update, delete) produce an audit log entry with user_id, tenant_id, timestamp, and change details.

---

## 10. Appendices

### 10.1 Glossary

| Term | Definition |
|------|-----------|
| Tenant | An organization/workspace with isolated data |
| Project | A collection of tasks within a tenant |
| Task | A unit of work with title, description, assignee, status |
| Member | A user belonging to a tenant with a specific role |

### 10.2 References

- [REST API Spec](../taskflow-specs/api/REST_ENDPOINTS.md)
- [WebSocket Events Spec](../taskflow-specs/events/WEBSOCKET_EVENTS.md)
- [VERSIONS.yaml](../VERSIONS.yaml)

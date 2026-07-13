# Multi-Tenant SaaS API

> A secure, scalable backend architecture for building organization-based SaaS applications with tenant isolation, authentication, role-based access control, audit logging, background processing, and configurable platform policies.

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Solution](#solution)
* [Key Features](#key-features)
* [Architecture](#architecture)
* [Technology Stack](#technology-stack)
* [Multi-Tenant Data Model](#multi-tenant-data-model)
* [Authentication](#authentication)
* [Authorization](#authorization)
* [Tenant Isolation](#tenant-isolation)
* [Organization Switching](#organization-switching)
* [Security Architecture](#security-architecture)
* [Two-Factor Authentication](#two-factor-authentication)
* [Email and Background Jobs](#email-and-background-jobs)
* [Audit Logging](#audit-logging)
* [API Structure](#api-structure)
* [Example API Endpoints](#example-api-endpoints)
* [Request Lifecycle](#request-lifecycle)
* [Database Design](#database-design)
* [Environment Configuration](#environment-configuration)
* [Local Development](#local-development)
* [Database Migrations](#database-migrations)
* [Testing Strategy](#testing-strategy)
* [Deployment](#deployment)
* [Scalability](#scalability)
* [Production Security Checklist](#production-security-checklist)
* [Future Enhancements](#future-enhancements)
* [Key Engineering Decisions](#key-engineering-decisions)
* [Author](#author)

---

## Overview

The **Multi-Tenant SaaS API** is a backend reference architecture for software products that serve multiple independent organizations from a shared application.

The API provides the foundation required for:

* User authentication
* Organization management
* Tenant memberships
* Active organization switching
* Role-based authorization
* Permission-based platform access
* Tenant-scoped resources
* Two-factor authentication
* Password recovery
* Security policy enforcement
* Audit logging
* Email queues
* Background workers
* Platform administration

The architecture is suitable for products such as:

* B2B SaaS platforms
* Agency management systems
* Monitoring applications
* Customer portals
* Subscription platforms
* White-label products
* Business management software
* Multi-location operational systems

---

## Problem Statement

A multi-tenant SaaS backend must solve several complex problems:

* How should users belong to one or more organizations?
* How should tenant data remain isolated?
* How should platform administrators differ from organization users?
* How should permissions be enforced consistently?
* How should users switch between organizations?
* How should existing sessions be invalidated?
* How should security policies such as mandatory 2FA be applied?
* How should background jobs and email delivery remain reliable?
* How should sensitive administrative actions be audited?
* How should the application scale without weakening security?

These requirements cannot be solved safely through frontend filtering alone. They require a carefully designed backend authorization and data-access model.

---

## Solution

The API uses a shared database with strict tenant-level isolation.

Each tenant is represented as an `Organization`.

Users are connected to organizations through an `OrganizationMembership` record rather than storing a single organization directly on the user.

This allows:

* One user to belong to multiple organizations
* Different roles in different organizations
* Secure organization switching
* Centralized identity management
* Separate platform and tenant authorization
* Tenant-scoped resource ownership

All tenant-owned queries are constrained by the active organization.

---

## Key Features

### Tenant Management

* Create and manage organizations
* Add and remove organization members
* Support multiple organizations per user
* Switch active organization
* Assign organization-level roles
* Suspend or deactivate organizations
* Store tenant-specific settings

### Authentication

* Email and password login
* Email verification
* JWT access tokens
* Password reset
* Password update
* Session invalidation
* Two-factor authentication
* Login attempt protection
* Password reset rate limiting

### Authorization

* Organization roles
* Platform roles
* Fine-grained platform permissions
* Owner-only operations
* Resource-level tenant checks
* Centralized guards and decorators

### Security

* TOTP-based 2FA
* Mandatory 2FA policy
* 2FA grace period
* Security audit logs
* Brute-force protection
* Token revocation through security timestamps
* Encrypted sensitive configuration
* Server-side validation

### Operational Features

* Database-backed email queue
* Delivery retry logic
* Scheduled background jobs
* Email delivery logs
* Security notifications
* CSV export support
* Retention cleanup jobs

---

## Architecture

```mermaid id="w9x3a7"
flowchart LR
    Client[Web or Mobile Client] -->|HTTPS REST API| Gateway[API Gateway / NGINX]
    Gateway --> API[NestJS API]

    API --> Auth[Authentication Module]
    API --> Tenant[Tenant Context]
    API --> Permission[Authorization Layer]
    API --> Modules[Business Modules]

    Auth --> DB[(PostgreSQL)]
    Tenant --> DB
    Permission --> DB
    Modules --> DB

    Modules --> Queue[(Job / Email Queue)]
    Worker[Background Worker] --> Queue
    Worker --> SMTP[SMTP Provider]

    API --> Audit[(Audit Logs)]
    Worker --> Audit
```

### Logical Layers

```text id="uf7lp4"
Client Application
       |
       v
Reverse Proxy / API Gateway
       |
       v
Controllers
       |
       v
Authentication and Authorization Guards
       |
       v
Tenant Context Validation
       |
       v
Application Services
       |
       v
Prisma Data Access Layer
       |
       v
PostgreSQL Database
```

---

## Technology Stack

| Category                  | Technology          |
| ------------------------- | ------------------- |
| Runtime                   | Node.js             |
| Backend Framework         | NestJS              |
| Language                  | TypeScript          |
| ORM                       | Prisma              |
| Database                  | PostgreSQL          |
| Authentication            | JWT                 |
| Two-Factor Authentication | TOTP                |
| Email                     | Nodemailer and SMTP |
| Scheduling                | NestJS Scheduler    |
| Validation                | DTO validation      |
| Containerization          | Docker              |
| Reverse Proxy             | NGINX               |
| Package Manager           | pnpm                |
| Source Control            | Git and GitHub      |
| CI/CD                     | GitHub Actions      |

---

## Multi-Tenant Data Model

```mermaid id="1smtkp"
erDiagram
    USER ||--o{ ORGANIZATION_MEMBERSHIP : belongs_to
    ORGANIZATION ||--o{ ORGANIZATION_MEMBERSHIP : has
    USER ||--o| PLATFORM_USER_PROFILE : may_have

    ORGANIZATION ||--o{ TENANT_RESOURCE : owns
    ORGANIZATION ||--o{ ORGANIZATION_SETTING : configures
    ORGANIZATION ||--o{ EMAIL_QUEUE : queues
    ORGANIZATION ||--o{ AUDIT_LOG : generates

    PLATFORM_ROLE ||--o{ PLATFORM_ROLE_PERMISSION : contains
    PERMISSION ||--o{ PLATFORM_ROLE_PERMISSION : assigned_to
    PLATFORM_USER_PROFILE }o--|| PLATFORM_ROLE : uses
```

### User

The `User` entity stores the central identity.

Typical fields include:

```text id="e88f4m"
id
email
passwordHash
firstName
lastName
emailVerifiedAt
isActive
securityChangedAt
passwordUpdatedAt
createdAt
updatedAt
```

A user is not automatically treated as a tenant user or platform administrator.

Access is derived from related profile and membership records.

### Organization

The `Organization` entity represents a tenant.

Typical fields include:

```text id="ur7vnk"
id
name
slug
status
ownerId
createdAt
updatedAt
```

### Organization Membership

The `OrganizationMembership` entity connects users and organizations.

```text id="ekvbgc"
id
userId
organizationId
role
status
joinedAt
createdAt
updatedAt
```

Example roles:

```text id="5irk6c"
OWNER
ADMIN
MEMBER
```

A user can hold different roles in different organizations.

### Platform User Profile

Platform-level access is stored separately.

```text id="m4gpeg"
id
userId
platformRoleId
status
createdAt
updatedAt
```

This prevents platform permissions from being mixed with tenant membership roles.

---

## Authentication

The API uses JWT-based authentication.

### Login Flow

```mermaid id="4dys6k"
sequenceDiagram
    participant U as User
    participant API as API
    participant DB as Database
    participant TFA as 2FA Service

    U->>API: Submit email and password
    API->>DB: Find user
    DB-->>API: User record
    API->>API: Verify password

    alt 2FA required
        API-->>U: Return 2FA challenge token
        U->>API: Submit TOTP code
        API->>TFA: Verify TOTP
        TFA-->>API: Verification result
    end

    API->>API: Generate JWT
    API-->>U: Access token and session data
```

### JWT Payload

A JWT may contain:

```json id="ubpnh5"
{
  "sub": "user-id",
  "organizationId": "active-organization-id",
  "organizationRole": "ADMIN",
  "platformRole": null,
  "issuedAt": 1760000000
}
```

The token should not be treated as the only source of authorization truth.

Critical access checks should also validate current database state.

---

## Authorization

The API supports two authorization systems.

### Organization Authorization

Organization authorization controls access within a tenant.

Roles:

| Role   | Typical Access                  |
| ------ | ------------------------------- |
| Owner  | Full organization control       |
| Admin  | Operational and user management |
| Member | Limited resource access         |

Example owner-only operations:

* Transfer ownership
* Delete organization
* Change sensitive billing settings
* Remove another owner
* Configure critical security settings

### Platform Authorization

Platform authorization controls access to SaaS administration features.

Example platform roles:

* Super Administrator
* Support
* Sales
* Marketing
* Developer
* Finance
* Operations

Instead of relying only on role names, platform access can be defined using granular permissions.

Example permissions:

```text id="gfcxmc"
organizations.view
organizations.manage
platform_users.view
platform_users.manage
security_logs.view
security_policy.manage
smtp.manage
billing.view
billing.manage
```

### Authorization Guard Sequence

```text id="kvt51y"
JWT Authentication
       |
       v
Account Status Check
       |
       v
Email Verification Check
       |
       v
Security Timestamp Check
       |
       v
2FA Policy Check
       |
       v
Tenant Membership Check
       |
       v
Role or Permission Check
       |
       v
Resource Ownership Check
```

---

## Tenant Isolation

Tenant isolation is the most important security requirement in the architecture.

Every tenant-owned query must include the active `organizationId`.

### Unsafe Query

```typescript id="91t8mk"
const resource = await prisma.resource.findUnique({
  where: {
    id: resourceId,
  },
});
```

This query is unsafe because it does not confirm that the resource belongs to the active tenant.

### Tenant-Scoped Query

```typescript id="p9iaxd"
const resource = await prisma.resource.findFirst({
  where: {
    id: resourceId,
    organizationId: activeOrganizationId,
  },
});
```

The organization identifier must come from validated authentication context rather than unrestricted request input.

### Recommended Service Pattern

```typescript id="8hp2kk"
async findResource(
  resourceId: string,
  organizationId: string,
): Promise<Resource> {
  const resource = await this.prisma.resource.findFirst({
    where: {
      id: resourceId,
      organizationId,
    },
  });

  if (!resource) {
    throw new NotFoundException('Resource not found');
  }

  return resource;
}
```

Returning `404 Not Found` instead of revealing that another tenant owns the resource can reduce information disclosure.

### Tenant Isolation Rules

* Never trust `organizationId` from the request body without validation.
* Derive the active tenant from authenticated context.
* Confirm that the user has an active membership.
* Scope list, read, update, and delete operations.
* Scope nested relations.
* Scope exports and reports.
* Scope background jobs.
* Scope audit-log visibility.
* Apply the same checks to administrative support workflows.

---

## Organization Switching

Users may belong to multiple organizations.

The API supports secure active-organization switching.

### Switching Flow

```mermaid id="o4o9su"
sequenceDiagram
    participant U as User
    participant API as API
    participant DB as Database

    U->>API: Request switch to organization
    API->>DB: Validate active membership
    DB-->>API: Membership and role
    API->>API: Generate updated JWT
    API-->>U: New session context
```

### Validation Requirements

Before switching, the API verifies:

* The organization exists.
* The organization is active.
* The user has an active membership.
* The membership is not suspended.
* The requested tenant is available to the user.

The new JWT contains the selected organization and role.

---

## Security Architecture

### Security Timestamp

JWTs are normally valid until expiration.

To invalidate existing sessions after a security-sensitive change, the user record contains a timestamp such as:

```text id="t5q2c4"
securityChangedAt
```

When validating a token:

```text id="ih1z0f"
if tokenIssuedAt < securityChangedAt:
    reject token
```

This supports session revocation after:

* Password changes
* 2FA reset
* Account recovery
* Administrative security action
* Suspicious activity
* Permission-sensitive account changes

### Brute-Force Protection

The API should track and limit:

* Failed login attempts
* Failed 2FA attempts
* Password reset requests
* Email verification requests
* Recovery-code attempts
* Sensitive administrative operations

Possible responses include:

* Temporary account lock
* Request throttling
* Security audit event
* Administrator notification
* User notification

### Sensitive Data Protection

Sensitive values should not be stored in plain text.

Examples include:

* Passwords
* SMTP passwords
* API secrets
* Integration tokens
* Recovery codes
* License keys

Recommended controls:

* Strong password hashing
* Application-level encryption
* Environment-managed encryption keys
* Secret rotation
* Restricted logging
* Masked API responses

---

## Two-Factor Authentication

The API supports TOTP-based two-factor authentication.

### Setup Flow

```mermaid id="v691tc"
sequenceDiagram
    participant U as User
    participant API as API
    participant DB as Database
    participant App as Authenticator App

    U->>API: Start 2FA setup
    API->>API: Generate TOTP secret
    API-->>U: QR code and setup token
    U->>App: Scan QR code
    App-->>U: Generate code
    U->>API: Submit verification code
    API->>API: Verify code
    API->>DB: Store encrypted secret and enable 2FA
    API-->>U: 2FA enabled
```

### Supported Capabilities

* TOTP secret generation
* QR-code enrollment
* Setup verification
* Login challenge
* Enable and disable
* Audit logging
* Mandatory 2FA policy
* Grace-period enforcement
* Recovery requests

### Mandatory 2FA Policy

A platform-wide policy may define:

```text id="7brq4o"
mandatory2faEnabled
gracePeriodDays
policyUpdatedAt
```

When mandatory 2FA is enabled, users without 2FA may be allowed temporary access during the configured grace period.

After the grace period expires, protected application routes are blocked until setup is completed.

---

## Email and Background Jobs

The API uses an email queue for asynchronous delivery.

### Why Use a Queue?

Sending email directly inside an API request introduces several problems:

* Slow responses
* SMTP timeouts
* Temporary provider failures
* Difficult retries
* Limited visibility
* Duplicate-send risk

The queue separates message creation from delivery.

### Email Queue Record

```text id="6y8exp"
id
organizationId
smtpScope
recipient
subject
templateKey
templateData
status
scheduledAt
attemptCount
lastError
sentAt
createdAt
updatedAt
```

Example statuses:

```text id="c2xxjd"
PENDING
PROCESSING
SENT
FAILED
CANCELLED
```

### Processing Flow

```mermaid id="eo3tj8"
flowchart TD
    Event[Application Event] --> Queue[Create Queue Record]
    Queue --> Worker[Background Worker]
    Worker --> Lock[Claim Pending Record]
    Lock --> Resolve[Resolve Template and SMTP]
    Resolve --> Send[Send Email]
    Send --> Result{Successful?}

    Result -->|Yes| Sent[Mark as SENT]
    Result -->|No| Retry{Attempts Remaining?}

    Retry -->|Yes| Reschedule[Schedule Retry]
    Retry -->|No| Failed[Mark as FAILED]
```

### SMTP Scopes

The queue may support:

```text id="a3j6zk"
PLATFORM
ORGANIZATION
```

Platform SMTP is used for core account and security messages.

Organization SMTP is used for tenant-branded operational communication.

---

## Audit Logging

Audit logs provide a traceable history of sensitive activity.

### Example Audit Events

```text id="nyebgm"
USER_LOGIN_SUCCESS
USER_LOGIN_FAILED
USER_LOGOUT
PASSWORD_CHANGED
PASSWORD_RESET_REQUESTED
PASSWORD_RESET_COMPLETED
PASSWORD_RESET_FAILED
TWO_FACTOR_SETUP_STARTED
TWO_FACTOR_ENABLED
TWO_FACTOR_DISABLED
TWO_FACTOR_CHALLENGE_FAILED
ORGANIZATION_CREATED
ORGANIZATION_UPDATED
MEMBER_INVITED
MEMBER_ROLE_CHANGED
MEMBER_REMOVED
PLATFORM_ROLE_UPDATED
SECURITY_POLICY_UPDATED
SMTP_CONFIGURATION_UPDATED
```

### Audit Log Fields

```text id="w1ke5j"
id
eventType
actorUserId
targetUserId
organizationId
ipAddress
userAgent
metadata
createdAt
```

Audit records should avoid storing:

* Plain-text passwords
* Authentication secrets
* Full access tokens
* Unmasked API keys
* Sensitive personal data not required for investigation

---

## API Structure

A modular NestJS structure may look like:

```text id="zbaoo8"
apps/api/src/
├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── guards/
│   ├── strategies/
│   └── dto/
├── users/
├── organizations/
├── memberships/
├── platform-users/
├── roles-permissions/
├── two-factor/
├── security/
├── audit-logs/
├── email/
│   ├── email-queue/
│   ├── email-templates/
│   ├── smtp/
│   └── delivery-logs/
├── settings/
├── common/
│   ├── decorators/
│   ├── guards/
│   ├── filters/
│   ├── interceptors/
│   └── middleware/
├── prisma/
├── app.module.ts
└── main.ts
```

### Module Responsibilities

| Module                | Responsibility                               |
| --------------------- | -------------------------------------------- |
| Auth                  | Login, token generation, verification        |
| Users                 | User account management                      |
| Organizations         | Tenant lifecycle                             |
| Memberships           | Organization-user relationships              |
| Platform Users        | Internal SaaS users                          |
| Roles and Permissions | Platform authorization                       |
| Two Factor            | TOTP setup and verification                  |
| Security              | Rate limits and security policies            |
| Audit Logs            | Security and administrative events           |
| Email Queue           | Asynchronous outbound messages               |
| SMTP                  | Platform and organization mail configuration |
| Settings              | Configurable application policies            |
| Prisma                | Database access                              |

---

## Example API Endpoints

### Authentication

```text id="fqy2q7"
POST   /auth/register
POST   /auth/login
POST   /auth/login/2fa
POST   /auth/verify-email
POST   /auth/forgot-password
POST   /auth/reset-password
POST   /auth/refresh
POST   /auth/logout
```

### User Profile

```text id="1phtmh"
GET    /users/me
PATCH  /users/me
PATCH  /users/me/password
GET    /users/me/security
```

### Two-Factor Authentication

```text id="t7as2j"
POST   /two-factor/setup
POST   /two-factor/verify
POST   /two-factor/disable
GET    /two-factor/status
POST   /two-factor/recovery-request
```

### Organizations

```text id="7puwht"
GET    /organizations
POST   /organizations
GET    /organizations/:id
PATCH  /organizations/:id
DELETE /organizations/:id
POST   /organizations/:id/switch
```

### Organization Members

```text id="4t6rk4"
GET    /organizations/:id/members
POST   /organizations/:id/members
PATCH  /organizations/:id/members/:membershipId
DELETE /organizations/:id/members/:membershipId
```

### Platform Administration

```text id="lv0t0e"
GET    /admin/platform-users
POST   /admin/platform-users
PATCH  /admin/platform-users/:id
GET    /admin/roles
PATCH  /admin/roles/:id/permissions
GET    /admin/security/audit-logs
GET    /admin/organizations
GET    /admin/organizations/:id
```

### Email Administration

```text id="wdkbgj"
GET    /email-queue
POST   /email-queue/:id/retry
POST   /email-queue/:id/cancel
POST   /email-queue/process-due
GET    /email-delivery-logs
GET    /email-delivery-logs/export
```

---

## Request Lifecycle

```mermaid id="ubrp50"
flowchart TD
    Request[Incoming Request] --> Validation[Validate DTO]
    Validation --> Auth{Public Route?}

    Auth -->|Yes| Controller[Controller]
    Auth -->|No| JWT[JWT Guard]

    JWT --> Account[Account Status Check]
    Account --> Security[Security Timestamp Check]
    Security --> Policy[2FA Policy Check]
    Policy --> Tenant[Tenant Membership Check]
    Tenant --> Permission[Role or Permission Check]
    Permission --> Controller

    Controller --> Service[Application Service]
    Service --> Database[(PostgreSQL)]
    Service --> Audit[Create Audit Event]
    Service --> Queue[Queue Background Work]

    Database --> Response[API Response]
```

---

## Database Design

### Recommended Constraints

The database should enforce important invariants.

Examples:

```text id="iq0glf"
UNIQUE user.email

UNIQUE organization.slug

UNIQUE organizationMembership:
    userId + organizationId

UNIQUE platformUserProfile.userId

INDEX tenantResource.organizationId

INDEX auditLog.organizationId + createdAt

INDEX emailQueue.status + scheduledAt
```

### Soft Delete Considerations

Soft deletion may be useful for:

* Organizations
* Memberships
* User accounts
* Tenant resources
* Email templates

However, soft deletion must be consistently applied to all queries to avoid unintentionally exposing inactive records.

### Transactions

Use database transactions for workflows that modify multiple dependent records.

Examples:

* Creating an organization and owner membership
* Transferring organization ownership
* Accepting an invitation
* Enabling 2FA
* Updating permissions
* Completing password recovery

---

## Environment Configuration

Example environment variables:

```bash id="xbz6zj"
NODE_ENV=development
PORT=4000

DATABASE_URL=postgresql://user:password@localhost:5432/saas_api

JWT_ACCESS_SECRET=replace-with-secure-secret
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_SECRET=replace-with-secure-secret
JWT_REFRESH_EXPIRES_IN=30d

APP_ENCRYPTION_KEY=replace-with-strong-encryption-key

FRONTEND_URL=http://localhost:3000

SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=example-user
SMTP_PASSWORD=example-password
SMTP_FROM_EMAIL=noreply@example.com

PASSWORD_RESET_EXPIRY_MINUTES=30
EMAIL_VERIFICATION_EXPIRY_HOURS=24

LOGIN_MAX_ATTEMPTS=5
LOGIN_LOCKOUT_MINUTES=15

TWO_FACTOR_ISSUER=MultiTenantSaaS
```

Never commit real secrets to source control.

Use separate credentials for development, staging, and production.

---

## Local Development

### Prerequisites

* Node.js 18 or later
* pnpm
* PostgreSQL
* Docker and Docker Compose

### Install Dependencies

```bash id="vu4wmz"
pnpm install
```

### Start PostgreSQL with Docker

```bash id="w8o53e"
docker compose up -d postgres
```

### Generate Prisma Client

```bash id="a9kt1l"
pnpm --filter api prisma generate
```

### Apply Database Migrations

```bash id="wcvqqb"
pnpm --filter api prisma migrate dev
```

### Start the API

```bash id="f4l2tk"
pnpm --filter api start:dev
```

The API will typically be available at:

```text id="0af64n"
http://localhost:4000
```

---

## Database Migrations

### Create a Migration

```bash id="om8s7o"
pnpm --filter api prisma migrate dev --name migration_name
```

### Apply Production Migrations

```bash id="p7l1vt"
pnpm --filter api prisma migrate deploy
```

### Generate Prisma Client

```bash id="10bt8g"
pnpm --filter api prisma generate
```

### Open Prisma Studio

```bash id="7wmt0r"
pnpm --filter api prisma studio
```

Production migrations should run before newly deployed application instances begin serving traffic.

---

## Testing Strategy

### Unit Tests

Unit tests should cover:

* Authentication services
* Password policy
* JWT validation
* 2FA verification
* Permission evaluation
* Tenant-context resolution
* Email-template rendering

### Integration Tests

Integration tests should verify:

* Organization creation
* Membership assignment
* Active organization switching
* Tenant-scoped CRUD operations
* Platform permission enforcement
* Password reset
* 2FA setup and login
* Email queue retry behavior

### Tenant Isolation Tests

Tenant isolation requires dedicated negative tests.

Example scenarios:

* User from Organization A cannot read Organization B resources.
* User from Organization A cannot update Organization B resources.
* User cannot switch to an organization without membership.
* Tenant administrator cannot use platform endpoints.
* Platform support role cannot perform unrestricted super-administrator actions.
* Background jobs cannot process another tenant’s resource using the wrong context.

### End-to-End Tests

End-to-end tests should cover complete workflows such as:

```text id="s65au1"
Register
→ Verify email
→ Login
→ Create organization
→ Invite member
→ Accept invitation
→ Switch organization
→ Create tenant resource
→ Verify cross-tenant isolation
```

---

## Deployment

The API can be deployed using Docker.

```mermaid id="t9jnvw"
flowchart TB
    Internet[Internet] --> Proxy[NGINX / Load Balancer]

    Proxy --> API1[API Container 1]
    Proxy --> API2[API Container 2]

    API1 --> DB[(PostgreSQL)]
    API2 --> DB

    API1 --> Queue[(Redis or Database Queue)]
    API2 --> Queue

    Worker1[Background Worker] --> Queue
    Worker2[Background Worker] --> Queue

    Worker1 --> SMTP[SMTP Provider]
    Worker2 --> SMTP
```

### Recommended Production Services

* NGINX or cloud load balancer
* Multiple API containers
* Managed PostgreSQL
* Redis
* Dedicated background worker
* Centralized logs
* Error tracking
* Metrics and alerting
* Automated backups
* Secret manager
* Container registry

### CI/CD Workflow

```text id="rjuov9"
Git Push
   |
   v
Run Tests and Linting
   |
   v
Build Docker Image
   |
   v
Push Versioned Image
   |
   v
Deploy to Target Environment
   |
   v
Apply Prisma Migrations
   |
   v
Run Health Check
   |
   v
Complete or Roll Back
```

Use immutable release tags such as:

```text id="5cpfmz"
api:2026.07.13-1
api:git-commit-sha
```

Avoid relying exclusively on the `latest` tag.

---

## Scalability

### API Scaling

The API should remain stateless so multiple instances can run behind a load balancer.

State that must be shared externally includes:

* Database records
* Rate-limit counters
* Background job queues
* Revoked-session information, if used
* Distributed locks
* Cache entries

### Background Workers

Long-running tasks should be moved outside HTTP requests.

Examples:

* Email delivery
* CSV generation
* Report generation
* Data cleanup
* Webhook delivery
* Subscription synchronization
* Tenant usage calculations

### Database Scaling

Potential improvements include:

* Query optimization
* Proper indexing
* Connection pooling
* Read replicas
* Table partitioning for large audit logs
* Archival policies
* Managed database services
* Point-in-time recovery

### Tenant Growth

For most SaaS products, a shared database with tenant-scoped rows is sufficient.

Larger tenants may later require:

* Dedicated database
* Dedicated schema
* Data-region isolation
* Custom retention
* Tenant-specific encryption keys

The application layer should avoid assumptions that make future tenant migration impossible.

---

## Production Security Checklist

* Use HTTPS everywhere.
* Store secrets in a secure secret manager.
* Rotate JWT and encryption keys.
* Use strong password hashing.
* Encrypt sensitive integration credentials.
* Apply strict tenant scoping.
* Validate every request DTO.
* Rate-limit authentication endpoints.
* Enable mandatory 2FA for privileged users.
* Protect administrative endpoints.
* Record security audit events.
* Restrict CORS origins.
* Configure secure HTTP headers.
* Use short-lived access tokens.
* Implement refresh-token rotation.
* Scan dependencies.
* Scan container images.
* Back up the database.
* Test backup restoration.
* Avoid logging tokens or passwords.
* Separate development and production infrastructure.
* Monitor repeated authentication failures.
* Review platform permissions regularly.

---

## Future Enhancements

### Authentication and Security

* Passkeys and WebAuthn
* Backup recovery codes
* Device management
* Active session management
* IP-aware login alerts
* Risk-based authentication
* Single sign-on
* SAML
* OAuth providers
* Enterprise identity federation

### Tenant Management

* Organization invitations
* Ownership transfer
* Custom organization roles
* Department or team hierarchy
* Tenant-level feature flags
* White-label domains
* Tenant-specific policies

### Billing

* Subscription plans
* Usage metering
* Feature entitlements
* Trial periods
* Invoice history
* Payment webhooks
* Seat-based billing
* Resource-based billing

### Infrastructure

* Redis-backed queues
* Distributed rate limiting
* Kubernetes deployment
* Multi-region architecture
* Read replicas
* Centralized observability
* Automated disaster recovery
* Feature-flag service

---

## Key Engineering Decisions

### Shared User Identity

A single user identity can belong to several organizations without creating duplicate accounts.

### Membership-Based Tenant Access

Tenant roles belong to membership records rather than directly to users.

### Separate Platform Authorization

Internal SaaS administration is separated from customer organization access.

### Backend-Enforced Isolation

Every tenant-owned resource is scoped by organization in the backend.

### Security-Based JWT Revocation

Security timestamps allow existing tokens to be invalidated after account changes.

### Fine-Grained Platform Permissions

Internal roles are assigned explicit permissions instead of universal administrator access.

### Asynchronous Email Processing

Email delivery is separated from API requests for reliability and performance.

### Audit-First Security Design

Sensitive actions are recorded for investigation and compliance.

### Stateless API Design

Application instances avoid local session state, enabling horizontal scaling.

---

## Repository Scope

This repository may contain:

* Backend architecture examples
* NestJS module structure
* Prisma data models
* Authentication patterns
* Authorization guards
* Tenant-scoped service patterns
* Security-policy examples
* Queue-processing patterns
* Deployment documentation

It should not contain:

* Production secrets
* Real customer data
* Private encryption keys
* SMTP credentials
* Access tokens
* Production database backups
* Proprietary integration credentials

---

## Author

**Ramesh Singh**
Senior Software Developer and Solutions Architect

Core experience:

* Multi-tenant SaaS architecture
* Backend and REST API development
* Node.js, NestJS, PHP, and TypeScript
* PostgreSQL and database design
* Authentication and authorization
* WordPress, Drupal, Shopify, and Magento
* Docker, NGINX, and Linux
* AWS, Azure, and Google Cloud
* Security and performance engineering

LinkedIn:
https://www.linkedin.com/in/ramesh-singh-tech-lead/

---

## Disclaimer

This project is an architectural case study and reusable backend reference.

The exact schema, modules, security controls, deployment strategy, and business rules should be adapted to the requirements of the product using it.

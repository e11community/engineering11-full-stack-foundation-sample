# Auth Service

Multi-tenant authentication and authorization infrastructure with enterprise-grade token management.

## What It Does

The Auth Service handles authentication, session management, and authorization for multi-tenant SaaS platforms. It automates the operational complexity of managing identity at scale — batch token updates, multi-tenant isolation, automated provisioning, and event-driven auditing.

## Key Capabilities

| Capability                       | Description                                                                            |
| -------------------------------- | -------------------------------------------------------------------------------------- |
| **Large-Scale Token Management** | Batch update JWT custom claims for 150+ users when products/roles change               |
| **Multi-Tenant Provisioning**    | Automatically provision Identity Platform tenants + reCAPTCHA keys per tenant |
| **Token Lifecycle Management**   | Automatic token revocation on user deactivation/reactivation/deletion                  |
| **Batch Processing**             | Background jobs handle bulk custom claims updates across customer bases                |
| **Event-Driven Architecture**    | Publishes auth events (user created/deleted, claims updated) for audit trails          |
| **Email Action Handling**        | Password reset and email verification with custom link generation                      |
| **Claims Validation**            | Enforces 1000-byte claims limit to prevent OAuth token bloat                           |
| **Task Queue Integration**       | Async processing for email verification and bulk operations                            |

## How It Fits Together

```
┌───────────────────────────────────────────────────────────┐
│                   Auth Service Core                       │
│                                                           │
│  ┌─────────────────┐      ┌──────────────────┐            │
│  │  AuthFactory    │      │  Bootstrap       │            │
│  │ (Tenant Router) │      │  (Provisioning)  │            │
│  └────────┬────────┘      └────────┬─────────┘            │
│           │                        │                      │
│           ▼                        ▼                      │
│  ┌─────────────────────────────────────────────┐          │
│  │   Identity Platform (Multi-Tenant) │        │          |
│  └─────────────────┬───────────────────────────┘          │
│                    │                                      │
└────────────────────┼─────────────────────────────────────-┘
                     │
        ┌────────────┼────────────┐
        │                         │
        ▼                         ▼
┌──────────────┐          ┌─────────────
│ Task Queues  │          │User Service │
│(Custom Claims│          │(Profile Sync│
│ Updates)     │          │             │ 
└──────────────┘          └─────────────┘
```

## Heavy-Lifting Features

### 1. Automated Token Management at Scale

When product catalogs or roles change, the auth service automatically updates JWT tokens for affected users:

```typescript
UpdateCustomClaimsJob:
  - Fetches 150 users per batch
  - Updates custom claims (products, roles, customerKey)
  - Updates jwtUpdatedAt timestamp
  - Integrates with customer service for product data
  - Handles errors gracefully without blocking
```

**What customers avoid:**

- Writing batch processing logic
- Managing API rate limits
- Coordinating with customer/product services
- Handling partial failures in batch updates

### 2. Multi-Tenant Infrastructure Provisioning

Each tenant gets isolated auth infrastructure automatically:

```typescript
AuthBootstrapService.createForHost(hostname):
  - Creates Identity Platform tenant
  - Provisions reCAPTCHA Enterprise key
  - Configures domain allowlists
  - Returns tenant ID + site key
  - Handles cascading deletion on tenant removal
```

**What customers avoid:**

- Manual tenant creation
- reCAPTCHA key provisioning and rotation
- Domain whitelist management
- Tenant-to-domain routing logic

### 3. Token Revocation & Session Management

Automatic token lifecycle handling:

- **User Deactivation**: Revokes refresh tokens + disables auth account
- **User Reactivation**: Revokes old tokens + re-enables auth account
- **User Deletion**: Revokes tokens before deletion
- **Claims Updates**: Publishes events to SSE channels for real-time client updates

### 4. Email Action Automation

Complete password reset and email verification flows:

```typescript
EmailActionService handles:
  - OOB (Out-of-Band) code generation
  - Custom link formatting for tenant domains
  - Display name lookup from email
  - Rate limit error handling
  - Integration with notifications service
```

**What customers avoid:**

- Building custom email verification systems
- Generating secure password reset links
- Managing email delivery infrastructure
- Handlind rate limits

### 5. Event-Driven Auditing

Automatic event publishing for all auth operations:

- `AUTH_USER_ADDED` - User created (Cloud Functions)
- `AUTH_USER_DELETED` - User removed
- `AUTH_USER_DEACTIVATED` - User disabled
- `AUTH_CLAIMS_SET` - Custom claims updated (with SSE broadcast)
- `AUTH_BOOTSTRAP_UPDATED` - Tenant config changed
- `AUTH_BOOTSTRAP_DELETED` - Tenant removed

Events trigger downstream services (billing, analytics, user profiles) without explicit coordination.

### 6. Claims Validation & Security

Built-in security guardrails:

- **Size Enforcement**: Max 1000 bytes per custom claims object (prevents OAuth token bloat)
- **Structure Validation**: Enforces required fields (`appUserId`, `customerKey`, `roles`, `products`)
- **Type Safety**: TypeScript guards validate claim shapes
- **reCAPTCHA Integration**: Enterprise-grade bot protection per tenant

### 7. Task Queue Integration

Async processing for scalability:

- Email verification requests queued via `AUTH_SEND_EMAIL_VERIFICATION`
- Bulk custom claims updates processed in background
- Decoupled from REST API response times
- Automatic retry on transient failures

## REST API Endpoints

### Public Endpoints

```
PUT  /v2/password-reset         - Self-service password reset
PUT  /v2/verify-email           - Email verification
PUT  /v2/verify-email/resend    - Resend verification (bypass throttle)
```

### Internal Endpoints

```
POST /internal/update-custom-claims  - Bulk update user claims
```

## Common Use Cases

- **Product catalog changes**: Automatically refresh JWT tokens for 1000s of users when product assignments change
- **Tenant onboarding**: Provision complete auth infrastructure (tenant + reCAPTCHA) in one API call
- **User deactivation**: Revoke all active sessions and disable login with single command
- **Custom claims sync**: Batch update user roles/products when organization structure changes
- **Email verification**: Send verification emails with custom branded links
- **Password reset**: Self-service password reset with tenant-specific redirect URLs

## What Customers Don't Have to Build

- Multi-tenant Identity Platform management
- Batch token refresh systems when product catalogs change
- Token revocation during user lifecycle events
- reCAPTCHA key provisioning and lifecycle management
- Email verification and password reset link generation
- Event publishing infrastructure for audit trails
- Custom claims validation and size enforcement
- Task queue integration for async operations
- Domain-based tenant routing
- Rate limit handling

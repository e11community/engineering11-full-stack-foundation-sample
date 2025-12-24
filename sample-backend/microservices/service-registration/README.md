# Registration Service

Complete user and organization onboarding with invitation workflows, multi-tenant provisioning, and account lifecycle management.

## What It Does

The Registration Service orchestrates the entire journey from invitation to active account, handling multi-step registration flows, email verification, account provisioning, fraud detection, and coordinated account deactivation/reactivation across the platform.

## Key Capabilities

| Capability                    | Description                                                                                                 |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Multi-Path Registration**   | Consumer self-service, business account creation, employee invitations, and platform admin provisioning     |
| **Invitation Management**     | Secure token-based invitations with expiration, resend, bulk operations, and domain-based self-registration |
| **Account Provisioning**      | Multi-tenant infrastructure with bootstrap creation, customer hierarchy, and client account setup           |
| **Lifecycle Orchestration**   | Coordinated deactivation and time-windowed reactivation with cascade operations                             |
| **Fraud Prevention**          | Email and IP validation with fraud scoring and analytics logging                                            |
| **Identity Generation**       | Automatic identicon creation for consumer users                                                             |
| **Event-Driven Architecture** | Background task orchestration with saga patterns for cross-service coordination                             |

## How It Fits Together

```
┌─────────────────────┐
│   Registration      │
│     Service         │
└──────────┬──────────┘
           │
    ┌──────┼───────────────┬──────────────┬──────────┐
    ▼      ▼               ▼              ▼          ▼
┌──────┐ ┌────┐ ┌──────────────┐ ┌──────────┐ ┌─────────┐
│ Auth │ │User│ │   Customer   │ │ Product  │ │Security │
└──────┘ └────┘ └──────────────┘ └──────────┘ └─────────┘
```

Registration coordinates across Auth, User, Customer, Product, and Security services to create fully configured accounts with proper roles, claims, and subscriptions.

## Heavy-Lifting Features

### 1. Multi-Path Registration Flows

Three distinct registration paths with coordinated account creation:

```typescript
Consumer Registration:
  - Creates auth user with password
  - Generates new customer record (Consumer type)
  - Creates user document with preferences
  - Sets custom claims with roles and products
  - Provisions default product subscriptions
  - Sends to finalize task (identicon generation, email verification)
  - Returns custom auth token

Business Registration:
  - Creates auth user with company details
  - Generates customer (Business type) and client records
  - Creates user with account creator flag
  - Sets custom claims with B2B roles
  - Provisions default product subscriptions
  - Sends to finalize task (email verification, analytics)
  - Returns custom auth token

Employee Registration:
  - Validates invitation token (15-day expiration)
  - Verifies customer is active
  - Creates auth user from invitation details
  - Creates user with pre-assigned roles
  - Sets custom claims from invitation
  - Deletes invitation after use
  - Sends to finalize task
  - Returns custom auth token
```

**What customers avoid:**

- Writing multi-path registration logic
- Coordinating account creation across services
- Managing customer type differences
- Handling invitation token validation
- Building rollback on failures
- Tracking registration completion

### 2. Secure Invitation System with Expiration

Comprehensive invitation lifecycle with multiple entry points:

```typescript
Invitation creation:
  - Generates cryptographic secret token
  - Sets deterministic ID (customerKey_email)
  - Validates no duplicate invitations
  - Checks user doesn't already exist
  - Stores invitation with 15-day expiration
  - Dispatches notification email immediately
  - Supports additional user attributes and custom claims

Invitation types:
  - Additional: Standard employee invitations
  - Provisioned: Admin-created accounts with elevated setup
  - Domain: Self-serve based on email domain matching

Invitation validation:
  - Secret token lookup with collision handling
  - Expiration check (15 days from last update)
  - Customer active status verification
  - Email uniqueness validation

Resend capabilities:
  - Updates expiration date to extend validity
  - Re-dispatches notification email
  - Preserves original invitation details
  - Supports custom host override
```

**What customers avoid:**

- Building secure token generation
- Implementing expiration logic
- Managing invitation states
- Writing duplicate detection
- Creating notification dispatch
- Handling invitation resend with date extension

### 3. Multi-Tenant Provisioning Infrastructure

Complete tenant and customer hierarchy creation:

```typescript
Tenant Business Provisioning:
  - Creates authentication tenant with custom domain
  - Generates tenant bootstrap configuration
  - Sets up theme, brand, and notification settings
  - Creates customer record (tenant customer flag)
  - Provisions default product subscriptions
  - Stores bootstrap with file storage integration
  - Sends invitation to first admin user
  - Returns bootstrap view model

Client Business Provisioning:
  - Links to existing customer client record
  - Validates parent-child relationship
  - Creates customer account for client
  - Updates client record with customer reference
  - Sends invitation to client admin
  - Provisions default products
  - Race condition handling (invite before customer created event)

Bootstrap management:
  - Host-derived settings (auth API keys, project IDs)
  - Additional host aliases
  - Customer key association
  - Type differentiation (Business vs Consumer)
  - File storage bucket configuration
```

**What customers avoid:**

- Building multi-tenant infrastructure
- Writing bootstrap configuration logic
- Managing customer hierarchies
- Implementing domain-based tenant isolation
- Coordinating tenant creation sequence
- Handling parent-child provisioning

### 4. Domain-Based Self-Registration

Automatic invitation for email domain matches:

```typescript
Self-invitation flow:
  - Extracts domain from email address
  - Looks up customer by registration domain
  - Validates customer allows domain registration
  - Verifies customer is not disabled
  - Creates invitation with customer's default roles
  - Uses customer's registration settings (host, roles)
  - Dispatches domain-specific notification
  - Provides user-friendly error messages

Email lookup:
  - Domain extraction and validation
  - Customer query by registration domain
  - Registration settings verification
  - Bootstrap tenant lookup
  - Disabled state checking
  - Custom error differentiation

Resend self-invitation:
  - Validates email domain again
  - Looks up existing invitation
  - Updates expiration date
  - Re-dispatches notification
  - Throttle and recaptcha protected
```

**What customers avoid:**

- Building domain matching logic
- Implementing customer domain lookup
- Writing registration settings evaluation
- Managing self-service permissions
- Creating domain validation rules
- Handling custom notification routing

### 5. Account Lifecycle Management with Time-Windowed Recovery

Coordinated deactivation and reactivation:

```typescript
Deactivation:
  - Validates requestor authorization (customer or parent)
  - Prevents tenant customers from self-deactivation
  - Sets 21-day deactivation window
  - Immediately deactivates requesting user (prevents race conditions)
  - Disables customer account
  - Publishes deactivation event for cascade operations
  - Stores deactivation metadata
  - Returns window duration

Reactivation availability:
  - Calculates reactivation deadline (endDate + 21 days)
  - Checks if current time is before deadline
  - Returns availability status with dates
  - Provides deadline for UI display

Reactivation:
  - Validates within 21-day window
  - Re-enables customer account
  - Publishes reactivation event
  - Cascade operations handled by tasks

Authorization:
  - Customer can deactivate themselves
  - Parent can deactivate child customers
  - Platform admins bypass checks
  - Tenant customers cannot self-deactivate
```

**What customers avoid:**

- Building time-windowed recovery logic
- Implementing parent-child authorization
- Managing cascade deactivation events
- Writing reactivation deadline calculations
- Coordinating cross-service deactivation
- Protecting privileged accounts

### 6. Background Finalization with Fraud Detection

Async post-registration operations with analytics:

```typescript
Business finalization:
  - Requests email verification from auth service
  - Runs email fraud validation (IPQS)
  - Runs IP fraud validation with geolocation
  - Logs registration to BigQuery with fraud scores
  - Publishes business registration completed event
  - Includes geographic data (city, region, lat/long)
  - Includes tenant host information

Consumer finalization:
  - Generates identicon SVG and PNG (300x300)
  - Uploads to cloud storage bucket
  - Makes identicons publicly accessible
  - Upserts customer record (idempotency)
  - Requests email verification
  - Runs fraud validation (email + IP)
  - Logs to BigQuery analytics
  - Publishes consumer registration completed event

Fraud detection:
  - Email validation with fraud scoring
  - IP validation with geolocation and fraud scoring
  - Parallel execution for speed
  - Graceful error handling (silent logging)
  - Enriches events with fraud scores
```

**What customers avoid:**

- Building fraud detection integration
- Implementing identicon generation
- Writing cloud storage upload logic
- Managing email verification dispatch
- Creating analytics logging infrastructure
- Coordinating async finalization tasks

### 7. Event-Driven Saga Orchestration

Automatic cross-service coordination with sagas:

```typescript
User update saga:
  - Deactivate auth user when user.disabled becomes true
  - Reactivate auth user when user.disabled becomes false
  - Update auth custom claims on user role changes
  - Rename auth user email on user.email changes
  - Multiple saga handlers running in parallel

User deletion saga:
  - Delete auth user when user deleted
  - Cascades from user service events

Customer lifecycle sagas:
  - CustomerDeactivatedEvent triggers user/auth deactivation
  - CustomerReactivatedEvent triggers user/auth reactivation
  - Schedules delayed invitation deletion (21 days)
  - Bootstrap update/delete propagation

Saga pattern:
  - Observable streams with RxJS
  - Event filtering and type checking
  - Command dispatch to handlers
  - Idempotent operation design
```

**What customers avoid:**

- Building saga pattern infrastructure
- Writing event stream processing
- Implementing cross-service coordination
- Managing command/event dispatch
- Creating idempotent handlers
- Coordinating async workflows

### 8. Delayed Task Scheduling

Time-based operations with scheduling:

```typescript
Delayed invitation deletion:
  - Queries all invitations for deactivated customer
  - Schedules deletion for each invite
  - Scheduled time: deactivationWindowEnd (21 days)
  - Includes customerKey for conditional delete
  - Only deletes if customer still disabled
  - Prevents deletion if customer reactivated

Bootstrap change propagation:
  - Bootstrap updated events routed to tasks
  - Bootstrap deleted cascade operations
  - Async processing prevents blocking

Task routing:
  - Cloud Tasks queue integration
  - Scheduled execution at specific timestamps
  - Endpoint and queue configuration
  - Service ID routing
```

**What customers avoid:**

- Building delayed task infrastructure
- Implementing scheduled execution
- Writing conditional deletion logic
- Managing task queue integration
- Creating time-based cascade operations
- Coordinating scheduled vs immediate tasks

### 9. Platform Admin Provisioning

Special path for platform administrators:

```typescript
Platform admin creation:
  - Creates auth user with temporary password
  - Generates secret token for password reset
  - Sets email as verified immediately
  - Creates platform admin user (no customer)
  - Sets custom claims with platform admin roles
  - No customer or subscription linkage
  - Returns temporary password token

User type handling:
  - UserType.PlatformAdmin (no customer association)
  - UserType.Business (linked to customer)
  - UserType.Consumer (linked to customer)
```

**What customers avoid:**

- Building admin provisioning flow
- Implementing temporary password logic
- Managing platform-level users
- Writing no-customer user creation
- Creating admin-specific claims

### 10. Invitation Bulk Operations

Batch invitation management:

```typescript
Bulk tenant invitation:
  - Accepts array of invitation requests
  - Adds standard tenant admin roles automatically
  - Processes invitations in parallel
  - Returns array of created invitations
  - Merges additional roles with defaults

Standard tenant roles:
  - DEFAULT_ROLE_B2B
  - TenantAdmin
  - ClientEditor
  - ClientViewer
  - Plus any additional requested roles

Query operations:
  - Get all invitations for customer
  - Filter by invitation type
  - Exclude secret token from responses
  - Support provisional and additional types
```

**What customers avoid:**

- Building bulk operation handling
- Implementing parallel invitation creation
- Managing role merging logic
- Writing type-based filtering
- Creating sensitive data exclusion

### 11. Search Key Generation for Client Isolation

Secure search API key generation with filtering:

```typescript
Client user search:
  - Validates customer client relationship
  - Checks parent-child customer linkage
  - Generates secured Algolia API key
  - Restricts to user index only
  - Filters by client customer key
  - Returns app ID and secured key package

Validation:
  - Customer client existence check
  - Client has linked customer account
  - Requestor authorized for parent customer
  - Prevents cross-customer data access
```

**What customers avoid:**

- Building search key generation
- Implementing client isolation logic
- Writing secured API key creation
- Managing search index restrictions
- Creating authorization validation

### 12. Registration Code Validation

Pre-registration validation endpoint:

```typescript
Validation flow:
  - Looks up invitation by secret token
  - Checks expiration (15 days)
  - Returns email, first/last name, customer key
  - Allows UI to pre-fill registration form
  - Does not consume invitation
  - Public endpoint with auth guard
```

**What customers avoid:**

- Building pre-validation logic
- Implementing token lookup without consumption
- Writing expiration checking without deletion
- Creating pre-fill data responses

### 13. Custom Claims Construction

Sophisticated claims building with inheritance:

```typescript
Claims composition:
  - Base claims (appUserId, customerKey, bootstrapTenantKey)
  - Role assignment (from invitation or defaults)
  - Product access (from customer or defaults)
  - Terms acceptance tracking
  - Client ID for client customers

Invitation claims:
  - Supports additionalCustomClaims spread first
  - Override with explicit values
  - Preserves extra properties from invitation
  - Allows per-invitation customization

User attributes:
  - Supports additionalUserAttributes spread first
  - Override with explicit values
  - Allows per-invitation user customization
```

**What customers avoid:**

- Building claims inheritance logic
- Implementing spread-based composition
- Writing override precedence rules
- Managing custom attribute injection

### 14. BigQuery Analytics Integration

Structured registration analytics:

```typescript
Registration logging:
  - User ID and customer key
  - User type (Business/Consumer/PlatformAdmin)
  - Geographic data (IP, city, region, lat/long)
  - Fraud scores (email and IP)
  - Request header information
  - IPQS validation results (full JSON)
  - Registration source tracking
  - Timestamp with timezone

Registration sources:
  - Standard (direct registration)
  - Invitation (via email invitation)
  - Domain (self-service via domain match)
  - Admin (platform admin created)

Data retention:
  - Structured for reporting
  - Queryable by date range
  - Fraud analysis ready
  - Geographic trending
```

**What customers avoid:**

- Building analytics infrastructure
- Implementing BigQuery integration
- Writing structured logging
- Managing data warehouse schema
- Creating fraud reporting queries

### 15. Authorization and Permission System

Fine-grained endpoint protection:

```typescript
Permissions model:
  - ProvisionCustomerTenant (platform admins only)
  - ProvisionConsumerTenant (platform admins only)
  - ProvisionCustomer (create business accounts)
  - ProvisionCustomerClient (create client accounts)
  - DeprovisionCustomer (deactivate accounts)
  - ReactivateCustomer (reactivate within window)
  - ViewClientUsers (read user data)
  - EditClientUsers (modify user data)
  - ViewAllInvites/EditAllInvites (invitation management)

Role-based permissions:
  - TenantAdmin gets full provisioning rights
  - ClientEditor gets client management
  - ClientViewer gets read-only access
  - Platform admins bypass customer checks

Grantable roles:
  - Validates roles can be granted by requestor
  - Checks encompassed role hierarchy
  - Prevents privilege escalation
  - Rejects non-grantable roles
```

**What customers avoid:**

- Building permission hierarchies
- Implementing grantable role validation
- Writing privilege escalation prevention
- Managing role encompassing logic
- Creating endpoint authorization

### 16. Registration Error Handling

Consistent error responses with type differentiation:

```typescript
Error types:
  - EMAIL_EXISTS → User-friendly duplicate message
  - INVITATION_NOT_FOUND → Suggests login or re-request
  - INVITATION_EXPIRED → Prompts for new invitation
  - INVITATION_ALREADY_EXISTS → Check email including spam
  - CUSTOMER_NOT_FOUND/DISABLED → Account status messages
  - TENANT_BOOTSTRAP_NOT_FOUND → Missing tenant configuration
  - DOMAIN_REGISTRATION_DISABLED → Contact admin message
  - REACTIVATION_WINDOW_CLOSED → Cannot reactivate
  - NOT_AUTHORIZED → Authorization failure

Error interceptor:
  - Catches all registration errors
  - Translates to HTTP status codes
  - Provides user-friendly titles
  - Includes debug information
  - Maintains error type for client handling
```

**What customers avoid:**

- Building error translation logic
- Implementing user-friendly messages
- Writing error type differentiation
- Managing HTTP status mapping
- Creating consistent error format

## Common Use Cases

- **Employee onboarding**: HR provisions accounts, sends invitations, employees self-register with pre-assigned roles
- **Multi-tenant SaaS**: Platform admins create tenant infrastructure, tenant admins manage client accounts
- **Domain-based signup**: Employees self-register using company email domain with automatic role assignment
- **Self-service registration**: Consumers create accounts with automatic identity generation and verification
- **Trial account management**: Business accounts with time-limited deactivation and recovery window
- **Fraud monitoring**: Track registration patterns, detect suspicious signups, analyze geographic trends
- **Client account creation**: Service providers create accounts for clients with proper parent-child hierarchy

## What Customers Don't Have to Build

- Multi-path registration flows with coordinated account creation
- Secure invitation system with token generation and expiration
- Multi-tenant provisioning infrastructure with bootstrap management
- Domain-based self-registration with customer lookup
- Account lifecycle management with time-windowed recovery
- Background finalization with fraud detection integration
- Event-driven saga orchestration for cross-service coordination
- Delayed task scheduling with conditional execution
- Platform admin provisioning with temporary passwords
- Bulk invitation operations with role merging
- Search key generation with client isolation
- Registration code validation for pre-filled forms
- Custom claims construction with attribute inheritance
- BigQuery analytics integration with fraud scoring
- Fine-grained permission system with role validation
- Consistent error handling with user-friendly messages
- Identicon generation with cloud storage
- Email verification dispatch and tracking
- Customer hierarchy authorization logic
- Cascade deactivation and reactivation
- Invitation resend with expiration extension
- Rollback on partial registration failures
- Deterministic invitation IDs to prevent duplicates
- Race condition prevention in deactivation flow

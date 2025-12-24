# Customer Service

Multi-tenant organization management with hierarchical client relationships and data isolation.

## What It Does

The Customer Service manages organizational entities in B2B and B2B2C platforms. It handles tenant accounts (customers), their hierarchical client relationships, operational details (departments, locations, contacts), and environment-specific data partitioning. Each customer represents an independent tenant with isolated data, users, and configuration.

## Key Capabilities

| Capability                   | Description                                                         |
| ---------------------------- | ------------------------------------------------------------------- |
| **Customer Management**      | Business and consumer customers with profile data and settings      |
| **Client Hierarchies**       | Nested client relationships with provisioned customer accounts      |
| **Organizational Structure** | Departments, locations, and contacts for both customers and clients |
| **Data Partitioning**        | Environment-specific data segregation with API key management       |
| **Domain Registration**      | Self-serve user registration with domain whitelisting               |
| **Lifecycle Management**     | Activation, deactivation with grace periods, and reactivation       |
| **Search Integration**       | Real-time search indexing with scoped API key generation            |

## How It Fits Together

```
┌─────────────────┐
│    Customer     │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┐
    ▼                 ▼              ▼
┌─────────┐   ┌─────────────┐  ┌──────────┐
│  User   │   │   Access     │  │  Auth    │
│ Service │   │   Service    │  │ Service  │
└─────────┘   └─────────────┘  └──────────┘
```

Customers provide organizational context for users, define access scopes through data partitions, and trigger authentication claim updates.

## Heavy-Lifting Features

### 1. Hierarchical Client Management with Account Provisioning

Multi-level organizational structures with automatic account creation:

```typescript
Customer-Client relationship:
  - Customers have many clients (one-to-many)
  - Clients can have provisioned customer accounts (clientCustomerKey)
  - Clients inherit tenant context from parent customer
  - Client users receive clientId in auth tokens

Operations:
  - Create client with full business profile
  - Optionally provision customer account for client
  - Link departments, locations, and contacts to clients
  - Disable client cascades to provisioned customer account
  - Delete client with cleanup of related entities
```

**What customers avoid:**

- Building multi-tenant hierarchy models
- Managing bidirectional customer-client relationships
- Implementing tenant context propagation
- Writing account provisioning logic
- Coordinating cascading deactivation across hierarchies

### 2. Automatic Location Creation on Customer Onboarding

Seamless initial location setup from customer address:

```typescript
CreateLocationHandler (triggered on CustomerCreatedEvent):
  - Detects Business customer type
  - Extracts address fields from customer record
  - Creates matching location with customer ID
  - Sets customer location as primary headquarters
  - Defaults to US country code if not specified
  - Handles failures gracefully with retry logic
```

**What customers avoid:**

- Writing customer onboarding workflows
- Building location initialization logic
- Managing address data synchronization
- Implementing type-based conditional setup
- Coordinating multi-step onboarding processes

### 3. Data Partition System with API Key Management

Environment-specific data segregation with secure access control:

```typescript
Data Partition features:
  - Create logical data partitions (dev, qa, prod, demo)
  - Partition groups for organizational hierarchy
  - Scoped to customer with strict validation
  - Independent API key management per partition
  - Key rotation without data migration
  - Automatic expiration on partition deletion

API Key operations:
  - Generate admin keys with read/write scopes
  - Create custom keys with granular scopes
  - Rotate keys with automatic expiration
  - TTL support for temporary access
  - Foreign key linking for external systems
  - Secure key delivery (plain text only on creation)
```

**What customers avoid:**

- Building data isolation systems
- Implementing secure API key generation
- Writing key rotation logic
- Managing scope-based access control
- Coordinating key expiration and cleanup
- Building TTL and foreign key tracking

### 4. Domain-Based Self-Service Registration

Controlled user onboarding with domain whitelisting:

```typescript
Registration Settings:
  - Enable/disable domain registration per customer
  - Whitelist registration domains (e.g., @company.com)
  - Auto-assign roles to self-registered users
  - Custom host overrides for registration flow
  - Validation against customer-controlled domains

Flow:
  - User registers with email on whitelisted domain
  - System validates domain against customer settings
  - Automatically grants configured roles
  - Creates user within customer tenant context
  - Applies customer-specific session expiration
```

**What customers avoid:**

- Building domain validation systems
- Implementing whitelist management
- Writing auto-role assignment logic
- Managing tenant-scoped registration
- Creating custom host routing

### 5. Deactivation with Grace Periods and Feedback Collection

Controlled account lifecycle with reactivation support:

```typescript
DeactivateCustomerHandler:
  - Accepts deactivation window end date
  - Collects structured feedback (reason code, text, NPS score)
  - Logs deactivation to audit trail
  - Expires all customer API keys immediately
  - Sets isDisabled flag on customer record
  - Publishes CUSTOMER_DEACTIVATED event
  - Supports reactivation within grace period

Feedback structure:
  - Reason code and custom text
  - Recommendation score (NPS-style)
  - Additional context field
  - Requestor user ID tracking
  - Timestamp logging
```

**What customers avoid:**

- Building deactivation workflow systems
- Implementing grace period logic
- Writing feedback collection models
- Managing API key expiration on deactivation
- Creating audit trail logging
- Building reactivation support

### 6. Custom Claims Synchronization on Product Changes

Automatic authentication token updates across user base:

```typescript
UpdateCustomClaimsHandler (triggered when products change):
  - Detects customer product list updates
  - Queries all users for customer
  - Batch updates custom claims via Auth service
  - Ensures all user tokens reflect new products
  - Handles empty user lists gracefully
  - Retries on failure for reliability

Saga logic:
  - Filters for 'products' field changes in update events
  - Executes command asynchronously
  - Prevents unnecessary updates when products unchanged
```

**What customers avoid:**

- Building token synchronization systems
- Writing user enumeration logic
- Implementing batch claim updates
- Managing change detection for specific fields
- Coordinating async user updates

### 7. Comprehensive Organizational Structure Management

Full operational detail tracking for customers and clients:

```typescript
Departments:
  - Scoped to customer or client
  - Internal code for external system integration
  - Active/inactive status tracking
  - CRUD operations with key validation

Locations:
  - Full address details with country codes
  - HQ designation support
  - Secondary name and internal codes
  - Website and contact info
  - Active/inactive status
  - Optional notes field

Contacts:
  - Typed contacts (Account Owner, Billing, Technical)
  - Optional location association
  - Phone and email details
  - Custom contact types supported
  - Scoped to customer or client
```

**What customers avoid:**

- Building organizational structure models
- Implementing multi-level scoping (customer vs client)
- Writing internal code mapping systems
- Managing location hierarchy
- Creating contact type systems
- Building active/inactive state management

### 8. Real-Time Search Index Synchronization

Automatic search index updates via event-driven sagas:

```typescript
AlgoliaSagas:
  - CUSTOMER_CREATED → Upsert to search index
  - CUSTOMER_UPDATED → Upsert with latest data
  - CUSTOMER_DELETED → Remove from index
  - CUSTOMER_CLIENT_CREATED → Upsert client to index
  - CUSTOMER_CLIENT_UPDATED → Update client in index
  - CUSTOMER_CLIENT_DELETED → Remove client from index

Process:
  - Database trigger publishes event
  - Saga catches event via RxJS stream
  - Fetches current entity state
  - Sends upsert/delete command to search service
  - Maintains eventual consistency
```

**What customers avoid:**

- Building database trigger systems
- Implementing search index sync logic
- Managing eventual consistency
- Writing saga event handlers
- Coordinating multi-entity indexing

### 9. Scoped Search Key Generation

Dynamic API key creation with query restrictions:

```typescript
Search key types:
  - All customers (platform admin only)
  - Business customers only (filtered by type)
  - Consumer customers only (filtered by type)
  - Tenant customers (isTenantCustomer filter)
  - Customer's own clients (filtered by customerKey)

Security features:
  - Restricted to specific indices
  - Query filters embedded in key
  - Scoped to user's claims
  - Permission-gated endpoints
  - Temporary key generation
```

**What customers avoid:**

- Building search security systems
- Implementing query restriction logic
- Writing permission-based key generation
- Managing search index scoping
- Creating type-based filtering

### 10. Event-Driven Architecture with Pub/Sub

Transparent event publishing for all entity lifecycle changes:

```typescript
Database functions publish:
  - CUSTOMER_CREATED → onCreate trigger
  - CUSTOMER_UPDATED → onUpdate trigger (with deactivate/reactivate detection)
  - CUSTOMER_DELETED → onDelete trigger
  - CUSTOMER_CLIENT_CREATED/UPDATED/DELETED → lifecycle triggers
  - CUSTOMER_PARTITION_DELETED → cascade cleanup trigger
  - CUSTOMER_DEPARTMENT_UPDATED → change notification
  - CUSTOMER_LOCATION_UPDATED → change notification

Topic-to-Task routing:
  - Events route to dedicated task queues
  - Async processing via task handlers
  - Saga-based multi-step workflows
  - Idempotent operation handling
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database triggers
- Implementing pub/sub integrations
- Managing async task queues
- Creating event-driven architectures
- Coordinating topic-to-queue routing

### 11. Background Task Orchestration with Sagas

Complex async workflows using CQRS patterns:

```typescript
Customer sagas:
  - Create location on customer creation
  - Delete all partitions when customer deleted
  - Update custom claims when products change
  - Sync search indices on all entity changes

Partition sagas:
  - Delete all customer partitions on customer deletion
  - Expire partition API keys on partition deletion

Client sagas:
  - Log client operations to analytics warehouse
  - Dissociate client accounts on deactivation
```

**What customers avoid:**

- Implementing CQRS/saga patterns
- Building async workflow orchestration
- Writing command/query separation logic
- Managing event stream filtering
- Creating idempotent handlers
- Coordinating multi-service operations

### 12. Cascade Deletion with Cleanup Tasks

Automatic cleanup of related entities on parent deletion:

```typescript
DeleteDataPartitionsForCustomerHandler:
  - Queries all partitions for customer
  - Deletes each partition
  - Triggers partition deletion events
  - Each partition event expires associated API keys
  - Maintains referential integrity

ExpirePartitionApiKeysHandler:
  - Fetches all API keys for partition
  - Expires keys via access service
  - Prevents orphaned credentials
```

**What customers avoid:**

- Writing cascade deletion logic
- Building cleanup task systems
- Managing referential integrity
- Implementing multi-step deletion
- Coordinating cross-service cleanup

### 13. Analytics Warehouse Integration

Automatic logging to data warehouse for reporting:

```typescript
BigQuery sagas:
  - CustomerCreatedEvent → Log to customer table
  - CustomerUpdatedEvent → Log to customer table
  - CustomerClientCreatedEvent → Log to client table
  - Includes operation type (Create, Update, Delete)
  - Captures full entity state
  - Enables historical reporting
```

**What customers avoid:**

- Building data warehouse integrations
- Writing event-to-warehouse mapping
- Managing analytics logging
- Creating historical tracking systems

### 14. Session Expiration Customization

Per-customer session timeout configuration:

```typescript
Customer model includes:
  - sessionExpirationSeconds field
  - Overrides platform default
  - Applied to all customer users
  - Used by Auth service for token TTL
  - Null value uses platform default
```

**What customers avoid:**

- Building per-tenant session management
- Implementing timeout override systems
- Managing token TTL configuration

### 15. Multi-Tenant Bootstrap Keys

Secure tenant provisioning with unique keys:

```typescript
Bootstrap tenant key:
  - Required field on every customer
  - Used for initial tenant setup
  - Enables multi-tenant architecture
  - Supports tenant isolation
  - Links to authentication tenant
```

**What customers avoid:**

- Designing tenant provisioning systems
- Building unique key generation
- Managing tenant isolation

### 16. Permission-Based Access Control

Granular endpoint protection with role-based permissions:

```typescript
Permissions:
  - EditCustomer, ViewCustomer
  - EditClient, ViewClient
  - ManagePartitions
  - PlatformAdminEditAllCustomers
  - PlatformAdminViewAllCustomers
  - PlatformAdminSearchAllCustomers
  - SearchTenants
  - EditRegistrationSettings

Default roles:
  - TenantAdmin (edit + view clients)
  - ClientEditor (edit + view clients)
  - ClientViewer (view clients only)

Guards:
  - @HasPermission decorator on endpoints
  - Claim validation (customerKey required)
  - Key mismatch error handling
```

**What customers avoid:**

- Building permission systems
- Implementing role hierarchies
- Writing authorization guards
- Managing default role mappings
- Creating claim validation

### 17. Strict Key Validation with Audit Trail

Comprehensive security checks on all operations:

```typescript
matchKeysOrThrowError:
  - Validates customerKey in claims matches request
  - Validates clientId matches client entity
  - Prevents cross-tenant data access
  - Throws KEY_MISMATCH errors with details
  - Applied to all CRUD operations

Error types:
  - NOT_AUTHORIZED (401)
  - NOT_FOUND (404)
  - KEY_MISMATCH (400)
  - DOMAIN_NOT_FOUND (404)
```

**What customers avoid:**

- Implementing multi-tenant security
- Writing key validation logic
- Building audit trail systems
- Creating error differentiation
- Managing cross-tenant protection

### 18. Business Classification and Industry Tracking

Comprehensive business profiling:

```typescript
Business customer fields:
  - Industry sector (11 GICS sectors)
  - Industry group (24 GICS industry groups)
  - Business classification (9 types: LLC, C-corp, etc.)
  - Business size range (8 tiers: 1-10 to 10,001+)
  - Social media profile URLs (6 platforms)
  - External ID for third-party integration
  - Logo image with URL and file receipt
```

**What customers avoid:**

- Building business taxonomy systems
- Implementing industry classification
- Managing size range categorization
- Creating social media integrations
- Writing external ID mapping

## Common Use Cases

- **B2B SaaS platforms**: Multi-tenant customer accounts with user management and data isolation
- **Reseller networks**: Hierarchical customer-client relationships with provisioned accounts
- **Multi-environment applications**: Data partitioning for dev/qa/prod with scoped API keys
- **White-label platforms**: Per-customer branding, session settings, and domain registration
- **Enterprise onboarding**: Automated location creation, department setup, and contact management
- **Marketplace platforms**: Customer and client accounts with organizational structure and search

## What Customers Don't Have to Build

- Multi-tenant customer management with data isolation
- Hierarchical client relationships with account provisioning
- Domain-based self-service registration with auto-role assignment
- Data partition system with environment segregation
- API key generation, rotation, and expiration
- Deactivation workflows with grace periods and feedback
- Custom claims synchronization on product changes
- Organizational structure management (departments, locations, contacts)
- Real-time search index synchronization
- Scoped search key generation with query restrictions
- Event-driven architecture with database triggers
- Saga-based background task orchestration
- Cascade deletion with referential integrity
- Analytics warehouse integration
- Per-customer session timeout configuration
- Bootstrap key management for tenant provisioning
- Permission-based access control with role hierarchies
- Multi-tenant key validation and security
- Business classification and industry tracking
- Social media profile management
- Logo image handling with CDN integration
- External ID mapping for third-party systems

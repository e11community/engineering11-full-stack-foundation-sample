# Access Service

Permission and authorization management.

## What It Does

The Access Service manages what users can do within the platform — defining permissions, grouping them into roles, and enforcing access policies. It provides the authorization layer that all other services use to make access decisions.

## Key Capabilities

| Capability                 | Description                                                   |
| -------------------------- | ------------------------------------------------------------- |
| **Permission Definitions** | Define granular permissions for actions and resources         |
| **Role Management**        | Group permissions into reusable roles                         |
| **Role Assignment**        | Assign roles to users at global, customer, or resource levels |
| **Policy Evaluation**      | Check if a user has permission to perform an action           |
| **Permission Inheritance** | Support hierarchical permission structures                    |

## Permission Naming Convention

Permissions follow a consistent pattern for organization:

```
(scope/service)/(action)-(target)
```

Examples:

- `customer/view-client`
- `user/manage-all-customer-users`
- `product/edit-billing-details`

## How It Fits Together

```
┌─────────────────┐
│  Access Service │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│      All Services           │
│  (permission checks)        │
└─────────────────────────────┘
```

Every service queries the Access Service (or uses cached policies) to authorize user actions.

## Heavy-Lifting Features

### 1. Multi-Level Permission Checking

Automatic permission evaluation with multiple strategies:

```typescript
PermissionService provides:
  - hasPermission()      - Single permission check
  - hasAnyPermission()   - OR logic (user needs ANY of these)
  - hasAllPermissions()  - AND logic (user needs ALL of these)
  - getPermissions()     - Lazy-loaded permission retrieval
```

**Automatic resolution:**

- Product → Role → Permissions mapping
- Multi-product permission merging (user has access to multiple products)
- Role encompassing calculations (which roles can this user assign to others)

**What customers avoid:**

- Writing permission check logic
- Implementing AND/OR permission combinations
- Managing product-to-role-to-permission hierarchies
- Calculating role qualification logic

### 2. Real-Time Permission Updates

Automatic permission cache management with live updates:

- **Reactive Listeners**: Permission maps automatically update via database listeners
- **Zero Restart Required**: Permission changes propagate without app deployment
- **Static Service Design**: Zero-instantiation overhead for permission checks
- **Change Detection**: Automatic invalidation when permission configurations update

**What customers avoid:**

- Building cache invalidation systems
- Implementing pub/sub for permission updates
- Writing listener management code
- Handling cache consistency

### 3. Decorator-Based Access Control

Automatic endpoint-level authorization enforcement:

```typescript
@ApiKeyRequireScope('read', 'write')  // Automatic validation
async getResource() { ... }

Guards automatically:
  - Parse and validate API keys (format: {id}_{randomBytes(32)})
  - Check API key scopes against required scopes
  - Validate TTL expiration
  - Verify temporary authorization tokens
  - Delete tokens after successful use
```

**What customers avoid:**

- Writing authorization middleware
- Implementing scope validation logic
- Managing token parsing and validation
- Building automatic cleanup mechanisms

### 4. Multi-Tenant API Key Isolation

Automatic tenant-level access isolation:

```typescript
ApiKeyManager handles:
  - customerKey field: Customer-level isolation
  - partitionId field: Logical partitioning within customers
  - foreignKey field: External reference tracking

Bulk operations:
  - expireAllForCustomer()    - Invalidate all customer keys
  - expireAllForPartition()   - Partition-scoped invalidation
  - expireAllForForeignKey()  - Cascading invalidation
```

**Security features:**

- Obfuscated failure reasons (prevents information leakage)
- All errors return identical messages regardless of root cause
- Prevents tenant enumeration attacks

**What customers avoid:**

- Building multi-tenant isolation logic
- Implementing cascading invalidation
- Writing secure error handling
- Managing partition hierarchies

### 5. Automatic Token Lifecycle Management

Complete token lifecycle automation:

```typescript
Token Lifecycle:
  - Expiration enforcement (date-based TTL)
  - Use limit tracking (allowedUses/useCount with transactions)
  - Token disabling/enabling (soft delete)
  - Automatic cleanup (scheduled jobs delete expired tokens)

Validation checks ALL conditions:
  - checkTokenExpiration()
  - checkTokenDisabled()
  - checkUseLimit()
```

**Background automation:**

- Scheduled job runs every 12 hours to clean expired authorization tokens
- Cloud Tasks schedule API key expiration events
- Transactional use counting prevents race conditions

**What customers avoid:**

- Writing expiration checking logic
- Implementing use limit counters
- Building scheduled cleanup jobs
- Managing token state transitions

### 6. Cryptographic Security

Built-in cryptographic operations:

```typescript
API Key Hashing:
  - Configurable salt rounds (default: 10)
  - One-time plaintext delivery, then encrypted storage
  - Constant-time comparison (encryption.compare())

Authorization Token Security:
  - 32-byte random key generation
  - Hashed storage
  - ID:Token format (separates identity from secret)

Secure Key Generation:
  - generateApiKey(id) → {id}_{randomBytes(32).toString('hex')}
  - 64-character hex strings
```

**What customers avoid:**

- Implementing secure hashing
- Managing salt rounds
- Writing constant-time comparison
- Building key generation logic

### 7. Event-Driven Architecture

Automatic event publishing for all access operations:

```typescript
API_KEY_CREATED  → Publish event → Check TTL → Schedule expiration
API_KEY_UPDATED  → Publish event
API_KEY_DELETED  → Publish event
TOKEN_EXPIRED    → Publish event
```

**Event-driven automation:**

- Expiration tasks automatically scheduled on key creation
- Downstream services notified of access changes
- Audit trails generated automatically

**What customers avoid:**

- Building event publishing infrastructure
- Implementing task scheduling
- Managing event-driven workflows
- Writing audit logging

### 8. Role Encompassing Calculations

Automatic qualification checking for role assignments:

```typescript
rolesEncompassedByUser(userPermissions, permissibleRolePermissions):
  - Calculates which roles a user qualifies to assign to others
  - Uses hasAll() utility to match complete permission sets
  - Prevents privilege escalation (users can't grant roles with permissions they lack)
```

**What customers avoid:**

- Writing role qualification logic
- Implementing privilege escalation prevention
- Managing hierarchical role relationships

### 9. Use Limit Enforcement

Transactional use counting with automatic enforcement:

```typescript
UseLimitService:
  - Fine-grained limits: per userId + type + itemId
  - Transactional increment (prevents race conditions)
  - Automatic limit checking before operations
  - USE_LIMIT_EXCEEDED error thrown when exceeded
```

**What customers avoid:**

- Writing distributed counters
- Implementing transactional updates
- Building rate limiting logic
- Handling concurrent access

### 10. Token Enrichment Pipeline

Extensible token enhancement system:

```typescript
TokenEnricher abstract class:
  - NoOpTokenEnricher: Default implementation
  - DynamicLinkTokenEnricher: Auto-generates dynamic links
  - Custom enrichers: Merge additional data into tokens

Example: Authorization tokens automatically embed:
  - holderInformation.appUserId
  - holderInformation.customerKey
  - holderInformation.roles
  - holderInformation.products
```

**What customers avoid:**

- Building token enrichment pipelines
- Writing dynamic link generators
- Implementing claim embedding

## Common Use Cases

- **Role-based access**: Admins can manage users, viewers can only read with automatic permission checking
- **Resource-level permissions**: User can edit only their own content with decorator-based enforcement
- **Customer-scoped roles**: Permissions that apply within a specific customer context with automatic isolation
- **Feature gating**: Control access to premium features via permissions with real-time updates
- **API key management**: Multi-tenant API keys with automatic TTL expiration and scope validation
- **Temporary access tokens**: Single-use or time-limited tokens with automatic cleanup
- **Use limits**: Per-user, per-resource rate limiting with transactional counting
- **Role assignment control**: Automatic prevention of privilege escalation

## What Customers Don't Have to Build

- Multi-level permission checking logic (single, any, all)
- Real-time permission cache management
- Decorator-based authorization guards
- Multi-tenant API key isolation
- Token lifecycle management (expiration, use limits, cleanup)
- Cryptographic hashing and secure key generation
- Event-driven access control workflows
- Role encompassing calculations
- Transactional use limit counters
- Token enrichment pipelines
- Cascading invalidation across tenants
- Privilege escalation prevention
- Scheduled token cleanup jobs
- Secure error handling with information leak prevention

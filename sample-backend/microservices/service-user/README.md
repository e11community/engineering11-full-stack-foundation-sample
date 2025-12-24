# User Service

Comprehensive user management for B2B, B2C, and B2B2C platforms.

## What It Does

The User Service handles the complete user lifecycle — from onboarding to profile management to offboarding. It works in coordination with the Auth Service to manage identity while maintaining rich user profiles, business-specific metadata, and compliance tracking.

## Key Capabilities

| Capability                          | Description                                                              |
| ----------------------------------- | ------------------------------------------------------------------------ |
| **User Profile Management**         | Store and manage user information, preferences, and business metadata    |
| **Multi-Tenant Architecture**       | Support for Business, Consumer, and Platform Admin user types            |
| **Role-Based Access Control**       | Flexible role assignment with grantable and encompassed role validation  |
| **Deactivation & Deidentification** | Automated user lifecycle with GDPR-compliant data removal                |
| **Terms of Service Tracking**       | Record and audit TOS acceptance per user with version history            |
| **Auth Coordination**               | Link and unlink authentication accounts with bidirectional sync          |
| **Business User Metadata**          | Track department, location, company role for organizational segmentation |

## How It Fits Together

```
┌─────────────────┐     ┌─────────────────┐
│   Auth Service  │◄───►│   User Service  │
└─────────────────┘     └────────┬────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Customer │ │  Search  │ │Analytics │
              │ Service  │ │  Index   │ │(BigQuery)│
              └──────────┘ └──────────┘ └──────────┘
```

The User Service publishes events when users are created, updated, disabled, or deidentified — allowing other services to react to user changes.

## Heavy-Lifting Features

### 1. Attribute-Level Permission Enforcement

Granular control over what users can view and update:

```typescript
Permission whitelists:
  - GET_ATTRS_FOR_SELF_STANDARD_USER: 21 fields consumer users can view
  - GET_ATTRS_FOR_SELF_BUSINESS_USER: 22 fields business users can view
  - GET_ATTRS_FOR_ADMIN: 24 fields admins can view for others
  - UPDATE_ATTRS_FOR_SELF_STANDARD_USER: 12 fields consumers can update
  - UPDATE_ATTRS_FOR_SELF_BUSINESS_USER: 13 fields business users can update
  - UPDATE_ATTRS_FOR_ADMIN: 15 fields admins can update for others

UserViewModelService provides:
  - buildStandardUser: Filters consumer user responses
  - buildSelfBusinessUser: Filters business user self-views
  - buildBusinessUserForAdmin: Filters admin views of business users
  - buildStandardUserUpdate: Filters consumer update payloads
  - buildSelfBusinessUserUpdate: Filters business user self-updates
  - buildBusinessUserUpdateForAdmin: Filters admin updates

Configurable extensions:
  - userConfig.userValidation.getAttributes.standardUser
  - userConfig.userValidation.getAttributes.businessUser
  - userConfig.userValidation.getAttributes.adminUser
  - userConfig.userValidation.updateAttributes (same three types)
```

**What customers avoid:**

- Building field-level permission systems
- Writing view model transformation logic
- Implementing different permission levels per user type
- Managing configurable attribute extensions
- Preventing privilege escalation through update payloads

### 2. Role Grant Validation System

Multi-layer role validation prevents privilege escalation:

```typescript
GrantableRolesService provides:
  - rejectNonGrantableRoles: Blocks blacklisted roles (platform admin)
  - rejectNonEncompassedRoles: Ensures grantor has all permissions being granted

Permission validation flow:
  1. Check against NON_GRANTABLE_ROLES blacklist
  2. Query PermissionService.getEncompassedRoles(roles, products)
  3. Verify all granted roles are encompassed by grantor's permissions
  4. Throw 403 with detailed error types if validation fails

Error types:
  - roles/non-grantable → 403 "Invalid Roles" (attempted platform admin grant)
  - roles/unencompassed → 403 "Invalid Roles" (attempted privilege escalation)
```

**What customers avoid:**

- Building role hierarchy validation
- Implementing permission encompassment logic
- Preventing privilege escalation attacks
- Writing role blacklist systems
- Managing product-aware permission checks

### 3. Automated Deactivation & Deidentification Pipeline

Multi-stage privacy-compliant user offboarding:

```typescript
Deactivation workflow:
  1. User.disable(): Sets disabled=Date, isDisabled=true
  2. onUpdate trigger: Detects isDisabled change, publishes USER_DISABLED event
  3. scheduleDeidentification(): Schedules future deidentification task
     - Immediate if deidentifyOnDeactivation=true
     - 90 days (ACCOUNT_DISABLED_DAYS) otherwise
  4. deidentifyUser() executes at scheduled time

UserDeidentificationService.deidentifyUser():
  - Runs in transaction to prevent race conditions
  - Verifies user.isDisabled before proceeding
  - Only deidentifies Consumer users (protects Business/PlatformAdmin)
  - Removes IdentifyingFields: email, firstName, lastName, phoneNumber,
    displayName, aboutMe, photoURL, address fields, pronouns
  - Replaces firstName='Unknown', lastName='User', displayName='Unknown User'
  - Sets deidentified=true flag
  - Publishes USER_DEIDENTIFIED event for downstream services

Automatic side effects:
  - Search index updated (removes PII from search results)
  - BigQuery logging skipped for deidentified users
  - Downstream services notified via pub/sub event
```

**What customers avoid:**

- Building GDPR-compliant data removal systems
- Implementing scheduled deidentification workflows
- Writing transactional field removal logic
- Managing identifying field inventory
- Coordinating deidentification across services
- Preventing accidental deidentification of active users

### 4. Coordinated Auth Service Integration

Bidirectional synchronization with authentication:

```typescript
Auth linking operations:
  - linkAuth(userId, authId): Updates user.authId, persists link
  - unlinkAuth(userId): Sets user.authId=null, breaks link
  - getByAuthId(authId): Queries users by auth account

Task handlers:
  - USER_LINK_AUTH queue: Processes auth link requests
  - USER_UNLINK_AUTH queue: Processes auth unlink requests
  - Idempotent handling (safe to retry)
  - Automatic JWT update tracking via jwtUpdatedAt field

Auth coordination:
  - Auth Service creates identity, publishes link event
  - User Service receives event, updates user.authId
  - User Service maintains authTenant for multi-tenant isolation
  - bootstrapTenantKey tracks original registration tenant
```

**What customers avoid:**

- Building bidirectional auth synchronization
- Implementing idempotent link/unlink operations
- Writing multi-tenant isolation logic
- Managing JWT update tracking
- Coordinating between auth and user systems

### 5. Terms of Service Audit Trail

Complete TOS acceptance tracking with history:

```typescript
Terms tracking:
  - User.termsVersionAccepted: Current TOS version number
  - UserTerms collection: Complete acceptance history

UserTermsRepository creates records:
  {
    id: unique_id,
    acceptDate: Date,
    userId: user_id,
    version: version_number,
    customerKey: tenant_id
  }

Task handler flow:
  1. User updates termsVersionAccepted field
  2. onUpdate trigger detects change
  3. USER_TOU_LOG_ACCEPTED queue receives event
  4. UserTasksService.touAccepted() creates audit record
  5. USER_FIRST_TOU_ACCEPTED event published on first acceptance

Query capabilities:
  - All TOS acceptances for a user
  - All users who accepted specific version
  - Users with outdated TOS acceptance
```

**What customers avoid:**

- Building TOS audit trail systems
- Implementing version tracking logic
- Writing acceptance history queries
- Managing first-time acceptance detection
- Creating compliance reporting infrastructure

### 6. Multi-Datastore Synchronization

Automatic synchronization across storage systems:

```typescript
Write operations trigger parallel updates:
  - UserService.update() → Firestore primary write
  - AlgoliaSearchService.upsert() → Search index update
  - UserBigQueryRepository.logFromUser() → Analytics append

BigQuery logging:
  - Captures all CRUD operations (Create, Update, Delete)
  - Preserves full user JSON in 'data' column
  - Extracts key fields: id, customerKey, firstName, lastName, email,
    userType, authTenant, disabled, timestamp, operation
  - Skips deidentified users (prevents PII in analytics)
  - Immutable append-only log for audit and analytics

Search index sync:
  - Real-time updates on user changes
  - Excludes PlatformAdmin users from search
  - Automatic deletion on user deletion
  - Customer-scoped search with secured API keys

Task-based indexing:
  - USER_ADDED queue: Initial index creation
  - USER_UPDATED queue: Incremental updates
  - USER_DELETED queue: Index removal
  - Deduplication keys prevent duplicate processing
```

**What customers avoid:**

- Building multi-datastore sync logic
- Implementing eventual consistency patterns
- Writing append-only audit logs
- Managing search index updates
- Creating analytics data pipelines
- Coordinating between transactional and analytical stores

### 7. Secured Search Key Generation

Dynamic, scoped search credentials for client applications:

```typescript
SearchKeyController provides:
  - Admin search keys: Scoped to customerKey via filters
  - Platform admin keys: Excludes platform admin users via NOT filters

Key generation:
  GET /search-key/admin-package:
    - Requires Permissions.SearchAllUsers
    - Restricts to SEARCH_INDICES.USER
    - Filters: customerKey:${customerKey}
    - Returns: {searchAppId, searchKey}

  GET /search-key/platform-admin-package:
    - Requires Permissions.PlatformAdminSearchAllUsers
    - Filters: NOT roles:PlatformAdminRole1 AND NOT roles:...
    - Returns global search excluding platform admins

Security features:
  - Generated keys inherit permission-based filters
  - Keys cannot access data outside scope
  - Automatic expiration via Algolia TTL
  - No credential exposure (generated on-demand)
```

**What customers avoid:**

- Building search credential systems
- Implementing row-level security for search
- Writing filter-based access control
- Managing search API key lifecycles
- Creating permission-aware search scoping

### 8. Customer Key Isolation

Tenant isolation with automatic validation:

```typescript
Multi-tenant protection:
  - User.customerKey: Primary tenant identifier
  - User.clientId: Optional sub-tenant for data sharing
  - User.bootstrapTenantKey: Original registration tenant

BusinessUserManagementController enforces:
  matchCustomerKeys(user, claims):
    - Compares user.customerKey to claims.customerKey
    - Throws 403 "key mismatch" if different
    - Prevents cross-tenant access attempts

Query isolation:
  - UserFirebaseRepository.getAllForCustomer(customerKey)
  - TypedFirestoreQueryBuilder with customerKey criteria
  - Search keys scoped by customerKey filters

Group operations:
  - userGroupDeactivationRequested: Bulk deactivate by customerKey
  - userGroupReactivationRequested: Bulk reactivate by customerKey
  - Parallel task dispatch for scalability
```

**What customers avoid:**

- Building tenant isolation systems
- Implementing customer key validation
- Writing bulk tenant operations
- Managing cross-tenant access prevention
- Creating tenant-scoped queries

### 9. SelfGuard Authorization

Declarative endpoint protection for user access:

```typescript
@Controller({path: 'user'})
@UseGuards(SelfGuard)
export class UserController {
  @Get(':id')
  async get(@Param('id') id: string)

  @Put(':id')
  async update(@Param('id') id: string, @Body() dto)
}

SelfGuard validates:
  - Extracts user ID from URL parameter
  - Compares to claims.appUserId
  - Allows access only if IDs match
  - Returns 403 if user attempts to access another user's data

Automatic URL parsing:
  - Works with any :id parameter name
  - Supports nested routes
  - No manual authorization code needed
```

**What customers avoid:**

- Writing authorization logic in every endpoint
- Implementing user-owns-resource checks
- Managing URL parameter extraction
- Building guard decorators
- Creating reusable authorization middleware

### 10. Business User Metadata Segmentation

Rich organizational metadata for B2B use cases:

```typescript
IBusinessUser extends IUser:
  - companyRole: Job title or function
  - department: Organizational department
  - location: Physical or virtual location

Admin management:
  - View all fields (GET_ATTRS_FOR_ADMIN)
  - Update department, location, companyRole for others
  - Query users by organizational metadata

Self-management:
  - View own companyRole, department, location
  - Update own companyRole, department (not location if restricted)

UserStage for bulk import:
  - jobFunction, department, location, class, division, union,
    clearance, jobCode
  - Chunked processing with audit trail
  - Supports organizational hierarchy import
```

**What customers avoid:**

- Building organizational metadata schemas
- Implementing department/location tracking
- Writing bulk import with business fields
- Creating organizational hierarchy support
- Managing B2B-specific user attributes

### 11. Preference Management

Centralized notification preferences:

```typescript
IUserPreferences:
  - emailEnabled: boolean (default: true)
  - pushNotificationsEnabled: boolean (default: true)

Integration:
  - User.userPreferences: Embedded preferences object
  - Self-updatable by both Consumer and Business users
  - Updateable by admins for others
  - Extends to custom preferences via validation config

Notification coordination:
  - Services query userPreferences before sending notifications
  - Respects user opt-out choices
  - Centralized preference storage (no service-specific flags)
```

**What customers avoid:**

- Building preference management systems
- Implementing notification opt-out logic
- Coordinating preferences across services
- Creating preference APIs
- Managing default preference values

### 12. Automatic Event-Driven Architecture

Transparent pub/sub for all user changes:

```typescript
Cloud Functions automatically publish:
  - USER_ADDED → onCreate trigger → tasks queue
  - USER_UPDATED → onUpdate trigger → tasks queue
  - USER_DELETED → onDelete trigger → tasks queue
  - USER_DISABLED → conditional onUpdate (isDisabled change)
  - USER_REENABLED → conditional onUpdate (isDisabled=false)
  - USER_DEIDENTIFIED → published after deidentification
  - USER_FIRST_TOU_ACCEPTED → published on first TOS acceptance

Event routing:
  - Database triggers publish to topics
  - Topics route to Cloud Tasks queues
  - Task handlers process with retries
  - Deduplication keys prevent double processing

Downstream coordination:
  - Other services subscribe to user events
  - Search index updates automatically
  - BigQuery logs created automatically
  - Auth service receives link/unlink events
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database triggers
- Implementing pub/sub integrations
- Managing task queue routing
- Coordinating event subscribers
- Creating retry and deduplication logic

### 13. Platform Admin Isolation

Special user type for system administrators:

```typescript
UserType.PlatformAdmin users:
  - Excluded from search indices
  - Excluded from BigQuery logging
  - Cannot be deidentified (type check prevents it)
  - Managed via separate PlatformAdminUserController

Platform admin permissions:
  - PlatformAdminViewAllUsers: Read any user
  - PlatformAdminEditAllUsers: Update any user
  - PlatformAdminSearchAllUsers: Search across tenants
  - No customerKey matching required

Protection:
  - NON_GRANTABLE_ROLES blacklist prevents granting platform roles
  - Cannot be assigned via regular user update endpoints
  - Search keys explicitly exclude platform admin roles
```

**What customers avoid:**

- Building system administrator isolation
- Implementing cross-tenant admin access
- Writing platform role protection
- Managing admin user type segregation
- Creating admin-specific permission systems

## Common Use Cases

- **Employee onboarding**: Create business users with department, location, and role metadata
- **Customer self-registration**: Create consumer accounts with profile management
- **Account deactivation**: Disable users with optional deidentification after retention period
- **Compliance tracking**: Record TOS acceptance with version history and audit trail
- **Organizational structure**: Query users by department, location, or role for reporting
- **Multi-tenant SaaS**: Isolate users by customerKey with automatic tenant validation
- **User search**: Generate scoped search credentials for client-side user lookup
- **Cross-service coordination**: React to user events (creation, updates, deidentification)

## What Customers Don't Have to Build

- Attribute-level permission whitelists for different user types
- Role grant validation with encompassment checking
- Automated deactivation and deidentification pipelines
- GDPR-compliant PII removal with transaction safety
- Terms of service acceptance audit trail
- Auth service bidirectional synchronization
- Multi-datastore synchronization (database, search, analytics)
- Append-only BigQuery audit logs
- Secured search key generation with filters
- Customer key isolation and validation
- SelfGuard declarative authorization
- Business user metadata schemas
- Preference management for notifications
- Event-driven architecture with database triggers
- Platform admin user type isolation
- Bulk user operations scoped by tenant
- View model transformation per permission level
- Configurable attribute extension system
- Deduplication for idempotent task processing
- Scheduled task execution for deidentification

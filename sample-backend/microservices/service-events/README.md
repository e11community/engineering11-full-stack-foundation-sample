# Events Service

Secure event publishing with permission validation, schema enforcement, and message bus routing.

## What It Does

The Events Service provides a centralized gateway for publishing events across the platform. It validates user permissions, enforces event schemas, and routes messages to pub/sub topics based on configurable mappings. Events flow through validation layers before being reliably delivered to subscribers.

## Key Capabilities

| Capability                 | Description                                          |
| -------------------------- | ---------------------------------------------------- |
| **Permission Validation**  | User-based authorization with role and type checking |
| **Schema Validation**      | JSON Type Definition validation for event payloads   |
| **Event Routing**          | Configurable event-to-topic mappings                 |
| **Multi-Topic Publishing** | Single event can publish to multiple topics          |
| **Message Archiving**      | Automatic archival of all published messages         |
| **Retry Logic**            | Exponential backoff for failed publishes             |
| **User Context**           | Enriches events with sender information              |

## How It Fits Together

```
┌──────────────────────────────────────┐
│           Events Service             │
│    (Validation + Routing Layer)      │
└──────────────────┬───────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌────────┐    ┌────────┐    ┌────────┐
│ Pub/Sub│    │ Pub/Sub│    │Archive │
│ Topic A│    │ Topic B│    │ Topic  │
└────────┘    └────────┘    └────────┘
```

Services send events through the Events Service gateway, which validates and routes them to configured pub/sub topics.

## Heavy-Lifting Features

### 1. Multi-Layer Permission Validation

Complex authorization logic with declarative configuration:

```typescript
EventPermissionService handles:
  - User existence validation
  - Disabled user blocking (default behavior)
  - Role-based authorization
  - User type restrictions (Consumer vs Business)
  - Terms of Service version enforcement
  - Negative conditions (NOT logic)
  - OR conditions (ANY logic)
  - Nested validation rules
  - Custom failure messages

EventValidator supports:
  - role: "Admin" → requires specific role
  - userType: "Consumer" → restricts by user type
  - allowDisabled: true → overrides disabled user blocking
  - termsVersionAtLeast: 2 → requires TOS acceptance
  - not: {condition, failureMessage} → inverted logic
  - any: [{condition1}, {condition2}] → OR evaluation
```

**What customers avoid:**

- Building complex permission evaluation systems
- Writing user validation logic
- Implementing role and type checking
- Managing terms of service enforcement
- Creating flexible authorization rules with AND/OR/NOT logic
- Handling nested permission conditions

### 2. JSON Type Definition Schema Validation

Rigorous payload validation with cached schemas:

```typescript
EventShapeValidationService provides:
  - JSON Type Definition (JTD) schema validation
  - Schema lookup by event type
  - Cached schema storage for performance
  - Automatic schema refresh on updates
  - Optional validation (schemas not required)
  - Detailed error messages with field-level info
  - AJV-powered validation engine

Validation flow:
  - Check if schema exists for event type
  - If no schema: pass validation (opt-in enforcement)
  - If schema exists: validate payload structure
  - Return detailed errors for invalid payloads
  - Reject malformed events before publishing
```

**What customers avoid:**

- Implementing schema validation systems
- Managing schema storage and caching
- Writing JSON validation logic
- Building error message formatting
- Handling optional vs required validation
- Integrating validation libraries

### 3. Configurable Event-to-Topic Routing

File-based routing with validation:

```typescript
EventMappingLoader features:
  - JSON configuration file loading
  - Schema validation for mapping files
  - One-to-many event-to-topic mappings
  - Runtime topic resolution
  - Clear error messages for missing mappings
  - Prevents publishing to unmapped event types

Configuration format:
{
  "mappings": [
    {"eventType": "USER_CREATED", "topic": "user-events"},
    {"eventType": "USER_CREATED", "topic": "analytics"},
    {"eventType": "ORDER_PLACED", "topic": "orders"}
  ]
}

Runtime behavior:
  - Load mappings at service startup
  - Validate JSON structure with AJV
  - Transform to Map<eventType, topic[]>
  - Throw TopicMappingNotFoundException for unmapped events
```

**What customers avoid:**

- Building routing configuration systems
- Writing topic mapping logic
- Implementing configuration validation
- Managing one-to-many relationships
- Handling missing mapping errors
- Creating configuration file parsers

### 4. Authorization Spec Loading System

Flexible permission configuration management:

```typescript
AuthSpecLoader capabilities:
  - Load single spec files or entire directories
  - Recursive directory traversal
  - JSON schema validation with AJV
  - Duplicate event type detection
  - Nested permission object support
  - Spec file hot reloading support

Spec file structure:
{
  "eventType": "SEND_MESSAGE",
  "role": "Member",
  "not": {
    "userType": "Business",
    "failureMessage": "Business users cannot send messages"
  }
}

Loading behavior:
  - Reads from file path or directory
  - Validates each spec against schema
  - Transforms to EventValidator instances
  - Maps by event type for O(1) lookup
  - Throws on duplicate event types
```

**What customers avoid:**

- Building configuration loading systems
- Writing recursive file traversal
- Implementing spec validation
- Managing configuration parsing
- Handling duplicate detection
- Creating validator factories

### 5. Reliable Message Publishing with Archiving

Enterprise-grade publish infrastructure:

```typescript
PubSubService features:
  - Exponential backoff retry (5 attempts)
  - Automatic message archiving
  - Topic existence checking
  - Ordering key support
  - UUID message tracking
  - Error wrapping with context
  - Optional non-blocking publishes

EventPublishingService orchestrates:
  - Event metadata enrichment (timestamp, senderId)
  - Multi-topic parallel publishing
  - Event type extraction
  - Promise.all for atomic multi-publish
  - Structured event envelope:
    {
      kind: 'event',
      item: payload,
      timestamp: ISO string,
      senderId: userId
    }

Archive payload:
  - e11MessageId: UUID
  - topic: original topic name
  - timestamp: publish time
  - message: full event payload
  - orderingKey: optional ordering
```

**What customers avoid:**

- Implementing retry logic with exponential backoff
- Building message archiving systems
- Writing parallel publish coordination
- Managing message envelope structures
- Handling publish failures gracefully
- Creating message tracking systems
- Integrating with pub/sub infrastructure

### 6. User Context Validation and Enrichment

Comprehensive user handling:

```typescript
User validation flow:
  - Extract appUserId from JWT claims
  - Fetch full user object from User Service
  - Validate user exists (400 if not found)
  - Check if user is disabled
  - Verify user roles match requirements
  - Confirm user type matches spec
  - Validate TOS acceptance if required
  - Enrich event with senderId

Error handling:
  - User not found → BadRequestException
  - User disabled → "User X is disabled"
  - Missing role → "Must have role Y"
  - Wrong type → "Incorrect user type"
  - Permission denied → Custom failure message
```

**What customers avoid:**

- Building user validation pipelines
- Integrating with user services
- Writing JWT claim extraction
- Implementing existence checks
- Managing user context enrichment
- Creating detailed error messages

### 7. RESTful Event Submission Endpoint

Production-ready HTTP interface:

```typescript
POST /events/outgoing (v1)

Features:
  - NestJS controller with validation pipe
  - JWT authentication required
  - Custom claim extraction (@Claim decorator)
  - Class-validator integration
  - HTTP 201 on success
  - Detailed error responses (400)
  - Automatic error type mapping

Request validation:
  - @RequireCustomClaim('appUserId')
  - IncomingEvent DTO validation
  - eventType required (string, non-empty)
  - Additional payload properties preserved

Error mapping:
  - PermissionsValidationError → 400
  - SchemaNotFoundError → passes (opt-in)
  - ObjectNotValidError → 400 with details
  - TopicMappingNotFoundException → 400
  - User not found → 400
```

**What customers avoid:**

- Building REST endpoints with validation
- Implementing JWT authentication
- Writing request validation pipelines
- Managing error response formatting
- Creating DTO classes
- Handling HTTP status codes
- Integrating validation libraries

### 8. Sophisticated Permission Composition

Advanced authorization patterns:

```typescript
NOT logic:
  - Inverts child validation result
  - Requires custom failure message
  - Supports nested conditions
  - Example: "Allow all except Business users"

ANY logic:
  - OR evaluation across conditions
  - At least one must pass
  - Short-circuit evaluation
  - Example: "Admin OR Moderator OR Owner"

Composition examples:
  // Admin OR user owns the resource
  {
    "any": [
      {"role": "Admin"},
      {"userType": "Business"}
    ]
  }

  // Must be Consumer but NOT free tier
  {
    "userType": "Consumer",
    "not": {
      "role": "FreeTier",
      "failureMessage": "Upgrade to use this feature"
    }
  }

  // Complex nested logic
  {
    "any": [
      {"role": "Admin"},
      {
        "userType": "Consumer",
        "termsVersionAtLeast": 2
      }
    ]
  }
```

**What customers avoid:**

- Implementing boolean logic evaluation
- Writing permission composition systems
- Building nested validation support
- Managing short-circuit evaluation
- Creating flexible authorization DSLs
- Handling complex permission scenarios

### 9. Type-Safe Event Modeling

Structured event type system:

```typescript
Base event classes:
  - BaseCreatedEvent<T> → wraps Event<T>
  - BaseUpdatedEvent<T> → wraps UpdatedEvent<T>
  - BaseDeletedEvent<T> → wraps Event<T>

Event envelope structure:
  interface Event<T> {
    kind: 'event'
    item: T
    timestamp: string
    senderId: string
  }

  interface UpdatedEvent<T> extends Event<T> {
    before: T
    after: T
  }

Benefits:
  - Type safety for event payloads
  - Consistent envelope structure
  - Generic support for any payload type
  - Clear semantic event types
  - Automatic timestamp injection
  - Sender tracking built-in
```

**What customers avoid:**

- Designing event envelope structures
- Building type-safe event systems
- Creating generic event classes
- Managing timestamp injection
- Implementing sender tracking
- Standardizing event formats across services

### 10. Environment-Based Configuration

Declarative configuration validation:

```typescript
EnvironmentConfigModel:
  - AUTH_SPEC_PATH: path to auth specs (file or directory)
  - EVENT_MAPPING_PATH: path to event mappings JSON
  - Both validated as existing files at startup
  - Class-validator integration
  - Automatic env var loading
  - Fail-fast on missing/invalid config

Validation decorators:
  @IsString() → ensures string type
  @IsFile() → validates file exists

Startup behavior:
  - Load config from environment
  - Validate all required fields
  - Check file existence
  - Load and parse configuration files
  - Validate JSON schemas
  - Build runtime mappings
  - Throw on any validation failure
```

**What customers avoid:**

- Building configuration validation systems
- Writing environment variable loaders
- Implementing file existence checks
- Creating validation decorators
- Managing startup validation
- Handling configuration errors
- Building fail-fast initialization

### 11. Healthcheck Endpoint

Simple service monitoring:

```typescript
GET /healthcheck

Returns: "ok"
Status: 200

Purpose:
  - Load balancer health checks
  - Kubernetes liveness probes
  - Uptime monitoring
  - Quick service availability test
```

**What customers avoid:**

- Implementing health check endpoints
- Writing monitoring integrations
- Creating liveness probes

## Common Use Cases

- **Cross-service events**: User registration triggers email, analytics, audit trail
- **Access control**: Block disabled users, enforce role requirements, check TOS
- **Multi-tenant publishing**: Route different event types to separate topic hierarchies
- **Validated workflows**: Ensure event payloads match expected schemas before processing
- **Audit trails**: Archive every published event for compliance and debugging
- **Permission testing**: Use complex permission specs without custom code

## What Customers Don't Have to Build

- Multi-layer permission validation with AND/OR/NOT logic
- JSON Type Definition schema validation
- Configurable event-to-topic routing
- Authorization spec loading from files/directories
- Reliable pub/sub publishing with retries
- Automatic message archiving
- User existence and context validation
- RESTful event submission API
- Event envelope standardization
- Permission composition and nesting
- Role and user type authorization
- Terms of Service enforcement
- Exponential backoff retry logic
- Multi-topic parallel publishing
- Configuration validation at startup
- Detailed permission error messages
- Event metadata enrichment
- JWT claim extraction
- Optional schema enforcement
- Healthcheck endpoints for monitoring

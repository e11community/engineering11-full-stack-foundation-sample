# Social Service

Professional networking with connections, invitations, and relationship management.

## What It Does

The Social Service provides bi-directional professional networking capabilities with connection request workflows, token-based external invitations, user tagging, and automatic synchronization of user information across connections. It combines invitation infrastructure with connection state management to create flexible networking environments.

## Key Capabilities

| Capability                     | Description                                                 |
| ------------------------------ | ----------------------------------------------------------- |
| **Connection Requests**        | Request, accept, and delete bi-directional connections      |
| **External Invitations**       | Token-based invites for users outside the platform          |
| **Connection Tagging**         | Organize connections with custom tags                       |
| **Automatic Reverse Matching** | Auto-accept when both users request connection              |
| **Invite Nudging**             | Scheduled reminder emails for pending invites               |
| **User Data Sync**             | Automatic propagation of profile changes across connections |
| **Privacy Protection**         | Progressive disclosure of personal information              |

## How It Fits Together

```
┌─────────────────┐
│     Social      │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┐
    ▼                 ▼              ▼
┌─────────┐   ┌─────────────┐  ┌──────────┐
│  User   │   │   Access     │  │Notifica- │
│ Service │   │   Service    │  │  tions   │
└─────────┘   └─────────────┘  └──────────┘
```

Social uses User for member information, Access for invite tokens, and Notifications for connection event notifications.

## Heavy-Lifting Features

### 1. Intelligent Connection Request Handling

Automatic matching when mutual interest exists:

```typescript
ConnectionService.requestConnection():
  - Validates against self-connection attempts
  - Fetches and validates both users exist and are active
  - Checks for reverse connection request (addressee → requester)
  - If reverse exists: auto-accepts and returns connected status
  - If not: creates pending connection request
  - Deterministic ID generation (requesterId_addresseeId)
  - Prevents duplicate requests via unique ID
  - Transaction-safe operations
```

**What customers avoid:**

- Writing mutual connection detection logic
- Implementing bi-directional relationship checks
- Managing automatic acceptance workflows
- Preventing duplicate connection requests
- Coordinating user validation across services

### 2. Transactional Connection Lifecycle Management

Thread-safe state transitions with validation:

```typescript
Connection operations:
  - Request: Creates pending connection with user info snapshot
  - Accept: Transactional status update with timestamp
  - Delete: Transactional removal with participant validation
  - Tag/Untag: Atomic array operations on connection metadata

Validation rules:
  - Only addressee can accept connection
  - Only participants can delete connection
  - Disabled users cannot accept connections
  - Self-connections are blocked
  - All operations wrapped in transactions
```

**What customers avoid:**

- Building connection state machines
- Writing concurrent access protection
- Implementing participant validation logic
- Managing atomic array updates for tags
- Coordinating timestamp generation

### 3. Privacy-Aware Connection View Models

Progressive disclosure based on connection status:

```typescript
connectionToVM() transformations:
  - Outgoing: Hides first/last name (only displayName visible)
  - Incoming: Shows full name (user requested you)
  - Connected: Shows full name and connection date
  - Automatic status mapping (Pending → Outgoing/Incoming)
  - User-specific perspectives on same data
  - Filters disabled users from results
  - Enriches with metadata (tags, photos)
```

**What customers avoid:**

- Building privacy filtering logic
- Implementing perspective-based transformations
- Managing personal information disclosure rules
- Writing user-specific view logic
- Coordinating status mapping

### 4. Token-Based External Invitation System

Comprehensive invitation workflow for external users:

```typescript
InviteService provides:
  - Token creation with expiration dates
  - Dynamic link generation for mobile apps
  - Email validation against self-invites
  - One-time acceptance with auto-cleanup
  - Duplicate connection prevention
  - Token invalidation on acceptance
  - Custom token enrichment with invite metadata

Token lifecycle:
  - Create invite with allowedUses and expireDate
  - Generate dynamic link for mobile deep linking
  - Send notification via NotificationDispatchService
  - Schedule 7-day nudge reminder automatically
  - Accept invite creates confirmed connection
  - Delete token on acceptance or manual deletion
```

**What customers avoid:**

- Building token generation and validation
- Implementing expiration logic
- Creating deep links for mobile apps
- Managing token-to-connection conversion
- Writing duplicate connection checks
- Coordinating automatic token cleanup

### 5. Automated Invite Nudge System

Scheduled reminders with deduplication:

```typescript
Nudge system features:
  - Automatic 7-day delayed task scheduling on invite creation
  - Manual nudge endpoint for immediate reminders
  - Deduplication key prevents duplicate scheduled nudges
  - Updates lastNudgeDate on token metadata
  - Owner validation (only invite creator can nudge)
  - Email requirement validation
  - Dynamic link regeneration for nudge emails

Scheduling logic:
  - sendToTask with scheduledTime (7 days in future)
  - dedupKeySpec prevents duplicate scheduled tasks
  - Idempotent handling for retry scenarios
  - Silently ignores scheduling conflicts
```

**What customers avoid:**

- Building task scheduling infrastructure
- Implementing deduplication logic
- Managing reminder delay calculations
- Writing owner permission checks
- Coordinating scheduled vs. manual nudges
- Tracking nudge history

### 6. Automatic User Information Synchronization

Cascade updates across all connections:

```typescript
AddresseeInformationService.updateAllForUser():
  - Triggered by user profile changes (name, photo, displayName)
  - Queries all connections where user is requester
  - Queries all connections where user is addressee
  - Batch updates requesterInfo or addresseeInfo fields
  - Chunks updates into batches (500 per batch)
  - Updates firstName, lastName, displayName, photoURL, isDisabled
  - Runs in parallel for requester and addressee sides

Batch processing:
  - Uses firestoreBatchChunk for optimal batch sizes
  - Parallel batch execution for performance
  - Atomic updates within each batch
  - Handles large connection lists efficiently
```

**What customers avoid:**

- Building cascade update logic
- Implementing batch processing for large datasets
- Managing bi-directional update coordination
- Writing efficient query and update patterns
- Optimizing for connection list size
- Coordinating parallel batch operations

### 7. Event-Driven Architecture with Automatic Notifications

Transparent pub/sub for connection lifecycle events:

```typescript
Cloud Functions automatically publish:
  - CONNECTION_CREATED → onCreate trigger
  - CONNECTION_UPDATED → onUpdate trigger
  - CONNECTION_DELETED → onDelete trigger

Topic subscriptions route to task queues:
  - Connection created → connection-created queue
  - Connection updated → connection-updated queue
  - Connection deleted → connection-deleted queue
  - User updated → user-updated queue
  - User disabled → user-deactivated queue
  - User re-enabled → user-reenabled queue
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database triggers
- Implementing pub/sub integrations
- Managing async task queues
- Coordinating event routing

### 8. Saga-Based Connection Notifications

Complex notification workflows with conditional logic:

```typescript
ConnectionSagas orchestrate:
  - Connection accepted → notify both users
  - Connection requested → notify addressee
  - User disabled → delete all pending requests
  - Status changed from pending to accepted → new connection notification

Notification types:
  - social_connection_invite: External invite email
  - social_connection_requested: Internal request notification
  - social_new_connection: Accepted connection celebration
  - social_connection_nudge: Reminder for pending invite

Saga coordination:
  - RxJS observable streams for event filtering
  - Conditional notification based on connection status
  - Automatic cleanup of disabled user requests
  - Idempotent command handlers
```

**What customers avoid:**

- Writing complex conditional notification logic
- Implementing saga patterns for workflows
- Building RxJS stream processing
- Managing notification type routing
- Coordinating multi-step async workflows
- Writing idempotent handlers

### 9. Automatic Pending Request Cleanup

Lifecycle management for disabled users:

```typescript
DeleteUserConnectionRequestsHandler:
  - Triggered when user is disabled
  - Queries pending connections (both requester and addressee)
  - Deletes all pending requests in parallel
  - Leaves accepted connections intact
  - Prevents disabled users from accumulating requests
  - Retry-safe with RetryCommandHandler
```

**What customers avoid:**

- Building user lifecycle hooks
- Implementing cascade deletion logic
- Writing status-based query filters
- Managing parallel deletion operations
- Coordinating with user service events

### 10. Connection Tagging with Atomic Array Operations

Organize connections with custom labels:

```typescript
Tag operations:
  - tagConnection: Adds tag with arrayUnion (auto-dedups)
  - unTagConnection: Removes tag with arrayRemove
  - Tags stored on opposite user's info (tagging perspective)
  - Transaction-wrapped for consistency
  - Supports multiple tags per connection
  - No duplicate tags (arrayUnion handles deduplication)

Tagging logic:
  - Determines which userInfo field to update (opposite side)
  - Uses dot notation for nested field updates
  - Atomic array operations prevent race conditions
  - Transaction ensures tag and connection exist during update
```

**What customers avoid:**

- Implementing atomic array operations
- Managing tag deduplication
- Writing perspective-based field selection
- Coordinating transaction-safe tag updates
- Building tag validation logic

### 11. Dynamic Link Generation for Mobile Apps

Cross-platform deep linking for invitations:

```typescript
DynamicLinkTokenEnricher provides:
  - API-based dynamic link creation
  - Android package configuration with minimum version
  - iOS bundle ID with App Store fallback
  - Domain URI prefix for branded links
  - Automatic link attachment to invite tokens
  - URL parsing for notification compatibility

Configuration:
  - DYNAMIC_LINK_API_KEY for authentication
  - DYNAMIC_LINK_URI_PREFIX for branded domains
  - Platform-specific app identifiers
  - Minimum version enforcement
  - App Store ID for iOS downloads
```

**What customers avoid:**

- Building dynamic link generation
- Implementing platform-specific deep linking
- Managing app version requirements
- Creating branded short links
- Coordinating mobile app routing
- Writing URL parsing logic

### 12. Comprehensive Error Handling with Typed Errors

Specific error types for client feedback:

```typescript
Error types:
  - SELF_CONNECTION: Cannot connect to yourself
  - REQUESTER_NOT_FOUND: Requester user missing
  - ADDRESSEE_NOT_FOUND: Addressee user missing
  - NOT_PARTICIPANT: Cannot edit non-participant connection
  - NOT_ADDRESSEE: Only addressee can accept
  - CONNECTION_NOT_FOUND: Connection ID invalid
  - INVITE_NOT_FOUND: Invite token invalid
  - NOT_INVITE_OWNER: Only owner can manage invite
  - ALREADY_CONNECTED: Duplicate connection attempt
  - USER_IS_DISABLED: Disabled user in connection flow
  - NUDGE_MISSING_EMAIL: Cannot nudge invite without email

Error enrichment:
  - Contextual error messages
  - Error type constants for client filtering
  - Additional data payloads for debugging
  - HTTP status code mapping
  - Silent error logging for async tasks
```

**What customers avoid:**

- Building typed error systems
- Implementing error type hierarchies
- Writing contextual error messages
- Managing error code constants
- Coordinating error enrichment
- Building async error handling

## Common Use Cases

- **Professional networking**: Connect with colleagues and industry peers
- **External user invitations**: Invite users via email before they join the platform
- **Contact organization**: Tag connections by relationship type, project, or interest
- **Team building**: Facilitate connections within organizations
- **Alumni networks**: Connect graduates with custom invitation flows
- **Conference networking**: Enable attendee connections with reminder nudges

## What Customers Don't Have to Build

- Bi-directional connection request handling
- Automatic mutual request detection and acceptance
- Token-based external invitation system
- Dynamic link generation for mobile deep linking
- Scheduled invite nudge reminders with deduplication
- Automatic user information synchronization across connections
- Privacy-aware progressive disclosure of personal information
- Transactional connection state management
- Connection tagging with atomic array operations
- Event-driven architecture with database triggers
- Saga-based notification orchestration
- Automatic cleanup of disabled user requests
- Comprehensive error handling with typed errors
- Batch processing for large connection lists
- Perspective-based view model transformations
- Token lifecycle management with expiration
- Self-connection prevention
- Duplicate connection detection
- Participant validation for all operations
- Integration with notification service
- User validation across connection lifecycle

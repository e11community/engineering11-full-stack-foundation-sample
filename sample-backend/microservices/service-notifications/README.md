# Notifications Service

Multi-channel notification delivery with templating, thresholding, and user preference management.

## What It Does

The Notifications Service handles all outbound notifications across email, push, in-app, and channel integrations (Slack, Teams). It provides template-based content generation, frequency thresholding, user preference enforcement, scheduled nudges, and transactional token management for push delivery.

## Key Capabilities

| Capability                      | Description                                                            |
| ------------------------------- | ---------------------------------------------------------------------- |
| **Multi-Channel Delivery**      | Email, push, in-app, Slack, Teams with unified configuration           |
| **Template Engine Integration** | DustJS-based template compilation and rendering with token replacement |
| **Frequency Thresholding**      | Per-entity, per-channel rate limiting to prevent notification spam     |
| **User Preference Enforcement** | Respects opt-in/opt-out preferences with non-opt-out overrides         |
| **Scheduled Nudges**            | Follow-up notifications with configurable day offsets and conditions   |
| **Device Token Management**     | Push token registration, validation, and automatic cleanup             |
| **Channel Notifications**       | Webhook-based delivery to Slack and Teams with formatted payloads      |

## How It Fits Together

```
┌─────────────────┐
│   Any Service   │
│ (triggers event)│
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  Notification Dispatch  │
│      (via Tasks)        │
└────────┬────────────────┘
         │
    ┌────┴──────┬─────────┬────────┐
    ▼           ▼         ▼        ▼
  Email       Push     In-App   Channels
(Template)  (Cloud    (Store)  (Webhooks)
           Messaging)
```

Services dispatch notification requests via task queues, and the service handles template rendering, threshold checking, user validation, and delivery.

## Heavy-Lifting Features

### 1. Configuration-Driven Multi-Channel Dispatch

Centralized notification configuration with per-channel control:

```typescript
INotification structure:
  - Unique ID (group_name format)
  - Group, name, and sentWhen metadata
  - Per-channel configuration (email, push, inApp)
  - Per-channel thresholds and enable/disable flags
  - Scheduled nudge definitions

NotificationDispatchService:
  - Sends to task queue with 2-minute deduplication window
  - Supports scheduled delivery via scheduleAt parameter
  - Automatic dedup key generation from request data
  - Separate queues for requests, resends, nudges
```

**What customers avoid:**

- Building multi-channel dispatch logic
- Implementing deduplication systems
- Managing scheduled notification queues
- Writing per-channel enable/disable logic

### 2. Saga-Based Multi-Channel Orchestration

CQRS saga pattern coordinates parallel channel delivery:

```typescript
NotificationSagas orchestrate:
  - SendEmailNotificationCommand (if email.notificationOn)
  - SendInAppNotificationCommand (if inApp.notificationOn)
  - SendPushNotificationCommand (if push.notificationOn)
  - SendNudgeNotificationCommand (if nudges defined)

Each saga:
  - Filters based on channel configuration
  - Fires independent command handlers
  - Runs in parallel for performance
  - Isolated failures per channel
```

**What customers avoid:**

- Implementing event-driven orchestration
- Building parallel execution logic
- Managing channel-specific command routing
- Coordinating saga-based workflows

### 3. Template-Based Content Generation

DustJS template engine with multi-layer token replacement:

```typescript
ContentPreparationService handles:
  - Template compilation with unique cache IDs
  - Template registration in engine cache
  - Token replacement from notification data
  - Tenant data injection (company name, platform name, year)
  - URL building from routes + tenant bootstrap host
  - Logo image URL replacement in HTML
  - Special character encoding/decoding

Template token types:
  - Data tokens: {{field}} from request data
  - Tenant tokens: {emailCompanyName}, {emailPlatformName}
  - Email tokens: {{messageText}}, {{callToActionText}}
  - Route tokens: URL generation from path + host
```

**What customers avoid:**

- Integrating template engines
- Writing template compilation logic
- Managing template caching
- Building token replacement systems
- Handling URL generation for multi-tenant apps

### 4. Transactional Frequency Thresholding

Per-entity, per-channel rate limiting with transactional queries:

```typescript
ThresholdService provides:
  - Separate thresholds for email, push, inApp, Slack, Teams
  - Entity ID override support for custom grouping
  - Default 60-second minimum threshold
  - Configurable per-notification thresholds

NotificationStoreRepository implements:
  - Transactional threshold checking
  - Query for most recent notification by entityId + notificationId
  - Time-based threshold comparison
  - Automatic store creation on threshold pass
  - Prevents duplicate sends within threshold window
```

**What customers avoid:**

- Building rate limiting systems
- Implementing transactional threshold checks
- Managing notification history storage
- Writing time-based comparison logic
- Preventing notification spam

### 5. User Preference and Validation Pipeline

Multi-layer validation with automatic enforcement:

```typescript
NotificationUserService handles:
  - User existence validation
  - Disabled user blocking
  - User preference fetching with caching
  - Non-opt-out notification whitelist
  - Tenant bootstrap resolution
  - Auth tenant validation

SendPushNotificationHandler checks:
  - Threshold validation
  - Suppress flag enforcement
  - User opt-out status
  - Push preferences enabled check

SendEmailNotificationHandler validates:
  - Email address availability
  - User or non-user request type
  - Email preferences enabled
  - Non-opt-out override application
```

**What customers avoid:**

- Building user validation pipelines
- Implementing preference enforcement
- Managing opt-out systems
- Writing disabled user checks
- Coordinating multi-tenant user data

### 6. Push Token Management with Automatic Cleanup

Device token lifecycle with validation and cleanup:

```typescript
TokenService provides:
  - Token registration with client type (Web, Mobile)
  - Upsert logic for token updates
  - Last-used timestamp tracking
  - Transaction-safe token removal

SendPushNotificationHandler orchestrates:
  - Query all tokens for user
  - Send via cloud messaging service
  - Collect failed token list
  - Delete failed tokens automatically
  - Update successful tokens with last-used timestamp
  - Increment badge count per user
```

**What customers avoid:**

- Building token management systems
- Implementing token validation
- Managing failed token cleanup
- Writing badge count tracking
- Coordinating cloud messaging integration

### 7. Scheduled Nudge System with Conditions

Follow-up notification scheduling with conditional evaluation:

```typescript
SendNudgeNotificationHandler:
  - Schedules task for each nudge in configuration
  - Calculates scheduledTime from offset in days
  - Sends to separate nudge queue
  - Marks request with nudge flag

NotificationsController filters:
  - Checks nudge conditions via injected handlers
  - Evaluates condition functions with nudge context
  - Blocks nudge if condition returns false
  - Supports custom condition logic per notification

Nudge configuration:
  - notificationId: ID of nudge notification config
  - offset: Days before sending (0 = immediate)
```

**What customers avoid:**

- Building scheduled notification systems
- Implementing conditional evaluation
- Managing follow-up workflows
- Writing day-offset calculation logic
- Coordinating nudge cancellation on events

### 8. HTML Email Generation with Tenant Branding

Multi-tenant email customization with template processing:

```typescript
EmailNotificationsService orchestrates:
  - Fetch email template by ID
  - Replace logo image placeholder with tenant URL
  - Inject messageText and callToActionText
  - Build URLs from routes + tenant host
  - Render templates with tenant data
  - Replace tenant tokens in subject and fromTitle

Template data includes:
  - Request data (dynamic notification content)
  - Tenant data (company name, platform name)
  - Current year for copyright
  - Generated URLs for call-to-action links
  - Email logo URL from tenant bootstrap
```

**What customers avoid:**

- Building multi-tenant email systems
- Managing email template repositories
- Implementing logo customization
- Writing URL generation for emails
- Handling tenant branding replacement

### 9. Channel Webhook Notifications

Structured notifications to Slack and Teams via webhooks:

```typescript
SlackNotificationService:
  - Uses Slack Block Kit format
  - Header block with subject
  - Section block with text
  - Divider for visual separation
  - Supports markdown formatting
  - Fetches webhook URL from secrets

TeamsNotificationService:
  - Uses legacy MessageCard format
  - Title and text fields
  - Schema.org extensions context
  - Fetches webhook URL from secrets

WebhookNotificationService base:
  - JSON body preparation
  - RestClient integration
  - URL parsing and origin extraction
  - Secret management via SecretsService
```

**What customers avoid:**

- Implementing Slack integration
- Building Teams integration
- Managing webhook secret storage
- Writing block formatting logic
- Handling webhook error cases

### 10. In-App Notification Storage with Badge Counting

Persistent in-app notifications with unread tracking:

```typescript
SendInAppNotificationHandler:
  - Creates IAppNotification document
  - Includes notification data + message info
  - Sets viewed: false for new notifications
  - Assigns unique appNotificationId
  - Links to notification type (config.id)
  - Increments user's in-app notification count

AppNotificationRepository:
  - Stores notifications in dedicated collection
  - Indexed by userId for efficient queries
  - Supports viewed status updates
  - Enables notification history retrieval

UserNotificationConfigRepository:
  - Tracks pushNotificationCount
  - Tracks inAppNotificationCount
  - Provides badge count for push delivery
```

**What customers avoid:**

- Building in-app notification storage
- Implementing badge counting
- Managing notification viewed status
- Writing notification history queries
- Coordinating count increments

### 11. Event-Driven Architecture with Database Triggers

Automatic event publishing via database lifecycle hooks:

```typescript
CloudFunctionGenerator creates:
  - APP_NOTIFICATION_CREATED (onCreate trigger)
  - NOTIFICATION_DELETED (onDelete trigger)

Event flow:
  - Database trigger fires on document change
  - Event published to topic
  - Topic handler routes to task queue
  - Task worker processes event

NotificationDeletedEvent triggers:
  - RemoveNudgeNotificationCommand
  - Cancels scheduled nudges for deleted notification
```

**What customers avoid:**

- Building database trigger infrastructure
- Implementing pub/sub event routing
- Managing topic subscriptions
- Writing event-to-task orchestration
- Coordinating cascade cleanup

### 12. Multi-Queue Task Processing

Specialized queues for different notification workflows:

```typescript
Queue structure:
  - NOTIFICATION_REQUEST: Primary notification dispatch
  - NOTIFICATION_RESEND_REQUEST: Resend with reduced threshold
  - NOTIFICATION_NUDGE_REQUEST: Scheduled follow-ups
  - NOTIFICATION_CHANNEL_REQUEST: Slack/Teams delivery
  - APP_NOTIFICATION_REQUEST: Direct in-app notifications
  - NOTIFICATION_TEST_REQUEST: Testing notifications
  - NOTIFICATION_DELETED: Cascade cleanup
  - USER_EVENTS: User lifecycle events

Queue benefits:
  - Independent scaling per queue
  - Isolated failure domains
  - Priority-based processing
  - Dedicated error handling
```

**What customers avoid:**

- Building task queue infrastructure
- Implementing queue routing logic
- Managing queue priorities
- Writing retry policies per queue
- Coordinating cross-queue workflows

### 13. Email Delivery via Mailgun

Transactional email sending with template tracking:

```typescript
EmailService:
  - Fetches Mailgun API key from secrets
  - Fetches Mailgun domain from secrets
  - NodeMailgun client initialization
  - Supports single or multiple recipients
  - Custom variables for template tracking
  - From email and from title configuration
  - Unsubscribe link control

Email parameters:
  - toEmail: Single address or array
  - fromEmail: Required by mail provider
  - fromTitle: Display name for sender
  - subject: Rendered from template
  - html: Fully rendered HTML content
  - templateId: Tracking variable (v:id)
```

**What customers avoid:**

- Integrating mail service providers
- Managing mail API credentials
- Building email sending abstraction
- Implementing multi-recipient handling
- Writing template tracking logic

### 14. REST API for Token Management

Public endpoints for device registration:

```typescript
TokenController endpoints:
  - POST /notifications-token: Register push token
  - POST /notifications-token/logout: Remove token

TokenService operations:
  - createToken: Upsert with client type and timestamp
  - removeUserFromToken: Transaction-safe removal
  - Validates userId matches before deletion

Authentication:
  - @RequireCustomClaim('appUserId')
  - Extracts userId from JWT claims
  - Validates user authorization per token
```

**What customers avoid:**

- Building token registration APIs
- Implementing transaction-safe token removal
- Writing authorization middleware
- Managing token-user association
- Coordinating logout cleanup

## Common Use Cases

- **Transactional emails**: Password resets, email verification, order confirmations
- **Push notifications**: Real-time alerts with badge counts and deep linking
- **In-app notifications**: Activity feed with viewed status tracking
- **Scheduled nudges**: Reminder emails after X days if action not taken
- **Team notifications**: Slack or Teams alerts for platform events
- **Multi-tenant branding**: Customized email templates per customer tenant

## What Customers Don't Have to Build

- Configuration-driven multi-channel dispatch with deduplication
- Saga-based orchestration for parallel channel delivery
- DustJS template engine integration with caching
- Multi-layer token replacement (data, tenant, URLs)
- Transactional frequency thresholding per entity and channel
- User preference enforcement with non-opt-out overrides
- Push token lifecycle management with automatic cleanup
- Failed token detection and removal
- Badge count tracking per user
- Scheduled nudge system with conditional evaluation
- HTML email generation with tenant branding
- Logo and URL customization per tenant
- Slack Block Kit formatting
- Teams MessageCard formatting
- Webhook secret management
- In-app notification storage with viewed tracking
- Database trigger event publishing
- Event-to-task queue routing
- Multi-queue task processing
- Mail provider integration with template tracking
- REST API for token registration with auth
- Transaction-safe token removal
- User validation pipeline with disabled user blocking
- Tenant bootstrap resolution
- Cloud messaging integration
- Email template repository management
- Notification configuration repository
- Notification history storage for rate limiting

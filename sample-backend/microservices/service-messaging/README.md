# Messaging Service

Real-time conversations with transactional consistency and flexible membership models.

## What It Does

The Messaging Service provides a complete conversation infrastructure with support for both direct messages and group chats. It handles message delivery, read tracking, emoji reactions, user mentions, and automatic participant synchronization with transactional consistency guarantees.

## Key Capabilities

| Capability                 | Description                                                              |
| -------------------------- | ------------------------------------------------------------------------ |
| **Flexible Conversations** | Fixed-membership (direct messages) and flexible-membership (group chats) |
| **Message Management**     | Send, edit, and delete messages with history preservation                |
| **Read Tracking**          | Per-participant last-read positions with unread count tracking           |
| **Emoji Reactions**        | Transactional reaction management with emoji validation                  |
| **User Mentions**          | Automatic mention parsing and validation with rich metadata              |
| **Real-Time Updates**      | SSE integration for instant message delivery                             |
| **Conversation Controls**  | Clear history, close conversations, block users                          |
| **System Messages**        | Send automated messages from system accounts                             |

## How It Fits Together

```
┌─────────────────┐
│   Messaging     │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┬─────────────┐
    ▼                 ▼              ▼             ▼
┌─────────┐   ┌─────────────┐  ┌──────────┐  ┌──────┐
│  User   │   │     SSE      │  │  Files   │  │Queue │
│ Service │   │   Service    │  │ Service  │  │Tasks │
└─────────┘   └─────────────┘  └──────────┘  └──────┘
```

Messaging uses User for participant information, SSE for real-time delivery, Files for attachments, and Queue Tasks for async operations.

## Heavy-Lifting Features

### 1. Deterministic Conversation ID Generation

Idempotent conversation creation for fixed-membership conversations:

```typescript
ConversationCreationService:
  - Generates deterministic IDs from participant list
  - Uses unordered hash (same participants = same ID)
  - Supports conversation keys for contextual grouping
  - Prevents duplicate conversations automatically
  - Validates unique participants
  - Enforces max participant limits
```

**What customers avoid:**

- Building conversation deduplication logic
- Implementing deterministic ID generation
- Managing conversation key contexts
- Writing duplicate prevention systems

### 2. Dual Membership Models

Flexible architecture supporting different conversation types:

```typescript
Fixed Membership (Direct Messages):
  - Participants cannot be added or removed
  - Deterministic conversation IDs
  - Perfect for 1-on-1 messaging
  - Supports conversation keys for context (e.g., support tickets)

Flexible Membership (Group Chats):
  - Dynamic participant management
  - Add/remove members at runtime
  - Random conversation IDs
  - Member information preserved after removal
```

**What customers avoid:**

- Building separate systems for DMs vs groups
- Implementing membership validation logic
- Managing conversation ID strategies
- Handling participant immutability

### 3. Transactional Read Position Tracking

Race-condition-free read tracking with temporal ordering:

```typescript
ConversationOverviewService.syncConversationOverview():
  - Updates lastRead position within transaction
  - Compares message timestamps to prevent stale updates
  - Only moves read head forward, never backward
  - Updates unread counts for all participants atomically
  - Increments unread for others, resets for sender
  - Updates latestMessage if newer than current

Temporal ordering prevents:
  - Out-of-order updates from network delays
  - Race conditions from concurrent message sends
  - Stale read positions overwriting newer ones
```

**What customers avoid:**

- Building transactional read tracking
- Implementing temporal ordering logic
- Managing concurrent update conflicts
- Calculating per-participant unread counts
- Handling race conditions in distributed systems

### 4. Message Edit History with Snapshot Preservation

Complete edit audit trail with limit enforcement:

```typescript
EditMessageService features:
  - Preserves original message text and mentions
  - Stores edit timestamp and previous version
  - Enforces configurable edit limit (default from config)
  - Validates edit window (time-based restrictions)
  - Guards prevent adding new attachments (only removal)
  - Updates conversation.latestMessage if edited message is latest
  - Transactional updates across message and conversation

Edit guards:
  - User must own the message
  - Conversation must not be closed
  - Edit limit not exceeded
  - Edit window not expired
  - Only message sender can edit
```

**What customers avoid:**

- Building edit history systems
- Implementing edit limit enforcement
- Managing edit time windows
- Writing transactional edit logic
- Coordinating conversation latestMessage updates

### 5. Smart Message Deletion

Context-aware deletion with automatic edit conversion:

```typescript
DeleteMessageService.delete():
  - Checks if user can delete (via ConversationGuard)
  - Hard deletes the message from database
  - Publishes MESSAGE_DELETED event for cleanup tasks

EditMessageService auto-delete:
  - If edit removes all text AND all attachments
  - Automatically deletes message instead of saving empty
  - Treats as deletion rather than invalid edit
```

**What customers avoid:**

- Building permission-based deletion
- Implementing auto-delete on empty edits
- Managing deletion vs edit logic
- Publishing cleanup events

### 6. Emoji Reaction System with Transactional Safety

Thread-safe reaction management with validation:

```typescript
ReactionService features:
  - Unicode emoji validation using \p{Emoji} regex
  - 100 unique reaction limit per message
  - Transactional add/remove operations
  - Automatic array management (arrayUnion/arrayRemove)
  - Empty reaction cleanup (deletes key when last user removes)
  - Prevents reactions in closed conversations
  - Validates user is participant before reacting

Transaction flow:
  - Read message state
  - Check reaction count limit
  - Update reactions atomically
  - Handle edge case: last user removing reaction deletes key
```

**What customers avoid:**

- Writing emoji validation regex
- Implementing transactional reactions
- Managing reaction user arrays
- Building reaction limit enforcement
- Handling concurrent reaction updates
- Cleaning up empty reaction keys

### 7. Rich User Mention Parsing

Automatic mention extraction with validation and enrichment:

```typescript
UserMentionParsingService.getMentionsForMessage():
  - Extracts user IDs via regex: /<@([a-zA-Z0-9]+)>/g
  - Validates each mentioned user exists
  - Filters out invalid mentions automatically
  - Enriches with participant information (name, photo, pronouns)
  - Returns structured mention metadata

MessagingUserMentionParsingService:
  - Validates mentioned user is in conversation
  - Conversation-aware mention validation
  - Integrates with participant information service

Mention metadata includes:
  - userId, displayName, firstName, lastName
  - photoURL, pronouns, isDisabled status
```

**What customers avoid:**

- Building mention parsing regex
- Implementing mention validation
- Writing user existence checks
- Enriching mentions with user data
- Managing conversation-specific validation

### 8. Automatic Participant Information Sync

Real-time participant updates across all conversations:

```typescript
ConversationSaga.updateParticipantInformation:
  - Triggers on UserUpdatedEvent
  - Monitors: displayName, firstName, lastName, photoURL, pronouns, isDisabled
  - Fetches latest participant information
  - Updates all conversations for that user
  - Batches updates (500 conversations per batch)
  - Updates messages with mentions atomically

UpdateParticipantInformationHandler:
  - Queries all conversations for participant
  - Supports conversation type filtering (except/only)
  - Runs batch updates for performance
  - Updates both conversation and mentioned messages
  - Uses MessageMentionRepository for efficient lookup
```

**What customers avoid:**

- Building cross-conversation sync systems
- Implementing batch update logic
- Managing user information propagation
- Writing mention update coordination
- Handling large-scale conversation updates

### 9. Conversation State Management

Comprehensive conversation lifecycle with validation:

```typescript
Conversation states:
  - closed: true/false (prevents new messages when closed)
  - clearHead: Per-user message ID (hides messages before this point)
  - closeReasons: Array of reasons for closure

ConversationCloseService:
  - Validates conversation exists
  - Sets closed flag to prevent messaging
  - Preserves message history

ConversationService.deleteForUser():
  - Sets clearHead to latest message ID
  - Hides messages from user's view
  - Does not actually delete messages
  - Configurable per conversation type (enableOneSidedConversationDelete)

Automatic closure triggers:
  - When only one active participant remains (others disabled)
  - Via ConversationSaga monitoring UserDeactivatedEvent
```

**What customers avoid:**

- Building conversation state machines
- Implementing soft deletion (clearHead)
- Writing automatic closure logic
- Managing per-user message visibility
- Coordinating closure triggers

### 10. Message Content Validation

Multi-layer validation with pluggable scanners:

```typescript
MessageValidationService:
  - Uses MessageTextScanner with pluggable functions
  - Default scanner: scanForBadUrls
  - Validates before message creation
  - Throws BadMessageTextError with reasons

MessageTextScanner features:
  - Configurable fail-on-first-issue behavior
  - Multiple scanner functions in sequence
  - Async scanner support
  - Accumulates failure reasons
  - Returns TextStatus.Passed or TextStatus.Failed

MessageCreationService validation:
  - Text or attachments required (not both empty)
  - User must be participant
  - Conversation must not be closed
  - User must not be disabled
  - ConversationGuard.userCanMessage() must pass
```

**What customers avoid:**

- Building content moderation systems
- Implementing URL scanning
- Writing validation orchestration
- Managing scanner plugins
- Creating validation error types

### 11. Conversation Guards and Permissions

Pluggable authorization system for conversation types:

```typescript
IConversationGuard interface:
  - userCanCreate(): Validate conversation creation
  - userCanAddOtherMembers(): Control member management
  - userCanMessage(): Validate sending permission
  - userCanEditMessage(): Control message editing
  - userCanDelete(): Control message deletion
  - editMessageConfigs: Optional editLimit and editWindow

DirectMessageGuard (default):
  - Only fixed-membership conversations allowed
  - No adding other members
  - Users can message if not disabled and conversation open
  - Users can edit/delete own messages only

Guard integration:
  - Keyed by conversation type
  - Injected via @ConversationGuards() decorator
  - Used throughout message and conversation services
  - Enables custom conversation types with different rules
```

**What customers avoid:**

- Building permission systems
- Implementing authorization checks
- Writing role-based access control
- Creating pluggable guard architecture
- Managing conversation type variations

### 12. Conversation Side Effects and Providers

Extensible lifecycle hooks for custom behavior:

```typescript
Provider system:
  - IClientMetadataProvider: Build custom metadata on creation
  - IParticipantInformationProvider: Add custom participant data
  - IConversationSideEffect: Run logic after creation
  - IConversationOptionsProvider: Configure conversation behavior
  - IConversationGuard: Control permissions and validation

ConversationOptionsProvider:
  - autoDelete: Enable/disable TTL for messages
  - enableOneSidedConversationDelete: Control clearHead feature

Provider injection:
  - Keyed by conversation type
  - Registered via constants (ConversationSideEffects, etc.)
  - Loaded dynamically at runtime
  - Enables conversation type customization without code changes
```

**What customers avoid:**

- Building plugin systems
- Implementing lifecycle hooks
- Writing extensibility frameworks
- Managing provider registries
- Creating custom conversation types from scratch

### 13. Flexible Participant Management

Dynamic membership with validation and information preservation:

```typescript
ConversationParticipantsService:
  - addMembers(): Add users to flexible conversations
  - removeMembers(): Remove without deleting participant info
  - setMembers(): Replace entire participant list
  - updateParticipantInformation(): Update specific user info

Participant information preservation:
  - Keeps participantInformation after removal
  - Maintains historical context for old messages
  - Updates individual fields without overriding all data
  - Atomic field updates: 'participantInformation.{userId}.{field}'

Member validation:
  - Blocks operations on fixed-membership conversations
  - Validates users exist before adding
  - Updates MessageMentionRepository on info changes
  - Uses transactions for consistency
```

**What customers avoid:**

- Building dynamic membership systems
- Implementing participant history preservation
- Writing atomic field update logic
- Managing mention synchronization
- Handling membership type validation

### 14. Conversation Blocking System

User-level blocking with conversation creation prevention:

```typescript
ConversationBlocked model:
  - id: User ID
  - blockedUsers: Array of blocked user IDs

ConversationValidationService:
  - Checks if sender can create conversation
  - Validates via ConversationGuard
  - Prevents conversation creation with blocked users
  - Returns clear error messages

Block enforcement:
  - Checked before conversation creation
  - User-specific block lists
  - Bidirectional blocking possible
  - Does not affect existing conversations
```

**What customers avoid:**

- Building user blocking systems
- Implementing block validation
- Managing blocked user lists
- Preventing blocked conversations

### 15. System Message Support

Automated messaging from system accounts:

```typescript
SystemMessageService.sendSystemMessage():
  - Accepts systemUserId or uses defaultSystemUserId
  - Creates message with isSystemMessage flag
  - Updates conversation.latestMessage if newer
  - Transactional latestMessage update
  - Supports metadata and attachments

System message features:
  - Special senderId for system messages
  - Does not trigger normal validation
  - Does not affect participant read tracking
  - Used for automated notifications and updates
```

**What customers avoid:**

- Building system messaging infrastructure
- Implementing special message types
- Managing system user IDs
- Writing automated message logic

### 16. Event-Driven Architecture

Automatic pub/sub for all entity lifecycle events:

```typescript
Functions publish events:
  - MESSAGE_CREATED → onMessageCreate trigger
  - MESSAGE_UPDATED → onMessageUpdate trigger
  - MESSAGE_DELETED → onMessageDelete trigger (via topic)
  - CONVERSATION_CREATED → onConversationCreate trigger
  - CONVERSATION_UPDATED → onConversationUpdate trigger
  - CONVERSATION_DELETED → onConversationDelete trigger

Event flow:
  - Database trigger → Pub/Sub topic
  - Topic → Cloud Tasks queue
  - Queue → Task handler endpoint

MessageSaga:
  - notifyMessageSent: Publishes to SSE channel for real-time delivery

ConversationSaga:
  - updateParticipantInformation: Syncs user changes
  - userDisabled/userReenabled: Updates activation status
  - closeConversationIfUsersAreDisabled: Auto-closes when <2 active users

UserMentionedEvent:
  - Published for each mentioned user
  - Enables mention notifications
  - Extracted from MessageCreatedEvent
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database triggers
- Implementing pub/sub integrations
- Managing async task queues
- Coordinating event routing
- Building saga patterns

### 17. Real-Time SSE Integration

Private channel generation for live message delivery:

```typescript
ConversationController.getAccessTokenForConversation():
  - Validates user is participant
  - Generates private channel name: messageSentChannelName(conversationId)
  - Creates SSE auth token for client
  - Returns client-usable token

NotifyMessageSentHandler:
  - Triggers on MessageCreatedEvent
  - Publishes message to SSE channel
  - Loads SSE module lazily (E11LazyLoader)
  - Delivers to all connected participants in real-time

Channel naming:
  - Convention: messageSentChannelName(conversationId)
  - Private channels require auth token
  - Per-conversation isolation
```

**What customers avoid:**

- Building real-time messaging infrastructure
- Implementing SSE channel management
- Writing auth token generation
- Managing channel naming conventions
- Coordinating message delivery

### 18. Attachment Management

File integration with validation and edit restrictions:

```typescript
Message attachments:
  - Array of IFileReceipt (from Files Service)
  - Supports multiple attachments per message
  - Preserved in edit history

Edit restrictions:
  - Can remove attachments during edit
  - Cannot add new attachments during edit
  - Validates new attachments are subset of old
  - Throws error if new attachment detected

Message validation:
  - Either text or attachments required (not both empty)
  - Empty message after edit triggers deletion
  - Attachments validated before save
```

**What customers avoid:**

- Building attachment validation
- Implementing edit restriction logic
- Managing file references
- Writing attachment subset checking

### 19. Message Confirmation IDs

Client-side message tracking for optimistic updates:

```typescript
IMessage.confirmationId:
  - Optional client-generated ID
  - Passed through from SendMessageModel
  - Preserved in created message
  - Enables client-side optimistic updates

Use case:
  - Client generates confirmationId before sending
  - Shows message immediately with confirmationId
  - Server returns message with same confirmationId
  - Client replaces optimistic message with real one
```

**What customers avoid:**

- Building optimistic update systems
- Implementing client-side message tracking
- Managing temporary message IDs

### 20. Built-In Caching Layer

Performance optimization with transaction-aware caching:

```typescript
ConversationService caching:
  - Uses NestJS cache manager
  - Keys conversations by conversationId
  - Bypasses cache during transactions
  - Cache invalidation on update

Cache operations:
  - get(): Check cache first, fallback to database
  - update(): Invalidate cache on update
  - Transaction-aware: Always read from DB in transactions

Cache note in code:
  - "lastRead may not be totally accurate in the cache"
  - Indicates read positions update frequently
  - Cache optimizes for conversation metadata, not real-time state
```

**What customers avoid:**

- Implementing cache invalidation logic
- Managing cache keys
- Coordinating cache with transactions
- Building transaction-aware caching

### 21. Conversation Metadata and Client Context

Extensible metadata system for application-specific data:

```typescript
Conversation fields:
  - clientMetadata: Unknown type (application-specific data)
  - type: String identifier for conversation type
  - conversationKeys: Optional context keys for grouping

IClientMetadataProvider:
  - build(): Generate metadata during creation
  - Keyed by conversation type
  - Enables custom data per conversation

Use cases:
  - Support ticket IDs
  - Group chat topics
  - Channel names
  - Application-specific context
```

**What customers avoid:**

- Building metadata systems
- Implementing extensible data models
- Managing conversation context

## Common Use Cases

- **Direct messaging**: One-on-one conversations with fixed participants
- **Group chats**: Dynamic group conversations with flexible membership
- **Support tickets**: Contextual conversations with conversation keys
- **Team collaboration**: Internal messaging with mentions and reactions
- **Customer support**: Direct communication between support and customers
- **Threaded discussions**: Message editing with full history preservation
- **Read receipts**: Per-conversation read tracking with unread counts
- **Real-time updates**: SSE integration for instant message delivery

## What Customers Don't Have to Build

- Deterministic conversation ID generation with deduplication
- Dual membership models (fixed vs flexible)
- Transactional read position tracking with temporal ordering
- Message edit history with snapshot preservation
- Smart deletion with auto-delete on empty edits
- Thread-safe emoji reaction management
- User mention parsing and validation
- Automatic participant synchronization across conversations
- Conversation state management (closed, clearHead)
- Multi-layer message content validation
- Pluggable conversation guards and permissions
- Extensible lifecycle hooks (providers and side effects)
- Dynamic participant management with history preservation
- User blocking system with creation prevention
- System message infrastructure
- Event-driven architecture with pub/sub
- Real-time SSE integration
- Attachment management with edit restrictions
- Message confirmation IDs for optimistic updates
- Transaction-aware caching layer
- Conversation metadata and client context systems
- Batch updates for large-scale operations
- Race condition prevention in concurrent updates
- Automatic closure triggers (disabled users)
- Per-conversation configurable behavior (options providers)

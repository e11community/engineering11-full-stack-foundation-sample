# Community Service

Structured group communication with channels, roles, and messaging.

## What It Does

The Community Service provides organized group spaces with role-based permissions, multi-channel discussions, member management, and moderation tools. It combines messaging infrastructure with access control to create flexible community environments.

## Key Capabilities

| Capability                 | Description                                                  |
| -------------------------- | ------------------------------------------------------------ |
| **Community Creation**     | Create communities with automatic default roles and channels |
| **Role-Based Permissions** | Custom roles with granular permission control                |
| **Channel Management**     | Multiple channels per community with positioning             |
| **Message Threading**      | Send, edit, and delete messages with mentions and reactions  |
| **Member Moderation**      | Mute, kick, and ban members with owner protection            |
| **Invite System**          | Token-based invites with expiration and usage limits         |
| **Read Tracking**          | Per-channel last-read positions for each member              |

## How It Fits Together

```
┌─────────────────┐
│   Community     │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┐
    ▼                 ▼              ▼
┌─────────┐   ┌─────────────┐   ┌──────────┐
│  User   │   │   Access    │   │Messaging │
│ Service │   │   Service   │   │  Engine  │
└─────────┘   └─────────────┘   └──────────┘
```

Communities use User for member information, Access for invite tokens, and Messaging for message infrastructure.

## Heavy-Lifting Features

### 1. Automatic Community Initialization

Create fully-configured communities in one operation:

```typescript
CommunityCreationService.create():
  - Creates community with default role system
  - Generates three built-in roles (Admin, Moderator, Everyone)
  - Creates default 'general' channel automatically
  - Adds owner as first member with full permissions
  - Initializes member count tracking
  - Rolls back automatically if any step fails
```

**What customers avoid:**

- Writing multi-step initialization logic
- Managing role creation and assignment
- Handling partial creation failures
- Coordinating channel and member setup

### 2. Dynamic Role-Based Permission System

Flexible permission model with runtime evaluation:

```typescript
Permission structure:
  - 10 granular permissions (ManageChannels, BanMember, DeleteMessage, etc.)
  - Custom roles with any permission combination
  - Owner bypass (automatic full permissions)
  - Locked roles (Admin, Moderator) cannot be deleted
  - Role inheritance through multiple role assignment

CommunityPermissionsService provides:
  - Single permission check
  - Multiple permission check (OR logic)
  - Full member permission resolution
  - Real-time permission aggregation across roles
```

**What customers avoid:**

- Building permission evaluation systems
- Writing role hierarchy logic
- Implementing permission aggregation
- Managing owner vs. member distinction

### 3. Intelligent Member Management

Comprehensive lifecycle handling with automatic side effects:

```typescript
Member operations:
  - Add: Validates against ban list, creates member, updates counts
  - Remove: Soft deletion preserves message history
  - Ban: Marks banned + removes from community
  - Kick: Removes without ban (can rejoin)
  - Mute: Prevents messaging but retains membership

Automatic side effects:
  - Member count increments/decrements
  - UserCommunities bidirectional mapping
  - Cache invalidation on updates
  - Owner protection on all removal operations
```

**What customers avoid:**

- Writing member lifecycle coordination
- Managing bidirectional relationships
- Implementing soft deletion logic
- Protecting privileged users from accidental removal

### 4. Message Validation and Mention Parsing

Multi-layered validation with automatic mention extraction:

```typescript
SendMessageService handles:
  - Content validation (text or attachments required)
  - Channel existence verification
  - Mute status checking with error feedback
  - Text validation for inappropriate content
  - Automatic mention extraction and validation
  - Last-read position updates
  - Rollback on failure (deletes message if side effects fail)

CommunityUserMentionParser provides:
  - Markdown mention syntax parsing (@username)
  - Member existence validation
  - Mention metadata enrichment (display name, profile picture)
  - Invalid mention filtering
```

**What customers avoid:**

- Building mention parsing systems
- Writing content validation logic
- Implementing mute enforcement
- Managing message creation transactions
- Coordinating last-read updates

### 5. Emoji Reaction System with Transactional Updates

Thread-safe reaction management:

```typescript
ReactionService features:
  - Emoji validation using Unicode property escapes
  - Transactional add/remove operations
  - 100 unique reaction limit per message
  - Automatic user array management
  - Empty reaction cleanup (deletes key when last user removes)
  - Mute enforcement on reactions

Transaction handling:
  - Read message state
  - Update reactions atomically
  - Prevent race conditions on concurrent reactions
```

**What customers avoid:**

- Writing transactional reaction logic
- Implementing emoji validation
- Managing reaction user arrays
- Handling concurrent reaction updates
- Building reaction limit enforcement

### 6. Secure Invite System with Token Management

Comprehensive invitation workflow:

```typescript
CommunityInviteService provides:
  - Token creation with expiration dates
  - Usage limit tracking
  - Dynamic link generation for mobile apps
  - Token validation with specific error types
  - One-time use consumption
  - Disabled invite handling
  - Cache invalidation on creation/deletion

Error handling:
  - TOKEN_EXPIRED → 400 "Invite has expired"
  - TOKEN_DISABLED → 400 "Invite is disabled"
  - TOKEN_USE_LIMIT_EXCEEDED → 400 "Use limit exceeded"
  - INVALID_TOKEN → 404 "Invite was not found"
```

**What customers avoid:**

- Building token generation and validation
- Implementing expiration logic
- Managing usage limits
- Creating deep links for mobile
- Writing error differentiation logic

### 7. Granular Last-Read Tracking

Per-channel read position management:

```typescript
LastReadService features:
  - Per-channel last-read timestamps
  - Transactional read position updates
  - Temporal ordering (prevents moving read head backward)
  - Automatic unread count derivation
  - Message timestamp comparison within transaction

Update logic:
  - Read current last-read position
  - Fetch last-read message to compare timestamps
  - Only update if new message is newer
  - Prevents stale updates from race conditions
```

**What customers avoid:**

- Building read receipt systems
- Implementing temporal ordering logic
- Managing per-channel state
- Preventing stale read position updates
- Calculating unread counts

### 8. Channel Positioning System

Sortable channel organization:

```typescript
ChannelPositionService:
  - Queries for highest position number
  - Auto-assigns next position to new channels
  - Supports manual reordering
  - Position-based channel listing

Channel operations:
  - Create with automatic position assignment
  - Reorder by updating position values
  - Query channels sorted by position
```

**What customers avoid:**

- Implementing manual ordering systems
- Writing position assignment logic
- Managing sort order calculations

### 9. Automatic Event-Driven Architecture

Transparent pub/sub for all entity changes:

```typescript
Cloud Functions automatically publish:
  - COMMUNITY_CREATED → onCreate trigger
  - COMMUNITY_UPDATED → onUpdate trigger
  - COMMUNITY_DELETED → onDelete trigger
  - COMMUNITY_MEMBER_CREATED → onCreate trigger
  - COMMUNITY_MEMBER_UPDATED → onUpdate trigger
  - COMMUNITY_MEMBER_DELETED → onDelete trigger
  - COMMUNITY_CHANNEL_CREATED/UPDATED/DELETED
  - COMMUNITY_MESSAGE_CREATED/UPDATED/DELETED

Events route to Cloud Tasks queues for async processing
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database triggers
- Implementing pub/sub integrations
- Managing async task queues
- Coordinating event routing

### 10. Background Task Orchestration

Async operations handled by task workers:

```typescript
Task handlers include:
  - Delete all channels when community deleted
  - Delete all members when community deleted
  - Delete all messages when channel deleted
  - Update community information across members
  - Remove community from user's community list
  - Sync default role changes across existing communities
  - Update search index on community changes

Saga coordination:
  - Multi-step workflows with rollback
  - Event-driven task chaining
  - Idempotent operation handling
```

**What customers avoid:**

- Writing cascade deletion logic
- Implementing saga patterns
- Building idempotent task handlers
- Managing multi-step async workflows

### 11. Built-In Caching Layer

Performance optimization with automatic invalidation:

```typescript
Caching strategy:
  - Community members cached by (communityId, memberId)
  - Community invites cached by communityId
  - Automatic cache invalidation on updates
  - Transaction-aware caching (bypasses cache in transactions)

Cache operations:
  - Get: Check cache first, fallback to database
  - Update: Invalidate cache entry
  - Batch update: Invalidate all affected cache entries
```

**What customers avoid:**

- Implementing cache invalidation logic
- Managing cache keys and namespaces
- Coordinating cache with database updates
- Building transaction-aware caching

### 12. Permission Guard Decorators

Declarative endpoint protection:

```typescript
Guards available:
  - @HasCommunityPermission(CommunityPermission.SendMessages)
  - @HasAnyCommunityPermission(Permission.A, Permission.B)
  - Automatic community ID extraction from URL
  - Owner bypass (owners always pass checks)
  - Member validation and permission resolution

Permission evaluation:
  - Extracts user ID from request
  - Parses community ID from URL pattern
  - Resolves all member roles
  - Aggregates permissions across roles
  - Returns 403 if insufficient permissions
```

**What customers avoid:**

- Writing authorization middleware
- Implementing permission checks in every endpoint
- Managing URL parsing for context extraction
- Building role aggregation logic

## Common Use Cases

- **Team collaboration**: Private communities with channels for different topics or projects
- **Customer support**: Public communities with moderation tools and role-based access
- **Learning groups**: Communities with custom roles for instructors, TAs, and students
- **Gaming guilds**: Communities with complex role hierarchies and channel organization
- **Professional networks**: Invite-only communities with granular permission control

## What Customers Don't Have to Build

- Multi-step community initialization with rollback
- Role-based permission systems with runtime evaluation
- Member lifecycle management with side effects
- Soft deletion preserving message history
- Mention parsing and validation
- Transactional emoji reaction management
- Token-based invite system with expiration and limits
- Dynamic link generation for mobile apps
- Per-channel read position tracking
- Temporal ordering for read receipts
- Channel positioning and ordering
- Event-driven architecture with pub/sub
- Background task orchestration with sagas
- Cascade deletion across related entities
- Multi-layer caching with automatic invalidation
- Permission guard decorators for endpoints
- Mute enforcement across message and reaction operations
- Owner protection on all destructive operations
- Ban list validation on member addition

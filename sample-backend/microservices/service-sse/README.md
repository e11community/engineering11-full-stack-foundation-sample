# SSE Service

Real-time server-to-client event streaming with channel-based pub/sub.

## What It Does

The SSE (Server-Sent Events) Service provides real-time communication from backend services to connected clients through persistent WebSocket connections. It acts as a central message bus that receives events from any backend service and pushes them instantly to subscribed clients organized by channels.

## Key Capabilities

| Capability                 | Description                                                      |
| -------------------------- | ---------------------------------------------------------------- |
| **WebSocket Gateway**      | Persistent connections for bidirectional real-time communication |
| **Channel-Based Pub/Sub**  | Clients subscribe to named channels and receive targeted events  |
| **Private Channels**       | JWT-authenticated channels with token-based access control       |
| **Multi-Tenant Isolation** | Project-scoped channels prevent cross-tenant data leakage        |
| **API Key Management**     | Scope-based API keys for publisher authentication                |
| **Connection Tracking**    | Track client-channel-apiKey associations in distributed memory   |
| **Horizontal Scaling**     | Socket.IO adapter for multi-instance deployments                 |
| **Automatic Cleanup**      | Remove clients from channels when API keys expire or are deleted |

## How It Fits Together

```
┌─────────────────────────────────────┐
│      Backend Services               │
│   (Messaging, Notifications, etc)   │
└────────────┬────────────────────────┘
             │ publish events
             ▼
┌─────────────────────────────────────┐
│        SSE Service                  │
│     (WebSocket Gateway)             │
│   - Publisher Controller            │
│   - Client Gateway                  │
│   - Channel Management              │
└────────────┬────────────────────────┘
             │ push to channels
    ┌────────┼────────┬────────┐
    ▼        ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Client 1│ │Client 2│ │Client 3│ │Client 4│
│(Web)   │ │(Mobile)│ │(Desktop)│ │(IoT)   │
└────────┘ └────────┘ └────────┘ └────────┘
```

Backend services publish events to channels via REST API, and the SSE Service delivers them in real-time to all subscribed clients via WebSocket.

## Heavy-Lifting Features

### 1. WebSocket Gateway with Socket.IO

Production-ready WebSocket infrastructure:

```typescript
ClientGateway provides:
  - Connection lifecycle management
  - Error handling and reconnection logic
  - Event-driven architecture
  - CORS configuration for cross-origin clients
  - Connection logging and debugging

Gateway events:
  - handleConnection: Client connects
  - handleDisconnect: Clean up client state
  - listen: Subscribe to channel
  - remove: Unsubscribe from channel
  - event: Broadcast messages to channel
```

**What customers avoid:**

- Setting up WebSocket server infrastructure
- Implementing connection lifecycle management
- Writing reconnection and error recovery logic
- Configuring CORS for WebSocket connections
- Building event routing systems

### 2. Channel-Based Room Management

Organize clients by subscription channels:

```typescript
ClientRoomService:
  - Builds namespaced room names: projectId:channelName
  - Ensures multi-tenant isolation at room level
  - Socket.IO room joins/leaves

Room operations:
  - client.join(roomName): Add client to channel
  - client.leave(roomName): Remove client from channel
  - server.to(roomName).emit(): Broadcast to all in channel
  - server.in(socketId).fetchSockets(): Find specific client
```

**What customers avoid:**

- Building channel subscription logic
- Implementing multi-tenant room isolation
- Writing broadcast and targeting systems
- Managing room membership state

### 3. Private Channel Authentication

JWT-based access control for sensitive channels:

```typescript
Private channel workflow:
  1. Publisher generates JWT for specific channel
  2. Backend service returns token to authorized client
  3. Client includes token in listen request
  4. SSE verifies token signature and channel match
  5. Client joins channel only if valid

ClientChannelAuthPipe validates:
  - Channel name starts with "private" prefix
  - Auth token is present for private channels
  - JWT signature using project secret
  - Token channel matches requested channel
  - API key is valid and not expired

ProjectSecretService provides:
  - sign(): Generate 10-minute JWT for channel
  - verify(): Validate JWT signature and claims
  - Audience validation (CHANNEL_AUTH_AUDIENCE)
  - API key TTL validation
```

**What customers avoid:**

- Building JWT signing and verification
- Implementing channel authorization logic
- Managing token expiration
- Writing secure secret storage integration
- Coordinating authorization between services

### 4. Publisher API with Scope-Based Access

REST endpoints for backend services to publish events:

```typescript
PublisherController endpoints:
  POST /v1/publishers/events/publish
    - Requires ApiKeyScope.PublishEvent
    - Validates event structure (channelName + payload)
    - Extracts project ID from API key
    - Broadcasts to all channel subscribers

  POST /v1/publishers/sign
    - Requires ApiKeyScope.SignToken
    - Generates JWT for private channel access
    - Returns signed token for client use

API key validation:
  - @ApiKeyRequireScope decorator enforcement
  - Project ID (partitionId) extraction
  - Automatic 400/401 error responses
```

**What customers avoid:**

- Building publisher authentication
- Implementing scope-based authorization
- Writing API key validation middleware
- Managing multi-tenant partition logic

### 5. Distributed Connection Tracking

Track client-channel-apiKey associations across instances:

```typescript
ClientAuthManager maintains bidirectional mappings:
  - clientApiKey:socketId → Set[room:apiKeyRecordId]
  - channelApiKeyRecord:apiKeyRecordId → Set[socketId:room]

Operations:
  - addClientToRoom(): Store in both indexes
  - getByApiKey(): Find all clients using an API key
  - removeClient(): Clean up on disconnect
  - removeApiKey(): Revoke access for expired key

Storage implementation:
  - Redis-backed for distributed state
  - Dual storage enables lookups by socketId or apiKeyRecordId
  - Encodes composite keys: "room:apiKeyRecordId" and "socketId:room"
  - Automatic cleanup on disconnection
```

**What customers avoid:**

- Building distributed state management
- Implementing bidirectional indexing
- Writing Redis set operations
- Managing cleanup on disconnect
- Coordinating state across instances

### 6. Horizontal Scaling with Redis Adapter

Scale WebSocket connections across multiple instances:

```typescript
RedisIoAdapter configuration:
  - Creates Redis pub/sub clients for Socket.IO
  - Shares connection state across all instances
  - Routes messages to correct instance
  - Enables room broadcasts across servers
  - Project-scoped key prefixes for multi-tenancy

Scaling features:
  - Clients connect to any instance
  - Messages broadcast to all instances
  - Room membership synchronized
  - Connection state distributed
  - Zero-downtime deployments possible
```

**What customers avoid:**

- Building distributed WebSocket coordination
- Implementing Redis pub/sub for Socket.IO
- Writing cross-instance message routing
- Managing connection state synchronization
- Configuring multi-instance deployments

### 7. Automatic API Key Lifecycle Management

Clean up connections when API keys expire or are deleted:

```typescript
Event-driven cleanup workflow:
  1. Access service publishes ApiKeyDeleted or ApiKeyExpired event
  2. Task service receives event via endpoint
  3. Saga converts event to RejectApiKeyCommand
  4. Handler queues task in Redis Bull queue
  5. Consumer fetches all clients using that API key
  6. Gateway removes clients from all private channels
  7. System message sent to affected clients
  8. Bidirectional state cleaned up

RejectApiKeyConsumer handles:
  - Find all clientRoomAndKey records for API key
  - Group associations by socketId
  - Remove each client from associated rooms
  - Send system message explaining removal
  - Clean up both Redis indexes
```

**What customers avoid:**

- Building event-driven cleanup workflows
- Implementing saga patterns for async operations
- Writing Bull queue configuration
- Managing cross-service event routing
- Coordinating distributed cleanup operations

### 8. Client Event Model

Structured message format for clients:

```typescript
Event types:
  - event: Regular channel messages (IPublisherEvent)
  - system_message: Service notifications (ISystemMessage)

IPublisherEvent:
  - channelName: Target channel
  - payload: Arbitrary JSON data from publisher

ISystemMessage:
  - code: SystemMessageCode enum
  - message: Human-readable description
  - data?: Additional context (e.g., removed channels)

System message codes:
  - Success: Operation succeeded
  - RemovedFromChannel: Kicked due to expired API key
  - Error: Operation failed
```

**What customers avoid:**

- Designing message formats
- Implementing message type distinction
- Building error notification system
- Writing client notification logic

### 9. Publisher Client SDK

Server-side SDK for publishing events:

```typescript
PublisherClient features:
  - buildChannel(): Create public channel
  - buildPrivateChannel(): Create private channel
  - publishMessage(): Send event to channel
  - getClientAuthToken(): Generate JWT for private channel
  - Static build(): Auto-loads API key from secrets

Channel abstraction:
  - IChannel interface for public channels
  - IPrivateChannel extends with auth token generation
  - Prefix validation ("private" enforcement)
  - HTTP client wrapper with authentication
```

**What customers avoid:**

- Writing HTTP client for SSE API
- Building channel abstractions
- Implementing private channel logic
- Managing API key configuration
- Creating type-safe publisher interfaces

### 10. Connection Lifecycle Management

Handle client connect/disconnect with cleanup:

```typescript
ClientGateway lifecycle:
  - handleConnection(client): Log new connection
  - handleDisconnect(client): Clean up all state

Disconnect cleanup:
  1. Retrieve all channels client is in (by socketId)
  2. Remove from channelApiKey index for each room
  3. Delete entire clientApiKey collection for socketId
  4. Silent error handling to prevent cleanup failures
  5. Logging for debugging connection issues

Connection error handling:
  - engine.on('connection_error'): Log transport errors
  - WebSocket validation errors via BaseWsExceptionFilter
  - Graceful error messages to client
```

**What customers avoid:**

- Writing disconnect cleanup logic
- Implementing error recovery
- Managing partial failure scenarios
- Building connection debugging tools

### 11. Multi-Tenant Project Isolation

Prevent cross-tenant data leakage:

```typescript
Project isolation mechanisms:
  - API keys scoped to partitionId (project ID)
  - Room names prefixed with projectId
  - Redis keys prefixed with projectId
  - JWT claims include project context
  - Channel names scoped per project

Isolation layers:
  - Network: API key validation at gateway
  - Application: Room name namespacing
  - Storage: Redis key prefixes
  - Authorization: JWT project claims
```

**What customers avoid:**

- Implementing tenant isolation logic
- Building namespace management
- Writing cross-tenant validation
- Managing project-scoped state

### 12. Event-Driven Task Orchestration

Async workflows with sagas and queues:

```typescript
Architecture:
  - Task service receives events via HTTP endpoints
  - CQRS EventBus dispatches to sagas
  - Sagas convert events to commands
  - CommandHandlers queue tasks in Bull
  - Queue consumers execute async work
  - Results communicated via WebSocket

Task examples:
  - API key revocation workflow
  - Batch client removal
  - System message broadcasting

Bull queue features:
  - Redis-backed job persistence
  - Retry logic on failure
  - Job priority and delays
  - Distributed task processing
```

**What customers avoid:**

- Building saga orchestration
- Implementing task queue systems
- Writing retry logic
- Managing distributed job processing
- Coordinating async workflows

## Common Use Cases

- **Live notifications**: Push new notifications to users without polling
- **Chat applications**: Real-time message delivery across conversations
- **Collaborative editing**: Broadcast document changes to all editors
- **Live dashboards**: Stream metrics and updates to monitoring UIs
- **Gaming**: Synchronize game state across players
- **IoT updates**: Push configuration or data to connected devices
- **Activity feeds**: Real-time activity streams for social features
- **Order tracking**: Push status updates to customers in real-time

## What Customers Don't Have to Build

- WebSocket server infrastructure with Socket.IO
- Connection lifecycle and error handling
- Channel-based subscription management
- Room isolation and multi-tenant namespacing
- JWT-based private channel authentication
- Token signing and verification
- Publisher REST API with scope validation
- Distributed connection tracking with Redis
- Bidirectional client-channel-apiKey indexing
- Horizontal scaling with Redis adapter
- Cross-instance message routing
- Event-driven cleanup workflows
- API key expiration handling
- Saga patterns for async operations
- Bull queue task orchestration
- Client removal and notification
- Publisher client SDK with abstractions
- System message broadcasting
- Connection debugging and logging
- Project isolation at multiple layers
- CORS configuration for WebSocket
- Error recovery and silent failure handling

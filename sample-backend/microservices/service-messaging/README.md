# Messaging Service

In-app messaging and conversation management.

## What It Does

The Messaging Service provides real-time messaging capabilities within the platform — direct messages, group conversations, and threaded discussions. It handles message delivery, read receipts, and conversation management.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Direct Messages** | One-to-one messaging between users |
| **Group Conversations** | Multi-participant chat rooms and channels |
| **Threaded Replies** | Nested conversations within messages |
| **Mentions** | @mention users to notify them in conversations |
| **Read Receipts** | Track when messages are read by recipients |
| **Message History** | Searchable archive of all conversations |

## How It Fits Together

```
┌─────────────────┐     ┌─────────────────┐
│    Messaging    │────►│   SSE Service   │
│    Service      │     │  (real-time)    │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│  Notifications  │
│    Service      │
└─────────────────┘
```

The Messaging Service pushes real-time updates through SSE and can trigger push notifications for offline users.

## Common Use Cases

- **Team collaboration**: Internal messaging between employees
- **Customer support**: Direct communication between support and customers
- **Activity feeds**: Comment threads on content or activities
- **Contextual messaging**: Conversations attached to specific resources or workflows

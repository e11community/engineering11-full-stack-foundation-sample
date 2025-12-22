# SSE Service

Real-time event streaming to clients.

## What It Does

The SSE (Server-Sent Events) Service provides real-time communication from the server to connected clients. It maintains persistent connections and pushes updates as they happen — enabling live dashboards, notifications, and collaborative features.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Real-Time Push** | Push events to connected clients instantly |
| **Channel Subscriptions** | Clients subscribe to specific channels of interest |
| **Private Channels** | Authenticated channels for sensitive data |
| **Scalable Connections** | Handle thousands of concurrent connections |
| **Cross-Service Events** | Receive events from any backend service |
| **Connection Management** | Track and manage client connections |

## How It Fits Together

```
┌──────────────────────────────────────┐
│           All Services               │
│        (publish events)              │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│           SSE Service                │
│        (message broker)              │
└──────────────────┬───────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌────────┐    ┌────────┐    ┌────────┐
│Client 1│    │Client 2│    │Client 3│
└────────┘    └────────┘    └────────┘
```

Backend services publish events, and the SSE Service delivers them to subscribed clients in real-time.

## Common Use Cases

- **Live notifications**: Show new notifications without polling
- **Collaborative editing**: Real-time updates in shared workspaces
- **Live dashboards**: Streaming metrics and status updates
- **Activity feeds**: Real-time activity streams

# Events Service

Event publishing and subscription management.

## What It Does

The Events Service is the backbone of the platform's event-driven architecture. It handles publishing events from any service, routing them to subscribers, and ensuring reliable delivery across the system.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Event Publishing** | Publish events from any service |
| **Event Routing** | Route events to interested subscribers |
| **Subscription Management** | Register and manage event subscriptions |
| **Event History** | Query past events for debugging and replay |
| **Dead Letter Handling** | Manage failed event deliveries |
| **Event Schemas** | Enforce consistent event structures |

## How It Fits Together

```
┌──────────────────────────────────────┐
│           Events Service             │
│         (Central Event Bus)          │
└──────────────────┬───────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌────────┐    ┌────────┐    ┌────────┐
│ User   │    │ Notif  │    │ Audit  │
│Service │    │Service │    │Service │
└────────┘    └────────┘    └────────┘
```

All services publish to and subscribe from the Events Service, enabling loose coupling and reactive workflows.

## Common Use Cases

- **Cross-service coordination**: User created → send welcome email → log audit entry
- **Real-time updates**: Publish changes for live UI updates via SSE
- **Audit logging**: Subscribe to all events for compliance tracking
- **Async processing**: Trigger background jobs from domain events

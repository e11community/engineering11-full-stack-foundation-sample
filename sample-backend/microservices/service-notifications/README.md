# Notifications Service

Multi-channel notification delivery and management.

## What It Does

The Notifications Service handles all outbound notifications to users — email, push notifications, SMS, and in-app alerts. It provides templating, delivery tracking, preference management, and ensures notifications reach users through their preferred channels.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Multi-Channel Delivery** | Send via email, push, SMS, or in-app |
| **Template Management** | Reusable notification templates with variable substitution |
| **User Preferences** | Respect per-user channel and frequency preferences |
| **Delivery Tracking** | Track sent, delivered, opened, and failed notifications |
| **Batching & Digests** | Aggregate notifications to reduce noise |
| **Scheduling** | Send notifications at optimal times |

## How It Fits Together

```
┌─────────────────┐
│   Any Service   │
│ (triggers event)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Notifications  │
│    Service      │
└────────┬────────┘
         │
    ┌────┼────┬────┐
    ▼    ▼    ▼    ▼
  Email Push SMS  In-App
```

Services publish events, and the Notifications Service determines how and when to notify affected users.

## Common Use Cases

- **Transactional emails**: Order confirmations, password resets, verification emails
- **Push notifications**: Mobile alerts for time-sensitive updates
- **Activity notifications**: "Someone commented on your post"
- **Digest emails**: Daily or weekly summary of activity

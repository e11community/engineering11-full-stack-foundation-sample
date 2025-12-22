# SMS Service

SMS messaging and delivery.

## What It Does

The SMS Service provides dedicated SMS capabilities for the platform — sending text messages, handling delivery receipts, and managing SMS-specific concerns like character limits and carrier compliance.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **SMS Delivery** | Send text messages to mobile numbers |
| **Delivery Receipts** | Track message delivery status |
| **Template Support** | Pre-defined message templates with variable substitution |
| **Number Formatting** | Handle international number formats |
| **Opt-Out Management** | Respect unsubscribe requests and compliance requirements |

## How It Fits Together

```
┌─────────────────┐
│  Notifications  │
│    Service      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│   SMS Service   │────►│   SMS Provider  │
└─────────────────┘     └─────────────────┘
```

The SMS Service is typically invoked by the Notifications Service when SMS is the appropriate delivery channel.

## Common Use Cases

- **Two-factor authentication**: Send verification codes via SMS
- **Appointment reminders**: Time-sensitive alerts that need immediate attention
- **Order updates**: Shipping notifications and delivery alerts
- **Emergency notifications**: Critical alerts when other channels may not reach users

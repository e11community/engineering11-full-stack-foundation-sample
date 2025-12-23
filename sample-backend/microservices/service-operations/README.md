# Operations Service

Platform operations background tasks.

## What It Does

The Operations Service runs scheduled maintenance tasks.

## Key Capabilities

| Capability            | Description                          |
| --------------------- | ------------------------------------ |
| **Maintenance Tasks** | Run scheduled maintenance operations |

## How It Fits Together

```
┌─────────────────┐
│     Cloud       │
│   Scheduler     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Operations    │
│    Service      │
└────────┬────────┘
         │
         ▼
   ┌───────────┐
   │  API      │
   │ Gateways  │
   └───────────┘
```

The Operations Service has elevated access to gather metadata on all API Gateways in a project

## Common Use Cases

- **Gateway Warming**: Periodically send keep-alive requests to all API Gateways in a project, to keep them from going idle. Prevents latency spikes when another service hasn't been invoked for a long time.

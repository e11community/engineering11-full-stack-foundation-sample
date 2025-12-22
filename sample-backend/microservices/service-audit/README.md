# Audit Service

Activity logging and compliance tracking.

## What It Does

The Audit Service records all significant actions within the platform — who did what, when, and what changed. It provides the audit trail needed for compliance, debugging, and security analysis.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Action Logging** | Record user and system actions with full context |
| **Change Tracking** | Capture before/after states for data changes |
| **Query & Search** | Search audit logs by user, action, resource, or time |
| **Retention Policies** | Manage log retention based on compliance requirements |
| **Export** | Export audit data for external analysis or compliance |
| **Immutability** | Ensure audit records cannot be modified after creation |

## How It Fits Together

```
┌──────────────────────────────────────┐
│           All Services               │
│      (publish audit events)          │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│           Audit Service              │
│      (stores & indexes logs)         │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│         Long-term Storage            │
│      (compliance archive)            │
└──────────────────────────────────────┘
```

## Common Use Cases

- **Compliance reporting**: Generate audit reports for regulatory requirements
- **Security investigation**: Track suspicious activity patterns
- **Change history**: See who modified a record and when
- **Debugging**: Trace the sequence of events leading to an issue

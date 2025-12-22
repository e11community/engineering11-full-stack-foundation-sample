# Registration Service

User and organization onboarding workflows.

## What It Does

The Registration Service manages the onboarding journey for new users and organizations. It handles invitations, sign-up flows, verification steps, and the coordination needed to properly set up new accounts across the platform.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Invitation Management** | Send, track, and expire invitations to join the platform |
| **Sign-Up Flows** | Multi-step registration with validation and verification |
| **Email Verification** | Confirm email ownership before account activation |
| **Onboarding Orchestration** | Coordinate user, customer, and auth account creation |
| **Bulk Invitations** | Import and invite multiple users from spreadsheets or integrations |

## How It Fits Together

```
┌─────────────────┐
│  Registration   │
│    Service      │
└────────┬────────┘
         │
    ┌────┼────────────┐
    ▼    ▼            ▼
┌──────┐ ┌──────┐ ┌──────────┐
│ Auth │ │ User │ │ Customer │
└──────┘ └──────┘ └──────────┘
```

The Registration Service orchestrates account creation across Auth, User, and Customer services to ensure consistent onboarding.

## Common Use Cases

- **Employee invitations**: HR sends invitations, employees complete registration
- **Self-service sign-up**: Public registration with email verification
- **Trial onboarding**: Time-limited access with conversion tracking
- **Partner onboarding**: Multi-step verification for B2B relationships

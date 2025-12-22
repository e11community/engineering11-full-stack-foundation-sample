# User Service

Comprehensive user management for B2B, B2C, and B2B2C platforms.

## What It Does

The User Service handles the complete user lifecycle — from onboarding to segmentation to offboarding. It works in coordination with the Auth Service to manage identity while maintaining rich user profiles and organizational relationships.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **User Profiles** | Store and manage user information, preferences, and settings |
| **Bulk Ingestion** | Import users at scale with chunked processing, full audit trails, and rollback support |
| **Segmentation** | Organize users by traits (department, location, role) and associations (content, membership) |
| **Activity Tracking** | Record and query user activity for analytics and personalization |
| **Terms of Service** | Track acceptance of terms and policy versions per user |
| **Consortium Support** | Manage groups of customers for VARs, resellers, and partner networks |

## How It Fits Together

```
┌─────────────────┐     ┌─────────────────┐
│   Auth Service  │◄───►│   User Service  │
└─────────────────┘     └────────┬────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Customer │ │  Events  │ │ Content  │
              │ Service  │ │ Service  │ │ Service  │
              └──────────┘ └──────────┘ └──────────┘
```

The User Service publishes events when users are created, updated, or segmented — allowing other services to react to user changes.

## Common Use Cases

- **Employee onboarding**: Bulk import users from HR systems with automatic segmentation
- **Customer self-registration**: Create and manage end-user accounts with profile enrichment
- **Audience targeting**: Query users by traits and associations for personalized experiences
- **Compliance tracking**: Record terms acceptance and maintain audit history

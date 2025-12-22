# Customer Service

Organization and tenant management for B2B and B2B2C platforms.

## What It Does

The Customer Service manages the organizational entities that users belong to — companies, teams, departments, and hierarchies. In multi-tenant systems, a "customer" typically represents a paying organization with its own users, configuration, and data isolation.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Customer Profiles** | Store organization details, settings, and metadata |
| **Hierarchies** | Model parent-child relationships between organizations |
| **User Association** | Link users to customers with role-based membership |
| **Configuration** | Per-customer settings, feature flags, and customization |
| **Billing Context** | Track subscription tier, entitlements, and usage limits |

## How It Fits Together

```
┌─────────────────┐
│    Customer     │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│ Users │ │ Access│
│       │ │ Rules │
└───────┘ └───────┘
```

The Customer Service provides the organizational context that other services use to scope data and enforce permissions.

## Common Use Cases

- **Enterprise onboarding**: Create customer accounts with admin users and initial configuration
- **Reseller hierarchies**: Model VARs, distributors, and their sub-customers
- **Feature entitlements**: Enable or disable features based on customer subscription tier
- **White-labeling**: Store customer-specific branding and customization settings

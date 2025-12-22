# Access Service

Permission and authorization management.

## What It Does

The Access Service manages what users can do within the platform — defining permissions, grouping them into roles, and enforcing access policies. It provides the authorization layer that all other services use to make access decisions.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Permission Definitions** | Define granular permissions for actions and resources |
| **Role Management** | Group permissions into reusable roles |
| **Role Assignment** | Assign roles to users at global, customer, or resource levels |
| **Policy Evaluation** | Check if a user has permission to perform an action |
| **Permission Inheritance** | Support hierarchical permission structures |

## Permission Naming Convention

Permissions follow a consistent pattern for organization:

```
(scope/service)/(action)-(target)
```

Examples:
- `customer/view-client`
- `user/manage-all-customer-users`
- `product/edit-billing-details`

## How It Fits Together

```
┌─────────────────┐
│  Access Service │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│      All Services           │
│  (permission checks)        │
└─────────────────────────────┘
```

Every service queries the Access Service (or uses cached policies) to authorize user actions.

## Common Use Cases

- **Role-based access**: Admins can manage users, viewers can only read
- **Resource-level permissions**: User can edit only their own content
- **Customer-scoped roles**: Permissions that apply within a specific customer context
- **Feature gating**: Control access to premium features via permissions

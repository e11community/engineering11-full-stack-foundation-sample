# Auth Service

Authentication and identity management for multi-tenant platforms.

## What It Does

The Auth Service handles all aspects of user authentication — login, logout, password management, and identity verification. It integrates with identity providers and coordinates with the User Service to maintain the link between authentication identities and user profiles.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Authentication** | Email/password, social login, SSO, and multi-factor authentication |
| **Session Management** | Token generation, refresh, and revocation |
| **Password Management** | Reset flows, strength requirements, and history tracking |
| **Identity Linking** | Connect multiple auth providers to a single user profile |
| **Multi-Tenant Isolation** | Ensure users can only authenticate within their tenant context |

## How It Fits Together

```
┌─────────────────┐
│  Identity       │
│  Providers      │
│  (OAuth, SAML)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│   Auth Service  │────►│   User Service  │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│   All Other     │
│   Services      │
│  (via tokens)   │
└─────────────────┘
```

The Auth Service issues tokens that all other services validate to authorize requests.

## Common Use Cases

- **Multi-provider login**: Allow users to sign in with email, Google, or enterprise SSO
- **Secure password reset**: Email-based password recovery with expiring tokens
- **Account linking**: Connect social logins to existing email-based accounts
- **Session invalidation**: Force logout across all devices when security requires it

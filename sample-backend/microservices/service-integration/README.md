# Integration Service

Secure webhook ingestion and event routing for external payment systems.

## What It Does

The Integration Service receives webhooks from external payment providers, verifies their authenticity using cryptographic signatures, and routes validated events into the platform's event bus. It acts as the secure entry point for external system notifications, transforming third-party webhooks into first-class platform events.

## Key Capabilities

| Capability                | Description                                               |
| ------------------------- | --------------------------------------------------------- |
| **Webhook Verification**  | Cryptographic signature validation using provider secrets |
| **Raw Body Preservation** | Maintains raw request body for signature verification     |
| **Event Publishing**      | Routes verified webhooks to pub/sub topics                |
| **Secret Management**     | Secure retrieval of webhook secrets from secret store     |
| **Event Filtering**       | RxJS-based type filtering for downstream consumers        |
| **Public Endpoints**      | Unauthenticated endpoints for external webhook delivery   |

## How It Fits Together

```
┌──────────────┐
│   Payment    │
│   Provider   │ (webhook)
└──────┬───────┘
       │
       ▼
┌─────────────────┐
│  Integration    │  verifies signature
│    Service      │  publishes to topic
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Pub/Sub       │
│     Topic       │
└────────┬────────┘
         │
    ┌────┴─────┬─────────────┐
    ▼          ▼             ▼
┌──────┐  ┌──────┐     ┌──────────┐
│ Saga │  │ Saga │     │Background│
│  #1  │  │  #2  │     │  Tasks   │
└──────┘  └──────┘     └──────────┘
```

External webhooks are verified, transformed into platform events, and consumed by sagas and background tasks throughout the system.

## Heavy-Lifting Features

### 1. Cryptographic Webhook Verification

Multi-layered signature validation for webhook authenticity:

```typescript
StripeEventService.dispatchEvent():
  - Preserves raw request body (required for signature verification)
  - Retrieves webhook secret from secure secret store
  - Uses provider SDK to verify signature (constructEvent)
  - Validates event structure and integrity
  - Throws on verification failure (prevents processing invalid webhooks)
  - Only publishes events that pass cryptographic verification
```

**What customers avoid:**

- Implementing HMAC signature verification
- Managing raw body parsing for signature validation
- Writing webhook verification logic
- Handling signature timestamp validation
- Building replay attack protection

### 2. Secure Secret Retrieval with Fallback

Flexible secret management with multiple resolution strategies:

```typescript
Secret resolution logic:
  - Primary: Direct secret value (stripeWebhookSecret)
  - Fallback: Secret name lookup (stripeWebhookSecretName)
  - Async retrieval from secret management service
  - Error handling with specific error types
  - Configuration validation on startup

Error handling:
  - Missing secret → "stripe/bad-config" error
  - Invalid configuration → Fails fast at initialization
  - Secret access failure → Detailed error message
```

**What customers avoid:**

- Building secret management integration
- Implementing secret resolution fallback logic
- Writing secret caching mechanisms
- Managing secret rotation handling
- Coordinating secret access across environments

### 3. Raw Body Preservation for Signature Validation

Critical request handling for webhook verification:

```typescript
Webhook controller:
  - @Public() decorator bypasses authentication
  - RawBodyRequest<Request> preserves original request body
  - Extracts stripe-signature header from request
  - Validates body exists before processing
  - Returns specific error for missing body
  - Passes raw buffer to verification service

Request validation:
  - Checks for req.rawBody presence
  - Throws "stripe/bad-webhook-request" if missing
  - Provides clear error messaging
  - Prevents processing of incomplete requests
```

**What customers avoid:**

- Configuring body parser middleware correctly
- Preserving raw body alongside parsed body
- Managing middleware execution order
- Implementing header extraction logic
- Writing request validation logic

### 4. Pub/Sub Event Publishing with Type Wrapping

Seamless event routing to platform event bus:

```typescript
Event publishing:
  - Wraps native provider events in platform event format
  - Uses buildEvent() for consistent event structure
  - Publishes to dedicated topic (integration-stripe-events)
  - Async publishing with error propagation
  - Logs verification and publishing lifecycle

Event structure:
  - StripeEvent class wraps native Stripe.Event
  - Maintains full event payload and metadata
  - Enables type-safe event handling
  - Preserves all provider-specific fields
```

**What customers avoid:**

- Building pub/sub integration
- Writing event transformation logic
- Implementing topic naming conventions
- Managing event serialization
- Coordinating async publishing

### 5. RxJS Event Type Filtering

Declarative event filtering for downstream consumers:

```typescript
isStripeEventOfType(type: string):
  - RxJS operator for event type filtering
  - Filters event stream by event.type field
  - Enables declarative saga composition
  - Type-safe event handling in consumers
  - Reduces boilerplate in event handlers

Usage in sagas:
  events$.pipe(
    isOfType(StripeEvent),           // Filter by class type
    isStripeEventOfType('customer.subscription.deleted'),  // Filter by event type
    map(({event}) => new Command())  // Transform to command
  )
```

**What customers avoid:**

- Writing event filtering logic
- Implementing type discrimination
- Building RxJS operator composition
- Managing event stream transformations
- Coordinating saga subscriptions

### 6. Automatic Cloud Function Topic Subscription

Generated subscribers for webhook event consumption:

```typescript
CloudFunctionGenerator.withPrefix('stripe'):
  - Generates pub/sub topic subscriber functions
  - Auto-wires topic name to function name (onStripeEvent)
  - Deploys as background function triggered by topic
  - Provides structured function exports
  - Enables async event processing

Function structure:
  - Topic: integration-stripe-events
  - Function name: stripe-onStripeEvent
  - Async handler with error handling
  - Automatic retry on failure
```

**What customers avoid:**

- Writing cloud function deployment configuration
- Managing topic subscription lifecycle
- Implementing function naming conventions
- Building retry and error handling
- Coordinating function deployment

### 7. Modular Integration Architecture

Extensible design for multiple payment providers:

```typescript
Module structure:
  - Integration-specific modules (IntegrationStripeModule)
  - Conditional module loading based on configuration
  - Dedicated controllers per integration
  - Isolated services per provider
  - Provider-agnostic base module

Configuration:
  - Per-integration options (stripe.stripeApiKey)
  - Optional integration activation
  - Secret name or value configuration
  - Environment-specific setup
  - Graceful handling of missing integrations
```

**What customers avoid:**

- Designing extensible integration architecture
- Implementing conditional module loading
- Writing provider abstraction layers
- Managing multi-provider configuration
- Building integration registry systems

### 8. Type-Safe Provider SDK Integration

Seamless integration with provider SDKs:

```typescript
StripeService integration:
  - Wraps official Stripe SDK
  - Provides dependency injection via STRIPE_SERVICE
  - Exposes stripe.webhooks.constructEvent()
  - Maintains full type safety with Stripe.Event
  - Enables testing with SDK mocking

SDK features leveraged:
  - Webhook signature verification
  - Event structure validation
  - Type definitions for all event types
  - Automatic timestamp tolerance checking
  - Built-in error handling
```

**What customers avoid:**

- Writing webhook verification from scratch
- Implementing timestamp tolerance logic
- Managing SDK version compatibility
- Building provider-specific type definitions
- Testing webhook verification logic

### 9. Comprehensive Error Handling with Typed Errors

Structured error responses for all failure modes:

```typescript
Error types:
  - stripe/bad-webhook-request → Missing request body
  - stripe/bad-config → Missing or invalid secret configuration
  - Stripe SDK errors → Signature verification failures

Error handling flow:
  - Request validation errors → 400 Bad Request
  - Configuration errors → Server startup failure
  - Verification errors → 400 Bad Request with details
  - Publishing errors → Propagated to caller
```

**What customers avoid:**

- Designing error taxonomy
- Implementing error type discrimination
- Writing error response formatting
- Managing error logging
- Building error documentation

### 10. Logging and Observability

Detailed event lifecycle logging:

```typescript
Logging points:
  - "Stripe Event Received" → Initial webhook receipt
  - "Stripe Event Verified" → Successful signature verification (includes event ID and type)
  - "Stripe Event Successfully Published" → Pub/sub publishing complete
  - Debug level for detailed event inspection

Logged metadata:
  - Event ID for tracing
  - Event type for filtering
  - Verification status
  - Publishing status
```

**What customers avoid:**

- Implementing structured logging
- Writing log aggregation logic
- Building event tracing
- Managing log levels
- Correlating webhook lifecycle

## Common Use Cases

- **Subscription management**: Process subscription lifecycle events (created, updated, canceled)
- **Payment processing**: Handle payment succeeded/failed notifications
- **Refund tracking**: Receive refund events and update order status
- **Customer updates**: Sync customer information changes from payment provider
- **Dispute handling**: Process chargeback and dispute notifications
- **Billing cycles**: Track billing period changes and invoice generation

## What Customers Don't Have to Build

- HMAC signature verification for webhooks
- Raw body preservation middleware
- Webhook secret management and rotation
- Signature timestamp validation
- Replay attack protection
- Pub/sub topic integration
- Event wrapping and transformation
- RxJS event filtering operators
- Cloud function topic subscription generation
- Provider SDK integration and testing
- Modular multi-provider architecture
- Conditional integration loading
- Type-safe event handling
- Error taxonomy for webhook failures
- Comprehensive webhook lifecycle logging
- Secret retrieval with fallback strategies
- Public endpoint configuration for webhooks
- Event routing to multiple consumers
- Saga-compatible event streaming

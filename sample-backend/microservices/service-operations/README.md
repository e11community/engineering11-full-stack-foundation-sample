# Operations Service

Platform infrastructure management for API warming, request telemetry, and scheduled operations.

## What It Does

The Operations Service provides critical infrastructure capabilities that keep the platform responsive and observable. It manages API gateway warming to reduce cold-start latency, captures request telemetry for analytics and debugging, and coordinates scheduled maintenance tasks across the platform.

## Key Capabilities

| Capability               | Description                                                         |
| ------------------------ | ------------------------------------------------------------------- |
| **API Gateway Warming**  | Automated keepalive requests to maintain low-latency response times |
| **Gateway Discovery**    | Multi-region gateway enumeration with pagination support            |
| **Request Telemetry**    | Automatic capture and publishing of HTTP request metadata           |
| **Scheduled Operations** | Cloud Scheduler-protected endpoints for maintenance tasks           |
| **Event Publishing**     | PubSub events for warming failures, successes, and exhaustion       |
| **Request Filtering**    | Extensible middleware chain for request metadata enrichment         |

## How It Fits Together

```
       ┌───────────┐
       │  Cloud    │
       │ Scheduler │
       └─────┬─────┘
             │
             ▼
    ┌─────────────────┐
    │   Operations    │
    │    Service      │
    └────────┬────────┘
             │
    ┌────────┴────────┐
    ▼                 ▼
┌──────────┐   ┌─────────────┐
│   API    │   │   PubSub    │
│ Gateways │   │   Topics    │
└──────────┘   └─────────────┘
```

The Operations Service discovers and warms API gateways across regions, publishes request events for analytics, and exposes scheduler-protected endpoints for maintenance operations.

## Heavy-Lifting Features

### 1. Automatic API Gateway Discovery with Pagination

Multi-region gateway enumeration with visitor pattern:

```typescript
ApiGatewayPaginator.visitAll():
  - Accepts project ID and location codes
  - Queries API Gateway service for all gateways
  - Handles paginated results automatically (25 per page)
  - Extracts gateway key from resource path
  - Visits each gateway with location context
  - Processes multiple locations sequentially
  - Resolves when all pages exhausted

Visitor pattern:
  - Gateway name extraction from full resource path
  - Default hostname capture for routing
  - Location code tracking for regional operations
  - Callback-based pagination handling
```

**What customers avoid:**

- Building gateway discovery infrastructure
- Implementing pagination logic for cloud APIs
- Managing multi-region resource enumeration
- Parsing resource paths for identifiers
- Coordinating async gateway queries

### 2. Scheduled Warmer Fanout with Error Tracking

Orchestrated warming across all discovered gateways:

```typescript
FanoutService.fanout():
  - Publishes warmer fanout started event
  - Discovers all gateways across configured locations
  - Enqueues Cloud Task for each gateway
  - Tracks failures with error messages
  - Publishes success or failure event with details

Fanout details tracking:
  - Discovered gateways array
  - Failures array with receive + error message
  - Locations to discover array
  - All-or-nothing success tracking

Task creation:
  - Retry with exponential backoff (3 attempts)
  - Custom X-E11-Warmer header for identification
  - HTTPS URL construction from gateway address
  - Queue targeting: operations-warmer-receive
```

**What customers avoid:**

- Building scheduled task orchestration
- Implementing multi-gateway coordination
- Writing exponential backoff retry logic
- Managing task queue operations
- Tracking partial failures across distributed operations

### 3. Cloud Tasks Integration with Service Account Auth

Secure task enqueueing with automatic authentication:

```typescript
TaskService features:
  - Lazy project ID resolution from client
  - Project number from configuration service
  - Service account email generation
  - Audience URL construction for Cloud Run
  - HTTP task creation with custom headers

Task configuration:
  - Parent: queue path in default location
  - HTTP method: GET for keepalive
  - URL: gateway address + warmer endpoint
  - Headers: X-E11-Warmer identification
  - Automatic service account attachment

Error handling:
  - Publishes warmer receive exhausted event
  - Returns undefined on failure
  - Allows fanout to continue despite individual failures
```

**What customers avoid:**

- Configuring Cloud Tasks client authentication
- Building service account email generation
- Implementing task creation with retries
- Managing queue path construction
- Handling partial enqueue failures

### 4. Request Telemetry Middleware with Event Publishing

Automatic HTTP request capture and publishing:

```typescript
RequestMiddleware.use():
  - Checks if publishing is available
  - Records request start time with hrtime
  - Initializes response.locals.extra for metadata
  - Runs request model filter chain
  - Attaches finish event handler
  - Non-blocking publish on response complete

RequestPublisher.publish():
  - Calculates request duration in milliseconds
  - Filters excluded headers (auth, forwarded-auth)
  - Extracts all request metadata (method, path, query, params)
  - Captures response status and content length
  - Includes custom extra metadata from filters
  - Publishes to operations-request-completed topic

Captured metadata:
  - baseUrl, originalUrl, url, path, routePath
  - method, protocol, statusCode, contentLength
  - ip, ips, userAgent
  - headers (filtered), query, params
  - durationMs, timestamp, sourceType
  - extra custom metadata
```

**What customers avoid:**

- Building request timing infrastructure
- Implementing non-blocking event publishing
- Managing header filtering for sensitive data
- Extracting comprehensive request metadata
- Coordinating middleware with response lifecycle

### 5. Extensible Request Filter Chain

Pluggable request metadata enrichment:

```typescript
Request filter execution:
  - Injected via REQUEST_MODEL_FILTERS provider
  - Executed in order before response
  - Can reject processing (breaks chain)
  - Failures logged but don't block original request
  - Access to req and res for metadata extraction

Filter interface:
  - use(req, res): boolean
  - Returns false to stop filter chain
  - Throws to stop chain with error
  - Modifies res.locals.extra for custom data

Error handling:
  - Filter rejection logged with warning
  - Filter failure caught and logged
  - Original request always proceeds
  - No latency impact from filter failures
```

**What customers avoid:**

- Designing extensible middleware systems
- Implementing chain-of-responsibility pattern
- Managing filter failure isolation
- Building error-tolerant enrichment pipelines
- Coordinating filter registration across services

### 6. Topic Reachability Checking with Graceful Degradation

Automatic topic validation before publishing:

```typescript
RequestPublisher.init():
  - Checks if REQUEST_COMPLETED topic exists
  - Sets internal canPublish flag
  - Warns if topic unreachable
  - Includes topic name in warning

Publishing guard:
  - Early return if topic unreachable
  - No middleware overhead when disabled
  - No request latency impact
  - Silent degradation in dev environments

Module initialization:
  - Async topic check on module init
  - Warning logged with error details
  - Includes missing topic name
  - Service continues operating normally
```

**What customers avoid:**

- Building PubSub topic validation
- Implementing graceful degradation logic
- Managing publishing availability state
- Writing environment-aware warning systems
- Coordinating async initialization checks

### 7. Cloud Scheduler Guard with Special Header Validation

Endpoint protection for scheduled operations:

```typescript
CloudSchedulerGuard.canActivate():
  - Checks for X-CloudScheduler or X-Appengine-Cron header
  - Logs warning if headers missing
  - Returns false to block unauthorized requests
  - Handles null/undefined request edge cases

Protected endpoints:
  - Warmer fanout task handler
  - Scheduled maintenance operations
  - Platform-wide coordination tasks

Security model:
  - Header-based authentication
  - Logged rejection attempts
  - No credentials in request body
  - Leverages platform-level security
```

**What customers avoid:**

- Building scheduler authentication
- Implementing header-based guards
- Managing authorized caller validation
- Writing endpoint protection logic
- Coordinating security with cloud services

### 8. Multi-Region Gateway Targeting

Location-aware gateway operations:

```typescript
Location configuration:
  - Configurable locations array
  - Default to Project.defaultLocation()
  - Per-location gateway discovery
  - Location code tracking in events

Gateway address resolution:
  - Extracts defaultHostname from gateway
  - Constructs HTTPS URLs for tasks
  - Appends warmer receive endpoint path
  - Stores location context for debugging

Fanout details:
  - Locations to discover array
  - Per-gateway location tracking
  - Failure association with location
  - Success reporting per region
```

**What customers avoid:**

- Building multi-region infrastructure
- Managing location-aware operations
- Implementing region failover logic
- Coordinating cross-region tasks
- Tracking regional failure patterns

### 9. Warmer Event Publishing System

Comprehensive warming lifecycle events:

```typescript
Event types:
  - WarmerFanoutStarted: Fanout operation began
  - WarmerFanoutSuccess: All gateways enqueued
  - WarmerFanoutFailed: Partial or complete failure
  - WarmerReceiveExhausted: Gateway not responding

Event payloads:
  - FanoutStarted: Simple message
  - FanoutSuccess: Discovered gateways + failures
  - FanoutFailed: Details + failure reason
  - ReceiveExhausted: Gateway address + location

Error handling:
  - Silent logging on publish failure
  - No impact on warmer operation
  - Continues despite event failures
```

**What customers avoid:**

- Building event publishing infrastructure
- Designing warming lifecycle tracking
- Implementing failure event correlation
- Managing event payload schemas
- Writing event publication error handling

### 10. Request Path Exclusion Configuration

Configurable request capture filtering:

```typescript
Path exclusions:
  - /_ah/* (App Engine health checks)
  - /task/* (Task handler endpoints)
  - /tasks/* (Task handler endpoints)

Middleware configuration:
  - Applied globally to all routes
  - Exclusion patterns in constants
  - Frozen configuration object
  - Express route matching

Purpose:
  - Avoids capturing internal traffic
  - Reduces noise in request analytics
  - Prevents task recursion in telemetry
  - Optimizes event volume
```

**What customers avoid:**

- Building path filtering logic
- Managing exclusion pattern configuration
- Implementing middleware route exclusion
- Optimizing telemetry volume
- Preventing telemetry recursion

### 11. High-Resolution Request Timing

Precise duration measurement:

```typescript
Timing implementation:
  - process.hrtime() for start time
  - [seconds, nanoseconds] tuple format
  - Duration calculation on response finish
  - Millisecond precision conversion

Calculation:
  - hrtime() difference from start
  - seconds * 1000 + nanoseconds / 1000000
  - Rounded to nearest millisecond
  - Attached to published request model

Accuracy:
  - Nanosecond resolution capture
  - Wall-clock time measurement
  - Includes full middleware stack
  - End-to-end request duration
```

**What customers avoid:**

- Implementing high-resolution timing
- Managing timing precision across platforms
- Converting time formats for analytics
- Calculating accurate request durations
- Handling timing edge cases

### 12. Lazy Initialization with Caching

Efficient resource initialization:

```typescript
TaskService caching:
  - Project ID cached after first fetch
  - Project number cached from config service
  - Audience URL cached for tasks
  - Service account email generated once

Pattern:
  - Check cached value first
  - Async fetch if not present
  - Store in protected property
  - Return cached on subsequent calls

Benefits:
  - Reduces API calls
  - Improves fanout performance
  - Simplifies configuration management
  - Tolerates transient failures on first call
```

**What customers avoid:**

- Building lazy initialization patterns
- Managing cached configuration state
- Implementing async value caching
- Optimizing repeated API calls
- Coordinating initialization timing

## Common Use Cases

- **API cold start mitigation**: Scheduled warming prevents cold starts during traffic spikes
- **Request analytics**: Track API usage patterns, latency, and error rates
- **Debugging production issues**: Correlate requests with errors using captured metadata
- **Performance monitoring**: Measure endpoint response times and identify slow operations
- **Multi-region coordination**: Manage warming across geographically distributed gateways
- **Scheduled maintenance**: Execute platform-wide operations at specific intervals

## What Customers Don't Have to Build

- Multi-region API gateway discovery with pagination
- Scheduled warmer fanout with failure tracking
- Cloud Tasks integration with service account auth
- Request telemetry middleware with event publishing
- Extensible request filter chain with isolation
- Topic reachability checking with graceful degradation
- Cloud Scheduler guard with header validation
- Multi-region gateway targeting
- Warmer event publishing system
- Request path exclusion configuration
- High-resolution request timing with hrtime
- Lazy initialization with caching
- Exponential backoff retry logic
- Gateway address resolution and routing
- Custom header propagation for task identification
- PubSub event publishing with error tolerance
- Visitor pattern for gateway enumeration
- Request duration calculation in milliseconds
- Header filtering for sensitive data
- Non-blocking event publishing on response finish

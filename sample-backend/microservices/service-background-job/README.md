# Background Job Service

Long-running and parallelizable job execution.

## What It Does

The Background Job Service executes long-running operations that don't fit the request/response model — data migrations, bulk processing, report generation, and other compute-intensive tasks. It supports controlled parallelism for processing large datasets efficiently.

## Key Capabilities

| Capability              | Description                                      |
| ----------------------- | ------------------------------------------------ |
| **Job Execution**       | Run standalone jobs with configurable parameters |
| **Parallelism Control** | Scale job execution across multiple workers      |
| **Progress Tracking**   | Monitor job progress and completion status       |
| **Retry Handling**      | Automatic retry with backoff for failed jobs     |
| **Job History**         | Track past job executions and results            |
| **Scheduled Jobs**      | Run jobs on a schedule or trigger manually       |

## How It Fits Together

```
┌─────────────────┐
│   Any Service   │
│ (triggers job)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Background Job  │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│Worker │ │Worker │ ...
│   1   │ │   2   │
└───────┘ └───────┘
```

Jobs can be triggered by other services or scheduled, then executed by a pool of workers.

## Heavy-Lifting Features

### 1. Automatic Parallel Job Execution

Launch multiple job instances automatically without managing infrastructure:

```typescript
// Customer doesn't write parallelization logic - the SDK handles it
OrchestrationService.kickOffJob():
  - Accepts array of parameter objects
  - Generates unique UUID for each job instance
  - Launches N parallel container executions automatically
  - Passes parameters independently to each instance
  - Scales from 1 to thousands of parallel jobs
```

**What customers avoid:**

- Writing parallelization logic
- Managing worker pools
- Distributing parameters across instances
- Coordinating multiple execution environments

### 2. Real-Time Status Tracking with SSE

Monitor job progress with server-sent events:

- **Status Management**: Three-state system (InProgress, Completed, Errored) per job instance
- **Status Aggregation**: Individual job statuses roll up to orchestration completion
- **SSE Streaming**: Real-time updates when jobs transition states
- **Reactive Monitoring**: Observable-based completion detection

**What customers avoid:**

- Building polling infrastructure
- Managing WebSocket connections
- Implementing status aggregation logic
- Writing completion detection algorithms

### 3. Batch Result Aggregation

Automatic collection and partitioning of results from parallel jobs:

```typescript
OrchestrationService.getJobResults():
  - Waits for all jobs to complete
  - Partitions results into success/failure arrays
  - Reconstructs failure objects with original parameters
  - Returns: [successResults[], failureResults[] | null]
```

**What customers avoid:**

- Writing result collection logic
- Implementing failure tracking
- Correlating failures with original parameters
- Managing partial success scenarios

### 4. Error Handling and Failure Recovery

Built-in error capture and reporting:

- **Automatic Error Capture**: Catches job errors and updates status to Errored
- **Error Context Preservation**: Stores error with job name, orchestration ID, job ID
- **Parameter Association**: Links failures to the specific parameters that caused them
- **Graceful Degradation**: Failures don't block other parallel jobs

### 5. Container Execution Integration

Transparent orchestration of containerized job execution:

```typescript
CloudRunTaskService handles:
  - Creating task executions
  - Injecting environment variables (JOB_NAME, ORCHESTRATION_ID, JOB_ID)
  - Managing containerized job lifecycle
  - Service name mapping conventions
```

**What customers avoid:**

- Manual container orchestration
- Environment variable management
- Service discovery and routing
- Container lifecycle management

### 6. Event-Driven Completion Notifications

Automatic pub/sub events when jobs complete:

- Cloud Functions monitor orchestration updates via database triggers
- Detects when all jobs transition from InProgress
- Publishes `JobCompletedEvent` to pub/sub topics
- Tags events with customer identifiers for workflow continuation

**What customers avoid:**

- Building completion detection logic
- Implementing pub/sub integration
- Writing trigger functions
- Managing event routing

### 7. Metadata-Based Job Selection

Automatic job discovery and routing:

```typescript
@JobName('process_batch')
class ProcessBatchJob {
  async run(params: Record<string, string>) { ... }
}

// SDK automatically finds and executes the right job
runE11Job() discovers job by name using reflection metadata
```

**What customers avoid:**

- Writing job routing logic
- Maintaining job registries
- Implementing job class discovery
- Managing job name mappings

## Common Use Cases

- **Data migration**: Move or transform large datasets across 100+ parallel workers
- **Bulk operations**: Update thousands of records efficiently with automatic result aggregation
- **Report generation**: Generate complex reports asynchronously with real-time progress tracking
- **Data cleanup**: Archive or delete old data in batches with automatic failure recovery
- **Multi-stage processing**: Chain jobs together using completion events

## What Customers Don't Have to Build

- Parallel job execution infrastructure
- Worker pool management and scaling
- Status tracking and aggregation systems
- Server-sent events for real-time monitoring
- Result collection and partitioning logic
- Error handling and failure recovery
- Container orchestration and lifecycle management
- Job discovery and routing systems
- Completion detection and event publishing
- Parameter distribution across workers

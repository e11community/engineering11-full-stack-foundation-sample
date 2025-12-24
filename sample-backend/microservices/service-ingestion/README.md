# Ingestion Service

Enterprise-grade data pipeline orchestration for high-volume imports.

## What It Does

The Ingestion Service provides a complete ETL (Extract, Transform, Load) pipeline framework for importing data from external sources into the platform. It handles multi-protocol data fetching, format conversion, schema mapping, custom enrichment logic, staging, and database persistence — all with comprehensive error tracking, automatic chunking, and resume capability.

## Key Capabilities

| Capability                  | Description                                                               |
| --------------------------- | ------------------------------------------------------------------------- |
| **Multi-Protocol Fetching** | Pull data from FTP/SFTP, REST APIs, streaming endpoints, or cloud storage |
| **Format Conversion**       | Parse CSV, JSON, JSONL, XML, and EDI into normalized format               |
| **Schema Mapping**          | Transform data using declarative mapping specifications                   |
| **Custom Enrichment**       | Execute custom business logic in tasks or background jobs                 |
| **Staged Persistence**      | Write to database or cloud storage with transactional safety              |
| **Automatic Chunking**      | Process large files in configurable chunks (default 500 records)          |
| **Comprehensive Ledgers**   | Track every record through every pipeline stage with error details        |
| **Cron Scheduling**         | Automated pull ingestion with cron expressions                            |
| **File Drop Support**       | Automatic processing of files uploaded to cloud storage                   |
| **Pause/Resume**            | Stop and resume pipeline execution mid-flight                             |

## How It Fits Together

```
┌──────────────────────────────────────────────────────────┐
│                    Data Sources                          │
│  FTP/SFTP  │  REST API  │  Cloud Storage  │  File Drop  │
└───────────┬──────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────┐
│                   Ingestion Pipeline                      │
│                                                           │
│  1. Fetch    → Pull/Push data from source                │
│  2. Convert  → Parse to JSONL (CSV/XML/JSON → JSONL)     │
│  3. Map      → Transform fields using mapper specs        │
│  4. Enrich   → Execute custom business logic              │
│  5. Stage    → Prepare database operations                │
│  6. Persist  → Write to database in batches               │
│                                                           │
│  Each step: Chunks → Temp Storage → Error Tracking       │
└───────────┬───────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────┐
│                   Output Targets                          │
│         Database Collections  │  Cloud Storage            │
└───────────────────────────────────────────────────────────┘
```

## Common Use Cases

- **Supplier data feeds**: Automated daily imports from FTP servers with CSV product catalogs
- **HR system sync**: Scheduled user imports from REST APIs with field mapping and validation
- **Legacy migration**: One-time bulk data migration with custom enrichment logic
- **Partner integrations**: Real-time file drops triggering automatic processing pipelines
- **Multi-source aggregation**: Combining data from multiple sources into unified records

## Heavy-Lifting Features

### 1. Multi-Protocol Data Fetching

Unified abstraction layer for pulling data from various sources:

```typescript
Fetcher implementations:
  - SFtpFetcher: Secure FTP with SSH key or password auth
  - FtpFetcher: Standard FTP connections
  - HttpFetcher: REST API polling with token auth
  - HttpStreamFetcher: Streaming API consumption
  - LocalBucketFetcher: Cloud storage file copying
  - BucketDrop: Automatic file detection via storage triggers

Automatic handling:
  - Secret management (credentials stored in secrets service)
  - Connection pooling and retry logic
  - File pattern filtering (regex-based)
  - Most-recent-only mode for incremental imports
  - Source file deletion after successful fetch
  - Destination path construction with customerKey/sourceId
```

**What customers avoid:**

- Building FTP/SFTP client implementations
- Managing secure credential storage
- Writing connection retry logic
- Implementing file filtering and selection
- Coordinating file cleanup after processing
- Handling streaming data sources
- Building storage event triggers

### 2. Format Conversion Pipeline

Streaming converters normalize diverse formats into JSONL:

```typescript
Converter implementations:
  - CSVConverter: Stream parsing with configurable separators
  - JSONConverter: Array and object unwrapping
  - JSONLConverter: Pass-through with validation
  - XMLConverter: Tag-based data extraction
  - EDI support via format type enum

Streaming architecture:
  - Chain: FileStream → Parser → Transformer → Output
  - Memory-efficient chunked processing (default 500 records)
  - Automatic error file generation (error-{filename}.jsonl)
  - Line-by-line processing prevents memory overflow
  - PipelineWritable handles automatic file chunking
```

**What customers avoid:**

- Building streaming CSV parsers
- Implementing XML data extraction
- Writing memory-safe large file processors
- Creating chunked output file management
- Handling parse errors with context preservation
- Converting between arbitrary formats

### 3. Declarative Schema Mapping

Transform data using declarative specifications without code:

```typescript
JsonMapper integration:
  - Field renaming: source → destination
  - Nested object flattening/expansion
  - Type coercion (string → number, date parsing)
  - Default value injection
  - Conditional transformations
  - Array transformations

Mapper processing:
  - Streams JSONL input line-by-line
  - Applies IMapperSpec transformations
  - Separate error stream for failed mappings
  - Tracks source vs. processed item counts
  - Stores error items with original data + error message
```

**What customers avoid:**

- Writing transformation logic for every integration
- Building field mapping UIs
- Implementing type coercion logic
- Managing transformation error handling
- Tracking which records failed transformation
- Writing custom ETL code for each data source

### 4. Custom Enrichment Framework

Execute arbitrary business logic within the pipeline:

```typescript
EnrichmentStep base class:
  - handleRecord(record, ingestionLedger): Transform individual records
  - Optional parallel batch processing
  - Automatic error isolation per record
  - Supports both task and job execution modes

Enricher orchestration:
  - Streams through JSONL chunk files
  - Executes custom step logic on each record
  - Parallel batch mode for performance (configurable batch size)
  - Promise.allSettled ensures partial batch success
  - Error records written to separate error stream
  - Updates ledger with source/processed counts

Multiple enrichment steps per pipeline:
  - Sequential execution with output chaining
  - Each step isolated in separate temp files
  - Custom step IDs for tracking in ledger
```

**What customers avoid:**

- Building plugin architectures for custom logic
- Writing parallel batch processing frameworks
- Implementing error isolation mechanisms
- Managing sequential step execution
- Coordinating temp file storage between steps
- Tracking enrichment performance metrics

### 5. Two-Phase Staging and Persistence

Decouples business logic from database writes for safety:

```typescript
Staging phase:
  - StagingStep.handleRecord(): Business logic determines operations
  - Output.upsert() / Output.delete(): Declares intended operations
  - Operations written to temp files (not database)
  - Multiple outputs per staging step
  - FirestoreOutput and CloudStorageOutput implementations

Persist phase:
  - Reads staged operations from temp files
  - BaseBatchSink processes in configurable batches (default 1,000)
  - FirestoreSink: Parallel upsert/delete with error tracking
  - CloudStorageSink: Bulk object creation
  - Operation counts tracked by type (Create/Update/Delete/Error)
  - Source vs. processed chunk tracking

Transaction safety:
  - Staging failures don't corrupt database
  - Can retry persist step independently
  - Failed operations logged with error details
```

**What customers avoid:**

- Building two-phase commit systems
- Writing batch database operation logic
- Implementing operation replay on failure
- Managing transactional safety across steps
- Tracking operation types and counts
- Building idempotent write logic

### 6. Comprehensive Ingestion Ledgers

Full observability into every pipeline execution:

```typescript
IIngestionLedger tracks:
  - Pipeline state machine (Start → Fetch → Convert → Map → Enrich → Stage → Persist → Complete)
  - Per-step ledgers with timing, counts, and outcomes
  - Source files, destination files, completed files, error files
  - Source items vs. processed items at each step
  - Error counts and error messages
  - Start and end timestamps

Per-step ledgers include:
  - IFetchLedger: Source files fetched, download errors
  - IConversionLedger: Parse errors with line context
  - IMappingLedger: Transformation failures
  - IEnrichmentLedger[]: One ledger per enrichment step
  - IStagingLedger: Output logs with operation counts
  - IPersistLedger: Operation counts by type, chunk progress

Automatic ledger updates:
  - Each pipeline step updates its ledger section
  - transitionLedger() advances state machine
  - Database update triggers publish events
  - Real-time visibility into pipeline progress
```

**What customers avoid:**

- Building pipeline observability systems
- Designing ledger data models
- Implementing state machine logic
- Tracking error context across steps
- Writing audit trail infrastructure
- Creating progress monitoring UIs

### 7. Automatic File Chunking System

Prevents memory overflow and enables incremental processing:

```typescript
PipelineWritable stream:
  - Configurable chunk size (records per file)
  - Automatic file rotation when chunk limit reached
  - Sequential file naming: {basename}-1.jsonl, {basename}-2.jsonl
  - Files written to temp directories by pipeline state
  - Returns ITempFileOutput[] with bucket, path, and name
  - Supports streaming writes to cloud storage

Chunking throughout pipeline:
  - Convert step: 500 records/chunk
  - Map step: 500 records/chunk
  - Enrich step: 500 records/chunk
  - Persist step: 1,000 operations/batch

Benefits:
  - Large files (millions of records) processed without memory limits
  - Failed chunks don't invalidate entire import
  - Parallel processing potential (not yet implemented)
  - Storage-efficient temp file management
```

**What customers avoid:**

- Building streaming file chunking systems
- Managing memory limits for large files
- Implementing automatic file rotation
- Coordinating chunk metadata
- Writing incremental processing logic
- Handling chunk failure isolation

### 8. Cron-Based Scheduled Ingestion

Automated pull-based data fetching:

```typescript
Scheduled function (hourly):
  - Queries all pull-method feed configs
  - Parses cron expressions for each config
  - Compares lastRun time with cron schedule
  - Dispatches tasks for configs due to run
  - Updates lastRun timestamp on execution

FeedConfig includes:
  - method: Pull vs. Push
  - interval: Cron expression (e.g., "0 2 * * *" for 2 AM daily)
  - lastRun: Timestamp of most recent execution
  - Automatic tracking prevents duplicate runs
```

**What customers avoid:**

- Building cron scheduling infrastructure
- Implementing schedule tracking logic
- Writing duplicate execution prevention
- Managing scheduled task dispatching
- Building cron expression parsing

### 9. File Drop Event Processing

Automatic ingestion triggered by file uploads:

```typescript
Storage event trigger:
  - onObjectFinalized monitors storage bucket
  - FILE_DROP_REGEX: ^[^/]+/{customerKey}/{sourceId}/{fileName}$
  - Matches customer/source from path
  - Dispatches to task queue for processing

BeginFromFileDropHandler:
  - Extracts customerKey/sourceId from file path
  - Looks up IBucketDropFeedConfig
  - Validates file type matches expected format
  - Creates ingestion ledger
  - Skips Fetch step (already have file)
  - Transitions directly to Convert step
  - Copies file to ingestion bucket if needed
```

**What customers avoid:**

- Building storage event handlers
- Implementing path parsing logic
- Writing file type validation
- Coordinating bucket-to-bucket copying
- Managing automatic pipeline initialization
- Handling file organization conventions

### 10. Pipeline State Machine with Pause/Resume

Controllable pipeline execution:

```typescript
IngestionState enum:
  Start → Fetch → Convert → Map → Enrich → Stage → Persist → Complete
  Also: Paused, Cancelled

transitionLedger() logic:
  - Automatically advances to next state after step completion
  - Fetch step skipped for Push-method configs
  - Enrich step waits for all enrichment ledgers to complete
  - Updates database with new state

ExecutePipelineStepHandler:
  - Checks for Paused state before executing
  - Returns early if paused (doesn't queue next step)
  - Checks for Complete state (idempotency)
  - Executes current step based on ingestionState
  - Transitions to next state
  - Queues next step execution via Cloud Tasks

Resume capability:
  - Unpause ingestion ledger (set state from Paused to previous state)
  - Re-run ExecutePipelineStepCommand with last known inputFiles
  - Pipeline continues from where it stopped
```

**What customers avoid:**

- Building state machine implementations
- Writing pause/resume coordination logic
- Implementing idempotent step execution
- Managing step-to-step transitions
- Building manual pipeline control interfaces

### 11. Multi-Step Error Tracking

Granular error context at every pipeline stage:

```typescript
Error tracking per step:
  - errorCount: Total errors encountered
  - errors: Array of error messages
  - errorFiles: ITempFileOutput[] pointing to error JSONL files
  - completedFiles: Successfully processed files
  - incompleteFiles: Partially processed files

Error file format:
  {
    item: {original record data},
    error: "error message"
  }

Error isolation:
  - Fetch errors: Connection failures, auth errors
  - Conversion errors: Parse failures with line context
  - Mapping errors: Transformation failures with original record
  - Enrichment errors: Custom logic exceptions per record
  - Staging errors: Business rule violations
  - Persist errors: Database write failures

Error files stored separately:
  - error-{filename}.jsonl alongside successful output
  - Allows reprocessing of failed records independently
  - Full original record preserved for debugging
```

**What customers avoid:**

- Building multi-layer error tracking
- Designing error context data models
- Implementing error isolation per record
- Writing error file generation logic
- Creating error reprocessing workflows
- Building error analysis tools

### 12. Secret Management Integration

Secure credential handling for external sources:

```typescript
FetcherBaseModel integration:
  - buildFeedSecretName(customerKey, sourceId)
  - SecretsService.accessSecretDocument<T>(secretName)
  - Secrets stored encrypted in secrets backend
  - Never exposed in logs or responses

Secret types:
  - IFtpSecret: userName, password
  - IHttpSecret: token
  - Custom secret schemas per fetcher type

Automatic secret loading:
  - Fetcher init() loads secret before execution
  - Error thrown if secret not found
  - Secret scoped to customer/source combination
```

**What customers avoid:**

- Building secret storage systems
- Implementing secret encryption
- Writing credential management UIs
- Managing secret rotation
- Coordinating secret access across services

### 13. Event-Driven Architecture

Automatic event publishing for external coordination:

```typescript
Cloud Functions publish events:
  - INGESTION_LEDGER_UPDATED: On every ledger update
  - PIPELINE_COMPLETED: When pipeline reaches Complete state
  - Events routed to pub/sub topics
  - Downstream services subscribe to events

Topic handlers:
  - Execute cleanup tasks on completion
  - Trigger dependent pipelines
  - Send notifications on success/failure
  - Update dashboards with progress
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database trigger logic
- Implementing pub/sub integrations
- Managing event routing
- Coordinating event-driven workflows

### 14. Background Job Support for Heavy Enrichment

Scale-out enrichment processing:

```typescript
BaseEnrichJob for long-running logic:
  - Runs in background job service (isolated from main pipeline)
  - Receives chunk file reference as input
  - Downloads and processes records
  - Uploads results to new chunk file
  - Returns IEnrichmentJobResult with ledger + output file

Job orchestration:
  - EnrichmentCommand dispatches jobs for each chunk
  - Waits for all jobs to complete via orchestration
  - Aggregates results back into pipeline
  - Updates enrichment ledger with combined metrics

Benefits over task execution:
  - No 10-minute timeout limit
  - Isolated compute resources
  - Retry and failure handling
  - Progress tracking per chunk
```

**What customers avoid:**

- Building job orchestration systems
- Writing chunk-to-job mapping logic
- Implementing job result aggregation
- Managing long-running process infrastructure
- Coordinating job-to-pipeline communication

## What Customers Don't Have to Build

- Multi-protocol data fetching (FTP/SFTP/REST/Storage)
- Secure credential storage and retrieval
- Streaming file format converters (CSV/XML/JSON → JSONL)
- Memory-safe large file processing with chunking
- Declarative schema mapping and transformation
- Custom enrichment plugin architecture
- Two-phase staging and persistence for safety
- Comprehensive pipeline observability with ledgers
- Automatic file chunking and rotation
- State machine for pipeline progression
- Pause/resume pipeline control
- Cron-based scheduled execution
- Storage event-triggered processing
- Multi-step error tracking and isolation
- Error file generation for reprocessing
- Batch database operations with retry
- Event publishing for pipeline lifecycle
- Background job orchestration for heavy processing
- Secret management integration
- Idempotent step execution
- Progress tracking and monitoring infrastructure

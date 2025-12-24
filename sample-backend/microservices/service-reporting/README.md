# Reporting Service

Real-time data warehouse synchronization with secure SQL query execution.

## What It Does

The Reporting Service provides automatic synchronization from operational databases to analytical data warehouses, with a secure query API that supports permission-based column access, SQL injection prevention, and automatic multi-tenant filtering.

## Key Capabilities

| Capability                  | Description                                                      |
| --------------------------- | ---------------------------------------------------------------- |
| **Automatic Data Sync**     | Real-time replication of all database changes to warehouse       |
| **Dynamic View Generation** | Automatically creates and updates views from JSON documents      |
| **Secure Query API**        | SQL injection prevention with operation whitelisting             |
| **Permission-Based Access** | Granular column-level permissions with runtime schema generation |
| **Time-Series Tracking**    | Event-sourced history with timestamp and action tracking         |
| **Multi-Tenant Isolation**  | Automatic customer key filtering across all queries              |
| **Query Caching**           | 2-hour cache with hash-based invalidation                        |
| **SQL Abstraction**         | Accept query AST nodes instead of raw SQL strings                |

## How It Fits Together

```
┌──────────────────────────────────────┐
│       Database Events                │
│    (create/update/delete)            │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│      Reporting Functions             │
│    (capture all collections)         │
└──────────────────┬───────────────────┘
                   │
              ┌────┴────┐
              ▼         ▼
        ┌─────────┐  ┌────────┐
        │  Topic  │  │ Tasks  │
        └────┬────┘  └───┬────┘
             │           │
             └─────┬─────┘
                   ▼
        ┌──────────────────────┐
        │   Data Warehouse     │
        │  (tables + views)    │
        └──────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   Query API          │
        │ (secure execution)   │
        └──────────────────────┘
```

The service uses database triggers to capture all document changes, routes them through pub/sub topics to task queues, then syncs to the warehouse and automatically maintains denormalized views.

## Heavy-Lifting Features

### 1. Universal Database Change Capture

Automatic replication of all document operations:

```typescript
Cloud Functions monitor:
  - Any collection in the database (wildcard matcher)
  - All CRUD operations (create, update, delete)
  - Auth context for sender tracking
  - Event IDs for deduplication

Captured metadata:
  - collectionId: Source collection name
  - documentId: Full document path (supports subcollections)
  - item/before/after: Document data snapshots
  - timestamp: Event time for temporal ordering
  - senderId: User who triggered the change
  - eventId: Unique event identifier

Event routing:
  - onCreate → FIRESTORE_DOCUMENT_CREATED topic
  - onUpdate → FIRESTORE_DOCUMENT_UPDATED topic
  - onDelete → FIRESTORE_DOCUMENT_DELETED topic
  - Topics → Cloud Tasks queues for reliable delivery
```

**What customers avoid:**

- Writing database trigger infrastructure
- Implementing change data capture logic
- Managing event publication and routing
- Building retry and delivery guarantees
- Tracking auth context in database events
- Creating deduplication systems

### 2. Automatic Table and View Provisioning

Self-healing warehouse schema:

```typescript
BigQueryTaskService.ensureTableExists():
  - Creates dataset if missing (configurable region)
  - Creates table with standardized schema
  - Enables time partitioning by timestamp (daily partitions)
  - Handles race conditions when multiple tasks create simultaneously

Base table schema:
  - id: Document ID (supports nested paths)
  - data: JSON column for full document
  - timestamp: Event timestamp
  - action: CREATE/UPDATE/DELETE
  - event_id: Deduplication key

ensureLatestViewsExists():
  - Creates _latest view automatically
  - Uses ROW_NUMBER window function for deduplication
  - Returns most recent version of each document
  - Updates in real-time as data changes
```

**What customers avoid:**

- Writing schema creation logic
- Managing dataset provisioning
- Implementing time partitioning strategies
- Building deduplication views
- Handling concurrent table creation
- Coordinating schema across services

### 3. Dynamic Expanded View Generation

Intelligent schema discovery with automatic view updates:

```typescript
Expanded view system:
  - Extracts all keys from JSON documents
  - Creates typed views from JSON columns
  - Two variants per subcollection:
    - _expanded_raw: All historical records
    - _expanded_latest: Most recent per document

Dynamic SQL generation:
  - Discovers keys with JSON_KEYS function
  - Filters keys to valid SQL identifiers
  - Special handling for customerKey (forced string type)
  - Regex-based subcollection filtering
  - Generates column list from discovered schema

View update workflow:
  1. Check if view exists for document path
  2. Compare existing view schema to new keys
  3. If new keys found, schedule view update
  4. Merge schemas without data loss
  5. Execute EXECUTE IMMEDIATE to recreate view
  6. Maintain backward compatibility

Subcollection support:
  - Parses nested paths (users/123/posts/456)
  - Creates views per subcollection level
  - Regex matching for path filtering
  - Table naming: users__posts_expanded_raw
```

**What customers avoid:**

- Writing schema discovery logic
- Implementing dynamic view generation
- Managing nested collection structures
- Building schema merging algorithms
- Handling view versioning
- Creating subcollection filtering systems

### 4. SQL Injection Prevention with Operation Whitelisting

Multi-layer query validation:

```typescript
Security layers:
  1. Operation whitelist enforcement
  2. Table name validation
  3. Column access control
  4. SQL parsing and AST validation
  5. Parameterized query generation

operationWhiteListCheck():
  - Parses SQL AST to extract operations
  - Ensures all operations start with "select::"
  - Rejects INSERT, UPDATE, DELETE, DROP, etc.
  - Throws reporting/invalid-operation on violation

ensureOnlyAllowedTables():
  - Extracts table names from query AST
  - Validates against schema allowlist
  - Throws reporting/invalid-table-name on violation
  - Prevents access to unauthorized data

Query sanitization:
  - Accepts QueryNode AST instead of raw SQL
  - Generates parameterized queries
  - Removes wrapping quotes from parameters
  - Parses back to pure SQL without injection vectors
```

**What customers avoid:**

- Implementing SQL injection protection
- Writing query parsing and validation
- Building operation whitelisting systems
- Managing safe parameter handling
- Creating table access control
- Coordinating security layers

### 5. Permission-Based Schema Generation

Runtime column-level access control:

```typescript
Permission schema config:
  - Per-table column permissions
  - Column whitelists (no permissions required)
  - Permission-gated columns (require specific permissions)
  - Table visibility based on accessible columns

generateSchemaFromPermissions():
  - Takes user permissions array as input
  - Generates runtime schema for query validation
  - Includes whitelist columns automatically
  - Adds permission-gated columns if user has permission
  - Excludes tables with zero accessible columns
  - Supports OR logic (any listed permission grants access)

Schema application:
  - Schema passed to query builder
  - Enforced during AST parsing
  - Validates column access at build time
  - Prevents unauthorized data exposure
  - Works with complex joins and subqueries
```

**What customers avoid:**

- Building column-level permission systems
- Implementing runtime schema generation
- Writing permission evaluation logic
- Managing whitelist and permission combinations
- Creating dynamic access control
- Coordinating permissions across queries

### 6. Automatic Multi-Tenant Query Filtering

Transparent customer key isolation:

```typescript
ReportingGlobalConfig.apply():
  - Intercepts all queries before execution
  - Extracts customerKey from JWT claims
  - Injects WHERE clause automatically
  - Handles global filters across all queries

Automatic injection:
  - Prepends customerKey filter to conditions
  - Uses table name from query for column reference
  - Applied to primary table in FROM clause
  - Prevents cross-tenant data leakage
  - Transparent to query authors
  - Cannot be bypassed by user input

Global filter support:
  - Define filters applied to all queries
  - Combine with tenant filters
  - Support complex filter expressions
  - Maintain filter precedence
```

**What customers avoid:**

- Writing tenant isolation logic
- Remembering to add customer filters
- Risk of cross-tenant data exposure
- Implementing global filter systems
- Managing filter injection
- Building query interception middleware

### 7. Query Caching with Hash-Based Invalidation

Performance optimization with automatic cache management:

```typescript
Caching strategy:
  - Hash query AST for cache key generation
  - Order-agnostic hashing (unorderedArrays: true)
  - 2-hour TTL for cached results
  - Cache lookup before query execution
  - Automatic cache population on miss

Cache key generation:
  - Uses object-hash library
  - Considers full query structure
  - Includes filters, joins, ordering
  - Ignores array and object order
  - Produces stable cache keys

Cache flow:
  1. Format query with global filters
  2. Generate hash of formatted query
  3. Check cache for existing result
  4. Return cached data if found
  5. Execute query on cache miss
  6. Store result with TTL
```

**What customers avoid:**

- Implementing query result caching
- Writing cache key generation logic
- Managing cache TTLs
- Building cache invalidation strategies
- Handling cache lookup and population
- Coordinating cache with query execution

### 8. Event-Sourced Data Warehouse

Temporal data tracking with full history:

```typescript
Event sourcing approach:
  - All changes recorded as immutable events
  - Timestamp-based ordering
  - Action tracking (CREATE/UPDATE/DELETE)
  - Never deletes historical data
  - Soft deletes preserve audit trail

Data structure:
  - Base table: All events chronologically
  - _latest views: Current state reconstruction
  - Time partitioning for query performance
  - Event IDs for deduplication

Historical queries:
  - Point-in-time queries via timestamp filtering
  - Change tracking between time ranges
  - Audit trail for compliance
  - Rollback capability from history
```

**What customers avoid:**

- Implementing event sourcing patterns
- Building temporal query systems
- Managing audit trails
- Creating point-in-time snapshots
- Writing history preservation logic
- Coordinating soft deletion

### 9. Saga-Based Sync Orchestration

Reliable multi-step workflows with automatic retries:

```typescript
BigQuerySagas coordinate:
  - FirestoreDocumentCreatedEvent → BigQuerySyncCommand
  - FirestoreDocumentUpdatedEvent → BigQuerySyncCommand
  - FirestoreDocumentDeletedEvent → BigQuerySyncCommand

Saga pattern benefits:
  - RxJS observable-based event streaming
  - Automatic command generation from events
  - Failure isolation per sync operation
  - Retry logic via RetryCommandHandler
  - Idempotent operation handling

BigQuerySyncHandler workflow:
  1. Ensure target table exists
  2. Ensure _latest view exists
  3. Prepare row data with metadata
  4. Insert via streaming API
  5. Schedule expanded view updates if needed
  6. Retry on transient failures
```

**What customers avoid:**

- Implementing saga orchestration patterns
- Writing retry logic for failures
- Building idempotent handlers
- Managing multi-step workflows
- Coordinating event-driven operations
- Creating failure isolation

### 10. Managed Streaming Inserts

High-performance data ingestion:

```typescript
BigQuery Storage Write API:
  - Uses managed writer for efficient streaming
  - JSON writer for schemaless data
  - Proto descriptor conversion for type safety
  - Default stream for automatic committing
  - Connection management and cleanup

Insert optimization:
  - Bypasses traditional insert API
  - Lower latency for real-time sync
  - Higher throughput for batch operations
  - Automatic schema evolution support
  - Built-in retry and error handling

Data transformation:
  - Stringifies JSON data for storage
  - Converts timestamp formats
  - Handles nullable event IDs
  - Validates against table schema
```

**What customers avoid:**

- Implementing streaming ingestion
- Managing write API connections
- Building high-throughput insert systems
- Handling schema conversions
- Writing connection pooling logic
- Optimizing for low-latency writes

### 11. Flexible SQL AST Query API

Type-safe query construction without string concatenation:

```typescript
QueryNode structure supports:
  - SELECT with column lists or expressions
  - FROM tables or subqueries
  - JOIN operations (INNER, LEFT, RIGHT, FULL, CROSS)
  - WHERE filters with complex conditions
  - GROUP BY with aggregations
  - HAVING for post-aggregation filtering
  - ORDER BY for result sorting
  - LIMIT and OFFSET for pagination
  - WITH clauses (CTEs)
  - UNION operations
  - QUALIFY for window function filtering

QueryBuilder features:
  - Converts AST to SQL
  - Generates parameterized queries
  - Validates syntax during build
  - Supports nested subqueries
  - Handles complex expressions
  - Type-safe parameter binding

parseFromSQL():
  - Reverse operation: SQL → AST
  - Enables query manipulation
  - Allows query optimization
  - Supports query analysis
```

**What customers avoid:**

- Building SQL query builders
- Implementing AST parsing
- Writing SQL generation logic
- Managing parameterization
- Creating type-safe query APIs
- Handling complex query syntax

### 12. View Admin Service for Custom Views

Programmatic view management:

```typescript
BigQueryAdminService provides:
  - Upsert views (create or update)
  - Table metadata management
  - View query configuration
  - Existence checking
  - Atomic view updates

upsertView():
  - Checks if view exists
  - Creates new view if missing
  - Updates metadata if exists
  - Returns consistent API response
  - Handles concurrent updates

Use cases:
  - Creating aggregation views
  - Building materialized views
  - Setting up custom reporting views
  - Managing view permissions
  - Updating view definitions
```

**What customers avoid:**

- Writing view management APIs
- Implementing upsert logic
- Managing view metadata
- Building admin tooling
- Coordinating view updates
- Handling view existence checks

## Common Use Cases

- **Analytics dashboards**: Real-time metrics without impacting operational databases
- **Compliance reporting**: Event-sourced audit trails with full history
- **Customer analytics**: Tenant-isolated queries with automatic filtering
- **Time-series analysis**: Historical data with timestamp-based partitioning
- **Business intelligence**: Secure SQL access with permission-based column filtering
- **Data exports**: Generate reports from denormalized views without complex joins
- **Change tracking**: Monitor all database changes with action and timestamp metadata

## What Customers Don't Have to Build

- Universal database change data capture infrastructure
- Event routing and pub/sub integration
- Automatic data warehouse table provisioning
- Time-based partitioning strategies
- Dynamic view generation from JSON documents
- Schema discovery and merging algorithms
- Nested collection and subcollection handling
- SQL injection prevention systems
- Operation whitelisting and validation
- Table and column access control
- Permission-based schema generation
- Runtime column-level permissions
- Automatic multi-tenant query filtering
- Query result caching with hash-based keys
- Event-sourced data warehouse design
- Temporal query and point-in-time snapshots
- Saga-based sync orchestration
- Retry logic with idempotent handlers
- High-performance streaming ingestion
- Managed writer connection management
- SQL AST query builder and parser
- Type-safe query construction APIs
- Parameterized query generation
- View admin and metadata management
- Cache invalidation strategies
- Deduplication view creation
- Audit trail preservation

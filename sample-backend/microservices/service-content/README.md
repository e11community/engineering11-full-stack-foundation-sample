# Content Service

Enterprise content management with collaborative workflows, versioning, and publication control.

## What It Does

The Content Service provides structured content lifecycle management with multi-language support, role-based collaboration, review/approval workflows, snapshot versioning, and automated publishing. It combines flexible content storage with workflow orchestration to enable teams to create, review, approve, and publish content safely.

## Key Capabilities

| Capability                        | Description                                                             |
| --------------------------------- | ----------------------------------------------------------------------- |
| **Structured Content Management** | Store and organize content with custom types and language variants      |
| **Role-Based Associations**       | Owner, Editor, Reviewer, and Approver roles per content item            |
| **Review Workflows**              | Multi-reviewer approval with request tracking and completion detection  |
| **Approval Workflows**            | Multi-approver gate with automatic request generation                   |
| **Snapshot Versioning**           | Point-in-time snapshots with file cloning for rollback                  |
| **Publishing System**             | Transactional publishing with snapshot creation and log tracking        |
| **Multi-Language Support**        | Per-language document storage with language code routing                |
| **State Management**              | Active, Archived, Locked, and Deleting states with transition workflows |
| **Search Integration**            | Automatic indexing of published content for search                      |
| **Access Control**                | Permission checks for read, write, and admin operations                 |

## How It Fits Together

```
┌─────────────────┐
│   Content       │
│    Service      │
└────────┬────────┘
         │
    ┌────┴─────────────┬──────────────┐
    ▼                  ▼              ▼
┌─────────┐   ┌──────────────┐  ┌──────────┐
│  User   │   │    Files     │  │  Search  │
│ Service │   │   Service    │  │  Index   │
└─────────┘   └──────────────┘  └──────────┘
```

Content uses User Service for association metadata, Files Service for media cloning in snapshots, and Search indexing for published content discovery.

## Heavy-Lifting Features

### 1. Multi-Language Document Storage with Language Code Routing

Per-language document instances with automatic ID generation:

```typescript
ContentDocumentRepository structure:
  - Document path: content/{contentId}/document/{contentId}_{languageCode}
  - Default language: en-US
  - Automatic ID composition: contentId + underscore + languageCode
  - Each language stored as separate document
  - Query by contentId + languageCode for retrieval

Document operations:
  - create(): Generates composite ID, stores at computed path
  - get(contentId, languageCode): Fetches specific language variant
  - update(): Automatic path resolution from composite ID
```

**What customers avoid:**

- Building multi-language storage schemas
- Implementing language code routing logic
- Managing composite ID generation
- Writing language-specific query patterns
- Coordinating language variant updates

### 2. Role-Based Content Association System

Flexible four-role permission model with user metadata:

```typescript
Association roles:
  - OWNER: Full control (create, edit, delete, manage associations)
  - EDITOR: Can modify content and create drafts
  - REVIEWER: Can review content and provide feedback
  - APPROVER: Can approve content for publication

ContentAuthService provides:
  - allowContentRead(): Checks if user has any association (Owner/Editor/Reviewer/Approver)
  - allowContentCreate(): Validates Owner or Editor role for creation
  - allowContentUpdate(): Enforces Owner or Editor role for modifications
  - isCustomerAdmin(): Administrator bypass for all permission checks

Association structure:
  - userId, firstName, lastName for display
  - associationTypes array allows multiple roles per user
  - Stored in Map<string, IContentAssociation> on content summary
```

**What customers avoid:**

- Building role-based permission systems
- Writing permission evaluation logic
- Managing user metadata in associations
- Implementing admin bypass patterns
- Creating role hierarchy validation

### 3. Automatic Content Initialization with Owner Assignment

One-step content creation with automatic owner association:

```typescript
PostItemService.createDocument():
  - Fetches user metadata from User Service
  - Creates association Map with user as OWNER
  - Sets createdDate timestamp
  - Sanitizes contentId if present (removes from body)
  - Creates content summary with associations
  - Creates content document
  - Links owner bidirectionally

UserNotFoundError handling:
  - Validates user exists before creation
  - Returns 400 BadRequest if user not found
  - Prevents orphaned content items
```

**What customers avoid:**

- Writing user lookup and validation
- Managing owner assignment logic
- Implementing association initialization
- Coordinating summary and document creation
- Handling user-not-found edge cases

### 4. Snapshot System with File Cloning

Point-in-time versioning with embedded file duplication:

```typescript
ContentSnapshotService.snap():
  - Reads current document (or uses provided document)
  - Validates document has contentId and customerKey
  - Calls FilesService.cloneDocument() to duplicate embedded files
  - Creates snapshot with cloned document
  - Stores publishDate if provided
  - Returns snapshot with unique ID

Snapshot structure:
  - id: Unique snapshot identifier
  - contentId: Parent content reference
  - document: Full document copy with cloned files
  - created: Snapshot timestamp
  - name: Optional snapshot label
  - publishDate: Optional publication timestamp

SnapshotRepository:
  - Path: content/{contentId}/snapshot/{snapshotId}
  - getAllForId(): Returns all snapshots for content item
  - get(contentId, snapId): Fetches specific snapshot
  - Supports rollback by loading snapshot as current document
```

**What customers avoid:**

- Building snapshot storage systems
- Implementing file cloning for versioning
- Managing snapshot metadata
- Writing rollback logic
- Coordinating file duplication with documents

### 5. Review Workflow with Request Generation

Multi-reviewer collaboration with automatic request distribution:

```typescript
ReviewCreatedTaskService.process():
  - Fetches content summary to find associations
  - Filters associations for REVIEWER role
  - Creates snapshot before sending for review
  - Generates review request for each reviewer
  - Supports optional reviewerUserIds filter (subset of reviewers)
  - Updates review with reviewerCount and snapshotId
  - Returns number of reviewers notified

Review request tracking:
  - One request per reviewer
  - completed flag for tracking completion
  - completedOn timestamp when marked complete

Review completion detection:
  - Increments requestsCompleted when request completed
  - Compares requestsCompleted to reviewerCount
  - Raises CONTENT_REVIEW_COMPLETED event when all done
```

**What customers avoid:**

- Building reviewer discovery logic
- Implementing request generation and distribution
- Managing review completion tracking
- Writing snapshot-before-review logic
- Coordinating review state updates

### 6. Approval Workflow with Completion Detection

Multi-approver gate with automatic completion events:

```typescript
ApprovalTasks.created():
  - Finds all APPROVER associations from summary
  - Creates snapshot for approval review
  - Generates approval request for each approver
  - Supports optional approverUserIds filter
  - Updates approval with approverCount and snapshotId

ApprovalTasks.requestCompleted():
  - Increments requestsCompleted atomically
  - Returns updated approval

ApprovalTasks.updated():
  - Compares before.requestsCompleted to after.requestsCompleted
  - Detects when requestsCompleted === approverCount
  - Raises CONTENT_APPROVAL_COMPLETED event automatically
  - Returns true if approval completed

Transactional completion:
  - Repository.incrementRequestsCompleted() uses atomic increment
  - Prevents race conditions with concurrent approvals
  - Guarantees correct completion detection
```

**What customers avoid:**

- Building approver discovery and filtering
- Implementing atomic increment logic
- Writing completion detection algorithms
- Managing approval state transitions
- Coordinating event publishing on completion

### 7. Transactional Publishing with Summary Updates

Safe publication with snapshot creation and log tracking:

```typescript
PublishingTaskService.publish():
  - Creates or validates snapshot (snapshotId optional)
  - Updates publish log state to Published
  - Captures document timestamp from snapshot
  - Updates content summary with lastPublishLog
  - Uses transaction to prevent race conditions
  - Compares timestamps to set mostRecent publish
  - Sets firstPublishLog if first publication
  - Raises CONTENT_PUBLISHED event

Transaction logic:
  - summaryRepository.runTransaction()
  - Fetches current summary inside transaction
  - Compares lastPublishLog.documentTimestamp
  - Only updates if new document is newer
  - Prevents stale publishes from overwriting recent ones

Snapshot creation:
  - If snapshotId provided, validates it exists
  - If no snapshotId, creates new snapshot from current document
  - Associates snapshot ID with publish log
  - Stores publishDate on snapshot
```

**What customers avoid:**

- Building transactional publish logic
- Implementing timestamp comparison for stale prevention
- Managing snapshot creation during publish
- Writing first/last publish tracking
- Coordinating publish log updates

### 8. State Machine with Workflow Processors

State-based lifecycle management with custom processors:

```typescript
Content states:
  - ACTIVE: Normal operating state
  - ARCHIVED: Soft deletion, hidden but recoverable
  - LOCKED: Prevents modifications during processing
  - DELETING: Triggers cascade deletion workflow

StateChangedTaskService:
  - Detects state changes (before.state !== after.state)
  - Uses StateFactory to build processor for new state
  - Delegates to processor.process(before, after, timestamp)

State processors:
  - Delete processor: Triggers TaskService.delete() for cascade
  - Archive processor: (custom logic if needed)
  - Active processor: (restore logic if needed)
  - Locked processor: (lock enforcement if needed)

Cascade deletion (Delete processor):
  - Fetches full content item
  - Calls TaskService.delete(contentItem, timestamp)
  - Deletes all snapshots
  - Deletes all documents
  - Removes from search index
```

**What customers avoid:**

- Building state machine infrastructure
- Implementing state transition processors
- Writing cascade deletion logic
- Managing state change detection
- Coordinating cleanup workflows

### 9. Content Summary with Metadata Aggregation

Lightweight content metadata for listing and search:

```typescript
IContentSummary structure:
  - contentId, contentType, customerKey
  - title, description (for display)
  - state (active/archived/locked/deleting)
  - associations: Map<userId, IContentAssociation>
  - workspaceKey: Optional workspace grouping
  - reviewId: Active review reference
  - title_stamp, description_stamp: Update timestamps
  - modified: Last modification time
  - lastPublishLog: Most recent publication
  - firstPublishLog: First publication

SummaryService operations:
  - create(): Initializes summary with ACTIVE state
  - updateAssociations(): Updates association map
  - removeAssociation(): Deletes user association with FieldValue.delete()
  - archive(): Sets state to ARCHIVED
  - unArchive(): Restores to ACTIVE
  - lock()/unLock(): Manages locked state
  - markForDelete(): Sets DELETING state for cascade

Association queries:
  - getAllByAssociation(userId, customerKey, contentType?)
  - Finds all content where user has association
  - Filters by contentType if provided
```

**What customers avoid:**

- Building metadata aggregation systems
- Implementing summary update logic
- Managing association removal with field deletion
- Writing state transition methods
- Creating association query patterns

### 10. Automatic Document Update Tracking

Timestamp management with protected fields:

```typescript
ContentDocumentUpdateModel:
  - toPartialDocument(): Converts update model to partial document
  - removeNonUpdatableProperties(): Masks protected fields
  - buildObjectMask(): Prevents customerKey, contentType, revisionId, id, timestamps from updates

Protected fields (cannot be updated):
  - customerKey: Immutable tenant identifier
  - contentType: Immutable type classification
  - revisionId: System-managed revision tracking
  - id: System-generated identifier
  - modifiedDate, createdDate, publishedDate: System timestamps

PutItemService.updateDocument():
  - Validates update model is non-empty
  - Calls toPartialDocument() to sanitize
  - Sets modifiedDate to current timestamp
  - Merges contentId from URL parameter
  - Prevents contentId in body (security)
```

**What customers avoid:**

- Building field protection logic
- Implementing timestamp automation
- Managing immutable field enforcement
- Writing update sanitization
- Coordinating modification tracking

### 11. Automatic Event-Driven Architecture

Database triggers with pub/sub routing:

```typescript
Cloud Functions automatically publish:
  - CONTENT_DOCUMENT_CREATED → onCreate trigger
  - CONTENT_DOCUMENT_UPDATED → onUpdate trigger
  - CONTENT_DOCUMENT_DELETED → onDelete trigger
  - CONTENT_ITEM_UPDATED → summary onUpdate trigger
  - CONTENT_ITEM_DELETED → summary onDelete trigger
  - CONTENT_STATE_CHANGED → conditional on state change
  - CONTENT_REVIEW_COMPLETED → conditional on review completion

Event routing:
  - Database trigger → Pub/Sub topic
  - Topic → Cloud Tasks queue
  - Queue → Task endpoint
  - Automatic retry and idempotency

Conditional events:
  - sendToTopicIf() for conditional publishing
  - State change detection before publishing
  - Review completion detection before publishing
```

**What customers avoid:**

- Building database trigger infrastructure
- Implementing pub/sub routing
- Writing event detection logic
- Managing task queue integration
- Coordinating event retries

### 12. Background Task Orchestration

Async operations with task workers:

```typescript
Task handlers include:
  - Document added → Update search index
  - Document updated → Update search index
  - Document deleted → Remove from search index
  - Snapshot updated → Update search index with snapshot data
  - State changed → Execute state processor
  - Review created → Generate review requests
  - Review request completed → Increment completion count
  - Review updated → Detect and raise completion event
  - Approval created → Generate approval requests
  - Approval request completed → Increment completion count
  - Approval updated → Detect and raise completion event
  - Publish request → Create snapshot and publish
  - Published → Update summary with publish log

Task coordination:
  - Cloud Tasks for reliable async execution
  - Automatic retry on failure
  - Idempotent handlers
  - Error logging and monitoring
```

**What customers avoid:**

- Writing async task handlers
- Implementing retry logic
- Building idempotent operations
- Managing search index updates
- Coordinating workflow state machines

### 13. Search Index Management

Automatic indexing with snapshot support:

```typescript
ContentIndexService:
  - add(snapshot): Adds snapshot to search index
  - update(snapshot): Updates existing index entry
  - delete(documentId): Removes from search index

ContentService.documentToSearch():
  - Queries current document
  - Sends to AlgoliaIndex for indexing
  - Returns false if document not found
  - Logs debug message on failure

Index structure:
  - One index entry per language code
  - Uses snapshot.document.id as index key
  - Stores full snapshot data for search

Task integration:
  - Document created → Add to index
  - Document updated → Update index
  - Document deleted → Remove from index
  - Snapshot updated → Update index with snapshot
```

**What customers avoid:**

- Building search indexing integration
- Implementing per-language index management
- Writing index update coordination
- Managing index cleanup on deletion
- Coordinating snapshot-based indexing

### 14. Permission Guard with Customer Key Validation

Declarative authorization with automatic checks:

```typescript
ContentAuthService integration:
  - Injected into all controllers
  - Validates user claims against content summary
  - Checks customerKey match for tenant isolation
  - Evaluates role-based permissions

Permission checks:
  - allowContentRead(): Any association (Owner/Editor/Reviewer/Approver)
  - allowContentCreate(): Owner or Editor role, or matching customerKey
  - allowContentUpdate(): Owner or Editor role only
  - isCustomerAdmin(): Admin role with matching customerKey bypass

Controller patterns:
  - checkPermissionsAgainstSummary(): Fetches summary, validates access
  - Throws NotFoundException if summary not found
  - Throws UnauthorizedException if permission denied
  - Returns summary for further processing

Admin bypass:
  - Administrator role with matching customerKey
  - Bypasses all association checks
  - Enables cross-content administration
```

**What customers avoid:**

- Writing authorization middleware
- Implementing tenant isolation checks
- Building role-based access control
- Managing admin bypass logic
- Coordinating summary-based permissions

### 15. File Cloning Integration for Snapshots

Automatic file duplication during versioning:

```typescript
FilesService.cloneDocument():
  - Called during snapshot creation
  - Scans document for embedded file references
  - Duplicates each file in storage
  - Updates document with new file URLs
  - Returns document with cloned file references

Snapshot workflow:
  - Create snapshot request
  - Read current document
  - Call FilesService.cloneDocument(document)
  - Store snapshot with cloned document
  - Original files remain unchanged

Rollback workflow:
  - Load snapshot with cloned files
  - Apply snapshot document as current
  - Cloned files become active
  - No conflicts with current files
```

**What customers avoid:**

- Building file cloning logic
- Implementing embedded file detection
- Managing file storage duplication
- Writing rollback coordination
- Coordinating file URL updates

### 16. Transactional Summary Updates with Optimistic Locking

Atomic updates to prevent race conditions:

```typescript
ContentSummaryRepository.runTransaction():
  - Wraps operations in database transaction
  - getWithTransaction(): Reads within transaction
  - updateWithTransaction(): Updates within transaction
  - Automatic retry on conflict

Publish log update transaction:
  - Fetches summary inside transaction
  - Compares lastPublishLog.documentTimestamp
  - Only updates if new document is newer
  - Prevents concurrent publishes from overwriting
  - Atomic read-compare-write operation

Field deletion:
  - removeAssociation() uses FieldValue.delete()
  - Removes field from document atomically
  - Prevents partial deletion
```

**What customers avoid:**

- Implementing transaction logic
- Writing optimistic locking
- Managing atomic read-compare-write
- Building field deletion patterns
- Coordinating concurrent update handling

## Common Use Cases

- **Knowledge bases**: Multi-language help articles with review workflows and version control
- **Documentation systems**: Technical docs with approval gates before publication
- **Content publishing**: Marketing content with editor/reviewer/approver workflows
- **Training materials**: Learning content with role-based access and version snapshots
- **Compliance content**: Regulated content with approval tracking and audit logs

## What Customers Don't Have to Build

- Multi-language document storage with language code routing
- Role-based association system (Owner/Editor/Reviewer/Approver)
- Automatic owner assignment on content creation
- Snapshot versioning with file cloning
- Review workflow with request generation and completion tracking
- Approval workflow with atomic completion detection
- Transactional publishing with timestamp comparison
- State machine with cascade deletion
- Content summary metadata aggregation
- Automatic document update tracking with protected fields
- Event-driven architecture with database triggers
- Background task orchestration for async workflows
- Search index management with snapshot support
- Permission guards with customer key validation
- Admin bypass for cross-tenant administration
- File cloning integration for rollback support
- Transactional summary updates with optimistic locking
- Association queries by userId and contentType
- Publish log tracking (first and last publication)
- Review/approval request distribution to role-based users

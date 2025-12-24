# Sync Service

Reliable data synchronization across database collections at scale.

## What It Does

The Sync Service manages data synchronization across database collections and documents, enabling bulk updates and deletions when source data changes. It handles denormalized data consistency, cascade operations, and large-scale batch processing through asynchronous task queues with built-in pagination and reliability.

## Key Capabilities

| Capability                    | Description                                                         |
| ----------------------------- | ------------------------------------------------------------------- |
| **Collection Patching**       | Bulk update all documents in a collection with new values           |
| **Collection Group Patching** | Update documents across all subcollections with same name           |
| **Document Patching**         | Update individual documents at specific paths                       |
| **Collection Deletion**       | Delete all documents in a collection via paginated batch processing |
| **Document Deletion**         | Delete individual documents at specific paths                       |
| **Paginated Processing**      | Process large collections in batches (default: 100 docs/page)       |
| **Task-Based Architecture**   | Reliable, asynchronous processing via task queues with retry logic  |

## How It Fits Together

```
┌─────────────────────────────────────────────────────┐
│              TaskService Entry Points               │
│  requestCollectionPatch, requestDocumentPatch,      │
│  requestCollectionDeletion, requestDocumentDeletion │
└───────────────────┬─────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │   Cloud Task Queue    │
         │  (Reliable delivery)  │
         └──────────┬────────────┘
                    │
                    ▼
    ┌───────────────────────────────────┐
    │  SyncCollectionPatchController    │
    │    (NestJS Task Processors)       │
    └───────────┬───────────────────────┘
                │
    ┌───────────┴──────────────────┐
    │                              │
    ▼                              ▼
┌─────────────────┐    ┌──────────────────────┐
│PatcherTask      │    │DeletionTask          │
│Generator        │    │Generator             │
│                 │    │                      │
│- Streams pages  │    │- Streams pages       │
│- Creates tasks  │    │- Creates delete tasks│
└─────────────────┘    └──────────────────────┘
    │                              │
    └──────────┬───────────────────┘
               │
               ▼
    ┌──────────────────────┐
    │  Firestore Updates   │
    │   (Atomic writes)    │
    └──────────────────────┘
```

The Sync Service uses a two-phase approach: first, it streams through collections in pages, then creates individual document tasks for reliable atomic updates.

## Heavy-Lifting Features

### 1. Intelligent Collection Streaming with Pagination

Efficient processing of collections of any size:

```typescript
BaseFirestoreCollectionStreamer features:
  - Automatic cursor-based pagination
  - Configurable page size (default: 100 documents)
  - Orders by document 'id' field for consistent pagination
  - Processes pages sequentially to avoid memory issues
  - Supports both regular collections and collection groups

Processing flow:
  1. Fetch page of documents
  2. Process each document in page
  3. Move cursor to next page
  4. Repeat until no more documents
```

**What customers avoid:**

- Writing pagination logic for large collections
- Managing cursor state across batches
- Implementing memory-efficient streaming
- Handling collection vs. collection group differences

### 2. Collection Group Query Support

Cross-subcollection updates with a single operation:

```typescript
Collection Group features:
  - Query all subcollections with the same name
  - Example: Update all 'members' subcollections across communities
  - Strips leading/trailing slashes from collection paths
  - Removes all slashes for collection group queries
  - Processes documents regardless of parent hierarchy

Use case:
  Collection: communities/{id}/members
  With collectionGroup=true: updates ALL members in ALL communities
  With collectionGroup=false: updates members in specific community
```

**What customers avoid:**

- Writing complex multi-collection traversal logic
- Managing hierarchical collection paths
- Implementing collection group query syntax
- Coordinating updates across subcollections

### 3. Two-Phase Task Architecture for Reliability

Decomposed task processing with independent retry:

```typescript
Phase 1 - Collection Task:
  - Receives collection patch/delete request
  - Streams through collection in pages
  - Creates individual document task for each document
  - Each document task is independent

Phase 2 - Document Tasks:
  - Process individual documents atomically
  - Can retry independently if failed
  - No cross-document dependencies
  - Guaranteed eventual completion

Benefits:
  - Partial failures don't restart entire collection
  - Individual documents can retry without affecting others
  - Progress persists across task failures
  - Natural rate limiting through task queue
```

**What customers avoid:**

- Building multi-phase task systems
- Implementing partial failure recovery
- Writing retry logic for large operations
- Managing task dependencies and ordering

### 4. Automatic Task Queue Management

Built-in task queue integration with routing:

```typescript
Task queues:
  - SYNC_COLLECTION_PATCH_REQUEST → /tasks/sync/request-collection-patch
  - SYNC_DOCUMENT_PATCH_REQUEST → /tasks/sync/request-document-patch
  - SYNC_COLLECTION_DELETION_REQUEST → /tasks/sync/request-collection-deletion
  - SYNC_DOCUMENT_DELETE_REQUEST → /tasks/sync/request-document-deletion

Task infrastructure handles:
  - Automatic retry on failure
  - Rate limiting and throttling
  - Task deduplication
  - Scheduled execution
  - GCP project routing
```

**What customers avoid:**

- Configuring task queue infrastructure
- Writing task routing logic
- Implementing retry mechanisms
- Managing task queue names and endpoints

### 5. Path Normalization and Validation

Automatic path cleaning for consistency:

```typescript
Path processing:
  - Strips leading slashes: /collection → collection
  - Strips trailing slashes: collection/ → collection
  - Removes all slashes for collection groups: a/b/c → abc
  - Handles both top-level and nested collection paths

Examples:
  - Input: /communities/123/members/
  - Output: communities/123/members

  - Input: /members/ (collectionGroup=true)
  - Output: members
```

**What customers avoid:**

- Writing path parsing logic
- Handling inconsistent path formats
- Validating collection path syntax
- Converting between path formats

### 6. Document Update with Partial Values

Merge-based updates preserve existing fields:

```typescript
Update behavior:
  - Uses Firestore update() not set()
  - Merges new values with existing document
  - Only specified fields are modified
  - Unspecified fields remain unchanged
  - Supports nested field updates with dot notation

Example:
  Document: {id: '123', name: 'John', age: 30, city: 'NYC'}
  Update: {age: 31}
  Result: {id: '123', name: 'John', age: 31, city: 'NYC'}
```

**What customers avoid:**

- Implementing merge logic
- Reading documents before updating
- Managing partial updates
- Preserving unmodified fields

### 7. Atomic Document Operations

Transaction-safe single-document operations:

```typescript
Document operations:
  - Each document patch is atomic
  - Each document deletion is atomic
  - No partial document states
  - Guaranteed consistency per document

Controller handlers:
  - documentPatchRequested: FirestorePaginationFunctions.update()
  - documentDeletionRequest: FirestorePaginationFunctions.delete()

Both use atomic Firestore operations
```

**What customers avoid:**

- Writing transaction wrappers
- Handling partial write failures
- Implementing rollback logic
- Managing document consistency

### 8. Command Pattern for Complex Operations

CQRS-based deletion handling with extensibility:

```typescript
Deletion uses Command/Handler pattern:
  - DeleteCollectionCommand: Request representation
  - DeleteCollectionHandler: Business logic execution
  - DeletionTaskGenerator: Streams and creates delete tasks

Benefits:
  - Separation of concerns
  - Easy to add pre/post-processing hooks
  - Testable business logic
  - Extensible for custom deletion logic
```

**What customers avoid:**

- Implementing command pattern infrastructure
- Writing command routing logic
- Managing handler registration
- Building extensible operation framework

### 9. Built-In Streaming Infrastructure

Reusable streaming abstraction for collection processing:

```typescript
BaseFirestoreStreamer features:
  - Abstract consume() method for custom processing
  - Automatic page fetching and iteration
  - Cursor management between pages
  - Pages processed counter
  - Optional custom orderBy field
  - Deep copy of items before processing

Implementations:
  - PatcherTaskGenerator: Creates patch tasks per document
  - DeletionTaskGenerator: Creates delete tasks per document

Both inherit streaming logic, only implement consume()
```

**What customers avoid:**

- Writing collection streaming boilerplate
- Implementing cursor-based iteration
- Managing page state across iterations
- Building reusable collection processors

### 10. Real-World Integration Examples

Production usage patterns from other services:

```typescript
Example 1 - Product name updates:
  UpdateProductSubscriptionsHandler:
    - Streams all subscriptions
    - Filters by productId
    - Creates patch tasks for matching subscriptions
    - Updates denormalized productName field
    - Processes in parallel with Promise.all

Example 2 - Community member cleanup:
  DeleteAllMembersHandler:
    - Builds collection path from communityId
    - Requests collection deletion
    - Sync service handles pagination and deletion
    - Guarantees all members removed

These demonstrate:
  - Filtering before patching for efficiency
  - Parallel task creation for performance
  - Integration with repository patterns
  - Cascade deletion coordination
```

**What customers avoid:**

- Building denormalized data sync systems
- Writing cascade deletion logic
- Implementing bulk update workflows
- Coordinating multi-service operations

### 11. Firestore Serialization Handling

Automatic data transformation for database operations:

```typescript
Serialization features:
  - Converts JavaScript objects to Firestore format
  - Deserializes Firestore data back to objects
  - Handles special Firestore types (Timestamp, GeoPoint, etc.)
  - Preserves type information through round-trips
  - Used automatically by all update/delete operations

FirestorePaginationFunctions handles:
  - serialize<T>() before writes
  - deserialize() after reads
  - Type-safe transformations
```

**What customers avoid:**

- Writing serialization logic
- Managing Firestore type conversions
- Handling timestamp transformations
- Building type-safe database operations

### 12. Document ID Ordering Requirement

Predictable pagination with indexed field:

```typescript
Ordering requirement:
  - All documents MUST have 'id' field
  - Used for orderBy in pagination queries
  - Enables consistent cursor-based pagination
  - Prevents duplicate processing
  - Ensures deterministic iteration order

WARNING in TaskService:
  "this collection data to patch must have an 'id' property
   which is used to sort paginated data"

Impact:
  - Documents without 'id' may be skipped
  - Pagination relies on indexed 'id' field
  - Collection must support orderBy('id') query
```

**What customers avoid:**

- Debugging pagination inconsistencies
- Handling unordered collection traversal
- Implementing custom sorting logic
- Managing cursor state without indexes

## Common Use Cases

- **Denormalized data updates**: When a product name changes, update all subscription documents that reference that product
- **Bulk data migrations**: Apply schema changes or data transformations across entire collections
- **Cascading updates**: Propagate user profile changes to all related records
- **Collection cleanup**: Delete all member documents when a community is removed
- **Cross-collection consistency**: Maintain referential integrity for duplicated data
- **Large-scale refactoring**: Update field names or structures across thousands of documents

## What Customers Don't Have to Build

- Cursor-based pagination for large collections
- Memory-efficient collection streaming
- Collection group query logic
- Two-phase task decomposition
- Independent document task retry
- Task queue configuration and routing
- Path normalization and validation
- Merge-based partial updates
- Atomic document operations
- Transaction management
- Command/handler pattern infrastructure
- Reusable streaming abstractions
- Firestore serialization handling
- Denormalized data sync systems
- Cascade deletion orchestration
- Bulk update workflows
- Pagination cursor management
- Task deduplication logic
- Rate limiting and throttling
- Ordered collection traversal
- Progress tracking across failures
- Partial failure recovery
- Cross-service update coordination

## API Reference

```typescript
class TaskService {
  /**
   * Update all documents in a collection
   * WARNING: Documents must have 'id' field for pagination
   */
  requestCollectionPatch(request: {
    collectionId: string; // Collection path (leading/trailing slashes stripped)
    values: unknown; // Partial document update (merged with existing)
    collectionGroup?: boolean; // Query across all subcollections with this name
  }): Promise<void>;

  /**
   * Update a single document
   * For slow-changing data and high-volume updates
   */
  requestDocumentPatch(request: {
    path: string; // Full document path
    values: unknown; // Partial document update (merged with existing)
  }): Promise<void>;

  /**
   * Delete all documents in a collection
   * Creates individual delete tasks for each document
   */
  requestCollectionDeletion(request: {
    collectionPath: string; // Collection path
  }): Promise<void>;

  /**
   * Delete a single document
   */
  requestDocumentDeletion(request: {
    path: string; // Full document path
  }): Promise<void>;
}
```

## Important Notes

### Collection ID Requirement

All documents in collections processed by the Sync Service must have an `id` field. This field is used for ordering during pagination to ensure consistent, deterministic traversal of large collections.

### Collection Group Behavior

When using `collectionGroup: true`, the service queries all subcollections with the specified name across the entire database hierarchy. Use this feature carefully as it can affect documents across many parent collections.

### Task Processing Model

Collection operations (patch/delete) create individual document-level tasks. Each document is processed independently, allowing for:

- Partial progress on failures
- Independent retry per document
- Natural rate limiting through task queue
- No memory issues with large collections

### Update Semantics

Document patches use Firestore's `update()` operation, not `set()`. This means:

- Only specified fields are modified
- Unspecified fields remain unchanged
- Partial updates are safe and efficient
- No need to read document before updating

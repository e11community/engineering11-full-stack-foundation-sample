# Files Service

Secure file storage, video processing, and intelligent lifecycle management.

## What It Does

The Files Service provides complete file lifecycle management from upload to deletion. It handles storage organization, video hosting integration, time-limited signed URLs, automatic interest tracking, and scheduled cleanup of orphaned files. The service abstracts storage complexity and coordinates with external video processors while maintaining security through secret tokens and validation.

## Key Capabilities

| Capability                   | Description                                                        |
| ---------------------------- | ------------------------------------------------------------------ |
| **File Receipts**            | Lightweight pointers with secret tokens for secure file references |
| **Signed URL Generation**    | Time-limited URLs with caching for both storage and video files    |
| **Video Processing**         | Automatic upload to video hosting with composite key tracking      |
| **Interest-Based Deletion**  | Track file references across documents and schedule cleanup        |
| **File Cloning**             | Deep copy files and receipts for document duplication              |
| **Public/Private Files**     | Separate storage paths with access control                         |
| **Stream Upload/Download**   | Efficient stream-based file operations with size limits            |
| **PDF Generation**           | URL-to-PDF conversion with streaming or bucket storage             |
| **Metadata Synchronization** | Automatic metadata updates from storage events                     |
| **Multi-Platform URLs**      | Separate signed URLs for web and mobile applications               |

## How It Fits Together

```
┌─────────────────┐
│  Files Service  │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┬─────────────┐
    ▼                 ▼              ▼             ▼
┌─────────┐   ┌──────────────┐  ┌───────┐  ┌──────────┐
│ Storage │   │Video Hosting │  │ Cache │  │ Database │
│(Public/ │   │  (External)  │  │(Redis)│  │Firestore │
│Private) │   └──────────────┘  └───────┘  └──────────┘
└─────────┘
```

Files uses cloud storage for object persistence, external video hosting for video processing, Redis for signed URL caching, and Firestore for metadata and interest tracking.

## Heavy-Lifting Features

### 1. File Receipt Security System

Secret-token based file references with validation:

```typescript
File Receipt structure:
  - Unique ID and secret token for security
  - Customer key and grouping ID for organization
  - File name, MIME type, and size metadata
  - Public/private flag determines storage path
  - Optional videoCompositeKey for hosted videos
  - Optional deletionDelayDays for custom cleanup windows

buildFileReceiptObject():
  - Generates unique ID and secret token (UUID)
  - Validates MIME type against allowed types
  - Determines storage path based on public flag
  - Creates signature for receipt validation

Validation on signed URL requests:
  - Verifies secret token matches metadata
  - Checks customer key consistency
  - Validates file ID matches
  - Rejects tampered or invalid receipts
```

**What customers avoid:**

- Building secure file reference systems
- Generating and validating secret tokens
- Implementing storage path conventions
- Writing receipt validation logic
- Managing customer-scoped access control

### 2. Intelligent Signed URL Management

Multi-layer caching with storage and video routing:

```typescript
SignedUrlService provides:
  - Batch signed URL generation (deduplicates by ID)
  - Secret token validation before URL creation
  - Automatic routing to storage or video URLs
  - Separate web and mobile URL formats
  - Multi-status responses (207) for partial success
  - Silent error logging for failed individual files

Caching strategy:
  - Cache key: signed-url/{fileId} or signed-url/video/{videoId}
  - TTL: configurable expiration (default ~7 days)
  - Separate cache keys for web vs. mobile
  - Automatic cache invalidation on file deletion
  - Transaction-aware bypass for consistency

Throttling protection:
  - 500 requests per 5 seconds (100/sec rate)
  - Applied to batch operations to prevent abuse
```

**What customers avoid:**

- Building batch URL generation with deduplication
- Implementing two-tier caching (storage + video)
- Managing separate web/mobile URL formats
- Writing partial success response handling
- Coordinating cache invalidation on deletions
- Implementing rate limiting for URL endpoints

### 3. Video Processing Integration

Automatic video hosting with composite key management:

```typescript
Video workflow:
  - Detects video MIME types (video/*)
  - Uploads to external video hosting provider
  - Receives composite key for hosted video
  - Stores composite key in file metadata
  - Generates signed playback URLs
  - Routes mobile vs. web video URLs differently

VideoService.sendToVideoHost():
  - Validates MIME type is video
  - Builds storage path to source file
  - Calls video provider API with bucket path
  - Sets private flag for access control
  - Returns composite key for metadata storage

Signed video URLs:
  - Cache separate from storage URLs
  - Support expiration time parameters
  - Automatic provider selection from composite key
  - Fallback to storage signing on video errors
  - Separate mobile URL generation for apps
```

**What customers avoid:**

- Integrating with video hosting platforms
- Managing video upload workflows
- Implementing composite key tracking
- Building video URL generation systems
- Writing provider selection logic
- Handling mobile vs. web video formats

### 4. Interest-Based File Lifecycle Management

Automatic reference tracking and scheduled deletion:

```typescript
FileInterest tracking:
  - Global database triggers on ALL document writes
  - Automatic file receipt extraction from documents
  - Bidirectional interest mapping (file ↔ documents)
  - Array-based interested document tracking
  - Scheduled deletion when no documents interested

Register Interest functions:
  onCreate: Add document to interested array for all files
  onUpdate: Calculate diff and update interested arrays
  onDelete: Remove document from all file interest arrays

Deletion scheduling:
  - Triggers when interestedDocuments becomes empty
  - Default 14-day deletion window (configurable per file)
  - Creates scheduled Cloud Task for deletion
  - Stores task ID and deletion date in metadata
  - Cancels scheduled deletion if interest re-added

Transaction guarantees:
  - Read latest document state within transaction
  - Prevents race conditions on concurrent updates
  - Atomic interest array modifications
  - Rollback on scheduling failures
```

**What customers avoid:**

- Building automatic file reference tracking
- Writing global document change listeners
- Implementing array-based interest tracking
- Managing scheduled deletion workflows
- Coordinating task creation and cancellation
- Handling race conditions on interest changes
- Building configurable deletion windows

### 5. Deep File Cloning with Storage Coordination

Recursive document cloning with storage object duplication:

```typescript
FileCloningService features:
  - Deep document traversal to find all file receipts
  - Automatic receipt cloning with new IDs and tokens
  - Storage object copying with metadata preservation
  - Donor ID tracking for clone relationships
  - Synchronous or asynchronous copy modes

cloneDocument() workflow:
  - Recursively scan document for file receipts
  - Generate new receipts with cloneReceipts()
  - Deep copy entire document structure
  - Replace all file receipts with clones
  - Copy storage objects from donor to clone paths
  - Upload videos to hosting for cloned receipts
  - Create metadata entries for all clones

Copy operations:
  - Preserve MIME type and content type
  - Copy metadata as custom storage metadata
  - Handle public files (make public after copy)
  - Revoke access tokens for private files
  - Async copy via Cloud Tasks for large files
```

**What customers avoid:**

- Writing recursive document cloning logic
- Building file receipt extraction systems
- Coordinating storage object copies
- Managing donor-clone relationships
- Implementing metadata preservation
- Handling async copy operations
- Synchronizing video hosting for clones

### 6. Public/Private File Separation

Storage path conventions with automatic access control:

```typescript
Storage paths:
  - Private files: content/{customerKey}/{groupingId}/{id}/{secretToken}/{filename}
  - Public files: assets/{customerKey}/{groupingId}/{id}/{secretToken}/{filename}

Public file workflow:
  finalizeUpload() for public files:
    - Calls storage.makePublic(path)
    - Generates static CDN URL
    - Stores URL in file receipt and metadata
    - No signed URL required (direct access)

Private file workflow:
  finalizeUpload() for private files:
    - Revokes public access tokens
    - Requires signed URL for access
    - URLs expire after configured time
    - Cache signed URLs for performance

Path parsing:
  - Extract customerKey, groupingId, fileId from paths
  - Determine public/private from path prefix
  - Parse secret token for validation
  - Reconstruct full paths from file receipts
```

**What customers avoid:**

- Designing storage path conventions
- Implementing public/private access logic
- Building CDN URL generation
- Managing access token revocation
- Writing path parsing utilities
- Coordinating storage permissions

### 7. Stream-Based File Operations

Efficient upload/download with size limiting:

```typescript
Upload streaming:
  - FileStream API for multipart form uploads
  - Progress tracking during upload
  - Size limit enforcement (default 500MB)
  - Automatic stream abortion on size exceeded
  - Direct streaming to cloud storage
  - File receipt creation during upload

uploadFileStream():
  - Creates writable stream to storage
  - Optional size limit with limitSize() pipe
  - Resolves on stream finish
  - Rejects on stream error
  - Automatic finalizeUpload() after stream

Download streaming:
  - Readable stream from storage
  - Efficient for large files
  - No memory buffering required
  - Supports range requests

File size validation:
  - Decorator-based validation (@FileReceiptSize)
  - Type-specific size limits
  - Async validation against metadata
  - Automatic metadata lookup if size missing
```

**What customers avoid:**

- Implementing stream-based uploads
- Building size limit enforcement
- Writing progress tracking systems
- Managing multipart form parsing
- Creating storage stream adapters
- Implementing size validation decorators

### 8. Temporary File Management

Time-limited files with automatic cleanup:

```typescript
Temporary file workflow:
  - Created with saveAsTemp flag on finalize
  - Stored in separate temporary files collection
  - Contains file receipt and creation timestamp
  - No immediate storage deletion on save

Use cases:
  - Files pending approval or processing
  - Preview files before permanent storage
  - Staged uploads for multi-step workflows
  - Files with uncertain final destination

Cleanup (implementation pattern):
  - Scheduled task checks creation timestamp
  - Deletes temp record after configured window
  - Triggers normal file deletion cascade
  - Removes storage object and metadata
```

**What customers avoid:**

- Building temporary file tracking
- Implementing staged upload patterns
- Writing cleanup scheduling logic
- Managing separate temp file collections
- Coordinating deletion cascades

### 9. PDF Generation from URLs

Headless browser integration with streaming:

```typescript
PDFGeneratorController endpoints:
  V1: Stream PDF directly to client
    - Generates PDF from URL
    - Streams response with Transfer-Encoding: chunked
    - Works within Cloud Run 32MB limits
    - No Content-Length header for streaming

  V2: Generate PDF to bucket
    - Creates unique PDF filename (UUID)
    - Streams to configured PDF_BUCKET
    - Returns signed URL for download
    - Automatic lifecycle deletion after 1 day

PDF generation:
  - Validates URL to prevent SSRF
  - Calls headless browser service
  - Streams response data
  - Handles chunked transfer encoding
  - Throttled at 100 requests/second

Error handling:
  - Validates bucket configuration
  - Handles stream errors
  - Logs PDF generation failures
  - Returns appropriate error codes
```

**What customers avoid:**

- Integrating headless browser services
- Implementing URL-to-PDF conversion
- Managing streaming responses
- Handling Cloud Run size limits
- Building SSRF protection
- Writing PDF lifecycle management

### 10. Metadata Synchronization from Storage Events

Automatic metadata updates from cloud storage:

```typescript
Storage event handlers:
  onFinalize: File written to storage
    - Triggered when upload completes
    - Extracts metadata from storage object
    - Merges with existing file metadata
    - Updates size, hashes, storage file ID
    - Only processes files with metadata

  onDelete: File deleted from storage
    - Triggered when storage object removed
    - Deletes corresponding file metadata
    - Only processes files with receipt metadata

Metadata merge:
  - Extracts file ID from storage metadata
  - Updates: size, crc32c, fileType, storageFileId
  - Updates: md5Hash, bucket, updated timestamp
  - Preserves existing file receipt data
  - Upserts to avoid overwriting

Event routing:
  Storage Event → Pub/Sub Topic → Cloud Task → Task Handler
  - Async processing prevents blocking
  - Retries on failure
  - Deduplication for deletions
```

**What customers avoid:**

- Building storage event handlers
- Implementing metadata synchronization
- Writing hash and checksum tracking
- Managing storage-database consistency
- Coordinating async event processing
- Building retry and deduplication logic

### 11. Cascade Deletion with Multi-Resource Cleanup

Comprehensive cleanup across storage, video, and database:

```typescript
FileDeletedTaskService.delete():
  Parallel cleanup operations:
    - Delete storage object (if exists)
    - Delete signed URL cache entry
    - Delete video from hosting (if video file)
    - Delete video signed URL cache entries
    - Delete file interest tracking document

  All operations are non-throwing:
    - Storage deletion succeeds if not exists
    - Cache deletion succeeds if key missing
    - Video deletion handles missing videos

Video cleanup:
  - Extracts video provider from composite key
  - Calls provider-specific delete method
  - Removes both web and mobile cache entries
  - Handles deleted video hosting records

Storage deletion triggers:
  - Database file deletion → storage cleanup
  - Storage file deletion → database cleanup
  - Both paths converge to same cleanup logic
```

**What customers avoid:**

- Writing cascade deletion logic
- Coordinating multi-resource cleanup
- Implementing non-throwing deletion
- Managing video hosting deletion
- Building cache cleanup coordination
- Handling bidirectional deletion triggers

### 12. Multi-Tenant Organization with Grouping

Hierarchical file organization by customer and grouping:

```typescript
File organization hierarchy:
  customerKey: Tenant/customer identifier
  groupingId: Document or resource ID
  fileId: Unique file identifier

Storage path structure:
  {root}/{customerKey}/{groupingId}/{fileId}/{secretToken}/{filename}

Grouping-based operations:
  - deleteAllForGroup(groupingId): Delete all files for a document
  - getAll(groupingId): Query all files for a document
  - collectFileObjects(document): Extract all file receipts recursively

Use cases:
  - Product with multiple images (groupingId = productId)
  - User profile with avatar and documents (groupingId = userId)
  - Post with attachments (groupingId = postId)

Query patterns:
  - All files for a customer: query by customerKey
  - All files for a document: query by groupingId
  - Specific file: direct lookup by fileId
```

**What customers avoid:**

- Designing multi-tenant storage structures
- Building hierarchical file organization
- Implementing grouping-based queries
- Writing recursive file extraction
- Managing customer isolation
- Building batch deletion by group

### 13. Automatic Event-Driven Architecture

Transparent pub/sub for all file operations:

```typescript
Cloud Functions automatically publish:
  Storage events:
    - FILE_UPLOADED → onFinalize trigger
    - FILE_DELETED_STORAGE → onDelete trigger

  Database events:
    - FILE_UPDATED → onUpdate trigger
    - FILE_DELETED → onDelete trigger

  Interest events:
    - FILE_INTEREST_CREATED → onCreate trigger
    - FILE_INTEREST_UPDATED → onUpdate trigger
    - FILE_INTEREST_DELETED → onDelete trigger

Event routing:
  Database/Storage → Pub/Sub Topic → Cloud Task Queue → Task Handler

  Benefits:
    - Async processing prevents blocking
    - Automatic retries on failure
    - Deduplication for deletions
    - Guaranteed execution
    - Decoupled architecture
```

**What customers avoid:**

- Building event publishing infrastructure
- Writing database and storage triggers
- Implementing pub/sub integrations
- Managing async task queues
- Coordinating event routing
- Building retry and deduplication logic

### 14. Type-Safe File Operations with Validation

Comprehensive MIME type support and validation:

```typescript
Supported file types:
  Images: JPEG, PNG, GIF, HEIC, HEIF
  Videos: MP4, MPEG, OGG, WebM, AVI, QuickTime, 3GPP, M4V
  Audio: MPEG, WAV, 3GPP
  Documents: TXT, MD, DOC, DOCX, RTF, PDF, PPT, PPTX,
             EPUB, XLS, XLSX, generic binary

File receipt validation:
  - @FileReceiptSize() decorator for size limits
  - Type-specific size limits (configurable)
  - Default 500MB maximum size
  - Async validation against metadata
  - Automatic size lookup from storage

MIME type detection:
  - Automatic MIME lookup from filename
  - Validation against allowed types
  - Video type detection (video/* prefix)
  - Content-Type preservation on copy

Storage metadata validation:
  - Secret token verification
  - Customer key matching
  - File ID consistency checks
  - Reject tampered receipts
```

**What customers avoid:**

- Building MIME type validation
- Implementing size limit enforcement
- Writing validation decorators
- Managing allowed file type lists
- Building automatic type detection
- Implementing security validation

## Common Use Cases

- **Media management**: Product images, user avatars, and marketing assets with automatic video processing
- **Document storage**: Contracts, invoices, and reports with secure time-limited access
- **Content publishing**: Blog images, videos, and attachments with public CDN delivery
- **Multi-tenant SaaS**: Customer file isolation with separate storage namespaces
- **File lifecycle**: Temporary uploads, approval workflows, and automatic cleanup of unused files
- **Document cloning**: Duplicate records with all associated files copied
- **PDF generation**: Convert web pages to PDFs for reports, receipts, and documentation
- **Video hosting**: Automatic upload to video processors with playback URL management

## What Customers Don't Have to Build

- Secret-token based file security system
- Batch signed URL generation with deduplication
- Multi-layer caching (storage + video + Redis)
- Automatic video hosting integration
- Interest-based reference tracking across documents
- Scheduled deletion workflows with cancellation
- Deep document cloning with storage coordination
- Public/private file access control
- Stream-based upload/download with size limits
- Temporary file management with cleanup
- PDF generation from URLs with streaming
- Metadata synchronization from storage events
- Cascade deletion across multiple resources
- Multi-tenant file organization with grouping
- Event-driven architecture with pub/sub
- Type-safe MIME type validation
- Separate web and mobile URL formats
- Storage path conventions and parsing
- Customer-scoped access validation
- Transaction-aware file operations
- Video composite key management
- Cache invalidation coordination
- Throttling and rate limiting for URLs
- Async file copy operations with tasks

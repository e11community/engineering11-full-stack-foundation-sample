# Files Service

File storage, processing, and delivery.

## What It Does

The Files Service manages all file operations in the platform — uploads, storage, processing, and secure delivery. It abstracts away cloud storage complexity and provides features like automatic cleanup, CDN delivery, and file transformations.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **File Upload** | Secure, resumable uploads with progress tracking |
| **Storage Management** | Organize files by tenant, user, and resource type |
| **Signed URLs** | Time-limited, secure URLs for file access |
| **CDN Delivery** | Fast global delivery through content delivery networks |
| **Automatic Cleanup** | Delete orphaned files when no longer referenced |
| **File Receipts** | Track file references across the system |

## File Receipt Pattern

Files are referenced using "receipts" — lightweight pointers that can be stored on any document:

```
Document (e.g., Product)
├── primaryImage: FileReceipt
└── attachments: FileReceipt[]
```

When a document is deleted, the system automatically cleans up files that are no longer referenced anywhere.

## How It Fits Together

```
┌─────────────────┐
│   Any Service   │
│  (stores files) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Files Service  │────►│  Cloud Storage  │
└────────┬────────┘     │     + CDN       │
         │              └─────────────────┘
         ▼
┌─────────────────┐
│  Signed URLs    │
│  for Clients    │
└─────────────────┘
```

## Common Use Cases

- **Profile images**: User avatars with automatic resizing
- **Document storage**: PDFs, spreadsheets, and attachments
- **Media libraries**: Images and videos with CDN delivery
- **Temporary uploads**: Files that need processing before permanent storage

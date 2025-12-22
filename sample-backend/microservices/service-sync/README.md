# Sync Service

Data synchronization across database collections.

## What It Does

The Sync Service manages data synchronization across Firestore collections and documents within the platform. It enables bulk patching and deletion of documents when data in one collection changes and needs to be reflected in related documents — maintaining referential consistency for denormalized data.

## Key Capabilities

| Capability               | Description                                                   |
| ------------------------ | ------------------------------------------------------------- |
| **Collection Patching**  | Bulk update documents across a collection with new values     |
| **Document Patching**    | Update individual documents at specific paths                 |
| **Collection Deletion**  | Bulk delete all documents in a collection                     |
| **Document Deletion**    | Delete individual documents at specific paths                 |
| **Paginated Processing** | Process large collections in batches (default: 100 docs/page) |
| **Async Task Queue**     | Reliable, asynchronous processing via task queues             |

## How It Fits Together

```
┌──────────────────────────────────────────────────────────────┐
│                     Request Entry Point                      │
│  TaskService: requestCollectionPatch, requestDocumentPatch,  │
│               requestCollectionDeletion, requestDocumentDeletion │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   Task Queue    │
                 └────────┬────────┘
                          │
                          ▼
          ┌───────────────────────────────┐
          │      Task Processor           │
          │  (Paginated Batch Processing) │
          └───────────────┬───────────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
            ▼                           ▼
   ┌─────────────────┐        ┌─────────────────┐
   │  Collection A   │        │  Collection B   │
   │   (source)      │        │   (target)      │
   └─────────────────┘        └─────────────────┘
```

The Sync Service propagates changes from source documents to related documents across collections.

## Common Use Cases

- **Denormalized data updates**: When a product name changes, update all subscription documents that reference that product
- **Bulk data migrations**: Sync fields across documents during data migrations
- **Cascading updates**: Propagate user profile changes to all related records
- **Collection cleanup**: Delete all member documents when a community is removed

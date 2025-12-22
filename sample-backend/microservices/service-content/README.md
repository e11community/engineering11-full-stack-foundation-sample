# Content Service

Content management and delivery.

## What It Does

The Content Service manages structured content within the platform — articles, pages, rich media, and other content types. It provides authoring workflows, versioning, and CDN-backed delivery for public content.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Content Storage** | Store and organize structured content |
| **Versioning** | Track content revisions and enable rollback |
| **Publishing Workflow** | Draft, review, and publish states |
| **CDN Delivery** | Fast global delivery for public content |
| **Access Control** | Control who can view, edit, and publish content |
| **Content Association** | Link content to users, products, or other resources |

## How It Fits Together

```
┌─────────────────┐
│ Content Service │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│ Files │ │  CDN  │
│Service│ │       │
└───────┘ └───────┘
```

The Content Service coordinates with Files Service for media and delivers through CDN for performance.

## Common Use Cases

- **Knowledge base**: Help articles and documentation
- **Marketing content**: Landing pages and promotional content
- **User-generated content**: Posts, reviews, and comments
- **Localized content**: Multi-language content with regional variants

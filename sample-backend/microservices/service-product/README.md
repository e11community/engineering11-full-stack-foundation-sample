# Product Service

Product catalog and inventory management.

## What It Does

The Product Service manages product information within the platform — catalogs, pricing, inventory, and product attributes. It supports both simple and complex product structures for e-commerce and marketplace scenarios.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Product Catalog** | Store and organize product information |
| **Variants** | Manage product variations (size, color, etc.) |
| **Pricing** | Flexible pricing including tiers and promotions |
| **Inventory** | Track stock levels and availability |
| **Categories** | Organize products into hierarchical categories |
| **Search & Filter** | Product discovery with faceted search |

## How It Fits Together

```
┌─────────────────┐
│ Product Service │
└────────┬────────┘
         │
    ┌────┴────────────┐
    ▼                 ▼
┌───────────┐   ┌───────────┐
│   Files   │   │  Search   │
│  Service  │   │  Index    │
└───────────┘   └───────────┘
```

The Product Service stores product images via Files and indexes products for search.

## Common Use Cases

- **E-commerce catalog**: Full product catalog with variants and pricing
- **Marketplace listings**: Multi-vendor product management
- **Service catalog**: IT or internal service offerings
- **Digital products**: Software, subscriptions, and digital goods

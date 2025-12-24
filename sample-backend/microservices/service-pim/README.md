# PIM Service

Product Information Management with intelligent product matching, variant management, and multi-supplier optimization.

## What It Does

The PIM (Product Information Management) Service provides comprehensive product catalog management with automatic product matching, similarity-based deduplication, supplier and manufacturer tracking, and intelligent ordering optimization across multiple suppliers. It combines product lifecycle management with advanced search capabilities and linear programming-based purchase recommendations.

## Key Capabilities

| Capability                  | Description                                                                            |
| --------------------------- | -------------------------------------------------------------------------------------- |
| **Product Hierarchy**       | Manage products, variants, and supplier-specific versions with automatic deduplication |
| **Manufacturer Tracking**   | Track manufacturers with similarity keys for matching across suppliers                 |
| **Supplier Management**     | Manage suppliers with shipping policies and free shipping thresholds                   |
| **Catalog Publishing**      | Build and publish catalog items from enriched products                                 |
| **Similarity Matching**     | Generate similarity keys for intelligent product and company matching                  |
| **Search Infrastructure**   | Multi-index search for products, variants, catalogs, manufacturers, and suppliers      |
| **Order Optimization**      | Linear programming-based recommendations for multi-supplier purchases                  |
| **Product Status Workflow** | Track products through Incomplete → Pending → Ready lifecycle                          |

## How It Fits Together

```
┌─────────────────┐
│   PIM Service   │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┐
    ▼                 ▼              ▼
┌───────────┐   ┌───────────┐  ┌──────────┐
│ Ingestion │   │  Product  │  │  Search  │
│  Service  │   │  Service  │  │  Engine  │
└───────────┘   └───────────┘  └──────────┘
```

PIM receives supplier data from Ingestion, enriches and normalizes it, publishes to Product Service, and maintains search indices for discovery.

## Heavy-Lifting Features

### 1. Intelligent Product Matching and Deduplication

Automatic product identification across suppliers:

```typescript
ProductLookupService.lookupOrCreateProduct():
  - Searches for existing product by manufacturer + part number + supplier
  - Falls back to manufacturer product key lookup
  - Creates new product if no match found
  - Generates similarity keys for fuzzy matching
  - Associates product keys across supplier variants
  - Maintains manufacturer key relationships

Product Key Structure:
  - Composite ID: {customerKey}_{manufacturerId}_{supplierId}_{mfrPartNumber}_{supplierPartNumber}
  - Links same manufacturer product across different suppliers
  - Enables cross-supplier product consolidation
```

**What customers avoid:**

- Building product matching algorithms
- Writing deduplication logic across suppliers
- Managing cross-supplier product relationships
- Implementing fallback lookup strategies
- Generating composite keys for unique identification

### 2. Similarity-Based Company Matching

Fuzzy matching for manufacturer name normalization:

```typescript
SimilarityKeyService provides:
  - Integration with external similarity API (Interzoid)
  - Company name normalization across variations
  - Configurable algorithms (wide/medium/narrow matching)
  - Fallback to text-based key generation
  - Automatic similarity key generation for manufacturers

Use case:
  - "3M Company", "3M Corp", "3M" → same simKey
  - Consolidates products from same manufacturer
  - Handles supplier naming inconsistencies
```

**What customers avoid:**

- Integrating third-party matching APIs
- Building company name normalization
- Writing fuzzy matching algorithms
- Managing API key secrets
- Implementing fallback strategies

### 3. Multi-Level Product Hierarchy

Three-tier product structure with automatic relationships:

```typescript
Product hierarchy:
  PimProduct (canonical):
    - Manufacturer-level product definition
    - Status tracking (Incomplete/Pending/Ready)
    - Aggregated attributes from variants
    - Multiple manufacturer keys
    - Similarity key for matching

  PimProductVariant (supplier-specific):
    - Links to canonical product
    - Supplier and manufacturer information
    - Supplier part number + manufacturer part number
    - Multiple UOM options with pricing
    - Stock codes and contract numbers
    - Industry codes (UNSPSC, EAN, UPC, NDC)

  CatalogItem (published):
    - Merges product + catalog enrichment
    - Adds cataloged timestamp
    - Ready for customer-facing search

Automatic updates:
  - Variant creation links to product
  - Product status changes cascade
  - Catalog rebuilt from latest product data
```

**What customers avoid:**

- Designing multi-tier product models
- Managing bidirectional relationships
- Implementing cascade update logic
- Building product merge strategies
- Coordinating hierarchy consistency

### 4. Linear Programming Purchase Optimization

Mathematical optimization for multi-supplier ordering:

```typescript
ProductOptimizationService.recommendProducts():
  - Accepts product requests with quantities
  - Considers all available variants and UOMs
  - Optimizes across multiple suppliers
  - Accounts for shipping policies:
    * Flat rate shipping costs
    * Free shipping thresholds
    * Minimum order quantities
    * Multiple order quantities
  - Applies multiple supplier penalty
  - Generates constraints and objective function
  - Solves using linear programming solver
  - Returns optimal quantity per variant/UOM

Constraint types:
  - Quantity fulfillment (must meet request)
  - UOM availability bounds (stock limits)
  - Shipping activation (binary: pay or free)
  - Free shipping threshold (cost-based)
  - Supplier ordering penalty (minimize suppliers)
  - Multiple order quantity constraints

Big M method:
  - Dynamically calculated from total quantity
  - Enables binary variable constraints
  - Accounts for over-purchasing when cheaper
```

**What customers avoid:**

- Building linear programming models
- Writing constraint generation logic
- Implementing Big M method
- Integrating LP solvers
- Handling shipping cost optimization
- Managing multi-supplier tradeoffs
- Calculating optimal purchase quantities

### 5. Comprehensive UOM and Pricing Management

Flexible unit of measure with supplier-specific pricing:

```typescript
UOM structure per variant:
  - Multiple UOM options (EA, CS, PK, etc.)
  - Items per unit (24/CS, 1/EA)
  - Quantity available
  - Formulary price (contract price)
  - List price (MSRP)
  - Currency specification
  - Minimum order quantity
  - Multiple order quantity (packaging increments)

Pricing features:
  - Contract-specific pricing via contract numbers
  - Multiple price points per product
  - Currency handling
  - Stock code tracking
  - UOM-specific identifiers for cart systems
```

**What customers avoid:**

- Designing flexible UOM systems
- Managing complex pricing structures
- Implementing contract pricing
- Building packaging increment logic
- Coordinating UOM across suppliers

### 6. Rich Product Attribute System

Structured and searchable product metadata:

```typescript
Product attributes include:
  - Custom name/value pairs (size, color, material)
  - Primary and secondary categories
  - Multiple category hierarchies
  - Classification codes
  - Short and long descriptions
  - Domain and brand information
  - Multiple images with URLs
  - Feature lists
  - Additional search terms
  - Industry standard codes:
    * UNSPSC (UN Standard Products and Services Code)
    * EAN (European Article Number - 13 digits)
    * UPC (Universal Product Code - 12 digits)
    * NDC (National Drug Code for pharmaceuticals)

Attribute indexing:
  - Flattened for search (name:value strings)
  - Full-text searchable across indices
  - Faceted filtering support
```

**What customers avoid:**

- Designing extensible attribute systems
- Implementing category hierarchies
- Managing industry code standards
- Building attribute search logic
- Creating flexible metadata schemas

### 7. Automatic Event-Driven Architecture

Transparent pub/sub for all entity lifecycle events:

```typescript
Database triggers automatically publish:
  - PIM_PRODUCT_CREATED/UPDATED/DELETED
  - PIM_PRODUCT_VARIANT_CREATED/UPDATED/DELETED
  - PIM_MANUFACTURER_CREATED/UPDATED/DELETED
  - PIM_MANUFACTURER_KEY_CREATED/UPDATED/DELETED
  - PIM_SUPPLIER_CREATED/UPDATED/DELETED

Event flow:
  1. Database onCreate/onUpdate/onDelete trigger
  2. Publishes to topic
  3. Topic routes to task queue
  4. Task worker executes command handler
  5. Updates search indices and analytics tables
```

**What customers avoid:**

- Building database trigger infrastructure
- Writing event publishing logic
- Implementing topic-to-queue routing
- Managing async event handlers
- Coordinating side effects

### 8. Multi-Index Search Architecture

Dedicated search indices for all entity types:

```typescript
Search indices:
  - pim_product: Canonical products
  - pim_product_variant: Supplier variants
  - pim_catalog: Published catalog items
  - pim_manufacturer: Manufacturer directory
  - pim_supplier: Supplier directory

SearchKeyController provides scoped keys:
  - Customer-filtered search keys
  - Product-specific variant search
  - Index-restricted queries
  - Secured API key generation per request
  - Automatic filter injection (customerKey)

Search capabilities:
  - Full-text search across attributes
  - Faceted filtering by category
  - Filter by manufacturer, supplier, brand
  - Code-based search (UPC, EAN, UNSPSC, NDC)
  - Part number search (mfr + supplier)
```

**What customers avoid:**

- Building multi-index search systems
- Implementing search key generation
- Writing filter injection logic
- Managing index synchronization
- Creating faceted search infrastructure
- Securing search with scoped keys

### 9. Task-Based Background Processing

CQRS pattern with saga coordination:

```typescript
Saga pattern for each entity:
  - Event triggers saga
  - Saga emits command
  - Command handler executes business logic
  - Idempotent operation handling

Create handlers:
  - Fetch latest from database
  - Update search index
  - Log to analytics table
  - Build catalog item (for products)

Update handlers:
  - Sync search index with changes
  - Update analytics logs
  - Propagate changes to dependents

Delete handlers:
  - Remove from search index
  - Mark deleted in analytics
  - Cleanup dependent records

Retry mechanism:
  - Automatic retry on failure
  - Exponential backoff
  - Dead letter queue for persistent failures
```

**What customers avoid:**

- Implementing CQRS architecture
- Building saga pattern orchestration
- Writing idempotent handlers
- Managing retry logic
- Coordinating async workflows

### 10. Analytics Data Warehouse Integration

Automated logging to analytics tables:

```typescript
BigQuery tables mirror all entities:
  - pim_product: Product history with attributes flattened
  - pim_product_variant: Variant history with pricing
  - pim_catalog: Published catalog snapshots
  - pim_manufacturer: Manufacturer directory
  - pim_supplier: Supplier directory with policies
  - pim_product_key: Product matching history

Table features:
  - Operation tracking (Create/Update/Delete)
  - Monthly partitioning by timestamp
  - Full JSON snapshots in data field
  - Attribute flattening for querying
  - Automatic timestamp tracking
  - Buffered batch inserts for performance

Use cases:
  - Product enrichment analytics
  - Pricing history tracking
  - Supplier performance analysis
  - Catalog growth metrics
```

**What customers avoid:**

- Building data warehouse pipelines
- Writing table partitioning logic
- Implementing buffered batch inserts
- Managing schema evolution
- Creating analytics snapshots

### 11. Permission-Based Access Control

Granular endpoint protection:

```typescript
Permissions:
  - ViewPimProducts: Access product details
  - SearchPimProducts: Get product search keys
  - ViewPimCatalog: Access catalog items
  - SearchPimCatalog: Get catalog search keys
  - ViewPimManufacturer: Access manufacturer details
  - SearchPimManufacturer: Get manufacturer search keys
  - ViewPimSupplier: Access supplier details
  - SearchPimSupplier: Get supplier search keys

Protection features:
  - @HasPermission decorator guards
  - Customer key validation on all operations
  - Automatic key mismatch detection
  - Consistent error responses (401/403/404)
```

**What customers avoid:**

- Implementing permission systems
- Writing authorization guards
- Building customer isolation logic
- Managing consistent error handling

### 12. Manufacturer Key Management

Supplier-scoped manufacturer normalization:

```typescript
ManufacturerKey structure:
  - Links manufacturer to supplier context
  - Tracks manufacturer name variations per supplier
  - Maintains similarity keys per relationship
  - Enables supplier-specific manufacturer matching

Use case:
  - Supplier A calls manufacturer "3M Company"
  - Supplier B calls manufacturer "3M Corp"
  - Both link to same Manufacturer via simKey
  - ManufacturerKey tracks supplier-specific names
  - Products match across suppliers despite name differences

Automatic management:
  - Created when new supplier products ingested
  - Updated when manufacturer names change
  - Indexed for fast lookup
  - Logged to analytics tables
```

**What customers avoid:**

- Building supplier-scoped normalization
- Managing manufacturer name variations
- Implementing cross-supplier matching
- Writing variation tracking logic

## Common Use Cases

- **Multi-supplier procurement**: Find best pricing across multiple suppliers for same product
- **Catalog consolidation**: Merge supplier feeds into unified product catalog
- **Order optimization**: Calculate optimal purchase quantities considering shipping and MOQs
- **Manufacturer normalization**: Identify same manufacturer across different supplier naming
- **Product enrichment**: Track products from incomplete to ready for publication
- **Healthcare supply chain**: Manage products with NDC codes and medical classifications
- **Industrial distribution**: Handle UNSPSC codes and complex UOM structures
- **Contract pricing**: Manage supplier-specific contract pricing and terms

## What Customers Don't Have to Build

- Product matching and deduplication across suppliers
- Similarity-based company name normalization
- Multi-tier product hierarchy with automatic relationships
- Linear programming purchase optimization
- Shipping cost and free shipping threshold handling
- Flexible UOM and pricing management systems
- Multiple order quantity and MOQ constraints
- Extensible attribute and category systems
- Industry standard code management (UNSPSC, EAN, UPC, NDC)
- Event-driven architecture with database triggers
- Multi-index search infrastructure with scoped keys
- CQRS pattern with saga orchestration
- Idempotent background task processing
- Data warehouse integration with partitioned tables
- Permission-based access control with customer isolation
- Manufacturer key management across suppliers
- Product status workflow tracking
- Catalog building and publishing pipeline
- Analytics table buffering and batch inserts
- Cross-supplier product consolidation logic

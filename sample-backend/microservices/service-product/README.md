# Product Service

Complete product catalog, pricing, and subscription billing infrastructure.

## What It Does

The Product Service provides end-to-end product lifecycle management with multi-tenant pricing, subscription billing, payment processing, and invoice management. It combines product catalogs with flexible pricing models, payment gateway integration, and automated subscription workflows.

## Key Capabilities

| Capability                     | Description                                                           |
| ------------------------------ | --------------------------------------------------------------------- |
| **Product & Price Management** | Product catalog with multi-currency pricing and SKU validation        |
| **Tenant-Specific Products**   | Per-tenant product availability with custom pricing                   |
| **Subscription Billing**       | Recurring subscriptions with auto-renewal and status tracking         |
| **Payment Processing**         | One-time and subscription payments with gateway integration           |
| **Invoice Management**         | Custom invoicing with line items, history tracking, and notifications |
| **Permission Integration**     | Product-based permission assignment and synchronization               |
| **Default Products**           | Automatic product provisioning for new customers                      |

## How It Fits Together

```
┌─────────────────┐
│   Product       │
│   Service       │
└────────┬────────┘
         │
    ┌────┴─────────────┬──────────────┬──────────────┐
    ▼                  ▼              ▼              ▼
┌──────────┐   ┌─────────────┐  ┌──────────┐  ┌──────────┐
│ Customer │   │   Access     │  │  Payment │  │  Tenant  │
│ Service  │   │   Service    │  │ Gateway  │  │ Service  │
└──────────┘   └─────────────┘  └──────────┘  └──────────┘
```

Product uses Customer for product assignments, Access for product permissions, Payment Gateway for transactions, and Tenant for multi-tenant product management.

## Heavy-Lifting Features

### 1. Transactional Product Creation with SKU Validation

Atomic product and price creation with global SKU uniqueness:

```typescript
ProductService.add():
  - Validates at least one price per product
  - Validates unique price IDs within product
  - Validates unique SKUs within product
  - Runs transaction to prevent race conditions
  - Checks for duplicate SKUs globally (up to 30 SKUs per batch)
  - Creates product document
  - Creates all price subcollection documents
  - Rolls back automatically if any step fails
  - Returns combined product-with-price view
```

**What customers avoid:**

- Building transactional product creation logic
- Implementing global SKU uniqueness validation
- Writing duplicate detection across subcollections
- Managing race condition prevention
- Coordinating product-price relationship creation
- Handling partial creation rollback

### 2. Multi-Tenant Product Distribution

Flexible per-tenant product availability with price synchronization:

```typescript
TenantProductService features:
  - Create tenant-specific product associations from SKU
  - Batch product assignment to multiple tenants
  - Automatic payment integration selection (Stripe or Custom)
  - Product permission inheritance to tenant scope
  - Global queries across all tenant products
  - Query by SKU, priceId, or productId across tenants

Creation flow:
  - Fetch product and price by SKU
  - Validate tenant existence
  - Create denormalized tenant-product with:
    • Product name and description
    • Price amount and currency
    • Payment integration details
    • Quantity transforms
    • Recurring billing information
    • Product permissions
```

**What customers avoid:**

- Building multi-tenant product distribution
- Implementing product denormalization for performance
- Managing tenant-product relationship synchronization
- Writing cross-tenant product queries
- Handling permission inheritance across tenants
- Coordinating price updates across tenant products

### 3. Subscription Lifecycle Management

Complete subscription workflows with payment integration:

```typescript
ProductPaymentService.createSubscription():
  - Checks for existing active subscriptions (prevents duplicates)
  - Handles pending subscription retrieval and payment method updates
  - Creates payment gateway subscription with metadata
  - Configures auto-renewal and payment behavior
  - Expands invoice and payment intent for client secret
  - Creates local subscription tracking with status
  - Returns unified subscription view

Subscription states:
  - Pending: Awaiting first payment
  - Active: Paid and current
  - Cancelled: Terminated (soft delete preserves history)

Update capabilities:
  - Change payment method
  - Enable/disable auto-renewal
  - Cancel at period end
  - Gateway sync via async tasks
```

**What customers avoid:**

- Building subscription duplicate detection
- Implementing payment gateway integration
- Writing subscription state machine logic
- Managing pending subscription updates
- Coordinating local and gateway subscription state
- Handling subscription renewal workflows

### 4. Intelligent Payment Intent Creation

Dual-mode payment handling with automatic fallback:

```typescript
Payment routing logic:
  - Gateway available + product configured → Gateway payment intent
  - Gateway unavailable OR not configured → Custom invoice

Gateway payment intent:
  - Validates payment method ownership
  - Creates payment intent with metadata tracking
  - Sets receipt email and customer
  - Returns client secret for frontend
  - Metadata includes: customerKey, productId, priceId, intent type

Custom invoice fallback:
  - Creates invoice with line items
  - Calculates tax (configurable rate)
  - Sets due date and status
  - Returns invoice for manual processing

Payment method validation:
  - Retrieves method from gateway
  - Verifies customer ownership
  - Returns 400 if invalid or not owned
```

**What customers avoid:**

- Building payment gateway abstraction
- Implementing payment method validation
- Writing dual-mode payment routing
- Managing payment intent metadata
- Creating invoice fallback logic
- Handling receipt and customer association

### 5. Automated Customer Product Synchronization

Event-driven product assignment with bidirectional sync:

```typescript
Automatic triggers:
  - Subscription activated → Create customer product
  - Subscription deactivated → Delete customer product
  - Customer product created/deleted → Sync customer record
  - Customer product updated (productId changed) → Sync customer record

SyncCustomerProductsHandler:
  - Queries all products for customer
  - Extracts product IDs
  - Updates customer document with product array
  - Enables permission checks via customer.products

Product sources tracked:
  - Subscription (includes subscriptionId)
  - OneTimePayment (purchase completed)
```

**What customers avoid:**

- Building event-driven product assignment
- Implementing bidirectional synchronization
- Writing customer product aggregation
- Managing subscription-to-product lifecycle
- Coordinating purchase completion tracking
- Handling product source attribution

### 6. Global SKU Management with Collection Group Queries

Cross-product SKU validation and lookup:

```typescript
ProductPriceRepository.findGloballyFromSKUs():
  - Accepts array of prices (max 30)
  - Performs collection group query across all products
  - Uses Firestore 'in' operator for batch lookup
  - Returns matching prices with product context
  - Used for duplicate detection in transactions

ProductPriceRepository.getPriceFromSKU():
  - Performs global SKU search
  - Validates exactly one match (errors on 0 or >1)
  - Enables SKU-based product discovery
  - Used for tenant product creation

SKU validation rules:
  - Globally unique across all products
  - Unique within product
  - Cannot be changed after creation
  - Required for each price
```

**What customers avoid:**

- Building cross-collection SKU validation
- Implementing collection group query logic
- Writing SKU uniqueness enforcement
- Managing SKU-to-product lookup
- Handling multi-product SKU searches
- Coordinating batch SKU validation

### 7. Invoice Management with Line Item Transformation

Comprehensive invoicing with order validation:

```typescript
CustomerInvoiceService features:
  - Create invoices with multiple line items
  - Transform product/price references to full line items
  - Validate product and price existence
  - Fetch customer information for invoicing
  - Track invoice history with diff preservation
  - Support multiple invoice statuses

CreateCustomerInvoiceDto transformation:
  - Accepts product/price/quantity DTOs
  - Fetches product name and description
  - Fetches price amount and currency
  - Builds enriched IProductWithPrice objects
  - Validates all references exist
  - Returns 400 if product or price not found

Invoice statuses:
  - DRAFT, OPEN, PAID, VOID, OVERDUE, UNCOLLECTABLE

Invoice includes:
  - Customer key and information
  - Order line items with quantities
  - Total due and tax calculation
  - Due date and paid date
  - Payment notes
```

**What customers avoid:**

- Building invoice line item transformation
- Implementing product/price reference validation
- Writing invoice history tracking
- Managing invoice status workflows
- Calculating invoice totals with tax
- Coordinating customer information fetching

### 8. Default Product Provisioning

Automatic product assignment for new customers:

```typescript
DefaultProductService.createDefaultProductSubscriptions():
  - Accepts array of default product IDs
  - Builds price ID from product ID convention
  - Fetches product and price documents
  - Checks for existing subscriptions (prevents duplicates)
  - Creates Active subscription (no payment required)
  - Associates with tenant and customer
  - Marks as system product
  - Used during customer provisioning

Subscription properties:
  - Status: Active (immediate access)
  - autoRenew: true
  - isSystemProduct: true (exempt from billing)
  - tenantId: Originating tenant

Use cases:
  - Free tier products
  - Trial period products
  - Platform base features
  - Onboarding products
```

**What customers avoid:**

- Building default product provisioning
- Implementing subscription duplication detection
- Writing convention-based price lookup
- Managing system product tracking
- Coordinating tenant-customer product assignment
- Handling free tier subscription creation

### 9. Payment Gateway Integration with Async Synchronization

Seamless gateway integration with eventual consistency:

```typescript
Product creation triggers:
  - ProductCreatedEvent → CreateStripeProductCommand (if Stripe integration)
  - Creates gateway product with metadata
  - Updates local product with gateway ID

Price creation triggers:
  - PriceCreatedEvent → CreateStripePriceCommand (if Stripe integration)
  - Creates gateway price with metadata
  - Updates local price with gateway ID

Product update triggers:
  - ProductUpdatedEvent → UpdateStripeProductCommand
  - Syncs name and description to gateway

Price update triggers:
  - PriceUpdatedEvent → UpdateStripePriceCommand
  - Archives old price and creates new one (gateway immutability)

Subscription sync:
  - Subscription status changes via webhook
  - UpdateSubscriptionStatusHandler maps gateway status
  - Updates autoRenew from cancel_at_period_end

One-time purchase completion:
  - payment_intent.succeeded webhook
  - Filters by PaymentIntentType.OneTimePayment
  - Publishes ONE_TIME_PURCHASE_COMPLETED event
```

**What customers avoid:**

- Building payment gateway synchronization
- Implementing webhook event handling
- Writing gateway status mapping
- Managing gateway ID association
- Coordinating async product/price sync
- Handling gateway immutability constraints

### 10. Tenant Product Update Propagation

Automatic cascade updates across tenant products:

```typescript
UpdateTenantProductHandler:
  - Triggered by ProductUpdatedEvent
  - Queries all tenant products for product ID
  - Updates product name, description, payment integration
  - Updates permissions across all tenants
  - Runs in parallel for all affected tenants

UpdateTenantProductHandler (price updates):
  - Triggered by PriceUpdatedEvent
  - Queries all tenant products for price ID
  - Updates amount, currency, quantity transforms
  - Updates recurring billing information
  - Updates gateway references
  - Maintains consistency across tenant products

Delete cascades:
  - ProductDeletedEvent → Delete all tenant products
  - PriceDeletedEvent → Delete tenant products for price
```

**What customers avoid:**

- Building cascade update logic
- Implementing tenant product queries
- Writing parallel update orchestration
- Managing denormalized data consistency
- Coordinating delete propagation
- Handling cross-tenant synchronization

### 11. Product Permission Integration

Automatic permission synchronization with access control:

```typescript
UpsertProductPermissionsHandler:
  - Triggered by ProductCreatedEvent and ProductUpdatedEvent
  - Checks if product has permissions defined
  - Calls PermissionAdminService.upsertProductPermission
  - Maps product ID to permission arrays
  - Enables product-based access control

DeleteProductPermissionsHandler:
  - Triggered by ProductDeletedEvent
  - Removes product from permission map
  - Cleans up orphaned permissions

Permission structure:
  - Product.permissions: Record<string, string[]>
  - Maps permission keys to allowed values
  - Inherited by tenant products
  - Synced to customer on product assignment

Access control flow:
  - Customer purchases product
  - Customer product created with productId
  - Customer.products array updated
  - Permission checks validate product ownership
```

**What customers avoid:**

- Building product-permission mapping
- Implementing permission synchronization
- Writing permission cleanup logic
- Managing product-based access control
- Coordinating permission inheritance
- Handling customer permission assignment

### 12. Subscription Invoice Automation

Scheduled invoice creation for recurring subscriptions:

```typescript
CreateSubscriptionInvoiceHandler:
  - Queries all active subscriptions
  - Filters by billing cycle (due for invoice)
  - Creates invoice for each subscription
  - Populates line items from subscription
  - Sets due date based on billing period
  - Sets status to OPEN
  - Publishes invoice notification event

Invoice notification saga:
  - Waits configured delay (default: 3 days)
  - SendInvoiceNotificationCommand triggered
  - Sends email/SMS to customer
  - Includes invoice details and payment link

History tracking:
  - UpdateCustomerInvoiceHistoryHandler
  - Records all invoice changes
  - Stores diff of old vs. new values
  - Maintains audit trail
  - Enables invoice dispute resolution
```

**What customers avoid:**

- Building scheduled invoice generation
- Implementing subscription billing cycles
- Writing invoice notification workflows
- Managing invoice history tracking
- Coordinating billing period calculations
- Handling notification delays and retries

### 13. Recurring vs. One-Time Product Classification

Automatic product type detection and filtering:

```typescript
ProductService.getAllEnabled():
  - Fetches enabled products (optionally including system products)
  - Fetches all prices for each product
  - Determines type from price.recurring field
  - Returns products with type: 'recurring' | 'one-time'

ProductService.getRecurring():
  - Fetches all products
  - Filters to products with at least one recurring price
  - Returns only recurring products
  - Used for subscription product listings

Price structure:
  - recurring?: IRecurringPriceInformation {
      interval: 'day' | 'week' | 'month' | 'year'
      intervalCount?: number
    }
  - Presence determines product classification

Product with multiple prices:
  - If any price is recurring → type: 'recurring'
  - If all prices are one-time → type: 'one-time'
```

**What customers avoid:**

- Building product type classification
- Implementing recurring price detection
- Writing product filtering by billing type
- Managing mixed pricing models
- Coordinating price type aggregation
- Handling subscription-eligible product queries

### 14. Payment Customer Management

Customer tracking across payment gateways:

```typescript
IPaymentCustomer model:
  - id: Customer key
  - stripeId?: Gateway customer ID
  - Created on first payment
  - Cached for subsequent payments

PaymentCustomerPipe:
  - Extracts customerKey from claims
  - Fetches or creates payment customer
  - Injects into controller

Validation:
  - validateUserPaymentMethod()
  - Retrieves payment method from gateway
  - Verifies method.customer === paymentCustomer.stripeId
  - Returns 400 if mismatch or not found

Used for:
  - Payment intent creation
  - Subscription creation
  - Payment method validation
  - Customer billing portal
```

**What customers avoid:**

- Building payment customer abstraction
- Implementing gateway customer mapping
- Writing payment method ownership validation
- Managing customer creation timing
- Coordinating cross-gateway customer IDs
- Handling customer association in payments

### 15. Quantity Transform Support

Advanced pricing with usage-based billing:

```typescript
IQuantityTransform:
  - divideBy: number (transform multiplier)
  - round: 'up' | 'down' (rounding strategy)

Use cases:
  - Usage-based pricing (API calls, storage, etc.)
  - Tiered volume pricing
  - Unit conversion (e.g., GB to TB)

Example:
  - divideBy: 1000, round: 'up'
  - 1,250 units → 2 billing units (rounded up)

Applied in:
  - Price creation and updates
  - Invoice line item calculations
  - Tenant product pricing
  - Subscription billing

Preserved across:
  - Product price documents
  - Tenant product denormalization
  - Invoice line items
```

**What customers avoid:**

- Building usage-based pricing calculations
- Implementing quantity transformation logic
- Writing rounding strategy handling
- Managing unit conversion in billing
- Coordinating transform across entities
- Handling tiered volume pricing

### 16. Event-Driven Architecture with Task Queues

Comprehensive pub/sub for all entity changes:

```typescript
Event types published:
  - PRODUCT_CREATED/UPDATED/DELETED
  - PRICE_CREATED/UPDATED/DELETED
  - CUSTOMER_PRODUCT_CREATED/UPDATED/DELETED
  - SUBSCRIPTION_CREATED/UPDATED/DELETED
  - SUBSCRIPTION_ACTIVATED/DEACTIVATED
  - CUSTOMER_INVOICE_CREATED/UPDATED/DELETED
  - ONE_TIME_PURCHASE_COMPLETED

Task queues:
  - product-product-events (product lifecycle)
  - product-price-events (price lifecycle)
  - product-customer-product-events (ownership)
  - product-subscription-events (subscriptions)
  - product-customer-invoice-events (billing)
  - product-stripe-events (gateway webhooks)
  - product-retry (failed task retry)

Saga orchestration:
  - StripeProductSagas (gateway sync)
  - CustomerProductSagas (ownership sync)
  - StripeSubscriptionSagas (subscription sync)
  - ProductSagas (tenant product updates)
  - InvoiceNotificationSagas (billing alerts)
```

**What customers avoid:**

- Building event publishing infrastructure
- Implementing task queue routing
- Writing saga orchestration
- Managing event-driven workflows
- Coordinating cross-service events
- Handling async task retries

### 17. Permission Guard Decorators

Declarative endpoint protection:

```typescript
Guards available:
  - @HasPermission(Permissions.ViewProducts)
  - @HasPermission(Permissions.EditProducts)
  - @HasPermission(Permissions.EditProductPrice)
  - @HasPermission(Permissions.ViewBillingDetails)
  - @HasPermission(Permissions.EditBillingDetails)
  - @HasPermission(Permissions.PlatformAdminViewBillingDetails)
  - @HasPermission(Permissions.PlatformAdminEditBillingDetails)
  - @HasPermission(Permissions.PlatformAdminViewSubscriptions)
  - @HasPermission(Permissions.PlatformAdminEditSubscriptions)
  - @HasPermission(Permissions.PlatformAdminViewTenantProducts)
  - @HasPermission(Permissions.PlatformAdminEditTenantProducts)

Permission evaluation:
  - Extracts user claims from request
  - Checks user permissions against required
  - Returns 403 if insufficient permissions
  - Supports platform admin vs. tenant scope
```

**What customers avoid:**

- Writing authorization middleware
- Implementing permission checks in endpoints
- Managing role-based access control
- Building permission validation logic
- Coordinating admin vs. user permissions
- Handling multi-scope authorization

## Common Use Cases

- **SaaS subscription billing**: Recurring subscription management with payment processing and invoicing
- **E-commerce platform**: Product catalog with flexible pricing, SKU management, and payment integration
- **Multi-tenant marketplace**: Tenant-specific product availability with custom pricing per tenant
- **Usage-based billing**: Quantity transforms for API calls, storage, or metered services
- **Freemium models**: Default product provisioning with automatic subscription creation
- **Custom invoicing**: Manual invoice creation for non-standard billing scenarios
- **Product-based permissions**: Access control tied to product ownership

## What Customers Don't Have to Build

- Transactional product creation with SKU validation
- Multi-tenant product distribution with denormalization
- Subscription lifecycle management with payment gateway integration
- Dual-mode payment processing (gateway + custom invoice)
- Automated customer product synchronization
- Global SKU management with collection group queries
- Invoice management with line item transformation
- Default product provisioning for new customers
- Payment gateway integration with async synchronization
- Tenant product update propagation
- Product permission integration with access control
- Subscription invoice automation with notifications
- Recurring vs. one-time product classification
- Payment customer management across gateways
- Quantity transform support for usage-based billing
- Event-driven architecture with task queues
- Permission guard decorators for endpoints
- Payment method ownership validation
- Subscription duplicate prevention
- Invoice history tracking with diffs
- Gateway status mapping and sync
- Price immutability handling for gateway updates
- Cascade updates across tenant products
- Cross-collection SKU duplicate detection

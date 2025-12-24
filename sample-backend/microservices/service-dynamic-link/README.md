# Dynamic Link Service

URL shortening with intelligent mobile app routing and social media preview optimization.

## What It Does

The Dynamic Link Service generates short, branded URLs that intelligently route users to mobile apps or web destinations based on device type. It provides customizable social media previews with Open Graph and Twitter Card metadata, handles app store fallbacks, and manages link storage for future reference.

## Key Capabilities

| Capability                    | Description                                                           |
| ----------------------------- | --------------------------------------------------------------------- |
| **URL Shortening**            | Generate collision-resistant short URLs with customizable ID length   |
| **Platform Detection**        | JavaScript-based user agent parsing to detect iOS vs Android          |
| **App Store Routing**         | Automatic routing to App Store or Play Store based on device          |
| **Intent-Based Deep Linking** | Android Intent URLs with automatic Play Store fallback                |
| **Social Media Previews**     | Customizable Open Graph and Twitter Card metadata                     |
| **Multi-App Support**         | Configure multiple apps with different bundle IDs from single service |
| **Custom Branding**           | Configurable thumbnails, backgrounds, and preview images              |
| **404 Handling**              | Branded 404 pages with custom imagery                                 |

## How It Fits Together

```
┌─────────────────┐
│  Dynamic Link   │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────────────┬──────────────┐
    ▼                 ▼              ▼
┌───────────┐   ┌───────────┐  ┌──────────┐
│   Token   │   │  Access   │  │Community │
│  Service  │   │  Service  │  │ Service  │
└───────────┘   └───────────┘  └──────────┘
```

Dynamic links are used by token generation for email verification, access invites, and community invite flows.

## Heavy-Lifting Features

### 1. Collision-Resistant Short URL Generation

Creates unique short URLs with automatic retry on collision:

```typescript
DynamicLinkService.createShortUrl():
  - Generates cryptographically random ID using nanoid
  - Configurable character count (default 10 characters)
  - Automatic collision detection on database insert
  - Silent retry with new ID if collision occurs
  - Recursive retry until unique ID found
  - Returns fully-qualified short URL

ID characteristics:
  - URL-safe character set (A-Za-z0-9_-)
  - 10 characters = 3.2 trillion possible combinations
  - Collision probability: ~0.000001% at 100k links
```

**What customers avoid:**

- Implementing collision detection logic
- Building retry mechanisms for unique ID generation
- Calculating collision probability for URL shorteners
- Managing cryptographically random ID generation
- Handling database uniqueness constraints

### 2. Intelligent Device-Based Routing

Client-side JavaScript routing with platform detection:

```typescript
Routing logic:
  - Loads Detect.js library for user agent parsing
  - Parses navigator.userAgent to identify OS family
  - Three routing modes:
    1. Store redirect: Routes to app stores
    2. App deep link: Routes directly to app content
    3. Web fallback: Routes to web URL

iOS handling:
  - Detects userAgent.os.family === 'iOS'
  - Redirects to https://apps.apple.com/us/app/{appStoreID}
  - Falls back to web URL if appStoreID empty

Android handling:
  - Detects userAgent.os.family.includes('Android')
  - Uses Android Intent URL scheme
  - Automatic Play Store fallback via browser_fallback_url
  - Format: intent:#Intent;scheme={scheme};package={bundle};S.browser_fallback_url={encoded};end
```

**What customers avoid:**

- Writing user agent parsing logic
- Implementing Android Intent URL formatting
- Building fallback URL encoding for Play Store
- Managing platform-specific routing rules
- Testing cross-platform deep linking behavior

### 3. Rich Social Media Preview Generation

Dynamic HTML generation with Open Graph and Twitter Card support:

```typescript
WebpageBuilderService.getHtml():
  - Reads HTML template from filesystem
  - Replaces template variables with link data
  - Generates full HTML page with metadata

Metadata included:
  - Open Graph: og:title, og:description, og:image, og:type, og:locale
  - Twitter Cards: twitter:card, twitter:title, twitter:description, twitter:image
  - Apple Web App: apple-mobile-web-app-capable, apple-mobile-web-app-status-bar-style
  - Viewport configuration for mobile optimization
  - Custom favicon and apple-touch-icon support

Visual elements:
  - Custom thumbnail for social previews (thumbnailUrl)
  - Background image for loading page (backgroundImageUrl)
  - Status indicator SVG during redirect (statusImage)
  - Configurable per-link or service-wide defaults
```

**What customers avoid:**

- Building HTML template rendering systems
- Implementing Open Graph and Twitter Card specifications
- Managing social media preview image CDN integration
- Writing mobile viewport optimization rules
- Testing social media link preview rendering

### 4. Multi-App Configuration System

Support for multiple mobile apps from single deployment:

```typescript
Environment configuration:
  - APPS JSON object with app-specific settings
  - Each app has unique key for lookup
  - Per-app Android bundle ID
  - Per-app iOS App Store ID
  - Per-app title and description defaults

DynamicLinkService.buildDynamicLink():
  - Accepts appKey to select app configuration
  - Retrieves app settings from APPS environment variable
  - Validates appKey exists in configuration
  - Returns error if app not found
  - Merges app defaults with link-specific overrides

Override hierarchy:
  1. Link-specific androidBundleId/appStoreId
  2. App-specific configuration from APPS
  3. Service-wide environment defaults (ANDROID_BUNDLE_ID, APP_STORE_ID)
```

**What customers avoid:**

- Building multi-tenant app configuration systems
- Implementing configuration hierarchy with overrides
- Managing environment variable parsing and validation
- Writing app-specific routing logic
- Coordinating multiple mobile apps from one service

### 5. Template-Based HTML Rendering

Filesystem-based HTML templates with variable substitution:

```typescript
Template replacement variables:
  - {{redirectUrl}} → Target URL for redirection
  - {{title}} → Page title and og:title
  - {{description}} → Meta description and og:description
  - {{appStoreID}} → iOS App Store ID for routing
  - {{androidBundleID}} → Android package name
  - {{backgroundImage}} → Full-screen background image URL
  - {{thumbnail}} → Social media preview image
  - {{statusImage}} → Loading indicator during redirect
  - {{androidScheme}} → URL scheme for Intent links
  - {{redirectToStore}} → Boolean flag for store routing

Template loading:
  - Reads from ./assets/html/index.html at runtime
  - Separate 404 template in ./assets/html/404.html
  - Uses Node.js fs.readFileSync for template access
  - UTF-8 encoding for Unicode character support
  - String.replaceAll() for variable substitution
```

**What customers avoid:**

- Implementing template rendering engines
- Building variable substitution systems
- Managing template file paths and loading
- Writing template variable escaping logic
- Coordinating multiple HTML templates

### 6. Branded 404 Error Handling

Custom 404 pages with service branding:

```typescript
NotFoundFilter:
  - NestJS exception filter for NotFoundException
  - Intercepts all 404 errors globally
  - Renders custom HTML instead of JSON error
  - Uses WebpageBuilderService.getNotFoundHtml()

404 HTML features:
  - Custom "Link Not Found" title and description
  - Branded 404 thumbnail for social previews
  - Not-found SVG graphic for visual feedback
  - Same background styling as redirect pages
  - Open Graph and Twitter Card metadata for shared 404s
  - Maintains brand consistency on invalid links
```

**What customers avoid:**

- Building custom error page rendering
- Implementing NestJS exception filters
- Managing 404 page branding and imagery
- Writing HTML error responses for API endpoints
- Coordinating error handling across controllers

### 7. Database Persistence with Repository Pattern

Link storage for retrieval and analytics:

```typescript
DynamicLinkRepository:
  - Extends CollectionRepository<IDynamicLink>
  - Firestore collection: 'dynamic_link/'
  - Stores all link metadata and configuration
  - Enables future analytics queries

Link model (IDynamicLink):
  - id: Short URL identifier
  - url: Original destination URL
  - title: Preview title
  - description: Preview description
  - androidBundleId: Android app package
  - appStoreId: iOS App Store ID
  - thumbnailUrl: Preview image
  - backgroundImageUrl: Page background
  - redirectToStore: Store routing flag

Repository operations:
  - create(): Stores new link with metadata
  - getOrError(): Retrieves by ID with error on missing
  - Automatic timestamp injection (createdAt, updatedAt)
  - Transaction support via CollectionRepository
```

**What customers avoid:**

- Building database repository abstractions
- Implementing Firestore collection access patterns
- Writing link storage and retrieval logic
- Managing timestamp injection and versioning
- Handling document-not-found error scenarios

### 8. REST API with NestJS Controllers

HTTP endpoints for link creation and redirection:

```typescript
DynamicLinkController endpoints:

POST /li/create:
  - Creates new short link
  - Body: IDynamicLinkCreateReq (url, title, description, etc.)
  - Returns: Full short URL string
  - Validates required fields
  - Uses service-configured base URL

GET /li/link?href={url}&appKey={key}&redirectToStore={bool}:
  - Generates redirect page without persistence
  - Query params: href (destination), appKey (app config), redirectToStore (bool)
  - Returns: HTML content (content-type: text/html)
  - Throws 404 if href or appKey missing
  - Uses buildDynamicLink() for on-demand generation

GET /li/:linkId:
  - Retrieves existing short link by ID
  - Param: linkId (short URL identifier)
  - Returns: HTML redirect page
  - Fetches from database via getLink()
  - Custom 404 page via NotFoundFilter if missing
  - Renders HTML with stored link metadata
```

**What customers avoid:**

- Building REST API controllers with NestJS decorators
- Implementing query parameter and path parameter parsing
- Writing HTTP response content-type handling
- Managing GET vs POST endpoint semantics
- Coordinating controller routing and service logic

### 9. Configurable Image CDN Integration

Centralized asset management with environment configuration:

```typescript
Image configuration:
  - BASE_IMAGE_URL: CDN base for all default assets
  - THUMBNAIL_URL: Default preview image (overridable per-link)
  - BACKGROUND_IMAGE_URL: Default page background (overridable per-link)

Default CDN assets:
  - background.png: Full-screen background for redirect pages
  - status.svg: Loading indicator shown during redirect
  - 404-thumb.jpg: Social preview for 404 pages
  - not-found.svg: 404 error graphic

Configuration precedence:
  1. Link-specific URLs (thumbnailUrl, backgroundImageUrl in model)
  2. Environment variables (THUMBNAIL_URL, BACKGROUND_IMAGE_URL)
  3. Default CDN paths (BASE_IMAGE_URL + filename)

Fallback logic:
  - backgroundImage = backgroundImageUrl ?? ${baseImageUrl}/background.png
  - thumbnail = thumbnailUrl ?? '' (empty if not specified)
  - All images served from CDN for optimal performance
```

**What customers avoid:**

- Building asset CDN integration logic
- Implementing configuration precedence hierarchies
- Managing default asset paths and fallbacks
- Writing URL construction for CDN resources
- Coordinating image loading performance optimization

### 10. Token Enrichment Integration

Seamless integration with token generation workflows:

```typescript
DynamicLinkTokenEnricher pattern:
  - Extends TokenEnricher<T, {dynamicLink: string}>
  - Used by token services to add dynamic links
  - Takes link function and dynamic link options
  - Returns {dynamicLink: string} to be merged with token

Usage example:
  new DynamicLinkTokenEnricher(
    token => `https://app.example.com/verify/${token}`,
    {domainUriPrefix: 'https://short.example.com'}
  )

Integration points:
  - Token service: Email verification links
  - Access service: Invite token links
  - Community service: Community invite links

Workflow:
  1. Token generated with unique token string
  2. Enricher builds full destination URL
  3. getDynamicLink() creates short URL
  4. dynamicLink property added to token object
  5. Short link stored in token for email sending
```

**What customers avoid:**

- Building token enrichment abstractions
- Implementing dynamic link + token coordination
- Managing async enrichment workflows
- Writing link generation for email campaigns
- Coordinating token services with URL shortening

### 11. NestJS Module Architecture

Production-ready microservice structure:

```typescript
AppModule configuration:
  - ConfigModule for environment variable loading
  - DynamicLinkController for HTTP endpoints
  - DynamicLinkService for business logic
  - WebpageBuilderService for HTML generation
  - NotFoundFilter for global 404 handling

Module organization:
  - SDK packages (dynamic-link-server-api, dynamic-link-functions)
  - Implementation package (service-dynamic-link/functions)
  - Utilities package (apis/dynamic-link)
  - Separation of concerns between SDK and implementation

Deployment:
  - Exports single Cloud Function (dynamicLink)
  - Uses Express adapter for HTTP handling
  - NestJS initialization on cold start
  - Persistent NestJS app across warm invocations
```

**What customers avoid:**

- Configuring NestJS module architecture
- Building Cloud Function deployment wrappers
- Managing Express + NestJS integration
- Implementing cold start optimization
- Coordinating SDK package structure

### 12. Security with Firestore Rules

Database-level access control:

```typescript
Firestore security rules:
  match /dynamic_link/{id} {
    allow get: if true;
  }

Access pattern:
  - Read-only public access for link lookups
  - No write access from client (server-only creates)
  - No list/query access (prevents enumeration)
  - No update/delete access (immutable links)

Security benefits:
  - Links publicly accessible by ID only
  - Cannot enumerate all links in database
  - Cannot modify existing links from client
  - Cannot delete links from client
  - Server authentication required for creation
```

**What customers avoid:**

- Writing Firestore security rules
- Implementing database access control
- Managing read-only vs write permissions
- Preventing link enumeration attacks
- Coordinating server-only write patterns

## Common Use Cases

- **Email verification**: Short links in verification emails that open mobile apps or web
- **Invite links**: Shareable invite URLs with social previews for inviting users
- **Password reset**: Short URLs for password reset flows that work across platforms
- **Deep linking**: Marketing links that open specific app screens or fall back to web
- **Share buttons**: Short URLs for social sharing with optimized previews
- **QR codes**: Short URLs for QR codes that fit in small squares
- **SMS campaigns**: Short links for text messages with character limits
- **Attribution tracking**: Shortened URLs that enable future analytics queries

## What Customers Don't Have to Build

- Cryptographically random short URL generation with collision detection
- Recursive retry logic for handling ID collisions
- User agent parsing for platform detection
- Android Intent URL formatting with Play Store fallbacks
- iOS App Store URL construction and routing
- Open Graph and Twitter Card metadata generation
- HTML template rendering with variable substitution
- Multi-app configuration with override hierarchies
- Custom 404 error pages with branding
- Social media preview image optimization
- Database repository pattern for link storage
- REST API controllers with NestJS decorators
- CDN integration with fallback asset logic
- Token enrichment patterns for email workflows
- NestJS + Express + Cloud Functions architecture
- Firestore security rules for read-only access
- Client-side JavaScript redirect logic
- Mobile viewport optimization for redirect pages
- Background loading indicators during redirect
- Environment-based configuration management

# Dynamic Link Service

Deep linking and URL management.

## What It Does

The Dynamic Link Service creates and manages smart URLs that route users to the right place — whether that's a web page, mobile app, or app store. It handles deep linking, deferred deep linking, and link analytics.

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **Deep Links** | Create links that open specific app content |
| **Deferred Deep Linking** | Route users after app install |
| **Short URLs** | Generate short, shareable URLs |
| **Platform Detection** | Route to web or app based on device |
| **Link Analytics** | Track clicks, conversions, and sources |
| **Custom Domains** | Use branded short domains |

## How It Fits Together

```
┌─────────────────┐
│  Dynamic Link   │
│    Service      │
└────────┬────────┘
         │
    ┌────┴────────────┐
    ▼                 ▼
┌───────────┐   ┌───────────┐
│    Web    │   │  Mobile   │
│    App    │   │   Apps    │
└───────────┘   └───────────┘
```

The Dynamic Link Service creates URLs that intelligently route users to the right destination.

## Common Use Cases

- **App marketing**: Links that open the app or fall back to app store
- **Email campaigns**: Track engagement from email links
- **Sharing**: Shareable links that work across platforms
- **Referrals**: Trackable referral links with attribution

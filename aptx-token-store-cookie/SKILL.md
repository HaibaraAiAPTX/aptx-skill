---
name: aptx-token-store-cookie
description: "Implement cookie-based token storage using @aptx/token-store-cookie. Use when code needs to store authentication tokens in browser cookies with: (1) Configurable token/meta cookie names, (2) Automatic expiresAt to cookie expires synchronization, (3) Cookie options (path/sameSite/secure), (4) createCookieTokenStore API integration"
---

# aptx-token-store-cookie

## Quick Start

Create store with default configuration:

```ts
import { createCookieTokenStore } from "@aptx/token-store-cookie";

const store = createCookieTokenStore({
  tokenKey: "token",
  metaKey: "token_meta",
  syncExpiryFromMeta: true,
  cookie: {
    path: "/",
    sameSite: "lax",
    secure: true,
  },
});
```

## Implementation Workflow

When integrating cookie token storage:

1. Create store using `createCookieTokenStore({ tokenKey, metaKey, cookie, syncExpiryFromMeta })`
2. Keep `syncExpiryFromMeta: true` (default) to auto-sync `meta.expiresAt` → cookie `expires`
3. Set `syncExpiryFromMeta: false` only if gateway controls cookie expiration independently
4. Inject store into `@aptx/api-plugin-auth`'s `store` field
5. Test: token/meta I/O, clear, expiry sync, and disabled sync scenarios

## Documentation

- **[API Reference](references/api-reference.md)** - Method signatures and descriptions
- **[Configuration](references/configuration.md)** - All options and expiry sync rules
- **[Testing](references/testing.md)** - Mock patterns, validation examples, and test coverage checklist
- **[Extensions](references/extensions.md)** - Custom fields and type imports

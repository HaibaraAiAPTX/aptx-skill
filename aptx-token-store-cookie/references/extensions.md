# Extensions

## Custom Meta Fields

`TokenMeta` supports custom fields beyond `expiresAt`:

```ts
store.setToken("jwt", {
  expiresAt: Date.now() + 3600000,
  userId: "u123",
  scopes: ["read", "write"],
  roles: ["admin", "user"],
});
```

Custom fields are automatically serialized to JSON in the meta cookie.

## Type Imports

Import types from `@aptx/token-store`:

```ts
import type { TokenMeta, TokenRecord } from "@aptx/token-store";
```

### Type Definitions

- `TokenMeta`: Metadata object associated with token (e.g., expiresAt, userId, scopes)
- `TokenRecord`: Combined object containing both token and meta

### Usage Example

```ts
import type { TokenMeta, TokenRecord } from "@aptx/token-store";

const record: TokenRecord = {
  token: "abc123",
  meta: {
    expiresAt: Date.now() + 3600000,
    userId: "u123",
  },
};
```

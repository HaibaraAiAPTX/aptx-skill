# Testing

## Mocking js-cookie

Use vi.mock to simulate js-cookie and track set/remove calls:

```ts
import { vi } from "vitest";

const jar = new Map<string, string>();
const setCalls: Array<{ key: string; value: string; options?: unknown }> = [];

vi.mock("js-cookie", () => ({
  default: {
    get: (key: string) => jar.get(key),
    set: (key: string, value: string, options?: unknown) => {
      jar.set(key, value);
      setCalls.push({ key, value, options });
    },
    remove: (key: string) => jar.delete(key),
  },
}));
```

## Validating Cookie Expires

Check `setCalls` to verify expires is correctly set:

```ts
const expiresAt = Date.now() + 1000;
store.setToken("t1", { expiresAt });
const tokenSet = setCalls.find(x => x.key === "aptx_token");
expect(tokenSet?.options?.expires?.getTime()).toBe(expiresAt);
```

## Testing Invalid JSON

Verify `getMeta` gracefully handles invalid JSON:

```ts
jar.set("aptx_token_meta", "not-json");
expect(store.getMeta()).toBeUndefined();
```

## Test Coverage Checklist

When testing cookie token store, ensure coverage for:

- **Token I/O**: Getting and setting token values
- **Meta I/O**: Getting, setting, and parsing metadata
- **Clear operations**: Clearing token and meta cookies
- **Expiry sync**: Verifying `meta.expiresAt` → `cookie.expires` synchronization
- **Disabled sync**: Testing behavior when `syncExpiryFromMeta: false`
- **Invalid values**: Handling non-positive integers in expiresAt
- **Invalid JSON**: Graceful degradation of malformed meta cookies
- **Atomic operations**: Testing `setRecord` with undefined values

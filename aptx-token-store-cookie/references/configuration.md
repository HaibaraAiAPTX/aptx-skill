# Configuration

## Options

### `tokenKey`
Cookie name for storing the token.
- **Default**: `"aptx_token"`

### `metaKey`
Cookie name for storing the token metadata.
- **Default**: `"aptx_token_meta"`

### `syncExpiryFromMeta`
Whether to synchronize `meta.expiresAt` to cookie `expires` timestamp.
- **Default**: `true`
- **Usage**: Set to `false` if your gateway controls cookie expiration independently.

### `cookie`
Cookie options passed to js-cookie (CookieAttributes).
- **Properties**: `path`, `sameSite`, `secure`, `domain`, etc.
- **Example**:
  ```ts
  {
    path: "/",
    sameSite: "lax",
    secure: true,
  }
  ```

## ExpiresAt Synchronization Rules

### Automatic Sync Behavior
When `syncExpiryFromMeta: true` (default):

- **Valid sync**: Only positive integer values of `meta.expiresAt` are synchronized to `cookie.expires`
- **Invalid values**: Non-positive numbers or invalid values are ignored, falling back to base cookie options
- **No expiry**: If `meta.expiresAt` is undefined, no cookie expiration is set

### Gateway-Controlled Expiration
When `syncExpiryFromMeta: false`:

- Cookie expiration is managed entirely by the gateway
- `meta.expiresAt` is ignored for cookie purposes
- Base cookie options (path, sameSite, secure, etc.) still apply

## Cookie Options Merge

Cookie `expires` from `meta.expiresAt` is merged with base cookie options using spread:
- Base options are always preserved
- Only `expires` is dynamically set when syncing

## Important Notes

- Do not store sensitive tokens in readable cookies in untrusted environments
- `setRecord` with `undefined` for token/meta will clear the corresponding cookie
- Invalid JSON in meta cookie returns `undefined` (graceful degradation)

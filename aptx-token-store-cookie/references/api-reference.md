# API Reference

## Methods

### `getToken(): string | undefined`
Get the token cookie value.

### `setToken(token: string, meta?: TokenMeta): void`
Set the token (optionally with metadata).

### `clearToken(): void`
Clear both token and meta cookies.

### `getMeta(): TokenMeta | undefined`
Get the parsed metadata object from cookie.

### `setMeta(meta: TokenMeta): void`
Set metadata (JSON serialized to cookie).

### `getRecord(): TokenRecord`
Get both token and meta simultaneously.

### `setRecord(record: TokenRecord): void`
Atomically set token and meta. Passing `undefined` for either will clear the corresponding cookie.


# Cache extension to LSP

The cache extension to the Language Server Protocol (LSP) enables the language server to cache arbitrary data persistently and across multiple workspaces.
Cache items are identified by a string key and have an associated value of arbitrary type.
The client should cache items in a key/value store that optimized for fast look up.
Different workspaces and instances of the same language servers MUST share the same cache namespace, but different language servers MUST NOT.
The client may clear cache items at any time and the language server MUST NOT depend on the presence of a cache item at any time, even directly after a `cache/set` (for example, an item might not have been saved because disk space was low).

### Example use case
This allows a language server for example to cache a self-contained definition/references index for each dependency at a specific version inside the workspace and reuse those indexes in multiple workspaces and instances of the language server.

### Cache Get Request

The cache get request is sent from the server to the client to request the value of a cache item identified by the provided key.

_Request_:
* method: 'cache/get'
* params: `CacheGetParams` defined as follows:
```typescript
interface CacheGetParams {
    /**
     * The key that identifies the cache item
     */
    key: string;
}
```

_Response_:
* result: `any` the value of the cache item or `null` if it was not found in the cache.
* error: code and message set in case an exception happens during the cache get request.

### Cache Set Notification

The cache set notification is sent from the server to the client to set the value of a cache item identified by the provided key.
This is a intentionally notification and not a request because the server is not supposed to act differently if the cache set failed.

_Request_:
* method: 'cache/set'
* params: `CacheSetParams` defined as follows:
```typescript
interface CacheSetParams {
    /**
     * The key that identifies the cache item
     */
    key: string;

    /**
     * The value that should be saved
     */
    value: any;
}
```

# VFS (virtual file system) extension to LSP

The VFS extension to the Language Server Protocol (LSP) allows a language server to operate without sharing a physical file system with the client. Instead of consulting its local disk, the language server can query the client with file system operations (read file, read dir, stat, lstat).

Use cases:

* In some deployment settings, the workspace exists on the client in archived form only, such as a bare Git repository or a .zip file. Using a virtual file system allows the language server to request the minimum set of files necessary to satisfy the request, instead of unarchiving the entire workspace.
* In multitenant deployments, the local disk may not be secure enough to store private data on. (Perhaps an attacker could craft a workspace with malicious `import` statements in code and reveal portions of any file on the file system.) Using a virtual file system ensures that the language server can operate without writing private code files to disk.
* In some deployment settings, language servers must avoid shelling out to untrusted (or sometimes any) programs, for security and performance reasons. Using a virtual file system helps enforce this by making it less likely that another programmer could change the language server to invoke an external program (since it would not be able to read any of the files anyway).
* For testing and reproducibility, it is helpful to isolate the inputs to the language server. This is easier to do if you can guarantee that the language server will not access the file system.

## JSON structures

### Initialization

`ClientCapabilities` has a new field to indicate client-side support
for this extension:

```typescript
interface VFSExtClientCapabilities extends ClientCapabilities {
	// vfs indicates that the client can satisfy file system requests (with 'fs/*' methods)
	// sent by the language server.
	vfs: boolean;
}
```

`ServerCapabilities` has a new field to indicate server-side support
for this extension:

```typescript
interface VFSExtServerCapabilities extends ServerCapabilities {
	// vfs indicates that the server supports querying the client for file system
	// operations (by sending 'fs/*' requests).
	vfs: boolean;
}
```

### FileInfo

A FileInfo contains the `stat` information about a file or directory.

```typescript
interface FileInfo {
	name: string; // the file's or dir's name (excluding its parent directory names)
	size: number; // the file's size in bytes (0 for dirs)
	dir:  boolean; // whether this is a dir
}
```

## Protocol

Note that unlike other requests in LSP, the VFS extension's requests are sent by the language server to the client, not vice versa. The client, not the language server, is assumed to have full access to the workspace's contents.

The language server is allowed to request file paths outside of the workspace root. (This is common for, e.g., system dependencies.) The client may choose whether or not to satisfy these requests.

### Initialization

VFS mode is either completely enabled or completely disabled, according to the following table:

| ClientCapabilities `vfs` field | ServerCapabilities `vfs` field | Description |
|:------------------|:------------|:------------|
| true    | true      | VFS mode is enabled; language server MUST perform all file system access using the VFS protocol described below and MUST NOT read from disk |
| true    | false     | VFS mode is disabled; the client may abort if it requires VFS mode |
| false   | true      | VFS mode is disabled; the server may abort if it requires VFS mode |
| false   | false     | VFS mode is disabled |

Even if VFS mode is enabled, it is OK for language servers to treat the local file system as scratch space (e.g., for caching data), as an implementation detail.

### fs/readFile request

The `fs/readFile` request is sent from the language server to the client to read the contents of a file.

_Request_:
* method: `fs/readFile`
* params: `string` the file's absolute file system path (not URI)

_Response_:
* result: `string` the file's contents, base64-encoded
* error: code and message set (TODO: specify error code mapping)

### fs/readDir request

The `fs/readDir` request is sent from the language server to the client to list the entries in a directory.

_Request_:
* method: `fs/readDir`
* params: `string` the directory's absolute file system path (not URI)

_Response_:
* result: `FileInfo[]` with one element for each child entry (files and directories)
* error: code and message set (TODO: specify error code mapping)

### fs/stat and fs/lstat

The `fs/stat` and `fs/lstat` requests are sent from the language server to the client to retrieve metadata about a file or directory.

`fs/lstat` is identical to `fs/stat`, except that if the path parameter refers to a symbolic link, then it returns information about the link itself, not the file it refers to.

_Request_:
* method: `fs/stat` or `fs/lstat`
* params: `string` the absolute file system path (not URI)

_Response_:
* result: `FileInfo` describing the file or directory at the given path
* error: code and message set (TODO: specify error code mapping)

## Known issues

* Many language servers need to traverse the entire workspace. With even a 3-5msec RTT between the client and server, this can become slow. Consider allowing language servers to provide globs to fetch information about multiple files at once, or similar.

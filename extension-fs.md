# `fs` (file system) extension to LSP

The `fs` (file system) extension to the Language Server Protocol (LSP) allows a language server to operate without sharing a physical file system with the client. Instead of consulting its local disk, the language server can query the client with file system operations (`x/fs/{read,stat,lstat}` LSP methods).

Use cases:

* In some deployment settings, the workspace exists on the client in archived form only, such as a bare Git repository or a .zip file. Using a virtual file system allows the language server to request the minimum set of files necessary to satisfy the request, instead of unarchiving the entire workspace to disk.
* In multitenant deployments, the local disk may not be secure enough to store private data on. (Perhaps an attacker could craft a workspace with malicious `import` statements in code and reveal portions of any file on the file system.) Using a virtual file system enforces that the language server can operate without writing private code files to disk.
* In some deployment settings, language servers must avoid shelling out to untrusted (or sometimes any) programs, for security and performance reasons. Using a virtual file system helps enforce this by making it less likely that another programmer could change the language server to invoke an external program (since it would not be able to read any of the files anyway).
* For testing and reproducibility, it is helpful to isolate the inputs to the language server. This is easier to do if you can guarantee that the language server will not access the file system.

## JSON structures

### Initialization

`ClientCapabilities` has a new field to indicate client-side support
for this extension:

```typescript
interface VFSExtClientCapabilities extends ClientCapabilities {
	// vfs indicates that the client can satisfy file system requests (with 'x/fs/*' methods)
	// sent by the language server.
	vfs: boolean;
}
```

`ServerCapabilities` has a new field to indicate server-side support
for this extension:

```typescript
interface VFSExtServerCapabilities extends ServerCapabilities {
	// vfs indicates that the server supports querying the client for file system
	// operations (by sending 'x/fs/*' requests).
	vfs: boolean;
}
```

### URIMatcher

A URIMatcher specifies a URI pattern that matches files and
directories on the client.

If only the `uri` field is set (i.e., `pathPattern` is empty), then an
exact match is performed. If the `pathPattern` field is also set, the
URIMatcher can match any number of files and directories.

Examples:

`{uri: "http://example.com/foo", pathPattern: "*.txt"}` with files
	http://example.com/foo/a.txt
	http://example.com/foo/b/c.txt
	http://example.com/foo/d.zip
matches only http://example.com/foo/a.txt.

`{uri: "http://example.com/foo/a.txt"}` with files
	http://example.com/foo/a.txt
	http://example.com/foo/b/c.txt
	http://example.com/foo/d.zip
also matches only http://example.com/foo/a.txt.

``` typescript
interface URIMatcher {
	// uri is the full URI to a file/directory (if no pathPattern is provided), or
	// the URI prefix to which the pathPattern glob is joined (if provided).
	uri: string;

	// pathPattern, if set, is a glob that is joined to the end of the uri field's
	// URI path component and matched against files/directories on the client's
	// file system.
	//
	// Glob matching is ONLY performed on the URI path component (and special
	// characters such as ?[]*-\ are interpreted as glob special characters, not
	// as URI special characters).
	//
	// TODO: is the ** glob operator (match zero or more directories) supported?
	pathPattern?: string;
}
```

### FileContents

ReadResult contains the contents of multiple files.

``` typescript
interface FileContents {
	// files contains each file's contents, base64-encoded, in the property
	// corresponding to its URI.
	contents: { [uri: string]: string };
}
```

### FileStat

A FileStat contains the `stat` information about a file or directory.

```typescript
interface FileStat {
	name:    string; // the file's or dir's name (excluding its parent directory names)
	size:    number; // the file's size in bytes (0 for dirs)
	dir:     boolean; // whether this is a dir
	symlink: boolean; // whether this is a symbolic link (only returned by lstat)
}
```

A FileStats contains the `stat` information about multiple matched files and directories.

``` typescript
interface FileStats {
	stats: { [uri: string]: FileStat };
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

### x/fs/read request

The `x/fs/read` request is sent from the language server to the client to read file contents.

_Request_:
* method: `x/fs/read`
* params: `URIMatcher` the URIs of files to read

_Response_:
* result: `FileContents` the contents of the files at the specified URIs
* error: if the `URIMatcher` is exact, an error code and message if the operation fails; if an error occurs on a specific file matched by glob, it is ignored

### x/fs/stat and x/fs/lstat request

The `x/fs/stat` and `x/fs/lstat` requests are sent from the language server to the client to retrieve metadata about a file or directory.

`x/fs/lstat` is identical to `x/fs/stat`, except that if a matched file is a symbolic link, then it returns information about the link itself, not the file it refers to.

_Request_:
* method: `x/fs/stat` or `x/fs/lstat`
* params: `URIMatcher` the URIs of files to retrieve metadata for

_Response_:
* result: `FileStats` describing the matched files and directories
* error: if the `URIMatcher` is exact, an error code and message if the operation fails; if an error occurs on a specific file matched by glob, it is ignored

## Design notes

* The protocol uses URIs, not file paths, to be consistent with the rest of LSP.
* To avoid requiring language servers to understand custom URI schemes, we only require that servers glob-match on the URI path component. (If we allowed globs in any part of the URI, how would we distinguish between `?` meaning glob zero-or-one or URI querystring?)
* We avoid needing to return multiple error objects by only returning an error if URI is exact. If we return multiple error objects, we're essentially reimplementing JSON-RPC 2.0 batching at the application level.

## Known issues

* Many language servers need to traverse the entire workspace. With even a 3-5msec RTT between the client and server, this can become slow. Consider allowing language servers to provide globs to fetch information about multiple files at once, or similar.
* There is no readlink for getting the destination path of a symlink.
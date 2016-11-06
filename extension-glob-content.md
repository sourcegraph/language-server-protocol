# Glob & content extension to LSP

The glob & content extension to the Language Server Protocol (LSP) allows a language server to operate without sharing a physical file system with the client. Instead of consulting its local disk, the language server can query the client for a list of all files/directories and for the contents of specific files.

Use cases:

* In some deployment settings, the workspace exists on the client in archived form only, such as a bare Git repository or a .zip file. Using virtual file system access allows the language server to request the minimum set of files necessary to satisfy the request, instead of writing the entire workspace to disk.
* In multitenant deployments, the local disk may not be secure enough to store private data on. (Perhaps an attacker could craft a workspace with malicious `import` statements in code and reveal portions of any file on the file system.) Using a virtual file system enforces that the language server can operate without writing private code files to disk.
* In some deployment settings, language servers must avoid shelling out to untrusted (or sometimes any) programs, for security and performance reasons. Using a virtual file system helps enforce this by making it less likely that another programmer could change the language server to invoke an external program (since it would not be able to read any of the files anyway).
* For testing and reproducibility, it is helpful to isolate the inputs to the language server. This is easier to do if you can guarantee that the language server will not access the file system.

## Protocol

Note that unlike other requests in LSP, these requests are sent by the language server to the client, not vice versa. The client, not the language server, is assumed to have full access to the workspace's contents.

The language server is allowed to request file paths outside of the workspace root. (This is common for, e.g., system dependencies.) The client may choose whether or not to satisfy these requests.

### Initialization

`ClientCapabilities` may contain two new fields to indicate client-side support for this extension:

```typescript
interface ClientCapabilities {
	/* ... all fields from the base ClientCapabilities ... */

	/**
	 * The client provides support for workspace/xglob.
	 */
	globProvider?: boolean;
	/**
	 * The client provides support for textDocument/content.
	 */
	contentProvider?: boolean;
}
```

### Content Request

The content request is sent from the server to the client to request the current content of any text document. This allows language servers to operate without accessing the file system directly.

_Request_:
* method: 'textDocument/xcontent'
* params: `ContentParams` defined as follows:

```typescript
interface ContentParams {
	/**
	 * The text document to receive the content for.
	 */
	textDocument: TextDocumentIdentifier;
}
```

_Response_:
* result: `Content` defined as follows
* error: code and message set in case an exception occurs

```typescript
interface Content {
	/**
	 * The text of the document.
	 */
	text: string;
}
```

### Glob Request

The glob request is sent from the server to the client to request a list of files in the workspace that match a glob pattern. The glob pattern must be matched only against the path (not other fragments of the URI) and must be relative to the workspace's `rootPath` (set during initialization).

The glob patterns must be interpreted according to the following rules:
* `*` matches any sequence of non-path separator characters
* `**` when alone in a path component, matches all files and zero or more directories and subdirectories ("globstar"); does not walk symlinks
* Only files, not directories, are matched
* `![]{}?` characters are interpreted literally and do not have any special meaning (TEMP NOTE: this is to simplify the implementation of globbing; we will need to write our own simple Go globbing library, and a brief survey of many languages' globbing libraries shows wide disparities in behavior)

A language server can use the result to index files by doing a content request for each URI. Usage of `TextDocumentIdentifier` here allows to easily extend the result with more properties in the future without breaking BC.

_Request_:
* method: 'workspace/xglob'
* params: `GlobParams` defined as follows:

```typescript
interface GlobParams {
	/**
	 * A list of glob patterns. A file is matched if it matches
	 * one or more glob patterns. An empty list or glob pattern
	 * matches no files.
	 */
	patterns: string[];
}
```

_Response_:
* result: `TextDocumentIdentifier[]`
* error: code and message set in case an exception occurs

Examples:

Relative (`rootPath` is `file:///some/project`):

```json
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "workspace/xglob",
	"params": {
		"patterns": ["**/*.php", "**/*.json"]
	}
}
```
```json
{
	"jsonrpc": "2.0",
	"id": 1,
	"result": [
		{"uri": "file:///some/project/1.php"},
		{"uri": "file:///some/project/composer.json"}
		{"uri": "file:///some/project/folder/2.php"},
		{"uri": "file:///some/project/folder/folder/3.php"}
	]
}
```

Absolute:

```json
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "workspace/xglob",
	"params": {
		"patterns": ["/usr/local/go/**/*"]
	}
}
```
```json
{
	"jsonrpc": "2.0",
	"id": 1,
	"result": [
		{"uri": "file:///usr/local/go/1.go"},
		{"uri": "file:///usr/local/go/folder/2.go"},
		{"uri": "file:///usr/local/go/folder/folder/3.go"}
	]
}


## Design notes

* The protocol uses URIs, not file paths, to be consistent with the rest of LSP.
* To avoid requiring language servers to understand custom URI schemes, we only require that servers glob-match on the URI path component. (If we allowed globs in any part of the URI, how would we distinguish between `?` meaning glob zero-or-one or URI querystring?)

## Known issues

* There is no readlink for getting the destination path of a symlink.
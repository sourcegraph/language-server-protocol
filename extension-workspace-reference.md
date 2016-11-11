# workspace/reference extension to LSP

The `workspace/reference` extension to the Language Server Protocol (LSP) enables a language server to export all of the references to dependencies that code inside of the workspace makes.

Use case: clients of a language server can invoke `workspace/reference` in order to find external references to dependencies. This information can then be stored inside of a database, which allows the caller to create a 'global mapping' of symbols in dependencies to the workspace they are used in (e.g. to see "how do other people use this symbol?").

### Initialization

`ServerCapabilities` may contain a new field to indicate server-side support for this extension:

```typescript
interface ServerCapabilities {
  /* ... all fields from the base ServerCapabilities ... */

  /**
   * The server provides workspace reference exporting support.
   */
  workspaceReferenceProvider?: boolean;
}
```

#### Workspace Reference Request

The workspace reference request is sent from the client to the server to export project-wide references to dependencies. That is, the response strictly returns references in the project to symbols defined in dependencies.

_Request_
* method: 'workspace/reference'
* params: `WorkspaceReferenceParams` defined as follows:
```typescript
/**
 * The parameters of a Workspace Reference Request.
 */
interface WorkspaceReferenceParams {
}
```

_Response_:
* result: `ReferenceInformation[]` defined as follows:
```typescript
/**
 * Represents information about a reference to programming constructs like
 * variables, classes, interfaces etc.
 */
interface ReferenceInformation {
	/**
	 * The location in the workspace where the `symbol` has been referenced.
	 */
	reference: Location;

	/**
	 * Metadata describing the symbol that is being referenced. It is up to the
	 * language server to define what exact data this object contains. In
	 * general, the response includes the same information that `workspace/symbol`
	 * would return, but that is only a guideline / not a hard constraint.
	 *
	 * The information does NOT always uniquely qualify the symbol. The caller
	 * should effectively consider the returned information to be information
	 * about the symbol which _generally_ (but NOT always) identifies a single
	 * symbol.
	 *
	 * The following keys are reserved to have special meaning, but are entirely
	 * optional:
	 *
	 * 	- `"name"` (same as `SymbolInformation.name`)
	 * 	- `"kind"` (same as `SymbolInformation.kind`)
	 * 	- `"file"` (same as `SymbolInformation.location.uri`)
	 * 	- `"containerName"` (same as `SymbolInformation.containerName`)
	 * 	- `"repo"`: A repository URI like `github.com/golang/go` or `bitbucket.org/jespern/django-piston`
	 *
	 * A language server is encouraged to include additional information about
	 * the symbol. Specifically, any information that a user querying against
	 * the returned information might be interested in. For example, with Go, a
	 * user may wish to include or exclude vendored packages from their search
	 * query, so the Go language server would include `"vendor": true` or `"vendor": false`
	 * entry in the response which someone indexing this object could filter
	 * based on. Another example would be including the canonical npm package
	 * name such that, for example, a user could search for symbols defined in `"npm": "react"`.
	 */
	symbol: Object;
}
```
* error: code and message set in case an exception happens during the workspace reference request.

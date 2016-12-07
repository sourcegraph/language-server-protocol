# workspace/xreference extension to LSP

The `workspace/xreference` extension to the Language Server Protocol (LSP) enables a language server to export all of the references to dependencies that code inside of the workspace makes.

Use case: clients of a language server can invoke `workspace/xreference` in order to find external references to dependencies. This information can then be stored inside of a database, which allows the caller to create a 'global mapping' of symbols in dependencies to the workspace they are used in (e.g. to see "how do other people use this symbol?").

### Initialization

`ServerCapabilities` may contain a new field to indicate server-side support for this extension:

```typescript
interface ServerCapabilities {
  /* ... all fields from the base ServerCapabilities ... */

  /**
   * The server provides workspace reference exporting support.
   */
  xworkspaceReferenceProvider?: boolean;
}
```

#### Workspace Reference Request

The workspace reference request is sent from the client to the server to export project-wide references to dependencies. That is, the response strictly returns references in the project to symbols defined in dependencies.

_Request_
* method: 'workspace/xreference'
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
	 * Metadata information describing the symbol being referenced.
	 */
	symbol: ReferenceSymbolInformation;
}
```
* error: code and message set in case an exception happens during the workspace reference request.

Where `ReferenceSymbolInformation` is defined as follows:

```typescript
/**
 * Represents information about a programming construct like a variable, class,
 * interface etc that has a reference to it. Effectively, it contains data similar
 * to SymbolInformation except all fields are optional and a metadata field is
 * present for language-specific data.
 *
 * ReferenceSymbolInformation does NOT always uniquely identify a symbol. The
 * caller should effectively consider the returned information to be
 * information about the symbol which _generally_ (but NOT always) identifies a
 * single symbol.
 */
interface ReferenceSymbolInformation {
    /**
     * The name of this symbol (same as `SymbolInformation.name`).
     */
    name?: string;

    /**
     * The kind of this symbol (same as `SymbolInformation.kind`).
     */
    kind?: number;

    /**
     * The URI of this symbol (same as `SymbolInformation.location.uri`).
     */
    uri?: string;

    /**
     * The name of the symbol containing this symbol (same as `SymbolInformation.containerName`).
     */
    containerName?: string;

    /**
     * The repository URI that the symbol is defined in, e.g. `github.com/golang/go`
     * or `bitbucket.org/jespern/django-piston`.
     */
    repo?: string;

    /**
     * The 'package', 'library', or 'crate' name that the symbol is defined in.
     * For example, in JS/TS this would be the npm module name. In Go, the full
     * package import path. In PHP, the Composer package name. etc.
     */
    packageName?: string;

    /**
     * Whether or not the symbol is defined inside of "vendored" code. In Go, for
     * example, this means that an external dependency was copied to a subdirectory
     * named `vendor`. The exact definition of vendor depends on the language,
     * but it is generally understood to mean "code that was copied from it's
     * original source and now lives in our project directly".
     */
    vendor?: boolean;
}
```

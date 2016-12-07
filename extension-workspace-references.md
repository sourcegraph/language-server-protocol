# workspace/xreferences extension to LSP

The `workspace/xreferences` extension to the Language Server Protocol (LSP) enables a language server to export all of the references made from the workspaces code to its dependencies. Additionally, a new `textDocument/xdefinition` method performs the equivalent of `textDocument/definition` while returning metadata about the definition and _optionally_ a concrete location to it.

Use case: clients of a language server can invoke `workspace/xreferences` in order to find references to dependencies. This information can then be stored inside of a database, which allows the caller to create a 'global mapping' of symbols in dependencies to the workspaces they are used in (e.g. to see "how do other people use this symbol?"). A user would perform `textDocument/xdefinition` in order to locate the metadata about the symbol they are interested in and find its references in the database.

### Initialization

`ServerCapabilities` may contain a new field to indicate server-side support for this extension:

```typescript
interface ServerCapabilities {
  /* ... all fields from the base ServerCapabilities ... */

  /**
   * The server provides workspace references exporting support.
   */
  xworkspaceReferencesProvider?: boolean;
}
```

#### Workspace References Request

The workspace references request is sent from the client to the server to export project-wide references to dependencies. That is, the response only returns references in the project to symbols defined in dependencies.

_Request_
* method: 'workspace/xreferences'
* params: `WorkspaceReferencesParams` defined as follows:
```typescript
/**
 * The parameters of a Workspace References Request.
 */
interface WorkspaceReferencesParams {
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
* error: code and message set in case an exception happens during the workspace references request.

Where `ReferenceSymbolInformation` is defined as follows:

```typescript
/**
 * Represents information about a programming construct like a variable, class,
 * interface etc that has a reference to it. Effectively, it contains data similar
 * to SymbolInformation except all fields are optional.
 *
 * ReferenceSymbolInformation usually uniquely identifies a symbol, but it is
 * not guaranteed to do so.
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
     * but it is generally understood to mean "code that was copied from its
     * original source and now lives in our project directly".
     */
    vendor?: boolean;
}
```

### Goto Definition Extension Request

This method is almost exactly the same as `textDocument/definition`, except that:

1. The method returns metadata about the definition (the same metadata returned by `workspace/xreferences`).
2. The concrete location to the definition (`location` field) is optional. This is useful because the language server might not be able to resolve to a concrete location (e.g. due to lack of dependencies) but still may know _some_ information about it (e.g. `name` and `containerName`).

_Request_
* method: 'textDocument/xdefinition'
* params: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_:
* result: `LocationInformation` defined as follows:
```typescript
interface LocationInformation {
    /* A concrete location at which the definition is located, if any. */
    location?: Location;
    /* Metadata about the definition */
    symbol: ReferenceSymbolInformation;
}
```
* error: code and message set in case an exception happens during the definition request.

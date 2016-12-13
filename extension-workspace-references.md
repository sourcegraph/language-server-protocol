# workspace/xreferences extension to LSP

The `workspace/xreferences` extension to the Language Server Protocol (LSP) enables a language server to export all of the references made from the workspace's code to its dependencies. Additionally, a new `textDocument/xdefinition` method performs the equivalent of `textDocument/definition` while returning metadata about the definition and _optionally_ a concrete location to it.

Use case: clients of a language server can invoke `workspace/xreferences` in order to find references to dependencies. This information can then be stored in a database, which allows the caller to create a "global mapping" of symbols in dependencies to the workspaces they are used in (e.g. to see "how do other people use this symbol?"). A user would perform `textDocument/xdefinition` in order to locate the metadata about the symbol they are interested in and find its references in the database.

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

The workspace references request is sent from the client to the server to locate project-wide references to a symbol given its description / metadata.

_Request_
* method: 'workspace/xreferences'
* params: `WorkspaceReferencesParams` defined as follows:
```typescript
/**
 * The parameters of a Workspace References Request.
 */
interface WorkspaceReferencesParams {
    /**
     * Metadata about the symbol that is being searched for.
     */
    query: SymbolDescriptor;

    /**
     * An optional list of files to restrict the search to.
     */
    files?: string[];
}
```

_Response_:
* result: `ReferenceInformation[]` defined as follows:
```typescript
/**
 * Represents information about a reference to programming constructs like
 * variables, classes, interfaces, etc.
 */
interface ReferenceInformation {
    /**
     * The location in the workspace where the `symbol` has been referenced.
     */
    reference: Location;

    /**
     * Metadata information describing the symbol being referenced.
     */
    symbol: SymbolDescriptor;
}
```
* error: code and message set in case an exception happens during the workspace references request.

Where `SymbolDescriptor` is defined as follows:

```typescript
/**
 * Represents information about a programming construct like a variable, class,
 * interface, etc that has a reference to it. It is up to the language server
 * to define the schema of this object.
 *
 * SymbolDescriptor usually uniquely identifies a symbol, but it is
 * not guaranteed to do so.
 */
interface SymbolDescriptor {
    [attr: string]: any
}
```

### Goto Definition Extension Request

This method is the same as `textDocument/definition`, except that:

1. The method returns metadata about the definition (the same metadata that `workspace/xreferences` searches for).
2. The concrete location to the definition (`location` field) is optional. This is useful because the language server might not be able to resolve a goto definition request to a concrete location (e.g. due to lack of dependencies) but still may know _some_ information about it.

_Request_
* method: 'textDocument/xdefinition'
* params: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_:
* result: `LocationInformation[]` defined as follows:
```typescript
interface LocationInformation {
    /* A concrete location at which the definition is located, if any. */
    location?: Location;
    /* Metadata about the definition. */
    symbol: []SymbolDescriptor;
}
```
* error: code and message set in case an exception happens during the definition request.

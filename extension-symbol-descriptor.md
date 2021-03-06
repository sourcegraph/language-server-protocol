# SymbolDescriptor extensions to LSP

A `SymbolDescription` contains metadata about a symbol to identify it as uniquely as possible.

The `SymbolDescriptor` extensions to the Language Server Protocol (LSP) allow a language server to able to interact with `SymbolDescriptor`s via the following methods:
- `textDocument/xdefinition` returns a `SymbolDescriptor` for a symbol at given location
- `workspace/xreferences` locates project wide references to a symbol, given a `SymbolDescriptor`
- `workspace/symbol` search for definitions of a symbol, given a `SymbolDescriptor`

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

  /**
   * The server provides extended text document definition support.
   */
  xdefinitionProvider?: boolean;

  /**
   * The server provides support for querying symbols by properties
   * with WorkspaceSymbolParams.symbol
   */
  xworkspaceSymbolByProperties?: boolean;
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
    query: Partial<SymbolDescriptor>;

    /**
     * Hints provides optional hints about where the language server should
     * look in order to find the symbol (this is an optimization). It is up to
     * the language server to define the schema of this object.
     */
    hints: {
        [hint: string]: any;
    }
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
     * The location in the workspace where the `symbol` is referenced.
     */
    reference: Location;

    /**
     * Metadata about the symbol that can be used to identify or locate its
     * definition.
     */
    symbol: SymbolDescriptor;
}
```
* error: code and message set in case an exception happens during the workspace references request.

Where `SymbolDescriptor` is defined as follows:

```typescript
/**
 * Represents information about a programming construct that can be used to
 * identify and locate the construct's symbol. The identification does not have
 * to be unique, but it should be as unique as possible. It is up to the
 * language server to define the schema of this object.
 *
 * In contrast to `SymbolInformation`, `SymbolDescriptor` includes more concrete,
 * language-specific, metadata about the symbol.
 */
interface SymbolDescriptor {
    /**
     * A list of properties of a symbol that can be used to identify or locate
     * it.
     */
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
* result: `SymbolLocationInformation[]` defined as follows:
```typescript
interface SymbolLocationInformation {
    /* The location where the symbol is defined, if any. */
    location?: Location;

    /**
     * Metadata about the symbol that can be used to identify or locate its
     * definition.
     */
    symbol: SymbolDescriptor;
}
```
* error: code and message set in case an exception happens during the definition request.


### Extended Workspace Symbol Request

The `workspace/symbol` request takes an optional parameter `symbol` that allows you to query by known properties about the symbol.
The string `query` parameter becomes optional.
If both `query` and `symbol` are provided, both should be matched with AND semantics.

#### Differences between `symbol` and `query`

 `query`                             | `symbol`
-------------------------------------|------------------------------------
 comes from user input in UI         | used programmatically
 matches as fuzzily as possible      | matches as exact as possible
 returns as many results as possible | returns as few results as possible


```typescript
/**
 * The parameters of a Workspace Symbol Request.
 */
interface WorkspaceSymbolParams {
    /**
     * A query string
     */
    query?: string;

    /**
     * Known properties about the symbol.
     */
    symbol?: Partial<SymbolDescriptor>;
}
```

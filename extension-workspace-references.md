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

### Extended `SymbolInformation`

This extension defines a few more properties on `SymbolInformation`.

```ts
interface SymbolInformation {

    // ... all properties from the base SymbolInformation

    /**
     * A globally unique identifier for the the symbol, if the language server is able to construct one.
     */
    id?: string;

    /**
     * Contains information about the package this symbol is contained in.
     * The properties that describe the package are language-dependent.
     */
    package?: { [key: string]: any; };
}
```

### Workspace References Request

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
    query: Partial<SymbolInformation>;

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
    symbol: Partial<SymbolInformation>;
}
```
* error: code and message set in case an exception happens during the workspace references request.

### Goto Definition Extension Request

This method is the same as `textDocument/definition`, except that instead of returning only the location, it returns a partial `SymbolInformation` with as many information about the symbol as possible.

The result can be passed into `workspace/xreferences` to search for.

_Request_
* method: 'textDocument/xdefinition'
* params: [`TextDocumentPositionParams`](#textdocumentpositionparams)

_Response_:
* result: `Partial<SymbolInformation>[]`
* error: code and message set in case an exception happens during the definition request.


### Extended Workspace Symbol Request

The `workspace/symbol` request takes an optional parameter `symbol` that allows you to query by known properties about the symbol.
The string `query` parameter becomes optional.
If both `query` and `symbol` are provided, both should both be matched with AND semantics.

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

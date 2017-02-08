# workspace/xreferences extension to LSP

The `workspace/xreferences` extension to the Language Server Protocol (LSP) enables a language server to export all of the references made from the workspace's code to its dependencies. Additionally, a new `textDocument/xdefinition` method performs the equivalent of `textDocument/definition` while returning metadata about the definition and _optionally_ a concrete location to it.

Use case: clients of a language server can invoke `workspace/xreferences` in order to find references to dependencies. This information can then be stored in a database, which allows the caller to create a "global mapping" of symbols in dependencies to the workspaces they are used in (e.g. to see "how do other people use this symbol?"). A user would perform `textDocument/xdefinition` in order to locate the metadata about the symbol they are interested in and find its references in the database.

### New Identifiers

#### Packages

The extension adds the concept of "packages" to LSP.
A package is specifically something that would usually be installed through a package manager like npm, Composer or Maven.
A workspace can contain multiple packages in the form of own packages and dependencies.
A package is identified by a language-specific interface, that is defined by the language server:

```ts
/**
 * The minimal set of properties that identify the package globally, as uniquely as possible.
 * Depending on the language, can contain a name, a publisher, a package manager, a URL, ...
 */
interface PackageIdentifier {
    [prop: string]: any;
}
```

Examples:

```ts
// TypeScript/JavaScript
{ name: "express" }
{ url: "github.com/some/repo" }
// PHP
{ name: "symfony/symfony" }
// Java
// ?
// Go
// ?
```

#### Symbols

```ts
/**
 * The minimal set of properties needed to identify the symbol globally as uniquely as possible
 */
interface SymbolIdentifier {
    /**
     * If the symbol is part of a package (in a dependency or root package), this property must contain the PackageIdentifier
     */
    package?: PackageIdentifier;

    [prop: string]: any;
}
```

Examples:

```ts
// TypeScript
// PHP
{ fqsen: "SomeNamespace\\SomeClass::someMethod()" }
// Java
// Go
```

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
     * Known properties about the identity of the symbol (for example, the package identifier)
     */
    query: Partial<SymbolIdentifier>;

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
     * Properties of the symbol to identify it
     */
    symbol: SymbolIdentifier;
}
```
* error: code and message set in case an exception happens during the workspace references request.

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
     * Properties of the symbol to identify it
     */
    symbol: SymbolIdentifier;
}
```
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
    symbol?: Partial<SymbolIdentifier>;
}
```

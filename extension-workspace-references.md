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
    symbol: SymbolDescriptor;
}
```
* error: code and message set in case an exception happens during the workspace references request.

Where `SymbolDescriptor` is defined as follows:

```typescript
/**
 * Represents information about a programming construct like a variable, class,
 * interface etc that has a reference to it. Effectively, it contains data similar
 * to SymbolInformation except all fields are optional.
 *
 * SymbolDescriptor usually uniquely identifies a symbol, but it is
 * not guaranteed to do so.
 */
interface SymbolDescriptor {
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
     * The container of this symbol. It is up to the language server to define
     * what exact data this object contains.
     */
    container?: Object;

    /**
     * Whether or not the symbol is defined inside of "vendored" code. In Go, for
     * example, this means that an external dependency was copied to a subdirectory
     * named `vendor`. The exact definition of vendor depends on the language,
     * but it is generally understood to mean "code that was copied from its
     * original source and now lives in our project directly".
     */
    vendor?: boolean;

    /**
     * Information about the package/library that this symbol is defined in.
     */
    package: PackageDescriptor;

    /**
     * Attributes describing the symbol that is being referenced. It is up to the
     * language server to define what exact data this object contains.
     */
    attributes: Object;
}
```

Where `PackageDescriptor` is defined as follows:

```
/**
 * Represents information about a programming code unit like a package,
 * library, crate, module, etc. It uniquely identifies a package at a specific
 * version within a given registry.
 */
interface PackageDescriptor {
    /**
     * An ID representing the package of code. For example, in JS this would be
     * the NPM package name. In Go, the full import path. etc.
     */
    id: string;

    /**
     * The version of the package in the registry.
     */
    version?: VersionDescriptor;

    /**
     * The registry for this package. Examples:
     *
     *  - JS: "npm"
     *  - Java: "maven" etc.
     *  - Go: "go"
     *  
     */
    registry: string;
}
```

Where `VersionDescriptor` is defined as:

```
/**
 * Represents a specific version. It can be either language-server / build tool
 * defined fields OR semantically version fields (which are preferable).
 */
type VersionDescriptor = Object | {
    commitID?: string
    major?: number
    minor?: number
    patch?: number
    tag?: string
};
```

### Goto Definition Extension Request

This method is almost exactly the same as `textDocument/definition`, except that:

1. The method returns metadata about the definition (the same metadata returned by `workspace/xreferences`).
2. The concrete location to the definition (`location` field) is optional. This is useful because the language server might not be able to resolve to a concrete location (e.g. due to lack of dependencies) but still may know _some_ information about it (e.g. `name` and `containerName`).

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

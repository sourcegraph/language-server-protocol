# `exec` extension to LSP

The `exec` extension allows the server to send a command (e.g. `git blame`) to the client for execution, and the client responds with the stdout of the command.

### `exec` Request

_Request_:
* method: 'xexec'
* params: `ExecParams` defined as follows:
```typescript
interface ExecParams {
    /**
     * The name of the command to run
     */
    name: string;
    /**
     * The arguments to the command
     */
    arguments: string[];
}
```

_Response_:
* result: `ExecResult` defined as follows:
```typescript
interface ExecResult {
    /**
     * The stdout of the process
     */
    stdout: string;
    /**
     * The exit code of the process
     */
    exitCode: string;
}
```
* error: code and message set in case the command was not found.

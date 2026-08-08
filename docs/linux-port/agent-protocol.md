# Agent protocol contract for the Linux port

This document is intentionally exact and source-grounded. It captures the current protocol as implemented by [Sources/InterAgent/InterAgentServer.swift](../../Sources/InterAgent/InterAgentServer.swift), [Sources/InterAgent/CLIRouter.swift](../../Sources/InterAgent/CLIRouter.swift), and the specific handlers under [Sources/InterAgent/Handlers/](../../Sources/InterAgent/Handlers/).

## 1. Transport and socket behavior

The current implementation exposes two local transports:

- a TCP listener on `127.0.0.1` with a dynamically assigned port
- a Unix domain socket at `~/.open-maestri/run/agent.sock` (the Swift code uses `PersistenceManager.shared.appDataURL/appendingPathComponent("run")/agent.sock`)

The Linux port should preserve the same local-only semantics and should prefer:

- `$XDG_RUNTIME_DIR/open-maestri/agent.sock` on Linux
- a fallback location when `$XDG_RUNTIME_DIR` is not set

The protocol is HTTP-like, not a bespoke binary framing. The server reads until it has a full request and then replies with a plain-text HTTP response.

## 2. Framing and encoding

### Request framing

The server parses requests using the following rules:

- it looks for `\r\n\r\n` to detect the header/body boundary
- it reads `Content-Length` from the headers when present
- it accumulates bytes until the body is complete
- it accepts a request body as UTF-8 text

### Request body shape

The request body is JSON and must contain an `args` array:

```json
{ "args": ["list", "foo", "bar"] }
```

### Response framing

The server replies with:

- `HTTP/1.0` or `HTTP/1.1` depending on the request line
- `200 OK`
- `Content-Type: text/plain; charset=utf-8`
- `Connection: close`
- a UTF-8 plain-text body

## 3. Exact request structure

The current router expects:

- a top-level JSON object with an `args` key
- `args` must be an array of strings
- `args[0]` is treated as the command name

The current implementation does not define a richer JSON envelope for request metadata. The `X-Terminal-ID` header is used as an optional routing key, but the body itself only carries `args`.

### Header requirements

- `X-Terminal-ID: <uuid>` is optional from the router’s perspective, but it is used to scope the request to a terminal session when present
- `Content-Type` is not parsed as a semantic field by the router
- `Content-Length` is used only for framing

## 4. Exact response structure

The router returns a string. The server wraps that string in an HTTP response body. In other words, the protocol is effectively:

- request -> JSON body with `args`
- response -> plain text body

There is no structured JSON response envelope at present. All commands return plain text and the CLI prints it directly.

## 5. Errors and failure modes

The current handlers return plain-text error strings that begin with `error:`. Examples include:

- `error: missing args`
- `error: unknown command 'foo'. Try 'omaestri list' for available commands.`
- `error: missing terminal ID`
- `error: agent 'foo' not found. Use 'omaestri list' to see connected agents.`

The Linux port should preserve the same plain-text error style. It should not invent a structured error envelope unless a compatibility migration is explicitly introduced.

## 6. Environment variables and terminal identity

The shell/agent process created for a terminal receives environment variables that allow it to discover the app endpoint and its own identity.

The current implementation injects environment variables including:

- `MAESTRI_SOCKET`
- `MAESTRI_TERMINAL_ID`
- `OMAESTRI_TERMINAL_ID`
- `TERM=xterm-256color`

The Linux port must preserve the identity semantics even if the names are aliased or normalized. In particular:

- the runtime must know which terminal instance is making the request
- the same terminal UUID must remain stable across workspace reloads and reconnects
- the environment variables must be available to the child shell or coding agent process

## 7. Routing behavior

The router in [Sources/InterAgent/CLIRouter.swift](../../Sources/InterAgent/CLIRouter.swift) dispatches the first argument to the following handlers:

- `list`
- `ask`
- `check`
- `note`
- `portal`
- `recruit`
- `dismiss`
- `connect`
- `role`
- `preset`
- `debug`

The Linux backend should preserve this command surface exactly.

## 8. Command contract by command

### 8.1 `list`

Usage: `omaestri list`

Behavior derived from [Sources/InterAgent/Handlers/ListHandler.swift](../../Sources/InterAgent/Handlers/ListHandler.swift):

- requires a terminal ID
- resolves the current terminal and its connected nodes
- reports the current terminal as `You:`
- reports connected agents, portals, and notes
- returns `No connections.` when nothing is connected

### 8.2 `ask`

Usage: `omaestri ask "TargetAgent" "prompt"`

Behavior derived from [Sources/InterAgent/Handlers/AskHandler.swift](../../Sources/InterAgent/Handlers/AskHandler.swift):

- the current terminal must be connected to the target terminal
- the target is matched by agent name, display name, command, or UUID prefix
- the prompt is injected into the target terminal as text, followed by carriage return
- for `generic_shell` terminals the handler waits for a prompt-like return state or a timeout of 30 seconds
- for non-shell agent types the handler waits for a `terminalBecameIdle` notification or a timeout of 30 seconds
- the returned text is a snapshot of the terminal buffer

### 8.3 `check`

Usage: `omaestri check "TargetAgent" [lines]`

Behavior derived from [Sources/InterAgent/Handlers/CheckHandler.swift](../../Sources/InterAgent/Handlers/CheckHandler.swift):

- requires a connected target terminal
- returns recent output from the target session
- strips ANSI escape sequences from the output
- defaults to 20 lines when no count is supplied

### 8.4 `note`

Usage: `omaestri note <read|write|edit|create> ...`

Behavior derived from [Sources/InterAgent/Handlers/NoteHandler.swift](../../Sources/InterAgent/Handlers/NoteHandler.swift):

- `read`: reads the content of a note by name
- `write`: replaces the content of a note by name
- `edit`: replaces one matching text fragment with another in a note
- `create`: creates a new markdown file under the workspace notes directory and returns its generated name

The current implementation resolves note paths through a runtime registry first and then scans the workspace note directories.

### 8.5 `portal`

Usage: `omaestri portal <subcommand> ...`

Behavior derived from [Sources/InterAgent/Handlers/PortalHandler.swift](../../Sources/InterAgent/Handlers/PortalHandler.swift):

The source confirms these implemented top-level subcommands:

- `create`
- `edit`
- `navigate`
- `back`
- `forward`
- `reload` / `refresh`
- `screenshot`
- `snapshot`
- `html`
- `text`
- `info`
- `click`
- `fill`
- `type`
- `key`
- `hover`
- `scroll`
- `drag`
- `wait`
- `evaluate`
- `select`
- `check`
- `uncheck`
- `focus`
- `scrollintoview`
- `selectall`
- `clear`
- `logs-start`
- `logs`

The Linux port should preserve the same command surface and argument patterns. The exact browser-side semantics are implemented by the portal runtime store and are not fully expressible in this document without inspecting that runtime in detail.

### 8.6 `recruit`

Usage: `omaestri recruit "Name" [--preset <agentType>] [--role <roleName>] [--command <cmd>]`

Behavior derived from [Sources/InterAgent/Handlers/MaestroHandlers.swift](../../Sources/InterAgent/Handlers/MaestroHandlers.swift):

- only works when invoked from a terminal that is in Maestro mode
- creates a new terminal session attached to the current workspace
- writes `export OMAESTRI_AGENT_NAME="..."` into the new terminal
- establishes a new connection between the Maestro terminal and the recruited terminal
- uses the requested preset or fallback values when no preset is given

### 8.7 `dismiss`

Usage: `omaestri dismiss "Name"`

Behavior derived from [Sources/InterAgent/Handlers/MaestroHandlers.swift](../../Sources/InterAgent/Handlers/MaestroHandlers.swift):

- resolves a connected agent by matching the requested name against connected agent names, commands, roles, or UUID prefix
- disconnects and removes the terminal from the runtime

### 8.8 `connect`

Usage: `omaestri connect "From" "To"`

Behavior derived from [Sources/InterAgent/Handlers/MaestroHandlers.swift](../../Sources/InterAgent/Handlers/MaestroHandlers.swift):

- resolves two terminals by matching names or commands
- creates a new terminal-to-terminal connection between them

### 8.9 `role`

Usage: `omaestri role <list|create|show|edit|write|assign> ...`

Behavior derived from [Sources/InterAgent/Handlers/MaestroHandlers.swift](../../Sources/InterAgent/Handlers/MaestroHandlers.swift):

- `list`: lists role presets
- `create`: creates a role preset with a name and prompt
- `show`: prints a role preset
- `edit` / `write`: updates a role prompt
- `assign`: assigns a role to an agent and prepares the role directory for the next restart

### 8.10 `preset`

Usage: `omaestri preset <list>`

Behavior derived from [Sources/InterAgent/Handlers/MaestroHandlers.swift](../../Sources/InterAgent/Handlers/MaestroHandlers.swift):

- `list`: prints the currently active agent presets

### 8.11 `debug`

Usage: `omaestri debug`

Behavior derived from [Sources/InterAgent/CLIRouter.swift](../../Sources/InterAgent/CLIRouter.swift):

- prints the current server port, the current terminal identifier if present, and the command list

## 9. Connection lifecycle and concurrency

The current implementation uses a runtime connection registry in [Sources/Connection/ConnectionManager.swift](../../Sources/Connection/ConnectionManager.swift). The Linux backend should preserve the same semantics:

- connections are stored by UUID and restored from the workspace document on reload
- `ask` and other commands operate on the currently connected terminal graph
- the implementation is not a stateless command queue; it depends on the in-memory connection graph

## 10. Uncertainties to keep explicit

The source makes the following behaviors clear, but the protocol spec should remain conservative about them:

- the portal automation features are implemented through the portal runtime store and not through a generic CLI contract documented in the router itself
- the current transport does not define a structured machine-readable success envelope
- the app uses various environment variables and notifications that are not part of a separate public protocol contract

The Linux port should preserve the observed behavior rather than inventing a new protocol surface.

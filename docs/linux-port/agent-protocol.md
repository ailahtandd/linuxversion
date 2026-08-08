# Agent protocol contract for the Linux port

The existing app uses a local, terminal-driven agent protocol that is semantically richer than a simple process launch. The Linux port must preserve the same behavior even if the transport implementation is rewritten.

## 1. Protocol overview

The inter-agent protocol is conceptually:

- a terminal node launches a shell or agent process
- the shell process inherits environment variables describing the terminal identity and the local IPC endpoint
- the `omaestri` CLI sends requests to the app through a local socket or HTTP endpoint
- the app routes the request to the correct workspace/terminal context and returns plain-text output

This is implemented in [Sources/CLI/main.swift](../../Sources/CLI/main.swift), [Sources/CLI/Transport.swift](../../Sources/CLI/Transport.swift), and [Sources/InterAgent/InterAgentServer.swift](../../Sources/InterAgent/InterAgentServer.swift).

## 2. Identity semantics

The current protocol uses two identity dimensions:

- terminal ID: a stable UUID identifying a specific terminal session
- workspace / app context: the application instance that owns the socket and workspace state

The protocol also uses the `X-Terminal-ID` header as a routing key. Linux must preserve the idea that a terminal has a stable identity across workspace reloads and that commands are scoped to that identity.

## 3. Request format

Requests are encoded as JSON bodies with an `args` array and are sent over a local transport. The current transport emits an HTTP-like request with:

- `POST /cli`
- `Content-Type: application/json`
- `X-Terminal-ID: <uuid>`
- a body shaped like `{ "args": [ ... ] }`

The Linux port should preserve the same command interface and header semantics while potentially replacing the TCP/Unix socket implementation with a Rust backend and Unix domain sockets.

## 4. Routing model

The router in [Sources/InterAgent/CLIRouter.swift](../../Sources/InterAgent/CLIRouter.swift) dispatches commands to handlers for:

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

The Linux port should preserve the same command surface and behavior. The app should not rely on the terminal process being directly aware of the workspace state; the backend should perform the routing.

## 5. Environment injection

The terminal process receives environment variables such as:

- `MAESTRI_SOCKET`
- `MAESTRI_TERMINAL_ID`
- `OMAESTRI_TERMINAL_ID`
- `TERM=xterm-256color`

This is the mechanism that allows the CLI to reach the runtime without a direct process API. The Linux port should preserve this contract or an equivalent aliasing mechanism.

## 6. Response semantics

The server returns plain text as the response body. The CLI prints it directly, which makes the protocol simple and composable for shells and agents.

The Linux port should continue to return UTF-8 text and should preserve the expectation that CLI commands are pipeline-friendly.

## 7. Compatibility requirements

The Linux implementation must preserve the following protocol properties:

- local-only communication
- stable terminal identity
- ability for shell processes to call into the app without a human UI action
- the same command names and argument shapes
- support for long-running interactive requests such as `ask`
- clear failure semantics when the backend is unavailable

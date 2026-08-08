# Proposed Linux architecture

This document describes the Linux architecture that should be implemented without touching the existing macOS Swift implementation. The design is intentionally Linux-native and preserves the current app’s observable behavior as the compatibility contract.

## 1. Architecture overview

The Linux port should be organized as a layered system:

- Tauri 2 provides the desktop shell, windowing surface, and native integration hooks.
- React + TypeScript powers the UI layer and the interactive workspace view.
- Rust provides the core domain services: workspace persistence, PTY lifecycle, IPC, orchestration, and agent routing.
- The frontend uses xterm.js for terminal rendering and PixiJS for the infinite canvas where it is beneficial.
- Unix domain sockets are used for Linux local IPC between the terminal processes and the Rust backend.

## 2. Layer responsibilities

### Shell layer (Tauri 2)

Responsibilities:

- host the React UI
- provide window lifecycle, menus, file dialogs, and OS integration hooks
- expose a thin command/event bridge to the Rust backend for workspaces, terminals, and agent commands

### UI layer (React + TypeScript)

Responsibilities:

- render the workspace canvas and nodes
- handle pan/zoom, selection, node creation, connection editing, and note editing
- render terminal panels with xterm.js
- render notes, file tree, and portal/browser surfaces using web-native primitives

### Backend layer (Rust)

Responsibilities:

- manage workspace models and persistence
- own PTY sessions for terminal nodes
- route `omaestri`/agent commands and preserve terminal identities
- maintain the connection graph and workspace reconstruction logic
- expose a service API for the UI over a local bridge

## 3. Runtime components

### Workspace service

A Rust workspace service should own the serialization and deserialization of the workspace model. It should preserve the current JSON contract and be validated by fixtures derived from the Swift implementation.

### PTY service

The PTY service should own shell lifecycle, process spawn, resize, input/output, and scrollback. It should be the canonical source of terminal state for the Linux port.

### Agent protocol service

The Linux port should expose a local agent-protocol service over Unix domain sockets. The Rust backend should accept the CLI payloads and route them to the correct workspace/terminal context while preserving the current `omaestri` behavior.

### Renderers

The frontend should have separate renderers for:

- canvas nodes and connections
- terminal sessions
- notes and file tree views
- portal/browser views

## 4. PTY architecture and process ownership

The terminal architecture must not accidentally use ordinary subprocess pipes. The Linux backend must own the PTY lifecycle.

The intended flow is:

1. xterm.js in the frontend captures keyboard input and resize events.
2. The frontend sends those events to the Tauri command/event boundary.
3. The Rust `TerminalManager` service receives the events and writes to a real PTY.
4. The child shell or coding agent process runs inside the PTY.
5. Output from the PTY is forwarded back to xterm.js.

This is the key architectural boundary:

- the frontend is responsible for presentation and user input
- the Rust backend is responsible for process creation, process lifecycle, resize handling, and data transfer

### Required PTY behavior

The Rust PTY backend should support:

- stdin: user keystrokes and pasted text should be written into the PTY
- stdout/stderr: child output should be streamed back to xterm.js without loss of UTF-8 text
- terminal resize: rows/columns events must be delivered to the child process
- Ctrl+C: should signal the foreground process group
- Ctrl+D: should close stdin and/or cause the shell to exit when appropriate
- Ctrl+Z: should suspend the foreground process group
- process exit: the backend should detect child termination and update the terminal state
- cwd: the initial working directory should be preserved from the current node/session metadata
- environment variables: the backend must inject the same terminal identity and routing env vars as the Swift implementation
- UTF-8: the PTY must preserve Unicode text and not treat it as a raw byte stream without decoding
- ANSI: the backend must pass through ANSI sequences so xterm.js can render them correctly
- large or rapid output: the backend should handle bursts of output without dropping data or causing UI stalls

## 5. Data serialization

The Linux implementation should use Serde for JSON serialization and model conversion. The Rust models should preserve the existing workspace schema and be validated against the same compatibility requirements.

## 6. Key design principles

- preserve the existing macOS behavior as the compatibility contract
- keep the Linux implementation isolated from the Swift codebase
- do not attempt to compile AppKit, SwiftUI, or WebKit on Linux
- use Unix domain sockets for Linux-only local IPC
- use Tauri 2 as the application shell rather than building a custom renderer from scratch
- keep the PTY backend authoritative for process state and lifecycle

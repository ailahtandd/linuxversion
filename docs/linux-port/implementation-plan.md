# Implementation plan for the Linux port

This plan is intentionally scoped to architecture and compatibility work. It does not begin application implementation yet.

## 1. MVP boundary

The first Linux MVP should prioritize:

- a Tauri 2 desktop application
- the infinite canvas
- terminal nodes
- multiple simultaneous PTYs
- Codex
- Claude Code
- OpenCode
- Gemini CLI
- a generic CLI shell
- workspace load/save
- notes
- inter-agent IPC
- `omaestri` CLI compatibility
- AppImage and/or `.deb` packaging

The following should be treated as post-MVP unless the existing Swift implementation proves that they are required by a core dependency:

- portal/browser automation parity
- advanced Git UI
- SSH
- updater
- routines
- floors
- exact visual parity
- advanced animation/polish

## 2. Milestone 0: lock the workspace compatibility contract

Goal: freeze the contract that the Linux port must preserve before any implementation begins.

Tasks:

- lock the schema version, field names, and field types for `WorkspaceDocument`, `WorkspacePayload`, `CanvasNode`, and `NodeContent`
- define the compatibility fixtures that represent the current Swift-generated workspace JSON
- document the required XDG-based filesystem layout for Linux while preserving the existing workspace serialization contract
- confirm the required terminal identity and agent protocol behavior

Dependencies:

- none; this milestone is the prerequisite for all following work

## 3. Milestone 1: scaffold the minimal Tauri 2 + React + Rust shell

Goal: produce a runnable desktop shell that can host the Linux UI without yet implementing the full feature set.

Tasks:

- create the Tauri 2 shell and React/TypeScript frontend skeleton
- wire a Rust backend command/event bridge
- provide a minimal window, workspace loading entry point, and basic app state

Dependencies:

- depends on Milestone 0

## 4. Milestone 2: implement the real Linux PTY backend

Goal: replace the macOS-oriented terminal runtime with a true Rust PTY implementation.

Tasks:

- create a `TerminalManager`-style Rust service that owns PTY processes
- implement stdin/stdout/stderr streaming, resize handling, process exit, cwd, and environment injection
- preserve the current terminal identity and CLI environment variables

Dependencies:

- depends on Milestone 1

## 5. Milestone 3: integrate xterm.js

Goal: connect the PTY backend to the frontend rendering surface.

Tasks:

- wire xterm.js into the React frontend
- stream PTY output into the terminal view
- send keyboard and resize events back to the Rust backend

Dependencies:

- depends on Milestone 2

## 6. Milestone 4: verify multiple concurrent terminals

Goal: prove that the Linux terminal backend can host more than one interactive PTY at the same time.

Tasks:

- create multiple terminal sessions in parallel
- validate independent input/output, resize behavior, and process lifecycle
- ensure the runtime can manage many terminals without cross-talk

Dependencies:

- depends on Milestone 3

## 7. Milestone 5: implement the infinite canvas core

Goal: produce the workspace canvas surface and the core node model without yet attaching every product feature.

Tasks:

- implement pan/zoom, selection, node placement, and basic node rendering
- use PixiJS where it improves performance and visual fidelity
- preserve the canvas coordinate semantics used by the current app

Dependencies:

- depends on Milestone 0 and Milestone 1

## 8. Milestone 6: integrate terminal nodes into the canvas

Goal: connect the terminal runtime to canvas nodes so terminals can be created and manipulated like the current app.

Tasks:

- map PTY sessions to canvas terminal nodes
- support create/show/hide/resize semantics for terminal nodes
- retain the current node identity and link behavior

Dependencies:

- depends on Milestone 4 and Milestone 5

## 9. Milestone 7: implement workspace persistence and reconstruction

Goal: make the Linux app load and save workspaces with the same schema semantics as the Swift implementation.

Tasks:

- implement workspace JSON load/save with Serde and the compatibility fixtures
- preserve note paths and scrollback behavior under the Linux storage layout
- implement atomic write semantics and recovery-friendly persistence

Dependencies:

- depends on Milestone 0 and Milestone 6

## 10. Milestone 8: implement agent adapters

Goal: support the agent command surface that the current app exposes to coding agents.

Tasks:

- support the current CLI command surface for `list`, `ask`, `check`, `note`, `portal`, `recruit`, `dismiss`, `connect`, `role`, `preset`, and `debug`
- preserve the terminal identity and environment injection behavior

Dependencies:

- depends on Milestone 2 and Milestone 7

## 11. Milestone 9: implement `omaestri` IPC/orchestration

Goal: make the Linux backend accept the same local IPC requests as the current app.

Tasks:

- expose a Unix-socket-based server that accepts the CLI request format
- route the requests to the correct workspace and terminal context
- preserve the plain-text response contract used by the current app

Dependencies:

- depends on Milestone 8

## 12. Milestone 10: implement notes and file tree

Goal: cover the core workspace productivity features beyond the terminal and canvas.

Tasks:

- implement note create/read/write/edit semantics
- implement file tree browsing and path mapping
- preserve note and workspace association semantics

Dependencies:

- depends on Milestone 7

## 13. Milestone 11: hardening and Linux packaging

Goal: turn the prototype into a usable Linux application.

Tasks:

- add end-to-end compatibility tests for workspace round-trip and agent protocol behavior
- package the app as AppImage and/or `.deb`
- verify the main workflows on Linux and fix regressions

Dependencies:

- depends on Milestones 9 and 10

## 14. Recommended milestone order

The recommended implementation order is:

1. Milestone 0: lock the workspace compatibility contract
2. Milestone 1: scaffold the minimal Tauri 2 + React + Rust shell
3. Milestone 2: implement the real Linux PTY backend
4. Milestone 3: integrate xterm.js
5. Milestone 4: verify multiple concurrent terminals
6. Milestone 5: implement the infinite canvas core
7. Milestone 6: integrate terminal nodes into the canvas
8. Milestone 7: implement workspace persistence and reconstruction
9. Milestone 8: implement agent adapters
10. Milestone 9: implement `omaestri` IPC/orchestration
11. Milestone 10: implement notes and file tree
12. Milestone 11: hardening and Linux packaging

This order separates the immutable compatibility contract from the broader persistence and product implementation work. It also keeps the PTY and agent protocol work ahead of the UI polish work.

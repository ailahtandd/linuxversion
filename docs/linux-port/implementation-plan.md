# Implementation plan for the Linux port

This plan is intentionally scoped to architecture and compatibility work. It does not start application implementation yet.

## 1. Phase 0: Freeze compatibility contracts

Goal: lock the portable behavior before implementation begins.

Tasks:

- confirm which parts of the current workspace format must remain unchanged
- finalize the required terminal and agent protocol semantics
- document the minimum canvas, note, portal, and Git behaviors that must be preserved

Output:

- a compatibility checklist that every Linux implementation step must satisfy

## 2. Phase 1: Workspace and persistence foundation

Goal: make the Linux app able to read and write the existing workspace structure.

Tasks:

- define the Rust models that correspond to the current workspace payload
- implement workspace file readers and writers with Serde
- implement atomic write behavior and workspace directory handling
- add compatibility tests using fixtures from the current repository

## 3. Phase 2: Agent protocol and terminal identity

Goal: make the Linux app understand the same local agent protocol as the macOS app.

Tasks:

- implement a Rust-side server listening on a Unix domain socket for Linux
- preserve the request/response contract for `omaestri` CLI commands
- preserve terminal identity and environment injection behavior
- confirm that existing CLI commands continue to work with the same semantics

## 4. Phase 3: Terminal runtime

Goal: provide interactive terminal sessions that match the current behavior.

Tasks:

- integrate a real Rust PTY implementation
- connect the PTY to the frontend using xterm.js
- preserve shell startup, resize handling, and scrollback expectations
- ensure the terminal node can be created, shown, and persisted like the current app

## 5. Phase 4: Canvas and node interaction

Goal: provide the core visual workspace experience.

Tasks:

- implement the workspace canvas and node placement in the Linux UI
- support pan/zoom, selection, node creation, and connection geometry
- use PixiJS for a renderer where it improves performance and visual fidelity
- keep the interaction semantics aligned with the existing canvas model

## 6. Phase 5: Notes, file tree, portal, and Git

Goal: bring the rest of the product surface into parity.

Tasks:

- implement note editing/storage
- implement file tree browsing and search
- implement portal navigation and automation with an equivalent browser runtime abstraction
- preserve basic Git integration semantics

## 7. Phase 6: Hardening and packaging

Goal: make the Linux version usable as a product rather than just a technical prototype.

Tasks:

- add end-to-end tests around workspace compatibility and protocol behavior
- package the app with Tauri 2 and the Rust backend
- verify the main workflows on Linux

## Recommended order

The recommended priority order is:

1. workspace compatibility
2. PTY/terminal correctness
3. agent orchestration
4. reliability
5. maintainability
6. Linux usability
7. visual parity

This order matches the repository’s stated Linux port priorities and avoids over-scaffolding the UI before the core contracts are stable.

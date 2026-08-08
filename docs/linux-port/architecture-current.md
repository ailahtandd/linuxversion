# Current macOS architecture

This document records the current implementation as it exists in the repository, before any Linux-specific implementation work begins.

## 1. High-level shape

The current product is a macOS-native Swift application with three package targets:

- [Package.swift](../../Package.swift): defines the `open-maestri` app target, the `omaestri` CLI target, and the test target.
- [Sources](../../Sources): contains the app runtime, canvas engine, terminal subsystem, persistence, inter-agent server, portal logic, notes, and file tree integration.
- [Tests/OpenMaestriTests](../../Tests/OpenMaestriTests): covers workspace persistence, terminal behavior, canvas rendering, connection logic, and inter-agent routing.

The product is organized around a workspace-centric runtime model rather than a single monolithic screen. The visible app UI is composed from SwiftUI, but the canvas layer is implemented as an AppKit-based `NSView` viewport.

## 2. Startup and runtime flow

The startup sequence is intentionally ordered by [CLAUDE.md](../../CLAUDE.md):

1. Start the local inter-agent server before the UI is fully loaded.
2. Inject skill assets into the local Claude skill directory.
3. Initialize app state and load the workspace manifest / workspace state.
4. Render UI and canvas views.

The runtime data model is centered on `WorkspaceManager` and a serialized `WorkspaceDocument` payload. The app is designed so that workspace changes are captured as a payload, then persisted through the shared persistence layer.

## 3. Core runtime layers

### App state

The global application state lives in the app layer and is responsible for workspace discovery, autosave scheduling, crash recovery markers, and top-level app lifecycle concerns.

### Workspace model

The workspace is backed by a set of runtime arrays stored in `WorkspaceManager`:

- nodes
- terminal connections
- note connections
- portal connections
- portal-to-portal connections
- note-to-note connections
- cross-floor connections
- floors
- drawings
- canvas origin / zoom

These values are serialized into a `WorkspacePayload` that is persisted to `workspace.json`.

### Canvas engine

The infinite canvas is implemented in [Sources/Canvas/Core/CanvasViewportView.swift](../../Sources/Canvas/Core/CanvasViewportView.swift). It uses a custom AppKit `NSView` pipeline with multiple layers for nodes, rope connections, selection, and interaction state. Important implementation characteristics:

- the coordinate origin is centered around `(9800, 8500)`
- viewport culling is used to limit rendered nodes
- sorted z-index caches are used to avoid repeated per-frame sorting
- hit testing is optimized for interactive performance

### Node types

The current node system is driven by `NodeContent` values and `CanvasNode` wrappers. The runtime currently supports terminal, note, portal, file-tree, and text-style content. The important compatibility point is that the serialized node frame uses the Maestri-compatible `[[x, y], [w, h]]` encoding and that node content is encoded in the existing wrapped object format.

## 4. Terminal subsystem

Terminal nodes are implemented through a custom layer over SwiftTerm and an AppKit terminal view. The main responsibilities are:

- start a PTY-backed shell session
- inject the `omaestri` or `maestri` CLI environment into the shell
- capture output for scrollback and status tracking
- mount the terminal into the canvas as a node view

The key implementation is in [Sources/Terminal/SwiftTermProvider.swift](../../Sources/Terminal/SwiftTermProvider.swift), which creates the PTY, sets environment variables, and coordinates shell-ready detection. It persists scrollback in workspace-local files under the app data directory.

## 5. Notes, file tree, Git

The note system stores markdown content through a note manager and writes it to workspace-local note files. The file tree is implemented as an AppKit `NSOutlineView` wrapper and exposes browsing, navigation, context-menu interactions, and search. The Git integration is currently lightweight and uses system `git` CLI calls rather than a native library.

Key files:

- [Sources/Note/NoteFileManager.swift](../../Sources/Note/NoteFileManager.swift)
- [Sources/FileTree/FileTreeOutlineAdapter.swift](../../Sources/FileTree/FileTreeOutlineAdapter.swift)
- [Sources/Git/GitStatusProvider.swift](../../Sources/Git/GitStatusProvider.swift)

## 6. Portal and browser functionality

Portal nodes use `WKWebView` inside the canvas node view. The portal subsystem manages multiple web views, navigation, shared sessions between portals, and automation commands such as navigation, click, fill, screenshot, and JavaScript evaluation. The portal automation is designed to be accessible by the `omaestri portal` CLI commands.

The critical implementation is in [Sources/Portal/PortalWebViewStore.swift](../../Sources/Portal/PortalWebViewStore.swift).

## 7. Inter-agent communication and CLI

The application exposes a local inter-agent server that accepts HTTP requests on localhost and a Unix socket path. The server routes commands such as `ask`, `check`, `note`, `portal`, `recruit`, `dismiss`, `connect`, `role`, and `preset` through the router.

Relevant files:

- [Sources/InterAgent/InterAgentServer.swift](../../Sources/InterAgent/InterAgentServer.swift)
- [Sources/InterAgent/CLIRouter.swift](../../Sources/InterAgent/CLIRouter.swift)
- [Sources/CLI/Transport.swift](../../Sources/CLI/Transport.swift)
- [Sources/CLI/main.swift](../../Sources/CLI/main.swift)

The CLI uses environment variables such as `MAESTRI_SOCKET` and `MAESTRI_TERMINAL_ID` to identify the terminal context and authenticate the request.

## 8. Persistence and storage layout

Workspace persistence is centralized through [Sources/Workspace/PersistenceManager.swift](../../Sources/Workspace/PersistenceManager.swift). The app writes to a per-workspace directory under `~/.open-maestri/` and uses atomic replacement when updating files. The storage layout is compatible with the existing Maestri workspace conventions and is explicitly treated as a compatibility contract.

## 9. Design implications for Linux

The current architecture is not a generic UI toolkit structure. It is a macOS-specific composition of SwiftUI, AppKit, SwiftTerm, WebKit, and Network.framework primitives. The Linux port must preserve behavior and data contracts while replacing the UI/runtime stack with a new platform-native implementation.

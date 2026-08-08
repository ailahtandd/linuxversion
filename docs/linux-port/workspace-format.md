# Workspace format compatibility contract

The Linux port must preserve the existing workspace serialization contract as much as possible. This document records the current on-disk format and the constraints that the Linux implementation must respect.

## 1. Top-level storage layout

The current app writes state under `~/.open-maestri/`:

- `manifest.json`: workspace index / manifest data
- `preferences.json`: user preferences
- `app-state.json`: crash-recovery and lifecycle state
- `run/agent.sock`: Unix socket path used by the inter-agent server
- `workspaces/{UUID}/workspace.json`: the main workspace document
- `workspaces/{UUID}/notes/`: note markdown files
- `workspaces/{UUID}/terminals/`: scrollback files and terminal-related state

The Linux implementation should preserve this layout or a deliberately versioned equivalent that remains compatible with the existing model.

## 2. Workspace document shape

The main workspace document is a payload-like object whose fields mirror the Maestri-compatible shape. The current implementation uses `WorkspacePayload` in [Sources/Workspace/Models/WorkspacePayload.swift](../../Sources/Workspace/Models/WorkspacePayload.swift).

Key persisted fields include:

- `id`, `name`, `icon`, `isPinned`
- `locationType`, `workingDirectory`, `preferredIDE`, `syncConfigFiles`
- `canvasOrigin`, `canvasZoom`
- `nodes`, `connections`, `noteConnections`, `portalConnections`, `portalToPortalConnections`, `noteToNoteConnections`, `crossFloorConnections`, `floors`, `drawings`
- timestamps such as `createdAt`, `lastOpenedAt`, `lastModifiedAt`

## 3. Node frame encoding

The current workspace format uses a frame array in the form:

```json
"frame": [[x, y], [w, h]]
```

This is not a standard `CGRect` JSON shape. It is encoded through the compatibility helpers in [Sources/Shared/Extensions/CGRect+Frame.swift](../../Sources/Shared/Extensions/CGRect+Frame.swift).

The Linux implementation must preserve this exact semantics, even if it uses a different in-memory geometry type internally.

## 4. Node content serialization

Node content is encoded in a wrapped object form that preserves Maestri compatibility. The current implementation uses a `NodeContent` enum that serializes as a wrapped object such as:

```json
{ "terminal": { "_0": { ... } } }
```

The Linux port should preserve the same overall object structure and the same semantic node categories. It should not introduce a new JSON shape that breaks compatibility with existing workspace files.

## 5. Connection geometry

Connection objects contain rope control points represented as arrays of coordinates. These are used by the rendering layer to draw connections and should be preserved as part of the workspace document.

## 6. Notes and scrollback

Notes are stored as markdown files under the workspace notes directory. Terminal scrollback is stored in workspace-local terminal files. These files should continue to exist in a predictable location or be recreated through equivalent migration logic.

## 7. Atomic writes and crash safety

The current implementation uses atomic replacement for workspace files and note files. The Linux port should preserve crash-safe write semantics even if it uses a different filesystem wrapper. The app’s persistence strategy is a compatibility contract of its own.

## 8. Compatibility constraints

The Linux implementation must preserve the following without breaking existing files:

- `schemaVersion: 2` semantics
- `CanvasNode.frame` encoding
- `NodeContent` serialization envelope
- workspace directory layout under `~/.open-maestri/`
- note and scrollback storage conventions
- stable node and terminal UUID semantics

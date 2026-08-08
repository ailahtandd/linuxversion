# Workspace format compatibility contract

The Linux port must preserve the existing workspace serialization contract even though the Linux filesystem layout can be modernized. This document separates schema compatibility from host-path conventions.

## 1. Compatibility vs filesystem layout

The workspace file schema is a hard compatibility contract. The host filesystem layout is not.

The current macOS implementation writes state under `~/.open-maestri/`, but the Linux port should prefer Linux conventions:

- `$XDG_CONFIG_HOME/open-maestri` for config-like state such as preferences and manifests
- `$XDG_DATA_HOME/open-maestri` for workspace data and note/scrollback files
- `$XDG_RUNTIME_DIR/open-maestri` for runtime state such as the Unix socket

Sensible fallbacks should be used when the XDG variables are unset:

- config: `$HOME/.config/open-maestri`
- data: `$HOME/.local/share/open-maestri`
- runtime: `$XDG_RUNTIME_DIR/open-maestri` when present, otherwise `$HOME/.local/run/open-maestri`

The preferred runtime socket path for Linux should be:

- `$XDG_RUNTIME_DIR/open-maestri/agent.sock`

Existing `~/.open-maestri` data should be treated as a compatibility source for import/migration, not as an immutable Linux requirement.

## 2. Exact workspace document schema

The current app writes a top-level workspace document shaped like this:

```json
{
  "payload": { "...": "..." },
  "schemaVersion": 2,
  "type": "workspace"
}
```

The schema is defined in [Sources/Workspace/Models/WorkspaceDocument.swift](../../Sources/Workspace/Models/WorkspaceDocument.swift), [Sources/Workspace/Models/WorkspacePayload.swift](../../Sources/Workspace/Models/WorkspacePayload.swift), and [Sources/Workspace/Models/CanvasNode.swift](../../Sources/Workspace/Models/CanvasNode.swift).

### 2.1 Top-level fields

- `payload`: object, required
- `schemaVersion`: integer, required, currently `2`
- `type`: string, required, currently `"workspace"`

### 2.2 Workspace payload fields

The persisted `payload` object contains:

- `id`: UUID string
- `name`: string
- `icon`: string
- `isPinned`: boolean
- `locationType`: string, currently `"local"` or `"ssh"`
- `workingDirectory`: string
- `preferredIDE`: string, currently `"cursor"`, `"vscode"`, or `"xcode"`
- `syncConfigFiles`: boolean
- `canvasOrigin`: object with `x` and `y` number fields
- `canvasZoom`: number
- `nodes`: array of `CanvasNode`
- `connections`: array of `TerminalConnection`
- `noteConnections`: array of `NoteConnection`
- `portalConnections`: array of `PortalConnection`
- `portalToPortalConnections`: array of `PortalToPortalConnection`
- `noteToNoteConnections`: array of `NoteToNoteConnection`
- `crossFloorConnections`: array of `CrossFloorConnection`
- `floors`: array of `FloorEntry`
- `drawings`: array of `Drawing`
- `createdAt`: ISO-8601 date string
- `lastOpenedAt`: ISO-8601 date string or `null`
- `lastModifiedAt`: ISO-8601 date string

### 2.3 Defaults and optional behavior

The current Swift initializer sets these defaults:

- `icon`: `"folder"`
- `isPinned`: `false`
- `locationType`: `"local"`
- `preferredIDE`: `"cursor"`
- `syncConfigFiles`: `false`
- `canvasOrigin`: `CGPoint(x: 9800, y: 8500)`
- `canvasZoom`: `1.0`
- `lastOpenedAt`: `nil`

The current decoder is not fully tolerant of missing optional fields for every type. In particular, most non-optional fields are required on decode. The Linux Rust implementation should treat missing required fields as invalid input unless a fixture proves otherwise.

## 3. Canvas node schema

Each node is encoded as a `CanvasNode` object with:

- `id`: UUID string
- `frame`: array of two arrays, `[[x, y], [w, h]]`
- `content`: one of the supported `NodeContent` envelopes
- `zIndex`: integer, default `0`
- `isLocked`: boolean, default `false`
- `createdAt`: ISO-8601 date string
- `lastModifiedAt`: ISO-8601 date string

### 3.1 Frame encoding

The current implementation uses a Maestri-compatible frame array:

```json
"frame": [[x, y], [w, h]]
```

The Linux port must preserve this exact shape. The Rust implementation should not switch to a standard object-based rectangle representation unless it can round-trip the existing workspace JSON without drift.

## 4. Node content envelope and payloads

Node content is encoded with the Maestri-compatible envelope used by [Sources/Workspace/Models/NodeContent.swift](../../Sources/Workspace/Models/NodeContent.swift).

The envelope is one of:

```json
{ "terminal": { "_0": { ... } } }
```

or the equivalent for `stickyNote`, `portal`, `fileTree`, `text`, `shape`, `stroke`, or `freehand`.

### 4.1 Terminal content

Persisted terminal metadata uses:

- `agentType`: string
- `command`: string
- `name`: string
- `icon`: string
- `color`: string
- `id`: UUID string
- `shellPath`: string
- `workingDirectory`: string
- `status`: string
- `isManager`: boolean
- `monitorWithOmbro`: boolean
- `autoScrollLocked`: boolean
- `shortcutMode`: object with `kind` set to one of `automatic`, `none`, `cmd1` … `cmd9`
- `assignedRoleId`: UUID string or `null`
- `scrollbackFile`: string or `null`
- `scrollbackLineCount`: integer
- `lastActiveAt`: ISO date or `null`
- `themeId`: string or `null`
- `fontFamily`: string or `null`
- `fontSize`: number or `null`

### 4.2 Sticky note content

- `color`: string
- `fileName`: string or `null`
- `fontSize`: integer
- `hasCustomName`: boolean
- `isPreviewing`: boolean
- `storageMode`: either `{ "managed": {} }` or `{ "custom": { "_0": "/path" } }`

### 4.3 Portal content

- `id`: UUID string
- `name`: string
- `currentURL`: string
- `source`: either `"none"` or `{ "url": { "_0": "https://..." } }`
- `status`: string
- `chromeHidden`: boolean
- `storageScope`: string

### 4.4 File tree, text, shape, stroke, and freehand

These use the same payload conventions as the Swift model:

- `FileTreeContent`: `name`, `rootPath`, `viewMode`
- `TextContent`: `text`, `fontSize`, `fontWeight`, `color`, `alignment`, `fontFamily`
- `ShapeContent`: `shapeType`, `fillColor`, `strokeColor`, `strokeWidth`, `strokeStyle`, `fillStyle`, `text`, `fontSize`, `rotation`
- `StrokeContent`: `strokeType`, `startPoint`, `endPoint`, `controlPoint`, `strokeColor`, `strokeWidth`, `strokeStyle`
- `FreehandContent`: `freehandType`, `points`, `strokeColor`, `strokeWidth`, `opacity`, `rotation`

## 5. UUID and identity behavior

UUIDs are persisted as canonical UUID strings. The Linux port must preserve stable UUIDs across save/load cycles.

Important semantic rules derived from the Swift implementation:

- `CanvasNode.id` is the stable node identity.
- For terminal nodes, `TerminalContent.id` is also the terminal identity used by the runtime.
- Connection IDs are also persisted as UUIDs and should be restored without re-generation.
- The terminal identity used by the inter-agent protocol must remain stable even if the workspace document is reloaded.

## 6. Connection representation

Connection records are persisted separately from nodes and are not inferred from the canvas alone.

Current connection types and fields:

- `TerminalConnection`: `id`, `createdAt`, `terminalIdA`, `terminalIdB`, `ropePoints`
- `NoteConnection`: `id`, `createdAt`, `terminalId`, `noteNodeId`, `ropePoints`
- `PortalConnection`: `id`, `createdAt`, `terminalId`, `portalNodeId`, `ropePoints`
- `PortalToPortalConnection`: `id`, `createdAt`, `portalIdA`, `portalIdB`, `ropePoints`
- `NoteToNoteConnection`: `id`, `createdAt`, `noteNodeIdA`, `noteNodeIdB`, `ropePoints`
- `CrossFloorConnection`: `id`, `createdAt`, `nodeIdA`, `floorIdA`, `nodeIdB`, `floorIdB`, `ropePoints`
- `FloorEntry`: `id`, `name`, `branchName`, `worktreePath`, `hooks`, `createdAt`
- `Drawing`: `id`, `points`, `color`, `lineWidth`, `createdAt`

`ropePoints` and `points` are arrays of `[x, y]` pairs encoded as JSON arrays of numbers.

## 7. Notes and terminal metadata

Notes are persisted as markdown files, not only as in-memory node data.

The current behavior uses:

- workspace-local notes directories under `workspaces/{workspaceId}/notes/`
- workspace-local scrollback files under `workspaces/{workspaceId}/terminals/{terminalId}.scrollback`

The Linux port should preserve these semantics through a compatible data layout, even if the base directory is moved under XDG data paths.

## 8. Unknown-field and schema evolution behavior

The existing decoder uses standard `Codable` behavior. Unknown fields are ignored on decode unless the app explicitly throws due to a missing required field or an invalid envelope.

Practical implications for the Linux port:

- unknown fields should be tolerated during load, but not assumed to be preserved on save
- the Rust implementation should not silently invent a new workspace shape without a migration plan
- schema evolution should be tested with fixtures produced by the current Swift implementation

## 9. Compatibility fixtures and tests

The Rust implementation should eventually have golden fixtures and round-trip tests for:

1. a minimal workspace with one terminal node and no connections
2. a workspace with multiple terminal nodes, note connections, and portal connections
3. a workspace containing text, shape, stroke, and freehand nodes
4. a workspace with a non-empty `frame`, `ropePoints`, and `createdAt`/`lastModifiedAt` values
5. a workspace that exercises `StorageMode.custom` and `PortalSource.url`

The goal is to prove that the existing Swift-generated workspace files can be parsed and then re-encoded without semantic drift.

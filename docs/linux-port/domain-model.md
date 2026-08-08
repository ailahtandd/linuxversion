# Domain model for the Linux port

This document captures the important domain concepts that the Linux port must preserve, even though the implementation language and runtime change.

## 1. Workspace

A workspace is the top-level unit of organization. It contains:

- a human-readable name
- a working directory on disk
- a set of nodes and connections
- a canvas origin and zoom level
- persistence metadata such as created/modified timestamps

The workspace is the unit of autosave, crash recovery, and cross-node communication.

## 2. Canvas node

A canvas node is a rectangular object placed on the infinite canvas. It has:

- a stable UUID
- a frame expressed in canvas coordinates
- content that determines its type
- a z-order and lock state
- timestamps for creation and modification

The frame is not stored as a native CGRect in the serialized workspace format. It is stored in the Maestri-compatible `[[x, y], [w, h]]` array form.

## 3. Node content

Node content is a variant-like payload for the node type. The current implementation distinguishes node kinds such as:

- terminal
- note
- portal
- file tree
- text / generic canvas content

The Linux port must preserve the same semantic categories and the compatibility wrapper structure used by the existing workspace format.

## 4. Connections

Connections are separate objects that link nodes. The current implementation supports several relation types:

- terminal-to-terminal connections
- terminal-to-note connections
- terminal-to-portal connections
- portal-to-portal connections
- note-to-note connections
- cross-floor connections

These connections are persisted independently from nodes and carry geometry information such as rope control points.

## 5. Terminal session

A terminal session represents a terminal node with:

- a terminal UUID
- a shell process or PTY-backed process
- output history and scrollback
- an identity that is used for CLI routing by the inter-agent system

The Linux port must preserve the fact that terminal nodes are long-lived interactive shells and that each terminal has a stable identity.

## 6. Portal session

A portal session represents a browser-like embedded node. It has:

- a portal UUID
- current URL state
- browser navigation state
- automation capabilities such as click, fill, evaluate, and snapshot
- optional shared-session semantics when two portals are linked

The Linux port must preserve portal state and URL persistence, even if the browser runtime is replaced.

## 7. Note document

A note document is markdown-backed content associated with a note node. It can be:

- stored in a managed workspace-local markdown file
- edited via the app UI
- read and written through the CLI protocol

Notes should continue to be addressable by name and independent of the UI surface.

## 8. File tree state

The file tree is a view model over a workspace directory. It exposes:

- a tree of files and directories
- expansion state
- search results
- branch and Git status metadata

The Linux port should preserve the same browsing experience and metadata semantics, even if the implementation is reworked in React or a native shell view.

## 9. Git state

The current implementation treats Git as a runtime capability around the workspace directory. It uses the system `git` CLI rather than a deep embedded library. The Linux port should preserve:

- repository detection
- branch name reporting
- status reporting
- diff/commit/pull/push semantics as needed by the UI and future features

## 10. Runtime vs persisted state

The following are runtime-only concerns and do not need to be part of the serialized workspace schema:

- active viewport origin / zoom
- in-memory terminal PTY handles
- portal web-view instances
- transient selection and interaction state

The following must remain persisted or representable in a future-compatible format:

- node positions and sizes
- node content type and metadata
- connection geometry
- note content
- portal URL state
- workspace identity and metadata

# Compatibility matrix

This matrix records the current implementation expectations and the recommended Linux replacement strategy.

| Area | Current implementation | Compatibility requirement | Linux replacement direction |
| --- | --- | --- | --- |
| Workspace file format | Swift workspace payload and `workspace.json` | Preserve schema-compatible structure with `schemaVersion: 2` | Rust/Serde model that reads and writes the same shape |
| Node geometry | `CGRect` in memory, `[[x, y], [w, h]]` in serialized form | Preserve exact frame semantics | Rust geometry model with explicit frame serialization |
| Node content | `NodeContent` enum with wrapped object serialization | Preserve visible node categories and existing JSON envelope | Rust enum / JSON mapping with the same external contract |
| Canvas view | AppKit `NSView` with custom viewport logic | Preserve pan/zoom, selection, node placement, connections | React + PixiJS or a custom canvas renderer behind a shell view |
| Terminal runtime | SwiftTerm PTY adapter | Preserve interactive shell sessions and scrollback | xterm.js front-end plus Rust PTY backend |
| Portal runtime | WKWebView | Preserve URL state, browser automation, and persistence | A web runtime exposed through the Linux shell or an equivalent embedded browser layer |
| Inter-agent transport | Local HTTP + Unix socket | Preserve CLI payload semantics and terminal identity | Tokio + Unix domain sockets and a compatible protocol shim |
| Persistence | Atomic file writes through the persistence manager | Preserve crash-safe writes and workspace directory layout | Rust filesystem service with atomic replacement |
| Notes | Markdown files in workspace-local directories | Preserve content and addressability | Rust/TypeScript file service |
| Git integration | System `git` CLI | Preserve repository detection and basic status behavior | Rust or shell wrapper around `git` |

## Additional notes

- The Linux port should preserve behavior and file formats even when the runtime is rewritten.
- The port should not attempt to make Apple-specific frameworks compile under Linux.
- Where the UI is different, the service contracts should remain stable so that the product behavior continues to match the current macOS app.

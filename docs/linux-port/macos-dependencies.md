# macOS-specific dependencies and their role

This document enumerates the concrete macOS dependencies found in the implementation and explains how they influence the Linux port strategy.

## 1. SwiftUI

Usage:

- top-level application shell views
- layout composition for panels, inspector surfaces, toolbar surfaces, and workspace controls
- reactive view composition around `@Observable` state

Why it matters:

- SwiftUI is not a cross-platform target for this Linux port plan.
- The Linux implementation should re-create equivalent UX with React + TypeScript and Tauri shell primitives.

## 2. AppKit and NSView

Usage:

- the infinite canvas viewport is a custom AppKit `NSView`
- terminal views and portal web view embedding rely on native view hierarchies
- file tree browsing is built with AppKit outline view components
- some selection, focus, and resizing behavior is implemented through AppKit event plumbing

Why it matters:

- This is the most significant macOS-specific layer. It must be replaced rather than ported.
- Linux should treat the canvas as a renderer-driven surface rather than attempting to compile AppKit classes on Linux.

## 3. NSWindow, NSViewRepresentable, and related AppKit/SwiftUI bridging

Usage:

- the app uses SwiftUI to host AppKit-backed views and vice versa
- interactive canvas editing flows depend on host view geometry and responder chains

Why it matters:

- These APIs are strictly macOS UI frameworks. They should not be part of the Linux implementation.
- The Linux UI should be built as a native shell with the renderer embedded in the shell surface.

## 4. CoreGraphics

Usage:

- geometry calculations, frame conversions, node layout, rope rendering helpers
- canvas coordinate transforms and hit testing

Why it matters:

- CoreGraphics concepts are portable in principle, but the implementation is tied to the macOS AppKit runtime environment.
- The Linux port should preserve the same geometry semantics while using a cross-platform graphics abstraction.

## 5. SwiftTerm

Usage:

- PTY-backed terminal node rendering
- shell startup, input handling, output capture, and scrollback

Why it matters:

- SwiftTerm is the current terminal runtime and strongly tied to the native macOS app.
- Linux should replace it with xterm.js plus a Rust PTY implementation so the port can run on Linux without depending on SwiftTerm.

## 6. WebKit and WKWebView

Usage:

- embedded browser/portal nodes
- navigation, JS evaluation, screenshot capture, shared storage, and browser automation

Why it matters:

- WebKit is Apple-specific and should not be part of the Linux runtime.
- The Linux replacement should use a browser engine exposed through the platform shell, or a renderer that can expose DOM automation and screenshots via a supported web runtime.

## 7. Network.framework and NWConnection / NWListener

Usage:

- the inter-agent server uses `NWListener` for local TCP listener behavior
- the app uses local network semantics plus HTTP request/response handling

Why it matters:

- These are Apple networking APIs. The Linux port should instead use Tokio-based network services and Unix domain sockets for local IPC.
- The protocol semantics should stay compatible even if the transport implementation changes.

## 8. Darwin and low-level POSIX interop

Usage:

- socket operations and process control across the CLI transport and local IPC paths
- direct `socket`, `connect`, `send`, `recv`, `bind`, and `unlink` operations

Why it matters:

- These APIs are portable at the conceptual level, but the Linux implementation should use the Linux equivalents through Rust stdlib or Tokio rather than trying to compile Darwin-specific code.

## 9. Sparkle

Usage:

- app update checking and update UI flow

Why it matters:

- Sparkle is a macOS-oriented update framework.
- Linux should avoid bringing Sparkle into the new stack and instead rely on a different update/installation story that matches the Linux packaging model.

## 10. Foundation, OSLog, and other Apple SDK pieces

Usage:

- filesystem, process execution, logging, and structured metadata

Why it matters:

- Some of these are portable or already have equivalents in Rust/TypeScript.
- The Linux port should preserve the cross-cutting behavior while replacing the runtime implementation with Rust and TypeScript primitives.

## 11. Portability conclusion

The macOS implementation is deeply coupled to Apple UI and networking layers. The Linux port should preserve behavior and file formats, but it should not attempt to make AppKit, SwiftUI, SwiftTerm, WebKit, or Sparkle compile on Linux. The replacement stack should be intentionally new and isolated from the existing macOS code.

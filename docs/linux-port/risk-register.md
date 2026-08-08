# Risk register

This document captures the major technical risks for the Linux port and the mitigation direction.

## 1. Terminal parity risk

Risk: the Linux terminal experience may diverge from the current SwiftTerm/PTTY setup if the new stack does not preserve shell semantics, resize behavior, scrollback, or startup timing.

Mitigation:

- preserve the same shell startup contract
- use a real Rust PTY implementation
- keep the terminal identity and environment injection contract stable

## 2. Canvas performance and interaction risk

Risk: the canvas may feel slower or less responsive if the Linux renderer cannot match the current optimized AppKit viewport behavior.

Mitigation:

- use PixiJS for the renderer where appropriate
- keep the viewport state model explicit and frame-budgeted
- preserve node z-order, selection, and connection behavior in the domain model rather than in view-specific code

## 3. Workspace compatibility risk

Risk: a rewritten backend may accidentally change the serialized workspace format or lose compatibility with existing files.

Mitigation:

- maintain the same JSON shape and field names wherever possible
- preserve frame encoding and node content envelope
- add fixtures and compatibility tests around real workspace files

## 4. Agent protocol drift risk

Risk: the `omaestri` CLI contract may drift if the Linux transport changes the request format or loses identity semantics.

Mitigation:

- preserve the current `args` array / plain-text response model
- preserve `X-Terminal-ID` or an equivalent terminal identity header
- keep the CLI environment variables stable

## 5. Portal automation risk

Risk: browser automation and shared-session behavior may be difficult to reproduce with a new web runtime.

Mitigation:

- treat portal automation as a compatibility contract first
- start with simple navigation and snapshot behavior before expanding automation capabilities
- define the browser runtime abstraction early

## 6. Platform integration risk

Risk: the Linux shell may not provide equivalent native behaviors to the current macOS UI integration points.

Mitigation:

- isolate the Linux implementation behind the Tauri shell and a Rust backend
- avoid mixing AppKit/SwiftUI logic into the Linux code path
- keep UI concerns and domain logic separate

## 7. Delivery risk

Risk: implementing the full product stack at once may create a large and fragile initial version.

Mitigation:

- follow a staged implementation plan
- start with workspace compatibility and protocol correctness
- expand into canvas and portal features once the core contracts are stable

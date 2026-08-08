# AGENTS

This file defines the working agreement for the coding agent in this repository.

## Goal

Keep all work incremental, reviewable, and reversible. Every meaningful round of changes must end with a Git commit so commits become the control surface for progress, rollback, and review.

## Required Workflow

1. Start each round by checking the current repository state with `git status -sb`.
2. Enter a topic worktree on a feature branch before editing. Do not edit files directly in the shared `main` worktree.
3. Read the relevant files before editing. Do not guess repository structure or behavior.
4. Before writing any Swift code, invoke the `swiftui-expert-skill` to load current best practices.
5. Before implementing a new feature, check `docs/reference/maestri-reference-index.md` for the relevant UI and interaction spec.
6. Keep each round focused on a single coherent change.
7. After making changes, run the most relevant verification available for that round (see Verification section).
8. Summarize what changed, including any verification gaps.
9. Commit the round on the feature branch before stopping.

## Commit Policy

- Every round that modifies files must end with a commit.
- Do not batch unrelated changes into one commit.
- Commit messages must be in English and follow Conventional Commits style with emoji prefixes, for example:
  - `feat: ✨ add portal node resize handle`
  - `fix: 🐛 fix canvas coordinate overflow on zoom`
  - `refactor: ♻️ extract node frame conversion to CGRect+Frame`
  - `docs: 📝 update AGENTS.md`
  - `chore: 🔧 bump SwiftTerm dependency`
- Do not amend existing commits unless explicitly requested.
- Create a feature branch for every independent change. Do not commit directly to `main`.
- Push feature branches and open PRs when the user asks for remote review or integration.
- When the user asks to open or submit a PR, open a normal ready-for-review PR by default.

## Safety Rules

- Never revert or overwrite user changes unless explicitly requested.
- If unexpected changes appear, inspect them and work around them when possible.
- If a conflict makes the task ambiguous or risky, stop and ask before proceeding.
- Never use destructive Git commands such as `git reset --hard` without explicit approval.
- All file I/O to `~/.open-maestri/` must go through `PersistenceManager.swift` (atomic writes). Never write workspace files directly.
- Do not change `schemaVersion` in serialized JSON without a corresponding migration path.

## Engineering Rules

- Prefer small end-to-end slices over large speculative scaffolding.
- Preserve a clean working tree after each round.
- Add documentation when making architectural or workflow decisions.
- All canvas state mutations must run on `@MainActor`.
- All `@State` properties must be `private`.
- Use `@Observable` (Swift 5.9 macro), never `ObservableObject` / `@Published`.
- The canvas layer (`CanvasViewportView`) is pure AppKit (`NSView`). Bridge to SwiftUI only via `NSViewRepresentable`.
- When adding a new `NodeContent` case, serialize it as `{ "type": { "_0": ... } }` to maintain Maestri compatibility.
- When reading or writing `CanvasNode.frame`, use the `[[x, y], [w, h]]` array format via the `CGRect+Frame.swift` helpers.
- New CLI commands require both a registration in `CLIRouter.swift` and a new Handler file under `Sources/InterAgent/Handlers/`.

## Branching Rules

- `main` is the stable integration branch. No direct commits.
- Branch naming conventions:
  - `feat/<topic>` — new features
  - `fix/<topic>` — bug fixes
  - `refactor/<topic>` — code restructuring without behavior change
  - `docs/<topic>` — documentation-only changes
  - `chore/<topic>` — dependency updates, build config
- Use `git worktree add` to create an isolated worktree for each feature branch. Keep the `main` worktree clean.
- Merge or rebase against `main` before opening a PR to avoid stale conflicts.

## Product Boundaries

The following constraints define what is in and out of scope for this repository.

**Platform target:** macOS 14.0+ only. No iOS code. Use `#available` only for macOS version branches, not iOS guards.

**Swift version:** Swift 5.9. Do not use language features requiring a higher compiler version without explicit approval.

**Out of scope — do not implement:**
- Ombro companion feature (requires Apple Foundation Models framework and macOS 26+).
- Any iOS, iPadOS, watchOS, or tvOS targets.

**Partially implemented — extend carefully:**
- Floors (git worktree isolation): confirm design with `docs/reference/` before adding functionality.
- Routines (scheduled tasks): confirm design with `docs/reference/` before adding functionality.
- Remote SSH: confirm design with `docs/reference/` before adding functionality.

**Data compatibility:** The `workspace.json` schema (`schemaVersion: 2`) must remain binary-compatible with Maestri v0.25.4. Do not change field names or types in `CanvasNode`, `NodeContent`, or top-level `WorkspaceDocument` without a migration path.

## Verification

Run the appropriate verification command before committing.

```bash
# Compile check (fast, use after every edit)
swift build

# Full test suite
swift test

# Run a single targeted test
swift test --filter OpenMaestriTests.<SuiteClass>/<testMethod>

# Xcode CI build (no code signing required)
xcodebuild -scheme open-maestri -destination 'platform=macOS' build \
  CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO
```

After making changes, run `swift build` at minimum. If tests cover the changed code, run `swift test`. Note any verification gaps in the commit summary.

UI and runtime behavior must be verified manually by the user in Xcode. Do not claim a visual or interactive fix is complete without flagging that manual testing is required.

## Default Expectation

Unless instructed otherwise, the agent should:

1. Check `git status -sb` first.
2. Create a feature branch and worktree for the change.
3. Read relevant source files before editing.
4. Invoke `swiftui-expert-skill` before writing any Swift/SwiftUI code.
5. Make the minimal change that satisfies the task.
6. Run `swift build` to verify compilation.
7. Commit with a conventional-style English message.
8. Report what changed, what was verified, and what still requires manual testing.

## Linux Port Exception

A cross-platform Linux port is being developed as an explicit exception to the
macOS-only Product Boundary above.

The existing Swift/AppKit/SwiftUI application remains the reference macOS
implementation and must not be removed, replaced, or broken.

Linux port code must be isolated from the existing macOS implementation.

For the Linux port:

- Target Linux first, with architecture suitable for future Windows support.
- Use Tauri 2 for the desktop application shell.
- Use Rust for backend/platform services.
- Use React + TypeScript for the frontend.
- Use xterm.js for terminal rendering.
- Use a real PTY implementation in Rust.
- Use PixiJS where beneficial for the infinite canvas.
- Use Tokio for asynchronous backend services.
- Use Serde for serialization.
- Use Unix Domain Sockets for local IPC on Linux.

The existing macOS implementation is the behavioral and compatibility reference.

Do not attempt to make AppKit, SwiftUI, SwiftTerm, WKWebView, Network.framework,
Metal, Sparkle, or other Apple-specific UI/platform code compile on Linux.

Instead, reproduce the observable behavior using cross-platform components.

The Linux port must preserve compatibility with the existing workspace schema,
including `schemaVersion: 2`, `CanvasNode`, `NodeContent`, `WorkspaceDocument`,
node frame encoding, terminal identity, and inter-agent communication semantics.

Do not modify existing serialized field names or types merely to make the Rust
implementation cleaner.

Existing macOS-specific engineering rules apply only to the macOS implementation.
For example:

- `@MainActor`
- `@Observable`
- `NSView`
- `NSViewRepresentable`
- SwiftUI property rules

do not apply to the Linux React/Rust implementation.

Linux-specific code should live outside `Sources/`.

Preferred location:

apps/desktop/

Before implementing the Linux application, inspect and document the existing
macOS behavior and compatibility contracts.

Linux port priorities are:

1. Workspace compatibility
2. PTY/terminal correctness
3. Agent orchestration
4. Reliability
5. Maintainable architecture
6. Linux usability
7. Performance
8. Visual parity

Do not delete or substantially refactor the existing macOS implementation as
part of the Linux port unless explicitly requested
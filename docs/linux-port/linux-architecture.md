# Proposed Linux architecture

This document describes the proposed Linux architecture for the port. It is intentionally platform-native for Linux and isolated from the existing macOS implementation.

## 1. Architecture overview

The Linux port should be organized as a layered system:

- Tauri 2 provides the desktop shell, windowing surface, and native integration hooks.
- React + TypeScript powers the UI layer and the interactive workspace view.
- A Rust backend provides core domain services, persistence, PTY management, local IPC, and workspace orchestration.
- A Rust-side inter-process protocol uses Unix domain sockets for terminal-to-app communication on Linux.
- The frontend uses xterm.js for terminals and PixiJS for the infinite canvas where it is beneficial.

## 2. Layer responsibilities

### Shell layer (Tauri 2)

Responsibilities:

- host the React UI
- provide window lifecycle, menu integration, and local-file access where needed
- expose a thin bridge to the Rust backend for workspace operations and terminal control

### UI layer (React + TypeScript)

Responsibilities:

- render the workspace canvas and nodes
- handle user interactions such as pan, zoom, selection, create, connect, and edit
- render terminal panels using xterm.js
- render notes, file tree, and portal views using web-native surfaces

### Backend layer (Rust)

Responsibilities:

- manage workspaces and their persistence
- implement PTY sessions for terminal nodes
- route agent protocol commands and preserve terminal identities
- manage notes, portal state, and connection data
- expose a service API to the frontend over a local bridge

## 3. Runtime components

### Workspace service

A Rust workspace service should own the serialization and deserialization of the workspace model. It should read and write workspace files using the existing compatibility constraints.

### PTY service

The PTY service should own shell lifecycle, process spawn, resize, input/output, and scrollback. It should be the canonical source of terminal state for the Linux port.

### Agent protocol service

The Linux port should expose an agent-protocol service over Unix domain sockets. The Rust backend should accept the CLI payloads and route them to the right workspace/terminal context. The transport should remain compatible with the existing CLI expectations.

### Renderers

The frontend should have separate renderers for:

- canvas nodes and connections
- terminal sessions
- notes and file tree views
- portal/browser views

## 4. Data serialization

The Linux implementation should use Serde for JSON serialization and model conversion. The Rust models should preserve the existing workspace schema and be validated against the same compatibility requirements.

## 5. Key design principles

- preserve the existing macOS behavior as the contract
- keep the Linux implementation isolated from the Swift codebase
- do not attempt to compile AppKit or SwiftUI on Linux
- use Unix domain sockets for Linux-only local IPC
- use Tauri 2 as the application shell rather than building a custom renderer from scratch

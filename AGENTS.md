# AGENTS.md

This file provides guidance to Qoder (qoder.com) when working with code in this repository.

## Project Overview

Moraya is a cross-platform Markdown editor with AI integration, built as a Tauri v2 desktop application. The frontend is Svelte 5 + SvelteKit (SPA mode via adapter-static) with a ProseMirror-based editor. The backend is Rust. Communication between frontend and backend is via Tauri IPC (`invoke`).

## Build & Development Commands

```bash
# Install dependencies
pnpm install

# Start dev server with Tauri shell (full app with Rust backend)
pnpm tauri dev

# Start frontend only (no Tauri shell, runs on http://localhost:1420)
pnpm dev

# Full production build (frontend + Rust + native bundle)
pnpm tauri build

# TypeScript/Svelte type checking (the project's lint/check command)
pnpm check

# Type checking in watch mode
pnpm check:watch

# Rust-only check
cd src-tauri && cargo check

# Bump version across package.json, tauri.conf.json, Cargo.toml
pnpm version:bump <patch|minor|major|x.y.z>
```

There is no ESLint or Prettier configured. `pnpm check` (svelte-check) is the primary code quality gate and is run in CI.

## CI Pipeline

CI runs on push/PR to `main` (.github/workflows/ci.yml):
1. `pnpm check` - TypeScript/Svelte type checking
2. `pnpm tauri build` - Full production build (requires system dependencies on Linux)

## Architecture

### Two-Layer Design

**Frontend (src/):** Svelte 5 SPA. SSR is disabled (`export const ssr = false` in `+layout.ts`). The entire app is a single route (`src/routes/+page.svelte`) that composes all panels and editors.

**Backend (src-tauri/):** Rust Tauri v2 application. Handles all privileged operations: file I/O, AI API proxying (keys never exposed to WebView), MCP process management, OS keychain access, object storage HMAC signing, speech WebSocket proxying, and plugin management.

### Frontend Structure (src/lib/)

- **components/** - Svelte components for UI panels (Sidebar, TabBar, TitleBar, Settings, AI, MCP, Voice, Plugins, Publishing, SEO)
  - **components/ai/** - AI-specific components: AIChatPanel, AISettings, MCPPanel, TemplateGallery
- **editor/** - ProseMirror editor core
  - `Editor.svelte` - Main WYSIWYG editor component (large, ~85KB)
  - `SourceEditor.svelte` - Raw markdown source editor
  - `schema.ts` - ProseMirror document schema
  - `setup.ts` - ProseMirror plugin configuration
  - `markdown.ts` - Markdown serializer/parser (markdown-it based)
  - `commands.ts` - Editor commands (heading, bold, etc.)
  - `doc-cache.ts` - Document caching for performance
  - **plugins/** - ProseMirror plugins (code blocks, syntax highlight, keybindings, mermaid, emoji, etc.)
- **services/** - Business logic layer, organized by domain:
  - **ai/** - AI chat, streaming, multi-provider support, templates (71+), tool bridge for MCP, rules engine, image generation, SEO
  - **mcp/** - MCP client (stdio/SSE/HTTP), server management, marketplace, dynamic container, presets
  - **voice/** - Speech-to-text service with speaker diarization
  - **plugin/** - Plugin system: manager, renderer loader, renderer registry
  - **publish/** - GitHub publisher, API publisher, RSS feed generation
  - **image-hosting/** - Multi-provider image upload (SM.MS, Imgur, GitHub, S3, GCS, etc.)
  - `file-service.ts` - Frontend file operations (via Tauri IPC)
  - `export-service.ts` - HTML/LaTeX export
  - `update-service.ts` - Auto-update service
- **stores/** - Svelte stores for application state:
  - `editor-store.ts` - Editor state (content, mode, dirty flag)
  - `files-store.ts` - File explorer state
  - `settings-store.ts` - User settings (persisted via @tauri-apps/plugin-store)
  - `tabs-store.ts` - Multi-tab state management
- **i18n/** - Internationalization (12 locales in `locales/*.json`)
- **styles/** - Global CSS, editor CSS, theme system (`themes.ts`, CSS variables)
- **utils/** - Shared utilities (platform detection, frontmatter parsing)

### Backend Structure (src-tauri/src/)

- `lib.rs` - App setup, window management, multi-window support, tab transfer data structures
- `main.rs` - Entry point
- `menu.rs` - Native platform menus
- `dock.rs` - macOS Dock menu integration
- `tray.rs` - System tray (Windows/Linux)
- **commands/** - Tauri IPC command handlers:
  - `ai_proxy.rs` - Proxies all AI API calls (streaming SSE), keeps API keys server-side
  - `file.rs` - File system operations with path traversal protection
  - `keychain.rs` - OS keychain read/write via `keyring` crate
  - `mcp.rs` - MCP process lifecycle management (spawn/kill child processes)
  - `object_storage.rs` - HMAC request signing for cloud storage uploads
  - `plugin_manager.rs` - Plugin download, install, permission checking
  - `speech_proxy.rs` - WebSocket proxy for speech-to-text providers
  - `update.rs` - App update checking and download
  - `macos_system_audio.rs` - macOS-specific audio capture

### Key Patterns

- **Version is defined in 3 places**: `package.json`, `src-tauri/tauri.conf.json`, `src-tauri/Cargo.toml`. Use `pnpm version:bump` to keep them in sync.
- **`__APP_VERSION__`** is injected at build time via Vite `define` (see `vite.config.ts`) and declared in `src/app.d.ts`.
- **Security boundary**: API keys are stored in OS keychain (Rust `keyring` crate) and injected into requests by the Rust backend. The WebView never sees raw API keys. CSP is enforced in `tauri.conf.json`.
- **SPA routing**: The app uses SvelteKit with adapter-static and `fallback: "index.html"`. There's only one page route; all UI is composed in `+page.svelte`.
- **Platform conditionals in Rust**: Several modules are conditionally compiled (`#[cfg(target_os = "macos")]`, etc.) for platform-specific features like Dock menus, system tray, and audio capture.
- **Vite dev server** runs on port 1420 (strict port). Tauri dev connects to this URL.

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | Tauri v2 (>=2.10,<2.11) |
| Backend | Rust 2021 edition |
| Frontend | Svelte 5 + SvelteKit (SPA) |
| Editor | ProseMirror (custom schema, not Milkdown at runtime) |
| Build | Vite 6, pnpm 10 |
| Language | TypeScript (strict mode, ~5.6.2) |
| Math | KaTeX |
| Diagrams | Mermaid (lazy-loaded) |

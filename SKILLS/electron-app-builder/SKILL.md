---
name: electron-app-builder
description: Build or refactor production-grade Electron desktop apps using a hardened reusable desktop shell pattern. Use when creating a new Electron app, porting a web app into Electron, adding a custom frameless title bar, wiring secure IPC/preload handlers, setting up Electron Builder packaging, configuring React/React Native Web with Webpack, implementing app data persistence, registering local media/file protocols, or hardening Electron main-process architecture.
---

# Electron App Builder

## Core Workflow

Use this skill to create Electron apps with a polished desktop shell: frameless desktop window, custom title bar, secure preload bridge, modular IPC, Electron Builder packaging, persistent app data, asset-safe Webpack output, and Linux-friendly taskbar/icon behavior.

Before implementing, inspect the target repo. If the task is creating a new project, scaffold from the patterns in [electron-shell-patterns.md](references/electron-shell-patterns.md). If the task is refactoring an existing Electron app, compare its current shell against that reference and patch only the missing pieces.

## Implementation Rules

- Keep Electron main-process files modular: `electron/main.js`, `electron/preload.js`, `electron/state.js`, and focused files under `electron/modules/`.
- Use `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`, and `webSecurity: true`.
- Expose renderer capabilities through a small `window.<appApi>` bridge with allowlisted `invoke`, `send`, and `on` channels.
- Use `frame: false` with a custom title bar. On Linux, leave `titleBarStyle` undefined; on macOS use hidden titlebar behavior when desired.
- Keep title-bar drag regions explicit: `WebkitAppRegion: 'drag'` on the title area and `no-drag` on buttons/controls.
- Use a shared main-process state module for the current `BrowserWindow` and active background jobs/downloads.
- Use `app.requestSingleInstanceLock()` for single-instance desktop apps unless there is a specific reason not to.
- Keep app state under Electron `app.getPath('userData')`, preferably split into settings and library/content files. Preserve legacy state files during migrations.
- Register local media/file schemes as privileged when the renderer needs reliable local playback or fetch behavior.
- Use Electron Builder with `dist/**/*`, `electron/**/*`, public/build assets, package metadata, and any backend assets required by the app.
- Configure Webpack for content-hashed production chunks, `runtimeChunk: 'single'`, vendor/common split chunks, inline image assets, and generous Electron-appropriate performance budgets.
- Do not copy app-specific domain logic from reference projects unless the new app needs that exact behavior.

## Build Checklist

1. Create or verify `package.json` scripts: `start`, `electron`, `dev`, `build`, and `dist`.
2. Create `electron/main.js` with hardened `BrowserWindow` settings, single-instance handling, protocol registration, Linux icon setup, and module registration.
3. Create `electron/preload.js` with channel allowlists and listener cleanup through a `WeakMap`.
4. Create `electron/state.js` for shared mutable main-process handles.
5. Create `electron/modules/system.js` for window controls, safe external links, folder/file dialogs, default paths, and reveal-in-folder behavior.
6. Create `electron/modules/storage.js` for split JSON state under `userData`, temp-file writes, legacy migration, remove, and backup.
7. Add app-specific modules for downloads, media, subprocesses, diagnostics, or file processing only after the shell is stable.
8. Add a custom renderer title bar and route all window controls through the preload API.
9. Add Webpack and Babel configuration compatible with React or React Native Web.
10. Run the real build command and any available tests/checks. Do not claim runtime behavior unless Electron was actually launched and tested.

## Reference

Read [electron-shell-patterns.md](references/electron-shell-patterns.md) when implementing the shell, packaging, IPC, storage, protocol, or renderer title-bar patterns. It contains the concrete reusable shell decisions.

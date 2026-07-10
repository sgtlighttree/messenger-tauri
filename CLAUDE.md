# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Tauri v2 desktop wrapper for the Facebook Messenger web app, optimized for macOS. It is a thin native shell, not a reimplementation of Messenger.

## Architecture

The single most important thing to understand: **the app window loads `https://www.messenger.com` directly.** The window's `url` is set to the remote site in `src-tauri/tauri.conf.json` (`app.windows[0].url`), along with a spoofed desktop Chrome `userAgent` for compatibility.

Consequences:
- **The `src/` frontend (`main.ts`, `index.html`, `styles.css`) is unused at runtime.** It is leftover Tauri scaffolding (the `greet` demo). Vite still builds it into `../dist` because `frontendDist` points there, but the window never navigates to it. Don't add product UI here expecting it to appear in the app.
- Behavior/appearance changes for the actual app come from **native Tauri config** (`tauri.conf.json`, `capabilities/`, the Rust side in `src-tauri/src/lib.rs`) or from what messenger.com itself serves — not from `src/`.
- The `greet` Tauri command (`src-tauri/src/lib.rs` ↔ `src/main.ts`) is demo boilerplate and can be removed if you build real native commands.

Security is enforced via a strict CSP in `tauri.conf.json` (`app.security.csp`) that allowlists only Messenger/Facebook/fbcdn origins (including `wss://` for realtime). Native capabilities (notifications, opener) are granted in `src-tauri/capabilities/default.json`; a plugin used from Rust must also be registered in `lib.rs` **and** permitted there.

## Common commands

```bash
npm install              # install JS deps (also triggers Cargo build on first tauri run)
npm run tauri dev        # run the app in debug mode with devtools
npm run tauri build      # production .app / DMG for current arch
npm run deploy           # scripts/deploy.sh: kill running instance, build, copy to /Applications
```

Universal (Apple Silicon + Intel) build:
```bash
rustup target add aarch64-apple-darwin x86_64-apple-darwin
npm run tauri build -- --target universal-apple-darwin
```
Bundles land in `src-tauri/target/<...>/release/bundle/`.

There is no test suite, linter, or formatter configured. `npm run build` (`tsc && vite build`) only type-checks/builds the unused `src/` scaffolding.

## Gotchas

- Cargo dependencies in `src-tauri/Cargo.toml` are pinned to exact versions (`=2.2.1`, etc.). Prefer keeping them pinned; loosening them has caused version-mismatch build issues before.
- Package names are still the scaffold defaults (`tauri-app`, crate `tauri_app_lib`) even though the product name is `Messenger` (set in `tauri.conf.json` / `deploy.sh`). The deploy script matches the process name `Messenger`.
- Project is macOS-focused (iOS/Android icons exist but there is no mobile build flow documented here).

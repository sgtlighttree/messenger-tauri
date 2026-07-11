# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A lightweight-for-Electron macOS desktop wrapper for Facebook Messenger. It is a thin native
shell, not a reimplementation of Messenger: the app window loads `https://www.messenger.com`
directly, and essentially all product logic (chat, unread counts, calls) lives on the remote site,
not in this repo.

Electron (Chromium), not Tauri/WebKit, was chosen for exactly one reason: Messenger's voice/video
calls depend on a Chrome-only WebRTC API (`createEncodedStreams`) that WebKit does not implement.
See `docs/HANDOFF.md` for the full research trail behind that decision.

**Calls-gate status: UNVERIFIED.** Camera/mic permission wiring, entitlements, and screen-share
plumbing are implemented, but end-to-end calls have not been confirmed working in a real call.
Don't claim calls work; check `docs/CALLS-RESULT.md` — it's PASSED/FAILED/PARTIAL only after a
manual test pass fills it in.

## Common commands

```bash
npm install       # install JS deps
npm start         # npm run build && electron .  — launch the app locally
npm test          # vitest run — unit tests for src/main/links.ts and src/preload/unread.ts
npm run dist      # npm run build && electron-builder --mac — produce the arm64 .dmg in release/
```

`npm run build` runs `tsc` (main + shared) and bundles the preload with esbuild into a single CJS
file at `dist/preload/index.js` (required because the preload runs under Electron's `sandbox:
true`, which needs a bundled, dependency-free script).

## Architecture map

- **`src/main/config.ts`** — the single source of truth for the wrapped URL: `TARGET_URL`
  (`https://www.messenger.com`) and `ALLOWED_HOSTS` (hosts kept inside the app window: messenger.com,
  facebook.com, fbcdn.net, fbsbx.com — everything else is treated as external). To point the app
  at a different URL (e.g. if messenger.com is ever retired in favor of facebook.com/messages),
  change `TARGET_URL` here — nowhere else.
- **`src/main/links.ts`** — pure, unit-tested URL predicates (`isExternalUrl`, `isHttpUrl`) used to
  decide whether a navigation/window-open should stay in the app or be handed to
  `shell.openExternal`. No Electron imports; easy to test in isolation (see `tests/links.test.ts`).
- **`src/main/index.ts`** — the app entrypoint: creates the `BrowserWindow` with the security
  webPreferences (see Security invariants below), wires `setWindowOpenHandler` and `will-navigate`
  to `links.ts`'s predicates, configures the default session (downloads to `~/Downloads`, a
  permission handler that grants only `media`/`display-capture` and denies everything else, and a
  `setDisplayMediaRequestHandler` for screen-share), and listens for the unread-count IPC message
  to set the dock badge (`app.setBadgeCount`).
- **`src/preload/`** — the sandboxed, esbuild-bundled preload. `unread.ts` is a pure function
  (`parseUnreadCount`) that extracts the unread count from the page `<title>` (e.g. `"(3)
  Messenger"` → `3`); `index.ts` wires it up via a `MutationObserver` on `<title>` and sends it to
  the main process over IPC on change.
- **`src/shared/channels.ts`** — the single place IPC channel name constants (`IPC.SET_UNREAD`)
  are defined, shared between main and preload so the string literal only exists once.

## Security invariants — never weaken these

The page is remote, untrusted content (messenger.com), so the app is deliberately locked down:

- `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` on the `BrowserWindow`'s
  `webPreferences` (`src/main/index.ts`). The preload must stay a single bundled file with no
  Node/Electron internals leaking to the page — this is *why* it's esbuild-bundled rather than run
  as raw TypeScript.
- The session's permission handler (`ses.setPermissionRequestHandler`) grants only `media` (camera
  /mic) and `display-capture` (screen share); every other permission request must be denied.
  Don't broaden this without a specific reason.
- External/non-allowlisted navigations are intercepted (`setWindowOpenHandler`, `will-navigate`)
  and either handed to the system browser (http/https only) or dropped entirely (other schemes).
  Don't let arbitrary URLs load inside the app window.
- `build/entitlements.mac.plist` grants only camera, microphone, and JIT entitlements for the
  packaged app — no broader sandbox exceptions.

## Testing

`npm test` runs `vitest` against `tests/links.test.ts` and `tests/unread.test.ts`, covering the
pure predicate/parsing logic in `src/main/links.ts` and `src/preload/unread.ts`. There is
currently no automated test for Electron wiring itself (window creation, IPC, permission
handlers) — that's exercised manually.

## Packaging

`npm run dist` uses `electron-builder` (config in `package.json`'s `"build"` block) to produce an
arm64-only, `en`-locale-only `.dmg` in `release/` (note: `directories.output` is set to `release`,
*not* `dist`, because `dist` is also the TypeScript/esbuild build output directory — electron-builder's
`files` glob would otherwise recursively bundle its own previous output into the asar). There is no
Apple Developer ID signing identity on the build machine; electron-builder ad-hoc-signs instead,
so first launch requires right-click → Open to get past Gatekeeper. Real measured `.app`/`.dmg`
sizes and RAM are recorded in `docs/BUILD-NOTES.md` — don't invent footprint numbers.

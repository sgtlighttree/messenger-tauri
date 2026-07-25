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

**Calls-gate status: PASSED (2026-07-11).** Voice + video calls verified bidirectionally in dev
and packaged builds; screen share works (whole screen only). Evidence and root-cause trail in
`docs/CALLS-RESULT.md`. Anything touching popups, permissions, or navigation guards risks this —
the calls regression check in `docs/MANUAL-TESTS.md` §9 must be re-run after such changes.

## Common commands

```bash
pnpm install                          # install JS deps
pnpm start                            # pnpm run build && electron .  — launch the app locally
pnpm test                             # vitest run — unit tests for the pure logic in src/
CSC_NAME="Mercury Dev" pnpm run dist  # signed arm64 .dmg in release/ — CSC_NAME is REQUIRED
                                      # (ad-hoc builds can't register notifications and reset
                                       # TCC grants every rebuild; see docs/BUILD-NOTES.md)
```

**Notifications are a known, evidence-backed limitation** — messenger.com posts them via Web
Push, which Electron cannot receive (no push service). Do NOT re-attempt page-Notification
interception without reading the "Notifications" section of `docs/HANDOFF.md` first; the
existing preload shim + IPC bridge is the tested end state, and the owner chose badge+sound
over synthetic banners.

`pnpm run build` runs `tsc` (main + shared) and bundles the preload with esbuild into a single CJS
file at `dist/preload/index.js` (required because the preload runs under Electron's `sandbox:
true`, which needs a bundled, dependency-free script).

This project uses pnpm. Do not run `npm` or `yarn` here; `npm install` will regenerate a lockfile that conflicts with `pnpm-lock.yaml`.

## Architecture map

Process model, runtime data/storage locations (userData layout, what's regenerable), and
footprint live in `ARCHITECTURE.md`; this map covers the source files.

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
- **`src/main/context-menu.ts`** — pure, unit-tested right-click menu template builder
  (spelling suggestions, Look Up, cut/copy/paste, link/image actions) with injected actions —
  no Electron imports; `attachContextMenu` in `index.ts` maps the actions onto
  webContents/clipboard/shell and pops the real Menu.
- **`src/main/splash.ts`** — a small frameless, theme-aware (`prefers-color-scheme`) splash
  window built from an inline data-URL page (no static assets to package); shown while the main
  window loads messenger.com hidden, then destroyed on reveal (`did-finish-load` / 15s cap).
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

`pnpm test` runs `vitest` against `tests/links.test.ts` and `tests/unread.test.ts`, covering the
pure predicate/parsing logic in `src/main/links.ts` and `src/preload/unread.ts`. There is
currently no automated test for Electron wiring itself (window creation, IPC, permission
handlers) — that's exercised manually.

## Packaging

`pnpm run dist` uses `electron-builder` (config in `package.json`'s `"build"` block) to produce an
arm64-only, `en`-locale-only `.dmg` in `release/` (note: `directories.output` is set to `release`,
*not* `dist`, because `dist` is also the TypeScript/esbuild build output directory — electron-builder's
`files` glob would otherwise recursively bundle its own previous output into the asar).

Builds are signed with the self-signed "Mercury Dev" certificate in the login Keychain — hence
`CSC_NAME="Mercury Dev"` on the `dist` command. This is not an Apple Developer ID and the app is
not notarized, but a stable signing identity means macOS treats each rebuild as the same app:
TCC grants (camera, microphone, notifications) persist across rebuilds instead of resetting, and
launching an updated build needs no Gatekeeper right-click → Open. Building without `CSC_NAME`
falls back to ad-hoc signing, which loses both properties — see `docs/BUILD-NOTES.md`. Real
measured `.app`/`.dmg` sizes and RAM are recorded there too — don't invent footprint numbers.

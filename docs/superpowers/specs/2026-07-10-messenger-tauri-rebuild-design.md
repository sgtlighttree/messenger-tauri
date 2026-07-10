# Messenger‑Tauri Rebuild — Design Spec

**Date:** 2026-07-10
**Status:** Approved (brainstorming complete; ready for implementation planning)
**Owner:** Matthew Oyan
**Scope:** macOS only, personal use until reliable.

---

## 1. Purpose & Goals

Rebuild the abandoned `messenger-tauri` prototype into a genuinely usable, native‑feeling
macOS desktop client for Facebook Messenger — a spiritual successor to the discontinued
native macOS Messenger app.

**v1 success = "I use it daily instead of a browser tab":** it stays logged in, shows real
native notifications with an accurate unread badge, opens external links in my browser, and
downloads attachments properly — all in a lightweight (non‑Electron) shell.

**Explicit non‑goal for v1:** voice/video calls (see §3 — they cannot work in the chosen
engine today; deferred to a possible future Electron variant per §8).

---

## 2. Architecture Decision: thin shell around the real web app

We do **not** build a protocol client. There is no official personal Messenger API, and
unofficial reverse‑engineered clients violate ToS, get accounts banned, break constantly, and
cannot do calls. Instead we embed the real `https://www.messenger.com` web app in the native
webview and build an excellent **native integration layer** around it.

All product behavior (chat, presence, typing, media) comes from messenger.com itself. Our code
observes and augments the page; it does not reimplement it.

**Engine: Tauri v2 (WKWebView).** Chosen for its ~10–20 MB disk / ~30–50 MB idle RAM footprint
versus Electron's ~130 MB disk / ~200–300 MB idle. Since v1 is text‑only, WKWebView's call
limitation (§3) is not disqualifying, and Tauri is strictly lighter with no functional downside
for messaging.

---

## 3. Load‑bearing research: why calls are out of scope for a WebKit engine

This decision is grounded in a cross‑checked fact‑finding pass (Claude WebSearch/WebFetch run in
parallel with an independent `agy`/Gemini pass; each claim corroborated across ≥2 independent
domains, with primary sources for the load‑bearing ones).

**Finding 1 — Messenger web calls are Chromium‑only; Safari/WebKit is explicitly unsupported.**
`[VERIFIED, multi‑source]`
Meta maintains a help page literally titled "Browsers **not** supported for Messenger calls";
Apple Support threads report a "calling is not supported in this browser" error in Safari; a
webcompat bug shows call buttons disappearing on non‑Chromium engines. The failure mode is
*disabling* calls, not degrading them — there is no fallback path.

**Finding 2 — Root cause: Messenger's E2EE calls depend on the Chrome‑only `createEncodedStreams()`
(WebRTC Insertable Streams) API, which WebKit does not implement.** `[VERIFIED, primary source]`
Mozilla Bugzilla 1896361 states directly: *"Possibly due to the use of non‑standard
createEncodedStreams,"* and notes Meta *"could support Firefox and Safari"* but has not migrated to
the now‑baseline `RTCRtpScriptTransform`. webrtcHacks and a WebRTC compatibility guide independently
confirm Insertable Streams is "supported in Chrome but still missing in Safari." Because Messenger
made E2EE the **default for calls** in 2024, this now applies to essentially all calls.

**Finding 3 — `getUserMedia` (camera/mic) itself *does* work in WKWebView** since macOS 12 (the
`requestMediaCapturePermissionFor` `WKUIDelegate` API is macOS 12.3), given `NSCameraUsageDescription`/
`NSMicrophoneUsageDescription` in Info.plist plus `com.apple.security.device.camera`/`audio-input`
entitlements for signed builds. `[VERIFIED]` So the mic/camera plumbing is solvable; the encryption
API is not.

**Implication:** A "spike Tauri for calls" would predictably fail — not on `getUserMedia`, but on the
encryption transform. Therefore calls require Chromium (Electron), which is deferred (§8).

**Consequence for config:** The prototype's Chrome User‑Agent spoof (`tauri.conf.json:20`) is
actively harmful — it makes Messenger serve the Chromium call path that WKWebView cannot execute.
The UA must not falsely present as Chrome (see §6, Task in §7).

**Honest uncertainty:** We have not empirically placed a Messenger call in WKWebView on this exact
macOS build; the Safari case is documented directly for Firefox and by shared‑missing‑API analogy for
WebKit. It is also not permanent — if Meta migrates to standard `RTCRtpScriptTransform` (now baseline
in WebKit), WKWebView calls could begin working, at which point the Electron path (§8) may become
unnecessary.

---

## 4. Component Design

One Tauri window loads `https://www.messenger.com`. Rust modules form the native layer; a single
injected TypeScript bridge script runs inside the page. Each unit has one purpose and a defined
interface.

### 4.1 `webview/` (Rust) — window & navigation
- Creates the main window, loads messenger.com, sets a truthful/neutral User‑Agent.
- **Session persistence:** uses WKWebView's default persistent data store so cookies/login survive
  restarts. Must verify the store is not ephemeral/incognito.
- **Navigation interception:** an `on_navigation` handler cancels navigation to any host outside the
  messenger/facebook/fbcdn allowlist and instead opens it in the default browser via the opener
  plugin — so clicking a shared link doesn't hijack the app window.
- **Downloads:** enables the webview download hook and routes attachments to `~/Downloads`.

### 4.2 `bridge/` (injected TypeScript) — the only shipping frontend code
An initialization script injected into the page (via the webview builder). This replaces the dead
`src/` scaffold entirely. Responsibilities:
- **Notification interception:** wraps `window.Notification` so that when messenger.com fires a web
  notification, the wrapper forwards `{title, body, tag}` to Rust via a single named `invoke`
  command, and forwards click events back to focus the window.
- **Unread count:** reads the unread count from `document.title` (Messenger renders it as
  `(N) Messenger`) and pushes changes to Rust to set the dock badge. Uses a `MutationObserver` on
  `document.title` rather than polling.
- Communicates only through a **minimal, named command surface** (§5) — never the global Tauri API.

### 4.3 `notifications/` (Rust) — native surfacing
- Receives bridge events; fires native notifications via `tauri-plugin-notification`.
- Sets the macOS dock badge unread count. **Verify‑item:** confirm the exact current Tauri v2 badge
  API; fallback is a small `objc`/`objc2` call to `NSApp.dockTile.badgeLabel`. Do not assume an API
  name until verified against installed Tauri version.

### 4.4 `permissions/` (Rust) — deferred stub
- Camera/mic (`getUserMedia`) grant wiring is **not built in v1** (no calls). Left as a documented
  extension point so the Electron variant (§8) or a future WebKit‑calls world can slot in.

---

## 5. Security Posture (fixes the audit's High finding)

Because the window intentionally runs a **remote origin**, the web↔native bridge must be least‑privilege:

- `withGlobalTauri: false` (currently `true` at `tauri.conf.json:13`) — remote page must not get the
  global Tauri API.
- Remove the scaffold `greet` command (`src-tauri/src/lib.rs:2-5,12`).
- Expose only a **minimal, explicitly named command set** the bridge needs (e.g. `notify`, `set_badge`).
  Do **not** grant the broad `opener` capability to the page; external‑link opening is handled in Rust
  (§4.1), not by exposing opener to messenger.com JS.
- Keep the existing strict CSP (`tauri.conf.json:24`) — it is a genuine strength; retain the
  messenger/facebook/fbcdn + `wss:` allowlist.
- Prune `capabilities/default.json` to only in‑use permissions.

---

## 6. Cleanup baked into the rebuild

Not unrelated refactoring — this is removing dead weight the rebuild sits on:
- Delete `src-tauri/metadata.json` (2.7 MB unused `cargo metadata` dump that leaks local absolute
  paths) and gitignore it.
- Delete the unused `src/` Vite scaffold (`main.ts`, `index.html` greet demo, template assets); the
  only surviving frontend code is the injected bridge (§4.2). Reconcile `frontendDist`/`beforeBuildCommand`
  accordingly.
- Rename template identity: crate `tauri-app`/`tauri_app_lib` and npm `tauri-app` → messenger‑appropriate
  names; remove the "Tauri App" window‑title fallback.
- Fix the User‑Agent (§3 consequence).
- Remove leftover iOS/Android icon artifacts (macOS‑only project).
- Fix README inaccuracies (references nonexistent `hybrid-logo.png`; `YOUR_USERNAME` placeholder; does
  not mention the shell/remote‑origin architecture).

---

## 7. Milestones

**M0 — Clean foundation & safety net.**
Delete `metadata.json` + `src/` scaffold; rename identity; tighten IPC/CSP (§5); fix UA; minimal CI
(GitHub Actions on macOS running `cargo build` + `tsc`, gating on success). *Done: app still builds and
loads messenger.com; CI green; no dead scaffold; no High security finding.*

**M1 — Session & links (usable baseline).**
Confirm persistent login across restarts; external links open in default browser; downloads land in
`~/Downloads`. *Done: log in once, quit, relaunch → still logged in; links/downloads behave.*

**M2 — Notifications & badge (the native feel).**
Injected bridge intercepts notifications and unread count; native notifications fire; dock badge
accurate; notification click focuses the window. *Done: a new message with the app unfocused produces a
native notification and a correct badge; clicking it focuses the app.*

**M3 — Polish (deferred, optional).**
Tray/menu‑bar presence, close‑to‑background, launch‑at‑login, keyboard shortcuts, code signing/notarization
if distribution is ever pursued.

---

## 8. Deferred: calls via an Electron variant

Documented so the reasoning isn't re‑litigated later. If working voice/video becomes a hard requirement,
the only viable path is a **Chromium engine (Electron)**, because Messenger's E2EE calling requires
`createEncodedStreams()` (§3). A slimmed single‑arch Electron build floors at ~130 MB disk / ~300–450 MB
RAM when running Messenger (the Chromium bytes are exactly the call capability and cannot be removed).
This would be a **separate build target**, ideally sharing the injected‑bridge logic. Not started in v1.

---

## 9. Testing Strategy

Calibrated to a personal wrapper app — the point is confidence the shell behaves, not exhaustive coverage:
- **CI build gate (M0):** `cargo build` + `tsc --noEmit` must pass on every push.
- **Manual acceptance checklist** per milestone (the "Done" lines in §7) — these are the real
  behavioral tests for a webview wrapper, since the product logic lives on messenger.com.
- **Injected bridge (§4.2)** is the one piece of custom logic worth unit‑testing in isolation
  (title→count parsing, Notification wrapping) since it's pure and deterministic.
- No attempt to test messenger.com itself.

## 10. Definition of Done (v1)

- Daily‑usable: persistent login, native notifications, accurate dock badge, links → browser,
  downloads work.
- Zero High security findings: `withGlobalTauri:false`, `greet` gone, minimal command surface,
  capabilities pruned, CSP retained.
- No unreferenced large files; no dead scaffold; identity renamed; README accurate.
- CI green on every push.

## 11. Open Questions

1. **Distribution:** personal‑only (skip code signing/notarization), or eventually shared (M3 signing
   becomes required)?
2. **Badge API:** exact dock‑badge mechanism to confirm against the installed Tauri v2 version
   (native `objc2` fallback if no first‑class API).
3. **Notification fidelity:** does messenger.com reliably fire `window.Notification` in WKWebView, or
   is additional DOM observation needed to detect new messages? To be validated in M2.
4. **Electron trigger:** what concrete event (Meta migrating to standard WebRTC APIs, or a hard personal
   need for calls) would justify starting the §8 Electron variant?

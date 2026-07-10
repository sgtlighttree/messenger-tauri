# Messenger Desktop (macOS) — Rebuild Design Spec

**Date:** 2026-07-10
**Status:** Approved direction (brainstorming); ready for implementation planning
**Owner:** Matthew Oyan
**Scope:** macOS only, personal use until reliable.
**Supersedes:** the earlier Tauri-based draft (engine changed from Tauri→Electron after call-engine research; see §3).

---

## 1. Purpose & Goals

Rebuild the abandoned `messenger-tauri` prototype into a genuinely usable, native-feeling macOS
desktop client for Facebook Messenger — a replacement for the discontinued native macOS Messenger
app (Meta removed the Mac/Windows desktop apps; their own help article now directs desktop users to
`messenger.com`).

**v1 success = "I use it daily instead of a browser tab":** stays logged in (incl. 2FA), real native
notifications with an accurate unread badge, external links open in my browser, downloads work — in
as lightweight a shell as an Electron app can be.

**The differentiator (stretch, gated): working voice + video calls, and screen share if achievable.**
This is the capability every existing wrapper fails at (§2), and the whole reason to accept Electron's
footprint (§3).

---

## 2. Competitive landscape — why build this

All existing options fail the one thing we care about (calls) or are about to break:

- **Caprine** (`sindresorhus/caprine`) — Electron, mature but maintenance-only ("feature complete";
  v2.61.0 Jan 2026, prior release Dec 2024). ~110 MB / ~570 MB RAM. **Calls broken** (issue #2393,
  May 2026: call/video buttons do nothing; screen share reported non-functional). Hardwired to
  `messenger.com`.
- **messenger-mac** (`stefanminch/messenger-mac`) — Electron, stale (last update ~Dec 28). Issues flag
  **2FA problems and broken calls**.
- **goofy** (`danielbuechele/goofy`) — advertises itself as *not* Electron (likely WebKit-based). If so,
  **predicted to fail calls** for the same `createEncodedStreams` reason as Tauri (§3). *To be tested.*

**Our niche:** a lightweight-as-possible Electron client targeting the live `messenger.com`, that treats
**calls, screen share, and 2FA login as first-class, correctly-wired features** — the exact things the
incumbents get wrong. Honest caveat: incumbents' broken calls are also a warning that this is hard even
on Chromium (§3, §6).

---

## 3. Architecture Decision

**Wrap the real web app; do not build a protocol client.** No official personal Messenger API exists;
unofficial protocol clients violate ToS, get accounts banned, break constantly, and cannot do calls. We
embed `https://www.messenger.com` and build a native integration layer around it. All product behavior
comes from messenger.com; our code observes and augments the page.

**Engine: Electron (Chromium).** Chosen specifically to enable calls. Rationale is a cross-checked
fact-finding pass (Claude WebSearch/WebFetch in parallel with an independent `agy`/Gemini pass; each
load-bearing claim corroborated across ≥2 independent domains, primary sources where possible):

- **Calls require Chromium.** `[VERIFIED, primary source]` Messenger's E2EE calling (default for calls
  since 2024) depends on the Chrome-only `createEncodedStreams()` (WebRTC Insertable Streams) API, which
  WebKit/WKWebView does not implement (Mozilla Bugzilla 1896361: *"Possibly due to the use of non-standard
  createEncodedStreams"*; corroborated by webrtcHacks and a WebRTC compat guide). Meta's help page lists
  supported call browsers as Chromium-only. → **A WebKit engine (Tauri) is a hard no for calls.**
- **Chromium is necessary but maybe not sufficient.** `[Observed]` Caprine and messenger-mac are
  Electron/Chromium yet their calls are currently broken (§2). So the engine unblocks calls but the
  wrapper must still wire media/screen permissions correctly, target the live URL, and avoid Meta gating.
  → **Calls remain a spike/gate even on Electron (§6), just far more likely to pass than Tauri's.**
- **Uploads do NOT justify the engine choice.** `[VERIFIED empirically]` A live test in WebKit (Safari,
  same engine as WKWebView) uploaded 20 MB and 70 MB videos to a 1:1 chat successfully; only an HTML file
  stalled (a rare, likely security-special-cased type). My earlier "large media flaky in WebKit" research
  over-predicted; the test corrected it. Uploads are struck from the Electron rationale — **calls are the
  sole reason.**

**Target URL: `messenger.com`.** `[Primary source over secondary reporting]` Secondary outlets (Feb 2026)
reported messenger.com shutting down April 15 2026 → `facebook.com/messages`. But Meta's own current help
article (`facebook.com/help/messenger-app/804132271957789`) directs desktop users to **messenger.com**,
and empirically messenger.com returns HTTP 200 and works today (July 2026, months past the reported date).
Live official guidance + working site override the reporting. **We target messenger.com**, but the target
is a **single config constant** (`TARGET_URL`) so switching to `facebook.com/messages` is a one-line change
if the redirect ever completes. Deprecation is a documented watch-item (§12), not a current reality.

**Honest footprint expectation.** Electron floors at ~90–130 MB disk / ~300–450 MB RAM running Messenger,
no matter how hard we optimize — the Chromium bytes *are* the call capability. "Lightweight" here means
best-in-class-*for-Electron* (§9), not Tauri numbers (~10 MB / ~150 MB). If branded "lightweight," be honest
about which kind.

---

## 4. Component Design

Standard Electron process split; each unit has one purpose and a defined interface.

### 4.1 `main` (Electron main process)
- Creates one `BrowserWindow` loading `TARGET_URL` with a hardened `webPreferences` (§5).
- **Navigation & links:** `setWindowOpenHandler` and a `will-navigate` handler — keep messenger/facebook
  auth navigation (incl. 2FA popups) **in-app**, but route genuinely external hosts to the default browser
  via `shell.openExternal`. (Getting this boundary right is what makes 2FA login work where messenger-mac
  fails.)
- **Downloads:** `session.on('will-download')` → save to `~/Downloads`.
- **Permissions:** `session.setPermissionRequestHandler` grants `media` (camera/mic) for calls;
  `session.setDisplayMediaRequestHandler` handles screen share (§6).
- **Badge:** exposes an IPC endpoint the preload calls to set `app.setBadgeCount(n)` (first-class on macOS).

### 4.2 `preload` (the injected bridge — runs per page load)
The only custom "frontend" code (replaces the dead Tauri `src/` scaffold). Minimal and context-isolated
(§5). Responsibilities:
- **Unread count:** observe `document.title` (Messenger renders `(N) Messenger`) via `MutationObserver`
  and send the count to main over a single named IPC channel → dock badge.
- **Notifications:** *verify first* — Electron natively maps the web `Notification` API to OS
  notifications, so messenger.com's own notifications may surface natively with **no interception**. If
  they do, do nothing. If they don't (e.g. permission/opacity quirks), fall back to wrapping
  `window.Notification` and forwarding to main. Do not build interception speculatively.
- Exposes **nothing** to the page beyond what the badge/notification bridge needs, via `contextBridge`.

### 4.3 `renderer` = messenger.com
Not our code. Loaded remotely, sandboxed, no Node access (§5).

**Key Electron advantage over the prior Tauri design:** notifications and dock badge are first-class
(`Notification` mapping + `app.setBadgeCount`), so the native-feel layer is *simpler* here — no `objc`
badge hack, likely no notification interception.

---

## 5. Security Posture

Because the window runs a **remote origin**, the renderer must be untrusted and isolated:
- `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` — messenger.com JS gets **no** Node
  or Electron internals.
- Preload exposes only a minimal, named `contextBridge` API (badge/notification), never the raw `ipcRenderer`.
- `setWindowOpenHandler` denies arbitrary `window.open` to new BrowserWindows; external hosts go to the OS
  browser, auth/messenger hosts stay in the main window.
- `webSecurity` on; no disabling of same-origin. No remote module.
- Permission handler grants **only** `media`/`display-capture` (for calls), denies the rest by default.

---

## 6. Calls + Screen Share (the differentiator — a gated milestone)

This is why we accept Electron; it is also the riskiest part (incumbents fail here), so it is an explicit
**gate with a documented fallback**.

- **Media (voice/video):** `session.setPermissionRequestHandler` grants camera/mic; `electron-builder`
  `mac.extendInfo` supplies `NSCameraUsageDescription` + `NSMicrophoneUsageDescription`; Hardened Runtime
  entitlements `com.apple.security.device.camera` / `audio-input`. Then **place a real 1:1 voice call, then a
  video call, on messenger.com** and confirm bidirectional media.
- **Screen share:** `session.setDisplayMediaRequestHandler` + `desktopCapturer.getSources`, gated by the
  macOS Screen Recording TCC permission. Attempt in-call screen share; if Meta's web UI doesn't offer it or
  it fails, mark screen share **best-effort/deferred** without blocking calls.
- **Gate:** if voice+video work → this is the shipping differentiator. If Meta blocks calls even on our
  correctly-wired Electron build → document the failure mode and reasses (the app is still a strong
  text+media client). **Do not claim calls work until verified in our own build.**
- **Cross-check:** compare against the `goofy` test result — if a non-Electron app can call, revisit the
  `createEncodedStreams` assumption.

---

## 7. Login & Session (incl. 2FA)

- Electron persists cookies/localStorage in `userData` by default → login survives restarts. Verify the
  session partition is persistent (not `partition: 'temp'`).
- **2FA must work:** the login/2FA flow may use popups or in-page redirects on facebook/messenger hosts.
  The `setWindowOpenHandler`/`will-navigate` logic (§4.1) must keep those **in-app**, not shunt them to the
  browser — the concrete bug that breaks 2FA in messenger-mac. Acceptance: log in with 2FA once, quit,
  relaunch → still logged in.

---

## 8. Migration & Cleanup (from the Tauri prototype)

- Remove the Tauri stack: `src-tauri/` (Rust), the Vite `src/` scaffold (`main.ts`, `index.html` greet demo),
  `vite.config.ts`, `tsconfig.json` as-is, and the 2.7 MB unused `src-tauri/metadata.json` (leaks local paths).
- New structure: `main` + `preload` (TypeScript), an `electron-builder` config, and a lean `package.json`
  (electron + electron-builder + minimal deps; no React/framework — the UI is messenger.com).
- Reuse the macOS app icons; delete leftover iOS/Android icon artifacts (macOS-only).
- Rename identity away from `tauri-app`; set product name "Messenger".
- Rewrite README (current one references a nonexistent `hybrid-logo.png`, has a `YOUR_USERNAME` placeholder,
  and describes the old Tauri/WKWebView design).

---

## 9. Lightweight Strategy (best-in-class *for Electron*)

- **Latest Electron** (baseline RAM improvements).
- **Single `BrowserWindow`** (each window = a renderer process).
- **Arch-specific builds** (arm64 primary), not universal — roughly halves the bundle.
- **Strip locales** to `en` (`electronLanguages: ['en']`) — trims Chromium `.pak` files.
- **asar** packaging; prune devDeps from the bundle; no heavy runtime deps.
- **`backgroundThrottling: true`**; consider discarding/reloading the renderer when hidden for long periods
  (trade-off: reload flash).
- Target: ~90–130 MB dmg, ~300–450 MB RAM active. Measure and record actuals; don't market numbers we
  haven't hit.

---

## 10. Milestones

**M0 — Electron foundation & clean slate.** Remove Tauri/scaffold/`metadata.json`; scaffold a hardened
single-window Electron app (§5) loading `TARGET_URL = messenger.com`; rename identity; CI on macOS building
the app + `tsc`. *Done: app builds, loads messenger.com in an isolated renderer, CI green.*

**M1 — Login, links, downloads (usable baseline).** Persistent session incl. **2FA**; external links →
browser while keeping auth in-app; downloads → `~/Downloads`. *Done: 2FA login persists across relaunch;
links/downloads behave.*

**M2 — Notifications & badge (native feel).** Verify native `Notification` mapping; add unread-count →
`app.setBadgeCount`; notification click focuses window. *Done: unfocused new message → native notification +
correct dock badge; click focuses app.*

**M3 — Calls + screen share (the differentiator, GATE).** Wire media permissions + Info.plist/entitlements;
place real voice + video calls on messenger.com; attempt screen share. *Done: bidirectional voice+video
verified in our build; screen share working or explicitly deferred with a documented reason.*

**M4 — Lightweight pass & polish.** Locale strip, arch-specific build, asar, RAM tuning (§9); optional tray,
close-to-background, launch-at-login. Record real footprint numbers.

---

## 11. Testing Strategy & Definition of Done

- **CI build gate (M0):** app build + `tsc --noEmit` pass on every push.
- **Manual acceptance checklist** per milestone (the "Done" lines above) — the real behavioral tests for a
  webview wrapper, since product logic lives on messenger.com.
- **Preload unit test:** the `document.title` → unread-count parser (pure, deterministic).
- **Calls (M3):** explicit manual verification is the gate; not shippable-as-"calls work" without it.

**v1 Done:** daily-usable (2FA-persistent login, native notifications, accurate badge, links, downloads);
calls either verified working (goal) or documented as blocked; secure renderer (isolation on, minimal bridge);
Tauri artifacts gone; README accurate; CI green; recorded footprint.

---

## 12. Open Questions / Watch-items

1. **messenger.com deprecation:** secondary reporting says it redirects to `facebook.com/messages`; Meta's
   live help article says use messenger.com. We target messenger.com via a single `TARGET_URL` constant —
   **watch for an actual redirect** and flip if it lands. Does `facebook.com/messages` also support calls
   and require a full Facebook account? (Login nuance if we ever switch.)
2. **Calls on Electron — will Meta let them run?** The gate in M3. Root cause of incumbents' broken calls is
   unconfirmed (Caprine issue #2393 has no diagnosis): missing permission wiring (fixable) vs Meta gating
   (not). Resolve empirically.
3. **`goofy` test result:** if a non-Electron app does calls, revisit the engine premise.
4. **Distribution:** personal-only (skip signing/notarization) or eventually shared (then Hardened Runtime +
   notarization become required, and interact with the camera/mic/screen entitlements from §6)?
5. **Screen share depth:** is in-call screen share exposed by messenger.com's web UI at all, or only in the
   native/mobile clients? Determines whether M3 screen share is even reachable.

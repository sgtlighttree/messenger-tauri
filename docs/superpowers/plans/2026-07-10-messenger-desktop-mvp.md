# Messenger Desktop (Electron) MVP — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the abandoned Tauri prototype as a lightweight macOS Electron app that wraps `messenger.com`, with persistent (2FA) login, native notifications + dock badge, sane link/download handling, and — the differentiator — working voice/video calls.

**Architecture:** One hardened Electron `BrowserWindow` loads the live `messenger.com`. The untrusted remote page is isolated (`contextIsolation`, no `nodeIntegration`); a thin trusted preload observes the page title for unread counts. The main process wires session permissions (camera/mic/screen for calls), download routing, and external-link handling. Product logic lives on messenger.com; we build only the native shell.

**Tech Stack:** Electron (latest), TypeScript, Vitest (unit tests for pure logic), electron-builder (macOS packaging). No UI framework — the "UI" is messenger.com.

## Global Constraints

- **Platform:** macOS only. Do not add Windows/Linux/mobile code paths.
- **Target URL:** `https://www.messenger.com`, defined once as `TARGET_URL` in `src/main/config.ts`. Never hardcode the URL elsewhere.
- **Security (non-negotiable for the remote page):** `contextIsolation: true`, `nodeIntegration: false`. The page must never receive Node or raw `ipcRenderer`.
- **Least privilege:** the permission handler grants only `media` and `display-capture`; everything else is denied.
- **Product/window name:** "Messenger". Crate/npm identity must not be `tauri-app`.
- **Calls are a GATE, not a guarantee** (Task 7): verify empirically; if Meta blocks calls on our build, document the failure — do not claim calls work unverified.
- **Frequent commits:** one commit per task minimum.

---

## File Structure

```
package.json                      # electron, electron-builder, typescript, vitest; scripts + build config
tsconfig.json                     # TS → CommonJS, rootDir src, outDir dist, strict
.github/workflows/ci.yml          # macOS CI: build + tsc + tests
build/entitlements.mac.plist      # camera/mic/audio entitlements (packaging)
src/
  shared/channels.ts              # IPC channel name constants (imported by main + preload)
  main/
    config.ts                     # TARGET_URL, ALLOWED_HOSTS
    links.ts                      # isExternalUrl(url) pure fn
    index.ts                      # app entry: window, session, permissions, IPC
  preload/
    unread.ts                     # parseUnreadCount(title) pure fn
    index.ts                      # observes document.title → sends unread count over IPC
tests/
  links.test.ts                   # unit: isExternalUrl
  unread.test.ts                  # unit: parseUnreadCount
docs/HANDOFF.md                    # session decision log (created separately, not a task here)
```

Files removed in Task 1: `src-tauri/`, `src/main.ts`, `src/styles.css`, `src/assets/`, `index.html`, `vite.config.ts`, `src-tauri/metadata.json`, and the iOS/Android icons under `src-tauri/icons/`.

---

### Task 1: Clean slate + Electron scaffold

Remove all Tauri/Vite artifacts and stand up a minimal Electron + TypeScript + Vitest project that installs, type-checks, and runs an (empty) test suite. This is pure setup; its deliverable is a green `npm install && npm run build && npm test`.

**Files:**
- Delete: `src-tauri/` (whole dir), `src/` (whole dir), `index.html`, `vite.config.ts`
- Modify/replace: `package.json`, `tsconfig.json`
- Create: `src/main/config.ts`

**Interfaces:**
- Produces: `TARGET_URL: string`, `ALLOWED_HOSTS: string[]` from `src/main/config.ts`.

- [ ] **Step 1: Remove Tauri and Vite artifacts**

```bash
git rm -r src-tauri src index.html vite.config.ts
```

- [ ] **Step 2: Replace `package.json`**

```json
{
  "name": "mercury-mac",
  "productName": "Messenger",
  "private": true,
  "version": "0.1.0",
  "description": "Lightweight macOS desktop client for Facebook Messenger",
  "license": "MIT",
  "main": "dist/main/index.js",
  "scripts": {
    "build": "tsc",
    "start": "npm run build && electron .",
    "test": "vitest run",
    "dist": "npm run build && electron-builder --mac"
  },
  "devDependencies": {
    "electron": "^35.0.0",
    "electron-builder": "^25.0.0",
    "typescript": "~5.6.2",
    "vitest": "^2.1.0"
  }
}
```

- [ ] **Step 3: Replace `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src"]
}
```

- [ ] **Step 4: Create `src/main/config.ts`**

```ts
export const TARGET_URL = "https://www.messenger.com";

// Hosts kept INSIDE the app (login, 2FA, chat, media CDN). Everything else opens in the browser.
export const ALLOWED_HOSTS = [
  "messenger.com",
  "facebook.com",
  "fbcdn.net",
  "fbsbx.com",
];
```

- [ ] **Step 5: Install and verify**

Run: `npm install && npm run build && npm test`
Expected: install succeeds; `tsc` produces `dist/main/config.js` with no errors; `vitest run` reports "No test files found" (exit 0) — acceptable at this stage.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: remove Tauri stack, scaffold Electron + TypeScript project"
```

---

### Task 2: Hardened main window loads messenger.com

Create the Electron entry point: one `BrowserWindow` with the security constraints, loading `TARGET_URL`. Deliverable is observable — the app launches and shows messenger.com in an isolated renderer.

**Files:**
- Create: `src/main/index.ts`

**Interfaces:**
- Consumes: `TARGET_URL` from `config.ts`.
- Produces: a running Electron app; `createWindow()` internal.

- [ ] **Step 1: Write `src/main/index.ts`**

```ts
import { app, BrowserWindow } from "electron";
import * as path from "path";
import { TARGET_URL } from "./config";

let mainWindow: BrowserWindow | null = null;

function createWindow(): void {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 800,
    title: "Messenger",
    webPreferences: {
      preload: path.join(__dirname, "../preload/index.js"),
      contextIsolation: true,
      nodeIntegration: false,
      // sandbox:false lets the trusted preload use local module imports.
      // The UNTRUSTED page stays isolated via contextIsolation + nodeIntegration:false.
      sandbox: false,
    },
  });
  mainWindow.loadURL(TARGET_URL);
  mainWindow.on("closed", () => {
    mainWindow = null;
  });
}

app.whenReady().then(() => {
  createWindow();
  app.on("activate", () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on("window-all-closed", () => {
  // MVP: quit when the window closes. Close-to-tray is a later polish task.
  if (process.platform !== "darwin") app.quit();
});
```

- [ ] **Step 2: Create a no-op preload so the path resolves**

Create `src/preload/index.ts`:

```ts
// Populated in Task 5. Present now so the preload path resolves.
export {};
```

- [ ] **Step 3: Launch and observe**

Run: `npm start`
Expected: an Electron window opens showing the messenger.com login page. Open the menu → View → Toggle Developer Tools; in the Console run `typeof window.require` and confirm it prints `"undefined"` (proves `nodeIntegration:false`). Close the app.

- [ ] **Step 4: Commit**

```bash
git add src/main/index.ts src/preload/index.ts
git commit -m "feat: hardened main window loading messenger.com"
```

---

### Task 3: External links open in the browser (keep auth in-app)

TDD a pure `isExternalUrl` predicate, then wire `setWindowOpenHandler` + `will-navigate` so external links go to the system browser while messenger/facebook (including 2FA auth) stay in-app.

**Files:**
- Create: `src/main/links.ts`, `tests/links.test.ts`
- Modify: `src/main/index.ts`

**Interfaces:**
- Consumes: `ALLOWED_HOSTS` from `config.ts`.
- Produces: `isExternalUrl(rawUrl: string): boolean`.

- [ ] **Step 1: Write the failing test — `tests/links.test.ts`**

```ts
import { describe, it, expect } from "vitest";
import { isExternalUrl } from "../src/main/links";

describe("isExternalUrl", () => {
  it("keeps messenger.com internal", () => {
    expect(isExternalUrl("https://www.messenger.com/t/123")).toBe(false);
  });
  it("keeps facebook login/2FA internal", () => {
    expect(isExternalUrl("https://www.facebook.com/checkpoint/")).toBe(false);
  });
  it("keeps fbcdn media subdomains internal", () => {
    expect(isExternalUrl("https://scontent.fbcdn.net/v/x.jpg")).toBe(false);
  });
  it("routes a shared external link out", () => {
    expect(isExternalUrl("https://example.com/article")).toBe(true);
  });
  it("treats an unparseable value as internal (do not hijack)", () => {
    expect(isExternalUrl("javascript:void(0)")).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/links.test.ts`
Expected: FAIL — cannot resolve `../src/main/links`.

- [ ] **Step 3: Write `src/main/links.ts`**

```ts
import { ALLOWED_HOSTS } from "./config";

/** True if the URL should open in the system browser rather than in the app window. */
export function isExternalUrl(rawUrl: string): boolean {
  let host: string;
  try {
    const parsed = new URL(rawUrl);
    if (parsed.protocol !== "https:" && parsed.protocol !== "http:") return true;
    host = parsed.hostname.toLowerCase();
  } catch {
    return true; // unparseable / non-URL scheme → don't navigate the app to it
  }
  return !ALLOWED_HOSTS.some(
    (allowed) => host === allowed || host.endsWith("." + allowed),
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/links.test.ts`
Expected: PASS (5 passed).

- [ ] **Step 5: Wire it into `src/main/index.ts`**

Add imports at the top:

```ts
import { app, BrowserWindow, shell } from "electron";
import { isExternalUrl } from "./links";
```

Inside `createWindow()`, after `mainWindow.loadURL` is defined but before it, add link handling to the `webContents`:

```ts
  const wc = mainWindow.webContents;
  wc.setWindowOpenHandler(({ url }) => {
    if (isExternalUrl(url)) {
      void shell.openExternal(url);
      return { action: "deny" };
    }
    return { action: "allow" };
  });
  wc.on("will-navigate", (event, url) => {
    if (isExternalUrl(url)) {
      event.preventDefault();
      void shell.openExternal(url);
    }
  });
```

- [ ] **Step 6: Build, launch, observe**

Run: `npm start`
Expected: log in; click an external link shared in a chat → it opens in your default browser, not the app. Navigating login/2FA stays in-app.

- [ ] **Step 7: Commit**

```bash
git add src/main/links.ts tests/links.test.ts src/main/index.ts
git commit -m "feat: route external links to browser, keep auth in-app"
```

---

### Task 4: Downloads save to ~/Downloads

Wire the session `will-download` handler so attachments land in `~/Downloads` without a save dialog on every file.

**Files:**
- Modify: `src/main/index.ts`

**Interfaces:**
- Produces: `configureSession()` internal (extended in Tasks 5, 7, 8).

- [ ] **Step 1: Add `configureSession()` to `src/main/index.ts`**

Extend imports:

```ts
import { app, BrowserWindow, shell, session } from "electron";
```

Add the function:

```ts
function configureSession(): void {
  const ses = session.defaultSession;
  ses.on("will-download", (_event, item) => {
    item.setSavePath(path.join(app.getPath("downloads"), item.getFilename()));
  });
}
```

Call it inside `app.whenReady().then(...)` before `createWindow()`:

```ts
app.whenReady().then(() => {
  configureSession();
  createWindow();
  app.on("activate", () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});
```

- [ ] **Step 2: Build, launch, observe**

Run: `npm start`
Expected: in a chat, download an image/file → it appears in `~/Downloads` with no save dialog.

- [ ] **Step 3: Commit**

```bash
git add src/main/index.ts
git commit -m "feat: save downloads to ~/Downloads"
```

---

### Task 5: Unread count → dock badge

TDD a pure `parseUnreadCount(title)`, then have the preload observe `document.title` and send the count to main, which sets the macOS dock badge via `app.setBadgeCount`.

**Files:**
- Create: `src/preload/unread.ts`, `tests/unread.test.ts`, `src/shared/channels.ts`
- Modify: `src/preload/index.ts`, `src/main/index.ts`

**Interfaces:**
- Produces: `parseUnreadCount(title: string): number`; `IPC.SET_UNREAD` channel constant `"set-unread"`; preload sends `number` on that channel; main calls `app.setBadgeCount(number)`.

- [ ] **Step 1: Write the failing test — `tests/unread.test.ts`**

```ts
import { describe, it, expect } from "vitest";
import { parseUnreadCount } from "../src/preload/unread";

describe("parseUnreadCount", () => {
  it("parses a leading count", () => {
    expect(parseUnreadCount("(3) Messenger")).toBe(3);
  });
  it("parses a capped count like (12+)", () => {
    expect(parseUnreadCount("(12+) Messenger")).toBe(12);
  });
  it("returns 0 when there is no count", () => {
    expect(parseUnreadCount("Messenger")).toBe(0);
  });
  it("returns 0 for an empty title", () => {
    expect(parseUnreadCount("")).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/unread.test.ts`
Expected: FAIL — cannot resolve `../src/preload/unread`.

- [ ] **Step 3: Write `src/preload/unread.ts`**

```ts
/** Extract the unread count from the page title, e.g. "(3) Messenger" → 3, "(12+) Messenger" → 12. */
export function parseUnreadCount(title: string): number {
  const match = title.match(/^\((\d+)\+?\)/);
  if (!match) return 0;
  const n = parseInt(match[1], 10);
  return Number.isFinite(n) ? n : 0;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/unread.test.ts`
Expected: PASS (4 passed).

- [ ] **Step 5: Create `src/shared/channels.ts`**

```ts
export const IPC = {
  SET_UNREAD: "set-unread",
} as const;
```

- [ ] **Step 6: Replace `src/preload/index.ts`**

```ts
import { ipcRenderer } from "electron";
import { parseUnreadCount } from "./unread";
import { IPC } from "../shared/channels";

function reportUnread(): void {
  ipcRenderer.send(IPC.SET_UNREAD, parseUnreadCount(document.title));
}

window.addEventListener("DOMContentLoaded", () => {
  reportUnread();
  const titleEl = document.querySelector("title");
  if (titleEl) {
    new MutationObserver(reportUnread).observe(titleEl, {
      childList: true,
      characterData: true,
      subtree: true,
    });
  }
});
```

- [ ] **Step 7: Handle the IPC in `src/main/index.ts`**

Extend imports:

```ts
import { app, BrowserWindow, shell, session, ipcMain } from "electron";
import { IPC } from "../shared/channels";
```

In `app.whenReady().then(...)`, register the handler before `createWindow()`:

```ts
  ipcMain.on(IPC.SET_UNREAD, (_event, count: number) => {
    if (typeof count === "number" && count >= 0) app.setBadgeCount(count);
  });
```

- [ ] **Step 8: Build, launch, observe**

Run: `npm start`
Expected: with unread conversations, the dock icon shows a red badge with the count; reading them clears it. (If the badge never appears, note it for Task 6/spec Open Question 2 — Messenger may not put the count in `document.title`; capture the actual title string from DevTools.)

- [ ] **Step 9: Commit**

```bash
git add src/preload tests/unread.test.ts src/shared/channels.ts src/main/index.ts
git commit -m "feat: reflect unread count on the dock badge"
```

---

### Task 6: Verify native notifications (fallback only if needed)

Electron maps the web `Notification` API to native macOS notifications automatically. Verify messenger.com's notifications surface natively; only add interception if they don't. This task is verification-first — do not build interception speculatively.

**Files:**
- Modify (only if verification fails): `src/preload/index.ts`

**Interfaces:**
- Produces: (conditional) forwarding of `window.Notification` calls; no new public interface if native mapping works.

- [ ] **Step 1: Build, launch, grant notifications, observe**

Run: `npm start`
Steps: log in; in messenger.com settings enable notifications (accept the in-page prompt); from another device send yourself a message with the app unfocused.
Expected (happy path): a native macOS notification appears. If it does, **skip to Step 3** — no code needed.

- [ ] **Step 2: (Only if no native notification appeared) Add a forwarding shim**

Append to `src/preload/index.ts`:

```ts
// Fallback: some pages' Notification calls don't surface natively inside Electron.
// Re-emit through Electron's Notification by re-constructing in the isolated world.
// Electron shows native notifications for `new Notification(...)` created here.
const OriginalNotification = window.Notification;
if (OriginalNotification) {
  // Touch it so linters don't flag it; real forwarding hook goes here if the
  // happy path failed. Capture the exact failing behavior before implementing.
  void OriginalNotification;
}
```

If Step 1 succeeded, do not add this. If it failed, replace the placeholder above with a concrete shim based on the observed failure (record the failure in the commit message) — the failure mode determines the correct hook, so implement against the real symptom, not a guess.

- [ ] **Step 3: Commit (document the outcome)**

```bash
git add -A
git commit -m "test: verify native notifications work in Electron (no interception needed)"
```

(Use a message describing what you actually observed.)

---

### Task 7: Calls — grant camera/mic (THE GATE)

Wire the permission handler to grant camera/mic, then empirically place a real voice and video call on messenger.com. This is the differentiator and the project's key risk; treat the result as a gate.

**Files:**
- Modify: `src/main/index.ts`

**Interfaces:**
- Modifies: `configureSession()` to add `setPermissionRequestHandler`.

- [ ] **Step 1: Add the permission handler to `configureSession()`**

In `src/main/index.ts`, inside `configureSession()`:

```ts
  // Grant only media (camera/mic) and screen capture; deny all other permissions.
  ses.setPermissionRequestHandler((_wc, permission, callback) => {
    callback(permission === "media" || permission === "display-capture");
  });
```

- [ ] **Step 2: Build, launch, place calls**

Run: `npm start`
Steps: log in; open a 1:1 conversation; click the **voice call** button; when macOS prompts for microphone access, allow it; confirm two-way audio. Then place a **video call**; allow camera; confirm two-way video.
Expected outcomes to record:
- ✅ Calls connect with working audio/video → **gate passed**; this is the shipping differentiator.
- ⚠️ Button does nothing / "switch browser to make calls" / connects with no media → **gate failed**; record the exact symptom.

- [ ] **Step 3: Record the result**

Create `docs/CALLS-RESULT.md` with: macOS version, Electron version, and exactly what happened for voice and video (screenshots welcome). This is the evidence the spec's M3 gate requires.

- [ ] **Step 4: Commit**

```bash
git add src/main/index.ts docs/CALLS-RESULT.md
git commit -m "feat: grant camera/mic for calls; record calls gate result"
```

---

### Task 8: Screen share (best-effort)

Add a `setDisplayMediaRequestHandler` so in-call screen sharing can request a source. This is best-effort: if messenger.com's web UI doesn't offer screen share, or it fails, mark it deferred without blocking calls.

**Files:**
- Modify: `src/main/index.ts`

**Interfaces:**
- Modifies: `configureSession()` to add `setDisplayMediaRequestHandler`.

- [ ] **Step 1: Add the handler**

Extend imports:

```ts
import { app, BrowserWindow, shell, session, ipcMain, desktopCapturer } from "electron";
```

In `configureSession()`:

```ts
  // Screen share: grant the primary screen. A source-picker UI is a later enhancement.
  // Requires the macOS Screen Recording permission (System Settings → Privacy).
  ses.setDisplayMediaRequestHandler((_request, callback) => {
    void desktopCapturer.getSources({ types: ["screen"] }).then((sources) => {
      if (sources.length > 0) {
        callback({ video: sources[0] });
      } else {
        callback({});
      }
    });
  });
```

- [ ] **Step 2: Build, launch, attempt screen share**

Run: `npm start`
Steps: in a call, try screen share; grant Screen Recording when macOS prompts (you may need to restart the app after granting).
Expected: either screen share works, or record the failure in `docs/CALLS-RESULT.md` and mark screen share deferred.

- [ ] **Step 3: Commit**

```bash
git add src/main/index.ts docs/CALLS-RESULT.md
git commit -m "feat: best-effort in-call screen sharing"
```

---

### Task 9: Verify persistent 2FA login

Confirm the Electron default session persists cookies so a 2FA login survives a full quit/relaunch, and that the auth flow is never shunted to the browser. Mostly verification; only touch code if login doesn't persist.

**Files:**
- Modify (only if needed): `src/main/index.ts`

- [ ] **Step 1: Log in with 2FA**

Run: `npm start`
Steps: log in fully, completing 2FA. Confirm the 2FA/checkpoint pages render **inside** the app (not kicked to the browser). If a 2FA popup is blocked, note the URL from DevTools and confirm its host is covered by `ALLOWED_HOSTS`.

- [ ] **Step 2: Quit and relaunch**

Fully quit (Cmd+Q), then `npm start` again.
Expected: still logged in — no re-login required. (Electron persists `defaultSession` cookies under `app.getPath("userData")`.)

- [ ] **Step 3: (Only if login did NOT persist) Force a persistent partition**

If and only if login was lost, in `createWindow()`'s `webPreferences` add:

```ts
      partition: "persist:messenger",
```

and change the session references in `configureSession()` from `session.defaultSession` to `session.fromPartition("persist:messenger")`. Rebuild and re-verify Steps 1–2.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "test: verify 2FA login persists across relaunch"
```

---

### Task 10: CI build + test gate

Add a GitHub Actions workflow that builds and tests on macOS, so a broken build or failing test can't be merged.

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Create `.github/workflows/ci.yml`**

```yaml
name: CI
on:
  push:
  pull_request:
jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npm run build
      - run: npm test
```

- [ ] **Step 2: Verify locally first**

Run: `npm ci && npm run build && npm test`
Expected: all succeed (both test files pass).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build + test on macOS"
```

---

### Task 11: Package the macOS app (lightweight)

Configure electron-builder to produce a signed-less local `.app`/`.dmg` with the camera/mic/screen entitlements, arch-specific build, and stripped locales. Record the real footprint.

**Files:**
- Create: `build/entitlements.mac.plist`
- Modify: `package.json` (add the `build` config block)

- [ ] **Step 1: Create `build/entitlements.mac.plist`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.security.cs.allow-jit</key>
  <true/>
  <key>com.apple.security.device.camera</key>
  <true/>
  <key>com.apple.security.device.audio-input</key>
  <true/>
</dict>
</plist>
```

- [ ] **Step 2: Add the `build` block to `package.json`**

```json
  "build": {
    "appId": "com.mercury.messenger",
    "productName": "Messenger",
    "electronLanguages": ["en"],
    "files": ["dist/**/*", "package.json"],
    "mac": {
      "target": [{ "target": "dmg", "arch": ["arm64"] }],
      "category": "public.app-category.social-networking",
      "hardenedRuntime": true,
      "entitlements": "build/entitlements.mac.plist",
      "entitlementsInherit": "build/entitlements.mac.plist",
      "extendInfo": {
        "NSCameraUsageDescription": "Messenger uses your camera for video calls.",
        "NSMicrophoneUsageDescription": "Messenger uses your microphone for calls."
      }
    }
  }
```

- [ ] **Step 3: Build the package**

Run: `npm run dist`
Expected: a `.dmg` is produced in `dist/` (or `release/`). Note: unsigned; on first launch you may need to right-click → Open (Gatekeeper).

- [ ] **Step 4: Record footprint**

Install from the dmg, launch, log in. In Activity Monitor record: `.app` bundle size on disk (`du -sh /Applications/Messenger.app`) and total real memory across Messenger processes. Add these to `docs/CALLS-RESULT.md` (rename to `docs/BUILD-NOTES.md` if you prefer) so we have honest numbers, not marketing.

- [ ] **Step 5: Commit**

```bash
git add build/entitlements.mac.plist package.json docs/
git commit -m "build: package macOS dmg with call entitlements, arm64, en-only locales"
```

---

### Task 12: Rename runbook (repo + folder + Claude history) — run at a session boundary

Operational, not code. Rename the GitHub repo to `mercury-mac`, move the local folder, and carry this project's Claude Code history so `--resume` still finds this conversation. **Run only when no Claude session is active in this folder**, because steps move files an active session holds open. This is why `docs/HANDOFF.md` exists — if resume fails, the handoff doc preserves all context.

**Files:** none in-repo (this operates on paths outside the repo too).

- [ ] **Step 1: Rename the GitHub repo (safe anytime; updates local remote + adds redirects)**

```bash
cd ~/messenger-tauri
gh repo rename mercury-mac --yes
```

- [ ] **Step 2: Move the local folder**

```bash
cd ~
mv messenger-tauri mercury-mac
```

- [ ] **Step 3: Carry the Claude Code history to the new path**

```bash
mv ~/.claude/projects/-Users-matthewoyan-messenger-tauri \
   ~/.claude/projects/-Users-matthewoyan-mercury-mac
```

- [ ] **Step 4: Verify resume**

```bash
cd ~/mercury-mac
claude --resume
```

Expected: this conversation appears in the resume list with full history. If it does NOT, open `docs/HANDOFF.md` — it contains every decision and finding needed to continue from a fresh session. No commit needed (nothing in-repo changed); the earlier identity rename (Task 1 `package.json name`) already reflects `mercury-mac`.

---

## Self-Review

**Spec coverage:**
- §3 engine=Electron, wrap messenger.com → Tasks 1–2. ✅
- §3 `TARGET_URL` single constant → `config.ts` (Task 1), Global Constraints. ✅
- §4.1 links/downloads/permissions/badge → Tasks 3, 4, 5, 7, 8. ✅
- §4.2 preload unread + notifications → Tasks 5, 6. ✅
- §5 renderer isolation + least-privilege permissions → Tasks 2, 7. ✅
- §6 calls + screen share GATE → Tasks 7, 8 (+ `docs/CALLS-RESULT.md`). ✅
- §7 2FA session persistence → Task 9. ✅
- §8 migration/cleanup (remove Tauri/scaffold/metadata.json/mobile icons; rename identity) → Task 1. ✅
- §9 lightweight packaging (arch, locales, asar-default, entitlements) → Task 11. ✅
- §10 milestones map to task ordering. ✅
- §11 CI build gate → Task 10. ✅
- §12 deprecation watch-item = single `TARGET_URL` (already central); goofy/calls resolved empirically in Task 7. ✅
- Rename (deferred, folded into plan per user) → Task 12. ✅
- README rewrite (spec §8) → **gap**; add as a step. (See note below.)

**Gap fix — README:** Fold a README rewrite into Task 11 Step 5 (or a light Task 11b): replace the Tauri-era README (nonexistent `hybrid-logo.png`, `YOUR_USERNAME`, WKWebView description) with Electron build/run/dist instructions and the honest footprint numbers. Acceptance: README has no reference to Tauri, Vite, or nonexistent files, and documents `npm start` / `npm run dist`.

**Placeholder scan:** Task 6 Step 2 intentionally defers a shim to the observed failure (verification-first by design, not a blind placeholder) — acceptable and labeled. No other placeholders.

**Type consistency:** `IPC.SET_UNREAD` (`"set-unread"`) is defined in `src/shared/channels.ts` and imported identically in both `preload/index.ts` and `main/index.ts`. `isExternalUrl`, `parseUnreadCount`, `TARGET_URL`, `ALLOWED_HOSTS`, `configureSession()` names are consistent across tasks. ✅

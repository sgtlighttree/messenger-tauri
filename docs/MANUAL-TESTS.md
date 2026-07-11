# Manual Verification Checklist (MVP)

Every check below needs a real Facebook login, so agents couldn't run them — they were
deliberately batched here (owner's decision during execution). Run top-to-bottom; each item
says where to record the result. App-under-test: `npm start` for dev checks, or install
`release/Messenger-0.1.0-arm64.dmg` for the packaged checks.

## 0. Install & first launch (packaged build)

- [x] Open the dmg, drag Messenger.app to /Applications, launch via **right-click → Open**
      (ad-hoc signed; Gatekeeper blocks plain double-click on first run).
- [x] App opens and shows the messenger.com login page.

## 1. Security spot-check (dev or packaged)

- [ ] View → Toggle Developer Tools → Console: `typeof window.require` prints `"undefined"`
      (page has no Node access).

## 2. Login & session (plan Task 9)

- [x] Log in fully, completing **2FA**. The 2FA/checkpoint pages render *inside* the app —
      never kicked to the browser.
- [x] Quit fully (Cmd+Q), relaunch → **still logged in**.
- [ ] If login did NOT persist: note it — the fallback is a `persist:messenger` partition
      (plan Task 9 Step 3, not yet applied).
      UPDATE from Matt: Login persisted.

## 3. Links & downloads (plan Tasks 3–4)

- [ ] Click an external link shared in a chat → opens in your **default browser**, not the app.
- [x] Download an image/file attachment → lands in `~/Downloads`, no save dialog.

## 4. Dock badge (plan Task 5)

- [x] With unread conversations, the dock icon shows the unread count; reading clears it.
- [ ] If the badge never appears: open DevTools Console and record `document.title` with
      unreads pending (the parser expects a `"(N) Messenger"` shape) — file the actual string
      so the regex can be fixed.

## 5. Native notifications (plan Task 6 — verification-first, no code was written)

- [x] Enable notifications in messenger.com settings (accept the in-page prompt).
- [x] With the app unfocused, send yourself a message from another device → a **native macOS
      notification** appears.
- [ ] If NO notification: record exactly what happened (prompt never appeared / accepted but
      silent). A preload shim is the planned fallback (plan Task 6 Step 2) — implement against
      the observed symptom.
      UPDATE: Notifications are limited/near impossible in Electron with out janky workarounds

## 6. 🎯 THE CALLS GATE (plan Task 7 — the project's key unknown)

Fill **docs/CALLS-RESULT.md** as you go; it is the spec's M3 evidence file.

- [x] 1:1 **voice call**: mic permission prompt → allow → two-way audio.
- [x] 1:1 **video call**: camera prompt → allow → two-way video.
- [x] In-call **screen share** (best-effort): macOS Screen Recording permission may require an
      app restart after granting (System Settings → Privacy & Security → Screen Recording).
- [ ] If a call button produces NO permission prompt at all, note it explicitly — permission-check wiring (setPermissionCheckHandler) is the first suspect.
- [ ] Record the verdict line (PASSED / FAILED / PARTIAL). A failure here is a real result,
      not a setback to hide — it decides what we do next.

## 7. Footprint (plan Task 11)

- [ ] With the packaged app running and logged in: Activity Monitor → sum real memory across
      the Messenger processes → record in **docs/BUILD-NOTES.md** (replaces "PENDING").

## 8. Operational follow-ups (not app tests)

- [x] `git push -u origin feat/rebuild-v1` → confirm the GitHub Actions CI run goes green
      (first live validation of `.github/workflows/ci.yml`).
- [ ] When ready (at a session boundary, no active Claude session in this folder): run the
      **rename runbook** — plan Task 12 in
      `docs/superpowers/plans/2026-07-10-messenger-desktop-mvp.md` (repo → `mercury-mac`,
      folder move, Claude history carry, `claude --resume` verification).

## 9. 2026-07-11 follow-up fixes (Matt's HANDOFF notes)

Round-1 results (Matt): 1:1 downloads silent ✓, group-chat downloads flashed a popup (fix v2:
hide every allowed popup until it commits) — **round 2 verified: downloads fully fixed ✓**.
Focus steal failed in rounds 1–2; Matt's screenshot proved the ring UI is in-page and the title
becomes "<name> is calling" → attempt 3 is title-based. Remaining checks:

- [x] **Group-chat download**: verified by Matt 2026-07-12 — downloads work as intended, no flash.
- [-] **Dock bounce**: when a download completes, the Downloads stack in the dock bounces
      (new — `app.dock.downloadFinished`).
- [=] **Incoming call steals focus (attempt 3, title-based)**: with another app frontmost,
      receive a call → the app comes to the foreground while ringing. If it fails: open DevTools
      and record `document.title` during a ring (the detector matches "… is calling …" and
      "Incoming call" — an unexpected string is the likely cause).
- [ ] **Calls regression check**: one outgoing voice call still connects, and the call window
      appears promptly (a call window failing to appear within ~5s is a regression — report it).

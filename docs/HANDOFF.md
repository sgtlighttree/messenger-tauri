# HANDOFF — Messenger Desktop Rebuild (session of 2026-07-10/11)

Redundant record of everything decided and discovered in the planning session, so work can
continue from a fresh Claude Code session (or by another person) even if `claude --resume` fails
to carry the conversation after the repo/folder rename. Read this top-to-bottom to reconstruct full
context.

## Matt's (maintainer's) notes

> Matt's scratchpad and notes for things observed outside an active coding session. If an item is addressed, click the checkbox, and/or add a ~~strikethrough~~ for emphasis.

- [ ] Not all links open externally, links to Facebook posts actually spawn a new window,
      triggering a sign in prompt instead of signing in with the current signed in account.
- [ ] Icon for app inside `build/icon.png`, may need variants for macOS 26/Liquid Glass
- [x] ~~Downloads spawn a blank white window  (presumably `about:blank`), but otherwise works.~~
      *Fix v2 2026-07-11: v1 only hid `about:blank` popups → Matt's retest found group-chat
      downloads still flashed (they open with a REAL fbcdn/fbsbx URL, then that navigation
      becomes the download). Now EVERY allowed popup is created hidden and revealed only on its
      first committed navigation, so both shapes download silently. Also added the browser-style
      dock bounce on completion (`app.dock.downloadFinished`). **Verified fixed by Matt
      2026-07-12.***
- [x] ~~Calls made to the account/app do not steal window focus, it should.~~
      *Attempts 1+2 (popup watchers) failed — Matt's screenshot (2026-07-12) proved the ring UI
      is IN-PAGE (dialog in the main window; no popup until accepted), so no popup trigger can
      fire. But the screenshot also showed the page title becomes "<name> is calling" during the
      ring. Attempt 3 (current): the preload's existing title observer detects that
      (`isIncomingCallTitle`, src/preload/incoming-call.ts), edge-triggered with a 5s re-arm so
      a blinking title can't spam, and IPC `INCOMING_CALL` makes main show + `app.focus({steal:
      true})`. Cross-checked 2026-07-12 via parallel web research (Sonnet + agy, agreeing):
      "[Name] is calling(…/you)" is the known ring title; the pattern set mirrors
      apotenza92/facebook-messenger-desktop's incoming-call-evidence.ts (the only wrapper found
      doing this — Caprine resets the title and can't); localized variants + title blinking
      remain unconfirmed (detector is en-only, blink-proof by design). **Verified working by
      Matt 2026-07-12.***
- [x] ~~Optional: In dark mode, about:blank is white, then the Messenger splash screen itself is white until the full app loads. If possible, there should be a splash screen that lives outside of `messenger.com` that follows the current OS light/dark mode~~
      *Done 2026-07-11: native theme-aware splash window (`src/main/splash.ts`, data-URL HTML,
      `prefers-color-scheme`) covers the load; the main window stays hidden until
      `did-finish-load` (15s cap) and all windows get a `nativeTheme`-matched `backgroundColor`.
      The preload also pins the page background dark in dark mode (best-effort — Messenger's own
      white splash *artwork* can't be restyled from outside, only its backdrop).*

## Notes for Matt

**State (end of 2026-07-11 session):** MVP done and merged to `main`. Calls ✅ (voice+video,
both directions, dev+packaged). Screen share ✅ (whole screen only). Badge ✅ (with anti-blink
stabilizer). Session/2FA ✅. Window size persists ✅ (shared between `npm start` and the
packaged app). Notifications ❌ — documented limitation, see the Notifications section below;
badge+sound is the chosen behavior.

**Build the real app** (signed — required, or notifications registration and TCC persistence
break, and hardened runtime won't load):
```bash
CSC_NAME="Mercury Dev" npm run dist   # → release/Messenger-0.1.0-arm64.dmg
```
The "Mercury Dev" self-signed cert lives in your login keychain (trusted for code signing).
Plain `npm run dist` without CSC_NAME produces an ad-hoc build that LAUNCHES but can't register
for notifications and resets TCC grants every rebuild — don't use it.

**The rename (run when NO Claude session is active in this folder):**
```bash
cd ~/messenger-tauri && gh repo rename mercury-mac --yes   # GitHub + remote + redirects
cd ~ && mv messenger-tauri mercury-mac                     # local folder
mv ~/.claude/projects/-Users-matthewoyan-messenger-tauri \
   ~/.claude/projects/-Users-matthewoyan-mercury-mac       # Claude history + memory follow
cd ~/mercury-mac && claude --resume                        # verify this conversation appears
```
If `--resume` doesn't show the conversation, this HANDOFF.md + the auto-loaded memory files
carry everything. Package/crate identity inside the repo is already `mercury-mac`.

**Where things live:** spec+plan `docs/superpowers/`; calls evidence `docs/CALLS-RESULT.md`;
manual checklist `docs/MANUAL-TESTS.md`; build facts `docs/BUILD-NOTES.md`; follow-up list
below (badge/notification enrichment, screen-share picker, will-redirect, etc.).

## Notifications: documented limitation (2026-07-11, evidence-based)

**messenger.com delivers desktop notifications exclusively via Web Push → service-worker
`showNotification`. Electron has no push service** (`AbortError: Registration failed — push
service not available`, electron#13041; Chrome ships Google's push backend, Electron doesn't),
so the push never arrives and no notification can be surfaced. Verified empirically in three
rounds: (1) OS layer proven working (signed app + main-process test notification appeared,
app registered in System Settings); (2) page-`Notification` interception shim installed at
dom-ready — nothing; (3) shim moved to preload/main-world (installs before any page script,
eliminating the capture race) — still nothing while the in-page sound played and Vivaldi (real
Chrome push) delivered the same message. Caprine has the same rot (multiple open "no
notifications" issues 2024–2026; their interceptor is the same design). Push-emulation libs
(electron-push-receiver et al) are FCM-project-scoped and cannot receive Facebook's encrypted
push — dead end.

**Owner's decision: accept badge + in-page sound; NO synthetic generic banner.** The dock badge
has an anti-blink stabilizer (rises instant, zero only clears after ~2.5s stable) so a quick
dock-peek is trustworthy. If richer notifications are ever wanted, the follow-up path is
synthesizing from the unread-count rise + conversation-list DOM scrape (sender/preview) — the
notification bridge (preload shim → IPC → native Notification, GC-hardened per electron#16922)
is already in place and tested; only the trigger source is missing.

## TL;DR

Reviving an abandoned Tauri prototype (a macOS wrapper for Facebook Messenger) into a serious,
lightweight **Electron** desktop client. Goal: native-feeling daily driver with persistent 2FA
login, native notifications + dock badge, sane links/downloads, and — the differentiator —
**working voice/video calls** (and screen share if possible). Personal use until reliable.

- **Spec:** `docs/superpowers/specs/2026-07-10-messenger-desktop-rebuild-design.md`
- **Plan:** `docs/superpowers/plans/2026-07-10-messenger-desktop-mvp.md`
- **Branch:** `feat/rebuild-v1`
- **Next action:** execute the plan (Task 1 onward). Rename is deferred to plan Task 12.

## The one decision that drove everything: engine = Electron, not Tauri

The prototype used Tauri (WKWebView = Apple's WebKit engine), chosen for its tiny footprint. We
changed to **Electron (Chromium)** for exactly one reason: **calls.**

**Verified finding (multi-source, primary where possible):** Facebook Messenger's voice/video
calls — now end-to-end encrypted by default (since 2024) — depend on the **Chrome-only
`createEncodedStreams()` (WebRTC Insertable Streams) API, which WebKit/WKWebView does not
implement.** So a WebKit engine physically cannot run Messenger's call path.

Evidence, corroborated across four independent angles:
1. **Mozilla Bugzilla 1896361** (primary): "Possibly due to the use of non-standard
   createEncodedStreams"; notes Meta could support Firefox/Safari via the now-standard
   `RTCRtpScriptTransform` but hasn't migrated.
2. **webrtcHacks / WebRTC compat guides**: Insertable Streams "supported in Chrome but still
   missing in Safari".
3. **Meta's own help page**: lists supported call browsers as Chromium-only (Chrome/Edge/Opera);
   Safari is on the unsupported list.
4. **goofy's issue history** (a maintained ~5MB WebKit Messenger wrapper): 2015 issues "Switch
   browser to make calls" (Messenger's own message) and "Use chrome(ium) webview instead of
   Safari", plus "Cannot make video and Audio call" (2020). Real-world confirmation WebKit can't
   call.

**Consequence:** Tauri is a hard no for calls. Electron's Chromium is **necessary**. It may not be
**sufficient** — Caprine and messenger-mac are Electron yet have broken calls right now — so a
wrapper still must wire camera/mic/screen permissions correctly and target the live URL. Therefore
**calls are a GATE to verify empirically in our own build (plan Task 7), not a guarantee.**

## Findings that were tested and corrected (intellectual honesty log)

- **Uploads in WebKit are FINE.** Early research predicted large media uploads would be flaky in
  WebKit. Empirical test (owner's Mac): Safari sent **20MB and 70MB videos** to a 1:1 chat
  successfully; only an HTML file stalled (rare, likely security-special-cased, and it worked in
  Chromium/Vivaldi). goofy (WebKit) also uploads HTML+PDF fine. **Uploads are NOT a reason to leave
  WebKit** — struck from the Electron rationale. Calls are the sole reason.
- **RAM: the website dominates, not the shell.** goofy Activity Monitor: the `facebook.com`
  WebContent process was ~472MB real; goofy's own shell processes were small. Confirms the
  Tauri-vs-Electron RAM gap is smaller than the headline "10x" — Electron overhead is additive to
  the same heavy page, not a multiplier. goofy bundle is 4.8MB (native Swift + WKWebView).
- **messenger.com is NOT dead (target stays messenger.com).** Secondary news (Feb 2026) reported
  messenger.com shutting down 2026-04-15 → facebook.com/messages. But Meta's **own current help
  article** (`facebook.com/help/messenger-app/804132271957789`) directs desktop users to
  **messenger.com**, and the site returns HTTP 200 and works today (mid-2026). Primary/live source
  beats secondary reporting. **Target = messenger.com**, kept as a single `TARGET_URL` constant so
  flipping to `facebook.com/messages` is a one-line change if the redirect ever lands. Watch-item,
  not current reality. (goofy loads facebook.com — mild evidence that surface also works.)

## Competitive landscape (why build our own)

- **Caprine** (`sindresorhus/caprine`): Electron, maintenance-only, ~110MB/~570MB RAM. **Calls
  broken** (issue #2393, May 2026). Hardwired to messenger.com.
- **messenger-mac** (`stefanminch/messenger-mac`): Electron, stale (~Dec 2025). **2FA + calls
  broken** per issues.
- **goofy** (`danielbuechele/goofy`): ~5MB native WebKit wrapper, maintained (last commit May
  2026, 983★). Great lightweight shell; **calls can't work (WebKit)**. Owner couldn't live-test
  calls yet (logistics) but issue history confirms the limitation.
- **Our niche:** lightweight-for-Electron client targeting live messenger.com that treats
  **calls, screen share, and 2FA login as first-class, correctly-wired features** — the exact
  things incumbents get wrong.

## Scope for v1 (owner's choices)

Must-haves confirmed: **persistent 2FA login, native notifications + dock badge, external links →
browser, downloads → ~/Downloads.** Stretch/differentiator: **calls + screen share** (gated).
Deferred polish: tray/menu-bar, close-to-background, launch-at-login.

## Honest footprint expectation

Electron floors ~90–130MB disk / ~300–450MB RAM running Messenger — the Chromium bytes ARE the call
capability and can't be removed. "Lightweight" here = best-in-class-for-Electron (single window,
arm64-only build, `en`-only locales, latest Electron), NOT Tauri numbers. Record real numbers; don't
market unverified ones.

## Repo state at handoff

- Branch `feat/rebuild-v1` off `main`. Commits: CLAUDE.md; the Tauri-era spec (superseded); the
  Electron spec; the MVP plan + this handoff; **then the full MVP implementation (2026-07-11,
  subagent-driven): Tauri removed, Electron 43 scaffold, hardened window (sandbox:true, bundled
  preload), links/downloads, unread dock badge, calls/screen-share permission handlers,
  CALLS-RESULT.md template, CI workflow, packaged arm64 dmg (release/), README+CLAUDE.md rewritten.**
- All automated checks green (11/11 unit tests, build, packaging). Every login-dependent check is
  batched in `docs/MANUAL-TESTS.md` — **none run yet**, including the calls gate.
- GitHub remote: `github.com/sgtlighttree/messenger-tauri` (rename to `mercury-mac` deferred to plan
  Task 12; branch not yet pushed).

## Rename runbook (deferred — plan Task 12, run at a session boundary)

New name chosen: **`mercury-mac`** (Mercury = messenger god + quicksilver/lightweight). Run only when
no Claude session is active in the folder:

```bash
cd ~/messenger-tauri && gh repo rename mercury-mac --yes      # updates remote + adds redirects
cd ~ && mv messenger-tauri mercury-mac                        # local folder
mv ~/.claude/projects/-Users-matthewoyan-messenger-tauri \
   ~/.claude/projects/-Users-matthewoyan-mercury-mac          # carry Claude history
cd ~/mercury-mac && claude --resume                           # verify this conversation follows
```

If `--resume` doesn't show this conversation, this HANDOFF.md is the fallback context.

## 🎉 CALLS GATE: PASSED (2026-07-11)

Voice + video calls work **bidirectionally** (made and received) in both `npm start` and the
packaged dmg. Screen share works (whole-screen only). Full evidence + the three stacked root
causes (about:blank popup denial → web-perms fine → TCC launcher-identity) in
`docs/CALLS-RESULT.md`. The project's differentiator — the thing Caprine/messenger-mac/goofy
all fail at — is empirically real. Key fixes: 3275a01, c252dfc, cb602bd.

Dev-run gotcha worth remembering: `npm start` inherits the **launcher's** TCC identity (VS Code
/ Terminal), so mic grants go to that app; the packaged app has its own identity. A self-signed
cert ("Mercury Dev", already created in Keychain) can be used via `CSC_NAME="Mercury Dev" npm run
dist` for stable signatures so TCC grants survive rebuilds.

## Known follow-ups (non-blocking, triaged by the final branch review 2026-07-11)

All fail safe, are unreachable in practice, or are cosmetic. None block merge; groom opportunistically:

1. Trailing-dot hostname (`facebook.com.`) classified external — fails safe (opens in browser).
2. Tab/newline-in-scheme URLs dropped rather than opened — fails safe.
3. `Infinity` would pass the badge-count guard — unreachable (only our preload sends, regex can't emit it).
4. No debounce on the title MutationObserver — cheap idempotent IPC.
5. CI pins Node 20 vs local Node 24 — bump when convenient.
6. esbuild `^0.24` pin is oldish — bump opportunistically.
7. Downloads overwrite silently on duplicate filenames — add a counter suffix if path exists.
8. `will-redirect` not intercepted (server-side 3xx to a non-allowlisted host would load in-window;
   low risk since Meta auth redirects stay in-allowlist) — add a handler reusing `isExternalUrl`.
9. **Screen share has no source picker** — shares the entire primary screen immediately (MVP
   choice). Chromium browsers offer tab/window/screen selection; Electron needs a small custom
   picker UI over `desktopCapturer.getSources({types:["screen","window"]})`. Best UX follow-up.
10. Sign builds with the "Mercury Dev" self-signed cert (`CSC_NAME="Mercury Dev" npm run dist`)
    so the signature — and TCC permission grants — stay stable across rebuilds.
11. README's "right-click → Open" note is slightly misleading: Gatekeeper only affects
    *downloaded* copies (quarantine xattr); locally built dmgs never trigger it (observed).

## Open questions to resolve during/after build

1. **Calls on our Electron build** — the Task 7 gate. Root cause of incumbents' broken calls is
   unconfirmed (fixable permission wiring vs Meta gating). Resolve empirically; record in
   `docs/CALLS-RESULT.md`.
2. **Screen share reachability** — does messenger.com web even expose in-call screen share, or only
   native/mobile? (Task 8.)
3. **Distribution** — personal-only (skip signing/notarization) vs shared (then notarization +
   entitlements interaction becomes required).
4. **Notifications** — verify Electron's native `Notification` mapping works for messenger.com
   before building any interception (Task 6).
5. **messenger.com deprecation** — watch for an actual redirect to facebook.com/messages; flip
   `TARGET_URL` if it lands.

## Method note

Engine/upload/deprecation/Caprine/goofy claims were cross-checked using parallel research (Claude
WebSearch/WebFetch + independent `agy`/Gemini pass), corroborating each load-bearing claim across ≥2
independent domains and going to primary sources (Bugzilla, GitHub issues, Meta help) for the
decisive ones. Several `agy` citations were discarded as tangential/parametric (e.g. a "messenger.com
phased out April 2026" claim that Meta's own live help article contradicts).

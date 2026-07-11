# HANDOFF — Messenger Desktop Rebuild (session of 2026-07-10/11)

Redundant record of everything decided and discovered in the planning session, so work can
continue from a fresh Claude Code session (or by another person) even if `claude --resume` fails
to carry the conversation after the repo/folder rename. Read this top-to-bottom to reconstruct full
context.

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

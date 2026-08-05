# Facebook content links open in the browser, not an in-app popup

**Status: implemented 2026-07-27. Pending manual verification (see below).**

> **This is a LIVE design doc, not frozen archive.** CLAUDE.md notes that everything under
> `docs/superpowers/` is a historical record left as-written — that rule refers to the
> 2026-07-10 spec/plan pair from the original MVP build. This file post-dates the rename and
> the pnpm migration, and its commands and behavior describe the app as it stands today.

## The problem

From Matt's notes in `docs/HANDOFF.md`:

> Not all links open externally, links to Facebook posts actually spawn a new window,
> triggering a sign in prompt instead of signing in with the current signed in account.

`facebook.com` is in `ALLOWED_HOSTS` (`src/main/config.ts`), so `isExternalUrl()` returns
`false` for a Facebook post link, `decideWindowOpen()` returns `"allow"`, and the link opens as
an in-app popup. That popup has no facebook.com session — the login cookies are scoped to
messenger.com — so it renders a sign-in prompt instead of the post.

`facebook.com` cannot simply be removed from `ALLOWED_HOSTS`: login/2FA/checkpoint redirects go
through facebook.com and must render inside the app (`docs/MANUAL-TESTS.md` §2, verified
passing). Removing the host would break the app's single most important behavior.

## The observation that decided the design

Two possible shapes had to be distinguished, because this codebase has already been surprised
once by exactly this question (1:1 downloads arrive as `about:blank`, group-chat downloads
arrive with a real URL — see the comment in `src/main/links.ts`):

1. the link arrives at `setWindowOpenHandler` already carrying a `facebook.com` URL, or
2. it arrives as `about:blank` and navigates to facebook.com afterward.

Temporary diagnostic logging in `wireNavigationGuards`, run by Matt on 2026-07-27, settled it:

```
[diag main] window-open url=https://www.facebook.com/share/r/1ETfmkj2u4/ frameName=  -> allow
[diag popup] will-navigate url=https://www.facebook.com/share/r/1ETfmkj2u4/ external=false
```

**Shape 1.** The URL is present at window-open time, so the fix belongs in `decideWindowOpen`
and fires before the popup is ever created.

The observed URL is `/share/r/<id>/` — Facebook's share-redirect form, **not** `/posts/` or
`/permalink.php`. This is why the fix is a **host** rule rather than a path rule: a denylist of
"content paths" would have missed the very first real-world sample.

## The change

`POPUP_EXTERNAL_HOSTS = ["facebook.com"]` in `src/main/config.ts`, alongside `ALLOWED_HOSTS`
(that file is the single source of truth for host policy). A new pure predicate
`isPopupExternalHost()` in `src/main/links.ts` matches the host or any subdomain of it.
`decideWindowOpen()` consults it and returns `"open-external"`, which `src/main/index.ts`
already routes to `shell.openExternal` — no change to `index.ts` was needed.

`isExternalUrl`, `isHttpUrl`, `ALLOWED_HOSTS`, and the `will-navigate` handler are all
untouched. In-place navigation to facebook.com still stays in-app, which is what preserves
login and 2FA.

## Why this cannot affect the calls gate

Ordering inside `decideWindowOpen` is load-bearing and is now documented in the function's
doc comment:

1. `about:blank` → `"allow"` (**first**) — voice/video call popups and 1:1 download popups.
2. `isPopupExternalHost` → `"open-external"` (**second**) — Facebook content links.
3. existing `isExternalUrl` logic.

Call popups arrive as `about:blank` (`docs/CALLS-RESULT.md`: denying that popup was the
original round-1 gate failure), so they return at step 1 and **never reach the new branch**.
Whatever host a call popup navigates to afterward is governed by `will-navigate`, which this
change does not modify. Group-chat download popups carry `fbcdn.net`/`fbsbx.com` URLs, which
are not `facebook.com`.

Three regression tests pin this explicitly, named so a future reorder fails loudly:
`about:blank` still allowed, `fbsbx.com` still allowed, `fbcdn.net` still allowed.

## Known limitation (accepted, not solved)

If Meta ever delivers an auth/checkpoint step as a **popup** to facebook.com rather than an
in-place navigation, this rule would send it to the browser and break that login step. No
evidence of that shape exists — 2FA was verified working with in-place navigation — and
guarding against it would mean reintroducing exactly the fragile auth-path matching this design
rejected. The login checks in `docs/MANUAL-TESTS.md` §2 are the detector if it ever happens.

## Verification

`pnpm test` — 48 tests pass, 11 new in `tests/links.test.ts`.

**Post-fix diagnostic run (2026-07-27), same link ID as the failing run:**

```
[diag main] window-open url=https://www.facebook.com/share/r/1ETfmkj2u4/ frameName= -> open-external
[diag main] window-open url=about:blank frameName= -> allow
[diag popup] will-navigate url=https://www.messenger.com/groupcall/ROOM:1362014242571258/?…&is_e2ee_mandated=true external=false
[diag main] window-open url=http://www.matthewoyan.com/ frameName= -> open-external
```

What this establishes:

1. **The bug is fixed.** The same URL that returned `allow` before now returns `open-external`,
   and **no `will-navigate` line follows it** — the popup is never created.
2. **The calls path is intact, and the safety argument is now empirical rather than inferred.**
   The call popup still takes the `about:blank` branch, and it navigates to a **messenger.com**
   `groupcall` URL. `POPUP_EXTERNAL_HOSTS` contains only facebook.com, so it cannot reach the
   call path even in principle. This had previously been assumed, not observed.
3. Ordinary external links still externalize correctly.

The temporary `[diag …]` logging has been removed; the observed call-URL shape is preserved as
a comment on the `will-navigate` handler in `src/main/index.ts`.

**Still owed by a human** — the log cannot show these:

- Two-way audio on a voice call (`docs/MANUAL-TESTS.md` §9 asks that the call *connects*, not
  merely that its window opens). The log proves the call window was created and navigated.
- A fresh login / 2FA cycle (§2), the detector for the accepted limitation above. Session
  persistence is evidenced circumstantially — chat links were clickable, so the session was
  live — but no re-authentication was exercised.

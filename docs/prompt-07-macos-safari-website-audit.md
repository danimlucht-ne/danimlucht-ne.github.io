# Prompt 7 — macOS / Safari audit of the three websites

Send to all three app sessions (bar-snap, trope-lit, play-place-finder). Each session does only
its own app's section plus the shared parts. Paste everything below the line.

---

You are auditing your app's **website** for things that break on a Mac. "On a Mac" means
**Safari on macOS** — a different rendering and JavaScript engine from Chrome, with different
defaults for popups, cookies, and cross-site tracking. It also means desktop viewport sizes,
not phone ones.

This is a check-then-fix task. Some of what follows is a confirmed defect; some is a
"look at this and tell me". They are labelled. **Do not fix what is already correct** — a
meaningful part of this prompt is a list of things I checked and found clean, so you can skip
them instead of rediscovering them.

## Standing rules for this workstream

**Verify every claim below before acting on it.** Line numbers were read on 2026-08-10 against
`origin/main`: bar-snap `5134ac1`, trope-lit `fe1e2a8`, play-place-finder `d1c5e72`. They drift.
**Do not assert the result of a search you did not run.** If a premise here is wrong, say so and
stop rather than building on it — an earlier prompt in this workstream shipped a false premise
and the session that pushed back was right to.

Where I say "verified" I ran the search and read the code. Where I say "inferred" or "unverified"
I did not, and you must check before acting.

---

## Part 1 — The structural gap (all three apps)

**Verified.** None of you test Desktop Safari except Play Spotter:

| App | WebKit projects | Viewport | Which specs |
|---|---|---|---|
| Bar Snap | `layout-mobile-webkit` | iPhone 12 | `layout/*.spec.ts` only |
| TropeLit | `mobile-safari` | iPhone 13 | `-responsive.spec.ts` only |
| Play Spotter | `webkit` + `mobile-webkit` | Desktop Safari + 375px | (check your config for scope) |

Bar Snap and TropeLit both run WebKit **only at a phone viewport, only on layout specs**. Both
config files carry a comment explaining that WebKit-at-phone-size was added because mobile layout
bugs kept reaching production. That reasoning was right and the coverage worked. It just does not
cover a person sitting at a MacBook, which is a different viewport *and* a different set of
engine behaviours (popups, storage, cross-site tracking).

**Bar Snap and TropeLit:** add a Desktop Safari project (`devices['Desktop Safari']`). Scope it
deliberately — you do not need all specs twice. Cover at minimum: sign-in, the advertise/checkout
flow, and any page carrying an ad or sponsor slot. Say in your PR what you scoped it to and why.

**Play Spotter:** you already have one. Report what it currently runs, and whether sign-in and
checkout are inside that scope or outside it.

---

## Part 2 — Google sign-in on Safari (app-specific; Bar Snap this is your priority)

All three of you use Firebase `signInWithPopup` for Google sign-in. You have solved its
Safari/COOP exposure three different ways, and one of you has not solved it at all.

**Verified, TropeLit — the reference.** `website/next.config.ts` does two things:
- sets `Cross-Origin-Opener-Policy: same-origin-allow-popups` on `/:path*`, with a comment
  stating that Vercel's default `same-origin` blocks the `window.closed` check `signInWithPopup`
  relies on;
- rewrites `/__/auth/:path*` to `trope-lit.firebaseapp.com`, so the auth handler is served from
  `tropelit.com` and the auth domain is first-party.

That second part is what matters most on a Mac: Safari ships **"Prevent cross-site tracking" on
by default**, so storage for a third-party `*.firebaseapp.com` auth domain is partitioned or
blocked. Serving auth from your own domain sidesteps it.

**Verified, Play Spotter — defensive but partial.** `app/lib/firebaseWebAuth.mjs:70-77` and
`:89-96` catch `auth/popup-blocked` and fall back to `signInWithRedirect`. Good instinct.
Two things to check yourself: (a) I found **no COOP header** in your `next.config`, and the COOP
failure mode does **not** raise `auth/popup-blocked` — it typically hangs — so your fallback
would not catch it; (b) whether your `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` is first-party or
`*.firebaseapp.com`. I could not check (b): it is an environment variable, not code.

**Verified, Bar Snap — no mitigation at all.** This is the highest-value item in this prompt.
- `website/src/lib/auth-context.tsx:127` calls `signInWithPopup`.
- It is reachable in the UI: "Continue with Google" at `website/src/app/login/page.tsx:198`.
- `website/next.config.js` is 17 lines and contains **only** a `/media/uploads/:path*` rewrite —
  no `headers()`, no COOP, no `/__/auth` proxy. I also found no `vercel.json` and no
  `Cross-Origin-Opener-Policy` string anywhere in the repo.
- You deploy to Vercel (`.github/workflows/deployToVercel.yml`).

So on Bar Snap, Google sign-in has no COOP override, no first-party auth domain, and no redirect
fallback. **Unverified and you must check first:** I did not confirm Vercel's default COOP header
myself — that claim is TropeLit's code comment, not my observation. Before changing anything,
check the actual response headers on the deployed site. If COOP is not `same-origin` there, the
COOP half of this is moot and you should say so; the Safari cross-site-tracking half still stands
on its own.

Note this is not purely a Mac issue — COOP affects every browser. It lands here because Safari is
where it compounds: stricter popup heuristics *and* cross-site tracking prevention on by default.

**Do not** copy TropeLit's config blindly. The `/__/auth` rewrite requires
`NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` set to your own domain in Vercel and that domain added to
Firebase Console → Authentication → Authorized domains. Those are deployment changes, not code
changes. If you cannot make them, say so and implement the redirect fallback instead — and flag
the env/console work as needing a human.

---

## Part 3 — Play Spotter only: unprefixed `backdrop-filter`

**Verified.** `website/app/globals.css` has four `backdrop-filter` declarations and only two
`-webkit-backdrop-filter` companions:

| Line | Prefixed? |
|---|---|
| `:95` | yes, `:96` |
| `:2781` | **no** |
| `:3266` | **no** |
| `:7309` | yes, `:7310` |

Safari only supported unprefixed `backdrop-filter` from **Safari 18**; before that the
`-webkit-` form is the only one that works. On an older-Safari Mac, the two unprefixed rules
produce no blur at all — whatever sits behind those surfaces shows through at full contrast,
which is a legibility problem, not a cosmetic one.

That the same file prefixes the other two is the tell: the prefix is the house pattern and these
two were missed. Add `-webkit-backdrop-filter` alongside both. Check whether any others have
crept in since I read the file.

**Bar Snap:** you use Tailwind's `backdrop-blur` utility rather than raw CSS (e.g.
`RecipeImageCarousel.tsx:89`). Tailwind emits the prefixed form itself, so I believe you are
fine — **but I did not verify your build output.** Confirm it once and move on.

**TropeLit:** no `backdrop-filter` anywhere. Nothing to do.

---

## Part 4 — Eyes-on, no change specified

These are not defects. I am asking you to look, on a real Mac or in Desktop Safari, and report.

1. **Horizontal scroll containers with no affordance.** Bar Snap has 15 `overflow-x:auto|scroll`
   containers, Play Spotter 9, TropeLit 0 (all verified counts). macOS hides scrollbars by
   default until a scroll gesture starts, so a container that scrolls sideways looks like a
   container that is simply cut off. Check the ones carrying real content — filter rows, tag
   strips, table-ish layouts — and say whether a Mac user can tell there is more to see.

2. **Bar Snap: the sticky header carries the presenting sponsor.**
   `NavigationShell.tsx:61` is `sticky top-0 z-50`, and `:147` mounts the presenting sponsor
   inside it with the comment "so it does not scroll away". `position: sticky` fails **silently**
   if any ancestor has `overflow: hidden|auto|scroll`. If it fails, the national sponsor scrolls
   off — which is Rule 1 broken. Verify it actually sticks in Desktop Safari at a tall viewport.

3. **Emoji and font metrics.** Both Bar Snap's and TropeLit's Playwright configs already document
   that Apple Color Emoji is wider and taller than Noto and that WebKit line boxes differ. That
   reasoning applies at desktop widths too, in places phone-viewport specs never render — wide
   nav rows, multi-column grids. Worth a look, not a code change.

---

## Part 5 — Verified clean; do not spend time here

I ran these across all three websites and found nothing. Skip them.

- **Regex lookbehind** (`(?<=`, `(?<!`) — none anywhere. This one matters because Safari below
  16.4 throws at *parse* time, taking the whole bundle down, so it is worth knowing it is absent.
- **Late-baseline JS**: `crypto.randomUUID`, `structuredClone`, `Object.groupBy`,
  `Promise.withResolvers`, `.toSorted(`, `.toReversed(`, `replaceAll(` — zero hits in all three.
  The JS baseline is conservative.
- **`-webkit-line-clamp`** — every occurrence in TropeLit and Play Spotter is correctly paired
  with `display: -webkit-box` and `-webkit-box-orient: vertical`. TropeLit's `globals.css:247`
  even documents why the standard `line-clamp` is deliberately *not* added alongside. Bar Snap
  has none in CSS.
- **Ad click-through popups.** All three open the ad destination synchronously inside the click
  handler, with impression/click tracking fire-and-forget rather than awaited
  (`bar-snap/src/lib/adTracking.ts:39-49`, `play-place-finder/app/lib/adTracking.js:64-74`,
  `trope-lit/src/components/ads/AdCard.tsx:70-80`). Safari blocks `window.open` only when the
  user gesture has been consumed by an `await` first. None of you do that, so ad clicks are safe.
  This one was worth checking precisely because breaking it would mean advertisers paying for
  clicks that never land.
- **Bar Snap `grid-rows-subgrid`** (`NearbyResultCard.tsx:81`, `NearbyExplorer.tsx:723`) — Safari
  has supported subgrid since 16, ahead of Chrome. Not a Mac risk.

---

## Report back

Per app: what you fixed, what you checked and found already correct, and anything above that did
not match the code you actually read. If you think one of my "verified clean" items is wrong,
say so with the search you ran — I would rather be corrected than have you work around me
quietly.

If something needs a deployment or Firebase Console change rather than a code change, do not
attempt it. Name it, say what it needs, and leave it for a human.

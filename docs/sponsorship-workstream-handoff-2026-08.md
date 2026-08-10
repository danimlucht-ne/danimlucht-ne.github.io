# Sponsorship & Advertising Workstream — Handoff

**Date:** 2026-08-10
**Scope:** Bar Snap, TropeLit, Play Spotter (play-place-finder). PlayBound is untouched by this workstream.
**Nature of the work:** coordination. Nothing was changed in the app repos from this session. The
output is a set of prompts dispatched to per-app sessions, which do the actual editing.

---

## 1. The three product rules

These came from the product owner and are invariants, not preferences. Every prompt in this
workstream enforces them.

**Rule 1 — The national/presenting sponsor lives in the header, not in a banner.**
Not a second banner below the state sponsor. Site chrome: a thin strip reading
`Sponsored by <Brand>` with the partner's logo, sitewide on web, and in the app's top bar.
This was always the intended design for national partners across all apps.

**Rule 2 — Ad-free removes everything except the national/presenting sponsor.**
Category, ingredient, seasonal, genre, state — all hideable. The national sponsor is never
hidden. This is already promised in the paid-tier copy, so it is a correctness matter, not a
design choice.

**Rule 3 — Never two ads stacked on one another, even banners.**
Adjacency is judged at *runtime*, not in source order. A conditional section that renders
empty collapses two distant ads into neighbours. Any check for this must be done signed-out
with optional sections empty, which is the state where collapse actually happens.

**Corollary the product owner supplied:** national sponsors do not buy impressions or clicks —
a header strip has effectively no click behaviour — so impression/click liability arguments do
not apply to Rule 1. Two of my earlier objections were retracted on this basis and should not
be re-raised.

---

## 2. Reference implementation

Play Spotter (`play-place-finder`) has the shipped version of the header format. Use it as the
model, but **not** as a clean reference — it has its own defects, listed in §4.

- `website/app/components/NationalSponsorStrip.js` — the strip. Resolves
  `placementType: 'HEADER_PRESENTED_BY'`, returns null when the row is missing or
  `sourceType === 'house'` (so no house fallback in the header), renders
  label + logo + partner name inside a single link. Impression tracked with a plain ref;
  no session storage needed, because the app-router shell persists across client navigation.
- `website/app/components/SiteNav.js:80` — mounted immediately after `</nav>`.
- `android/.../ui/composables/PresentedByStrip.kt:32` — `PRESENTED_BY_PLACEMENT = "HEADER_PRESENTED_BY"`.
- `server/src/services/campaignLifecycleService.js:203-213` — `national_partner` maps to
  `placementTypes: ['HEADER_PRESENTED_BY']`, `isExclusive: true`, `global: true`.
  `HEADER_PRESENTED_BY` is a first-class placement enum, not a bolt-on flag.

**Quote-terminus intake** (also Play Spotter, also the model): a national campaign goes through
the *same* self-serve builder — business info, logo, creative — but the final step requests a
quote instead of taking payment. An administrator then issues a priced Stripe Checkout Session
carrying the terms URL. Availability is checked *before* quoting (409 on conflict), not at
signature time. See `server/src/routes/adminRoutes.js:3050`
(`POST /advertiser-leads/:id/send-quote`) and `assertQuoteWindowLooksAvailable` at `:3074`.
The reminder/expiry sweep is `server/src/services/nationalPartnershipQuoteExpiryService.js`,
and its idempotency guard belongs in the update *filter*
(`{ quoteReminderSentAt: { $exists: false } }`), not an if-statement.

---

## 3. Prompts dispatched

Five prompts were written and handed to the product owner for dispatch to per-app sessions.
The files themselves are gone from this container's scratchpad; copies exist in the chat
downloads. Their substance is reproduced here.

| # | Prompt | Target | Status |
|---|--------|--------|--------|
| 1 | Bar Snap presenting sponsor | bar-snap | **landed** — `33c4929` (PR #72), 2026-08-05 |
| 2 | TropeLit site sponsorship + quote terminus | trope-lit | **landed** — branch `claude/sponsorship-quote-flow`, merged `111b810` |
| 3 | Play Spotter stacking / ad-free / preview | play-place-finder | **landed** — `7964567` (PR #16), 2026-08-05 |
| 4 | TropeLit advertise copy rewrite | trope-lit | **landed** — `7c3847f`, `355387f` (PR #128) |
| 5 | Manual test plan generator | all three | **not landed anywhere** |

Status column re-verified 2026-08-10 against `origin/main` in all three repos — see §6.
The rest of §3 below is the *as-dispatched* record and describes pre-fix state. Read §6 first.

### 3.1 Bar Snap presenting sponsor — four parts

1. **Header strip.** Build the `HEADER_PRESENTED_BY` equivalent. Bar Snap's chrome is
   `website/src/components/NavigationShell.tsx:58` (`sticky top-0 z-50`), with `minimalChrome`
   at `:29-34` and `isLandingPage` at `:36`. Android has a single `Scaffold` with a `topBar`
   slot at `android/.../ui/BarSnapApp.kt:197/199`.
   Retire the second banner: `website/src/app/page.tsx:38-41` currently renders
   `<StateSponsorBanner />` and `<PartnerSponsorBanner dimension="country" value="US" />`
   inside one `<section>` — that pair *is* the double banner.
2. **Ad-free rule.** `website/src/components/StateSponsorBanner.tsx:53` reads
   `if (adFree && type !== 'paid') return null;` — this gates the *house* variant but lets the
   *paid* state sponsor through to ad-free users. Ad-free must gate the surface, not the fill.
   The promise being violated is stated at `website/src/app/premium/page.tsx:125` and
   `android/.../ui/screens/PremiumScreen.kt:179`.
3. **No-stacking rule.** Enforce runtime adjacency, checked signed-out.
4. **Quote-terminus intake.** Mirror Play Spotter. Reuse the existing intake endpoint with a
   discriminator (`intakeKind: 'national_partnership'`) rather than standing up a parallel
   lead store. Intake lives at `backend/src/routes/partnership.routes.ts:142`.

**Dead fields to be aware of.** `backend/src/models/SponsorshipBooking.ts:37-40` (schema
`:99-102`) declares `bannerHeadline`, `bannerImageUrl`, `bannerCtaText`, `bannerCtaUrl`.
**Nothing in the repo writes any of them** — the only non-test readers are the model and
`backend/src/services/partnershipServing.service.ts`. Note that `bannerImageUrl` *is* the
logo field the header strip needs; do not drop it. (I initially specced the header as
text-only and had to reverse that.)

**Copy defect.** `backend/src/constants/adPlacements.ts:89` is a house state-sponsor variant
reading *"Lock in your state before another brand does - no rotation, no competing sponsors."*
State and national are separate inventory but adjacent on screen, so this line overpromises.
The exclusivity copy is the real defect, not the inventory model. The other house variant is `:84`.

**Self-serve reach.** `website/src/components/AdvertiserHub.tsx:87` —
`const SELF_SERVE_REACH: AdReach[] = ['local', 'state'];`, applied at `:1639`. National is
deliberately excluded from self-serve checkout, which is what makes the quote terminus necessary.
Pricing: `backend/src/constants/adPricing.ts` — local 99, state 299, national 999; `global` removed.

### 3.2 TropeLit site sponsorship — **read the correction first**

**This prompt shipped with a false premise and was rewritten.** The original asserted that
nothing in `website/src/` calls `/partnerships/current`, and instructed the session to verify
with a grep I never ran. It does call it. `website/src/components/layout/SponsorStrip.tsx`
exists, is mounted in the root layout at `website/src/app/layout.tsx:26` (so it is sitewide),
calls `getCurrentSponsor()` → `website/src/lib/api.ts:913`, tracks clicks, and returns null
when unsold with no house fallback. The TropeLit session caught this and reported that a
literal reading would have had it build a second, duplicate strip. It was right.

The rewritten Part 1 is therefore **not** "build the strip" but:
- **Android parity.** `android/.../ui/components/SponsorStrip.kt` renders only from
  `ui/home/HomeScreen.kt:166` — home-only, while web is sitewide. Bring Android to parity.
- **`labelText`.** Support the configurable label the way Play Spotter's strip does.

Remaining parts: ad-free rule, no-stacking rule, quote terminus (models are largely present —
`server/src/models/SitePartnership.ts` already has `logoUrl`, `tagline`, `clickUrl`,
`requestedWindow`, start/end dates, quote/deposit/balance fields, and `accessToken`).

**Bounce bug.** `website/src/components/advertise/CampaignForm.tsx:142` does
`window.location.href = '/advertise#site-sponsorship'` — a full-page bounce that discards
whatever the advertiser had already typed.

**Intake is creative-less.** `website/src/components/advertise/PartnershipInquiry.tsx:22` has
six fields and collects no creative, so the quote-terminus flow has nothing to preview.

### 3.3 Play Spotter — it is not the clean reference

Audited on request rather than assumed good. Three findings:

1. **Confirmed stacking violation.** `website/app/discover/page.js:1260-1268` renders
   `CATEGORY_HEADER` and `:1354-1360` renders `RegionSponsorSlot`. *Everything between them is
   conditional.* Signed-out with empty sections, the two collapse adjacent. This is Rule 3
   broken in the reference implementation.
2. **No ad-free tier exists.** The only equivalent is `hideAdsForRecording` in
   `AdDisplayConfig.kt`, an administrator demo toggle. Rule 2 has nothing to apply to yet;
   it needs to be built.
3. **Advertiser preview overstates the format.**
   `website/app/components/NationalPartnershipLeadForm.js` — `NationalAdPreview` at `:26-95`
   shows both a "Header placement" mock carrying a `Learn more →` affordance the real strip
   does **not** have, and a "Home / category card" that `national_partner` never receives at
   all (its only placement is `HEADER_PRESENTED_BY`). The preview needs to be tier-aware.

Inline cadence for reference: `website/app/lib/adTracking.js:77-78` —
`INLINE_AD_FIRST_BREAK = 2`, `INLINE_AD_CADENCE = 6`.

### 3.4 TropeLit advertise copy

Two problems the product owner identified from a screenshot: the voice is too formal for the
app, and the copy assumes every advertiser is an author selling a book. Many adjacent
categories buy this inventory.

**The fix that generalises:** describe *reader* behaviour, not *advertiser identity*. "Readers
browsing enemies-to-lovers" is category-neutral; "your book" is not.

Scope: six placement descriptions, two label renames, Android parity, and the author-assuming
strings at `AdvertisePage.tsx:42`, `AdvertiserHub.tsx:1137`, `CampaignForm.tsx:952`.

Canonical copy lives in `website/src/components/advertise/placementCopy.ts`
(`PLACEMENT_DESCRIPTIONS`, `PLACEMENT_LABELS`), mirrored at `server/src/services/placementCopy.ts`,
with `server/tests/placementCopy.parity.test.ts` enforcing the mirror. **Both files must change
together or the parity test fails.** The header comment in `placementCopy.ts` states the house
rules — no em-dashes, every description states occupancy explicitly — and also contains a stale
pointer to an Android `placementDescriptions` map; Android's copy actually lives inline in
`CAMPAIGN_TYPE_OPTIONS` at `android/.../ui/advertise/CampaignSetupScreen.kt:94-101`.

### 3.5 Manual test plan generator

One app-agnostic prompt sent to all three sessions; each generates `docs/manual-test-plan.md`
from its own code, 20-25 scenarios.

Written for one person working alone, holding both tester and administrator roles, who is not a
developer and will never open a terminal, query a database, read a log, or use browser developer
tools. **This is a security requirement, not a convenience one** — the product owner's stated
goal is to hire QA testers without granting any control or administrative rights. If a check
can't be done by looking at a screen, it is out of the plan and moved to a "Needs a developer"
list so it becomes an automated test rather than silently disappearing.

Format per scenario: **Sign in as / Where / Set up first / About [time] / Steps / Passes if /
Fails if / Why this one matters**, with `- [ ]` checkboxes on every step. Hard bans on
technical jargon. Every scenario must name a *visible* marker — "the row shows a green dot and
the word Active", never "verify the campaign is active". Both pass and fail outcomes required.
All typed test data prefixed `ZZTEST`. Payment provider test mode only.

**Ranking rule:** damage-if-broken × likelihood-of-breaking-unnoticed. Favour silent failures
over crashes — a crash is reported within the hour; money going to the wrong place runs for
months. Cut anything a unit test already covers. Do not pad to 25.

Required coverage includes the three rules from §1 plus: money in; money that shouldn't move
(abandoned checkout, declined card, double-press); money back out; price shown equals price
charged; test/real data separation; permissions and immediate revocation; exclusive really
being exclusive; app/website parity.

**Expect two of the three to report gaps.** Play Spotter has no ad-free tier (§3.3); Bar Snap
and TropeLit may report the same. The prompt requires them to say so explicitly rather than
skip silently, so a suspiciously complete plan is itself a signal.

---

## 4. Open decisions for the product owner

> **CLOSED 2026-08-10 — this was not a disagreement.** See §6.3. The three numbers are a list
> price, a superseded list price, and a promotional rate off that list price. Web and Android
> both read **45** and have since `af5da0d` (2026-08-05), which changed both in one commit —
> five days *before* this document called them contradictory. Nothing here is blocking.

~~**`home_featured` pricing disagrees three ways** in TropeLit:
`android/.../ui/advertise/CampaignSetupScreen.kt` says **45**, web `campaignTypes.ts` says **99**,
and the live founding rate is **$18/mo**. The TropeLit session was instructed to report and stop,
not to guess. Someone has to name the authoritative source.~~

**Decided 2026-08-10 by the product owner:**
- Bar Snap's adjacency helper **is** meant to have call sites — it was a miss, not staging.
  Prompt 6 (`docs/prompt-06-bar-snap-wire-adjacency-guard.md`) is written and ready to dispatch.
- **Per-screen coverage is the standard**, not app-wide. TropeLit and Play Spotter guarding one
  screen each is acceptable; neither needs an Android guard. No new work for those two.
- Prompt 5 (manual test plans) is **on hold** until the Rule 3 work settles, so the plans get
  written against final behaviour.

No open decisions remain.

---

## 5. Verification status of everything above

Bar Snap citations were verified against `origin/main = 88f333a`. TropeLit and Play Spotter
citations were verified at the head of their default branches on the day the prompts were written.
**Line numbers drift.** Re-verify before acting.

This matters more than usual here, because this session produced two verification failures:

- **Stale checkout.** I reported Bar Snap files from a tree three commits behind `origin/main`,
  and had to re-verify every citation after fetching. One correction survived:
  `adPlacements.ts` exclusivity copy is at `:89`, not `:88`; the other variant is `:84`, not `:83`.
- **An asserted grep I never ran.** Described in §3.2. I inferred a file's absence from two
  searches that used different terms than the file's actual name, then wrote the conclusion into
  a prompt as a verified fact, complete with a command to confirm it. The receiving session
  caught it.

Every prompt after that failure carries a standing instruction to verify each claim in the
document before acting on it, and the test-plan prompt states the rule directly:
**do not assert the result of a search you did not run.** That instruction applies to whoever
picks this up next, including future sessions of me.

---

## 6. Verified status — 2026-08-10 (coordinator pass 2)

Everything in this section was checked against `origin/main` fetched on 2026-08-10:
bar-snap `5134ac1`, trope-lit `d3a7dc2`, play-place-finder `7698178`. All three working
trees started this session behind `origin/main` — the §5 stale-checkout failure again — so
every claim below is post-fetch. Where I did not verify something, it says so.

### 6.1 Four of the five prompts had already landed

They landed on 2026-08-05/06, i.e. *before* §3 was written declaring them merely dispatched.
The correct read is that §3's defect descriptions are a snapshot of pre-fix state, not a
to-do list. Anyone acting on §3 literally would re-fix fixed code.

Spot-verified in current code rather than trusted from commit titles:

- **Rule 1 (header, not banner).** Bar Snap `website/src/components/PresentingSponsorLine.tsx`
  and `android/.../ui/components/PresentingSponsorLine.kt` exist; `PartnerSponsorBanner.tsx`
  and `PartnerSponsorCard.kt` were deleted; `website/src/app/page.tsx` lost the second banner.
- **Rule 2 (ad-free spares only the national sponsor).** Bar Snap
  `StateSponsorBanner.tsx:57` now reads `if (adFree) return null;` — the surface, not the fill.
  The old `if (adFree && type !== 'paid')` is gone. `PresentingSponsorLine.tsx` contains no
  ad-free check, so the header is exempt as required. TropeLit
  `website/src/components/layout/SponsorStrip.tsx` likewise contains no ad-free reference,
  with gating centralised in `website/src/components/ads/useAdFree.ts`.
- **TropeLit Android parity.** `SiteSponsorStrip()` is now mounted at
  `android/.../navigation/TropeFinderNavigation.kt:142` — navigation level, not home-only.
  `labelText` is supported on both surfaces (`SponsorStrip.tsx:65`, `SponsorStrip.kt:102`).
- **TropeLit bounce bug fixed.** The `window.location.href` navigation is gone from
  `CampaignForm.tsx`; the call site is now in-form state, and the removed behaviour is
  documented in a comment at `:157-162`.
- **TropeLit copy.** `placementCopy.ts` and its server mirror plus the parity test are all
  present. The three author-assuming strings named in §3.4 no longer match. Remaining
  "your book" hits are in profile/badges/share/lists — reader-facing, out of scope.

### 6.2 Prompt 5 did not run anywhere

There is no `docs/manual-test-plan.md` in any of the three repos, and neither
`play-place-finder/docs/SESSION_HANDOFF_2026-08-10.md` nor
`trope-lit/docs/SESSION_HANDOFF_2026-08-09.md` mentions a test plan at all. This is the one
prompt that produced nothing. It is also the one whose §3.5 spec is the most detailed, so
re-dispatch should be cheap. **The §3.5 warning still stands and is now sharper:** Play
Spotter's ad-free tier is a stub that always returns true (§6.3), so any Play Spotter plan
claiming ad-free coverage is wrong on its face.

### 6.3 Pricing: closed, and it was never three-way

`af5da0d` (2026-08-05) changed web `campaignTypes.ts` from 99 to 45 *and* touched
`android/.../CampaignSetupScreen.kt` in the same commit. Current values, read today:

| Source | Value |
|---|---|
| `website/src/components/advertise/campaignTypes.ts:7` | `price: 45` |
| `android/.../ui/advertise/CampaignSetupScreen.kt` (`home_featured`) | `45` |
| `server/src/services/stripe.service.ts:13` | `home_featured: 45` |
| `server/src/services/foundingRate.service.ts:72` | floor `15`, commented *"list dropped from $99 to $45; 60% off = $18, so floor must be ≤$18"* |

So: **$45 is the list price and both surfaces agree**; **$18/mo is the founding rate, which is
60% off $45**; **$99 is the superseded list price**. A list price and a promotional rate are
not in conflict, and the founding-rate service states the relationship in a comment.

One loose end, minor and *not* a pricing conflict: two server test fixtures still carry the
old 99 — `server/tests/stripe.webhook.payments.test.ts:82` and
`server/tests/ad.controller.test.ts:90`. They are local mock price tables, so they do not
affect what a customer is charged, but a fixture pinned to a dead price can mask a real
pricing regression. Worth a cleanup ticket, not a blocker.

### 6.4 Rule 3 diverged three ways — and Bar Snap's guard is dead code

All three apps built a computed runtime guard, which is the right shape. None of them built
the same one, and none share an implementation:

| App | Mechanism | Platform coverage | Applied at |
|---|---|---|---|
| Bar Snap | pure order-driven fn, `website/src/lib/adAdjacency.ts` + Kotlin mirror `android/composeApp/.../ads/AdAdjacency.kt` | web + Android | **nowhere** |
| TropeLit | `keepAds` from `website/src/components/ads/adLayout.ts` | web only | `SearchResults.tsx:11` only |
| Play Spotter | React context `AdSlotSequence` + `useAdSlotVisible` | web only | `discover/page.js`, via `RegionSponsorSlot` / `SponsorPlacement` |

**The finding that matters: Bar Snap's adjacency guard has no production call sites on either
platform.** `resolveAdSlots` / `hasAdjacentAds` are imported only by
`website/tests/ad-adjacency.spec.ts`; the Kotlin `AdAdjacency` is referenced only by
`AdAdjacencyTest.kt`. Both are well-tested — 122 and 137 lines of test respectively — and both
are unreachable from the app. Bar Snap's actual Rule 3 compliance today rests on having
*deleted* the second banner from `page.tsx`, which is a source-order fix. That is precisely
the reasoning the helper's own docstring rejects: *"static source order does not prove
non-adjacency."* Green tests are actively hiding this.

I am not calling this a defect on my own authority, because it may be deliberate staging —
land the helper with tests, wire it when a second placement lands. **This needs the product
owner to say which.** If it is meant to be wired, that is a new Bar Snap prompt.

Secondary divergence, lower stakes: TropeLit and Play Spotter each guard exactly one screen
(search results; discover) and neither has an Android guard. Bar Snap is the only app that
wrote the Kotlin side at all. If Rule 3 is meant to hold app-wide rather than screen-wise,
two of the three are under-covered.

### 6.5 Rule 2 diverged honestly

Play Spotter's `website/app/lib/adEligibility.js` is a stub returning `true` unconditionally,
with a comment saying no premium tier exists yet and naming the single place to wire one in.
That matches §3.3 finding 2 exactly and is the right call — it is a seam, not a claim. Bar
Snap and TropeLit both have real ad-free tiers and both spare the sponsor strip. No conflict
between the three; Play Spotter is simply a tier behind, deliberately.

### 6.6 What I did not verify

- Whether the landed code *works* — I read source, ran nothing, and there is no test run in
  this session's record.
- Play Spotter's quote-terminus and Bar Snap's quote-terminus beyond file existence
  (`quoteExpiry.service.ts`, `quoteAvailability.test.ts` are present in the #72 diff; I did
  not read their logic or check the idempotency-guard-in-the-filter point from §2).
- The §3.1 dead-banner-fields question — whether `bannerImageUrl` survived as the logo field
  the header needs. The #72 diff touches `SponsorshipBooking.ts` (+15/-…) but I did not read it.
- Anything in PlayBound, which remains out of scope.

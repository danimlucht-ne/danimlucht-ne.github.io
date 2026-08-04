# Pricing Audit — Bar Snap, TropeLit, PlayBound, Play Spotter

**4 August 2026** · Prices read from repo constants at HEAD · Verdicts weighted by actual current reach

> Every ad and purchase SKU across the four apps: list price, verdict against market, and what to change.
> The rates themselves are mostly well-constructed. What they are not calibrated to is the audience that exists today.

---

## The number every verdict hangs on

| App | Users today | Status |
|---|---:|---|
| **Play Spotter** | 600 | Live, Play Store production. The only app with real reach. |
| **PlayBound** | 5 | Live. Premium is the only revenue line; no ad inventory. |
| **Bar Snap** | 0 | Pre-launch. Most elaborate ad stack of the four. |
| **TropeLit** | 0 | Pre-launch. Ads are the *only* planned revenue path. |

Ad pricing is not set by what a placement is worth in the abstract — it is set by impressions delivered. Below roughly 5,000 monthly actives, a display rate card stops being a price list and becomes a negotiating anchor.

Here is the arithmetic on your best-positioned SKU, the Play Spotter Official Regional Partner slot at $299/mo:

```
600 registered  ×  ~50% monthly active        ≈    300 MAU
300 MAU  ×  3 sessions  ×  6 ad surfaces      ≈  5,400 impressions / month, all regions
flagship metro share (~60%)                   ≈  3,200 impressions / month

$299  ÷  3.2k impressions                     =    $93 effective CPM
```

Programmatic in-app native runs about **$3.30 CPM**; North American in-app eCPM about **$6.50**; private marketplace about **$8.20**. Direct-sold, hyper-targeted, category-exclusive local inventory can legitimately command **$30–65 CPM** — that is the ceiling, and it is the framing your exclusive slots deserve. At $93 you are still 1.5–3× above that ceiling, and the absolute delivery — 3,200 impressions for $299 — is the part an advertiser will actually notice.

The newsletter heuristic gives the harsh read: flat fee of 2.5–5% of audience size = **$15–30/mo** at 600 users. The truth sits between the two, which is where the recommendations land.

---

## Play Spotter — live, 600 users

Phase-based regional pricing (growing → mature), a 60% beta discount for the first 10 advertisers per region, radius surcharges, and a national rate card. Structurally the best-built of the four. Two things are off: the exclusive slot is the one placement excluded from the beta discount, and the website quotes higher prices than checkout actually charges.

| Ad / purchase type | List price | Verdict | Recommendation |
|---|---|---|---|
| **Official Regional Partner** — one exclusive per metro | $299/mo (no beta discount) | 🔴 High | **Drop to $99–149/mo** until the metro clears ~2,500 MAU, or fold it into the beta discount. Most important change in the audit. |
| **Prime Placement — growing** (`featured_home`) | $149/mo → $59.60 beta | ⚪ In range | **Confirm.** $59.60 against ~2,700 impressions ≈ $22 CPM — defensible for direct-sold top-of-home. |
| **Prime Placement — mature** | $199/mo | ⚪ In range | **Confirm as future state.** No region is mature yet, so this isn't being charged. |
| **Inline Placement — growing** (`inline_listing`) | $49/mo → $19.60 beta | 🟢 Low | **Floor the beta price at $25/mo.** $19.60 doesn't cover creative review and support for a month-long campaign. |
| **Inline Placement — mature** | $59/mo | ⚪ In range | **Confirm.** Correct ladder step. |
| **Event Partner — 7 day** | $13 → $5.20 beta ($12 floor) | 🟢 Low | **Exempt events from the beta discount** or raise the net floor to $12. A $5.20 transaction costs more to process than it earns. |
| **Event Partner — 14 day** | $25 → $10.00 beta ($22 floor) | 🟢 Low | **Same treatment.** Net floor $20. Events are your best low-friction entry SKU — cheap, not free. |
| **Radius extension** (20 mi included) | +$15 / +$25 / +$35 (30/40/50 mi) | ⚪ In range | **Confirm.** One-time rather than per-month is the right call. |
| **Reduced radius** | ×0.50 / ×0.70 (5 mi / 10 mi) | ⚪ In range | **Confirm.** Gives a small business a way in at ~$10–25/mo effective. |
| **Duration ladder** (1/2/3/6 mo) | 0% / 5% / 15% / 25% | ⚪ In range | **Confirm.** Well-shaped — 15% at 3 months is where most local advertisers land. |
| **Beta region discount** | 60% off, first 10 advertisers/region | ⚪ In range | **Confirm the mechanism**, and state the sunset in checkout copy: "founding rate, locked for your first 12 months." |
| **Category Partner** (national) | From $149/mo | 🔴 High | **$49–99/mo** at current reach. |
| **Seasonal Partner** | $249 / $599 / $999 (1/3/6 mo) | 🔴 High | **Roughly halve: $129 / $299 / $499.** Ladder shape is good; only the base is too high. |
| **National — Header** (exclusive, every screen) | $800–2,000/mo | 🔴 High | **Keep as a published anchor, don't chase it.** A visible rate card makes local prices look like a bargain. |
| **National — other 9 placements** | $99–1,200/mo | 🔴 High | **Same treatment — anchor, not a price list.** Internal ordering is sensible. Revisit at 25k MAU. |
| **Remove Ads** (one-time, permanent) | $9.99 | 🔴 High | **$4.99 now, $9.99 later.** Mature-app price against a light ad load — users pay to remove something they barely see. |
| **Booking deposit** (>30 days out) | 25% non-refundable | ⚪ In range | **Confirm.** Standard, correctly applied to exclusive placements regardless of lead time. |

**Fix first — the exclusive slot is the only placement excluded from the beta discount.** In `getPhasePrice`, `region_sponsor` returns early with `betaRegionDiscountActive: false` and never reaches `buildRegionBetaPricingResult`. So the hardest thing to sell at 600 users — a $299 exclusive — is sold at full list, while a $49 inline card sells for $19.60. That is backwards.

**Leak — the website quotes $99/mo for placements that check out at $49 list and $19.60 net.** `advertiseContent.js` says "Starting at $99/mo" for both Prime and Inline. The drift test only guards against the site quoting *lower* than reality, so this passes — but every small business that bounces off "$99" never discovers the real price. Change to "From $19/mo during founding rates."

**Note.** "Event Partner — From $12" likewise overstates the real charge ($5.20 with beta active). Directionally safe, and raising the net floor makes the $12 claim true again.

**Confirmed.** Locking `totalPriceInCents` and `cityPhaseAtPurchase` at booking is right even though a distant booking runs at today's rate — being re-priced after quote is far more damaging than the margin given up.

**Confirmed.** Deriving event prices from the monthly sponsored rate (25% / 50%) rather than storing them independently means one rate edit moves the whole card coherently.

---

## Bar Snap — pre-launch, 0 users

Reach-tiered campaigns, an exclusive state sponsor, event spotlights, and a quote-based partner program — priced for an app with a national footprint. The rate card is coherent internally. Its problem is that it has no launch mode: unlike Play Spotter, there is no beta or founding discount anywhere in the code, so day-one advertisers are quoted mature-product prices.

| Ad / purchase type | List price | Verdict | Recommendation |
|---|---|---|---|
| **Local campaign** (~25 mi radius) | $99/mo | 🔴 High | **$29–49/mo founding rate.** $99 is a fair mature price — it needs a launch discount attached, not a rewrite. |
| **State campaign** | $299/mo | 🔴 High | **$99–149/mo founding rate.** Same structure: keep the list, add the discount. |
| **National campaign** (US-wide) | $999/mo | 🔴 High | **Pull from self-serve; make it quote-only.** A $999 self-serve button on a zero-user app reads as unserious. It belongs in the partner flow you already built. |
| **Global campaign** | $1,999/mo | 🔴 High | **Remove it, or merge into National.** Content model is US-centric (US states, US retailers). "Global" at 2× national is a weak ladder step selling reach the product doesn't have. |
| **State Sponsor** (one business per state) | $299/mo | 🔴 High | **$149–199 founding, but it must exceed the metro rate.** Priced identically to Play Spotter's single-metro exclusive while covering a whole state. |
| **Event Spotlight** | $15/day ($105/7d, $189/14d) | 🔴 High | **Cut to $3–5/day.** The identical SKU on Play Spotter is $13 for 7 days. 8× spread inside one portfolio with no difference in reach. |
| **Duration ladder** (1/2/3/6 mo) | 0% / 5% / 15% / 25% | ⚪ In range | **Confirm.** Matches Play Spotter exactly — what you want across a portfolio. |
| **Partner tiers** (National / Category / Ingredient) | Quote-based | ⚪ In range | **Confirm the model, add published "from" anchors.** A quote flow with no visible floor filters out serious inbound as effectively as a too-high price. |
| **Premium — monthly** | $4.99/mo | ⚪ In range | **Confirm.** Standard for a hobby utility; free-tier limits (25 items, 5 scans, 2 receipts) are well-drawn. |
| **Premium — yearly** (built, not exposed) | $53.99/yr | 🔴 High | **Expose it at $44.99 (25% off).** $53.99 is only 9.8% off monthly — below the 15–25% users expect. Shipping an annual option at all is worth more than the discount. |
| **Premium — lifetime** (built, not exposed) | $99.99 | ⚪ In range | **Optional — hold it back.** Selling it pre-launch caps LTV on the early users most likely to stay. |
| **Standalone ad removal** | Does not exist | ⚪ In range | **Confirm; don't add one.** Premium carries real non-ad value (unlimited scans, analytics, export). A $9.99 ad-removal SKU would cannibalise a $4.99/mo subscription. |

**Fix first — Bar Snap has no founding-rate mechanism at all.** Play Spotter ships a 60% beta discount with a slot limit and an advertiser counter. Bar Snap has discount codes but no structural intro pricing. Port the concept before launch — it's the difference between "our prices are $99" and "our prices are $99, yours is $39 because you are first."

**Note.** The `premium_plus` tier exists in `isPremiumUser` / `isAdFreeUser` but has no price, no checkout, and no distinct entitlements. Either define it or drop the branch — a dormant paid tier in the auth path is where bugs hide.

**Confirmed.** The pro-rated refund policy — full refund before start, nothing after completion, undelivered remainder in between, separate non-refundable path for advertiser-initiated cancellation — is well-designed and worth putting in the advertiser terms verbatim as a selling point.

**Confirmed.** Charging from stored `totalPriceInCents` rather than recomputing from the live rate table is the right architecture.

---

## TropeLit — pre-launch, 0 users

Six placements, a singleton rate card with an audit history, and a $5–5,000 validation band. The cleanest pricing implementation of the four. But it sells into the author-promotion market — a market with well-known public prices — and its monthly rates sit above services that deliver 30,000 targeted readers today.

| Ad / purchase type | List price | Verdict | Recommendation |
|---|---|---|---|
| **Genre Banner** (`genre_sponsor`) | $149/mo | 🔴 High | **$39–59/mo founding.** Freebooksy charges $40–100 for a blast to 30k genre-matched subscribers. You're asking 1.5× the top of that against zero readers. |
| **Home Featured** (`home_featured`) | $99/mo | 🔴 High | **$29–49/mo founding.** Correct as mature price; wrong as launch price. |
| **Trope Feature** (`trope_sponsor`) | $79/mo | 🔴 High | **$25–39/mo founding.** Your most differentiated placement — trope-level targeting is what no competitor sells. Price it to get filled, then raise. |
| **Author Spotlight** (`author_spotlight`) | $59/mo | 🔴 High | **$19–29/mo founding.** Indie authors are individuals spending their own money — needs to clear an impulse threshold. |
| **Search Inline** (`search_inline`) | $49/mo | 🔴 High | **$15–25/mo founding.** Highest-intent surface you have. |
| **New Release Spotlight** (`new_release`) | $15/week ($60 for 4 weeks) | 🟢 Low | **Keep exactly as is.** Best-designed SKU in the portfolio: matches how authors actually buy (launch week, not calendar month) and undercuts every newsletter promo on the market. This is your wedge — lead with it. |
| **Multi-month discount** | 5% per additional month (~4.2% at 6 mo, ~4.6% at 12 mo) | 🔴 High | **Replace with the portfolio ladder: 0/5/15/25%.** A 12-month commitment currently earns 4.6%. The other apps give 25% at six months. Nobody will take a 12-month term. |
| **Reservation deposit** (>30 days out) | 25% non-refundable | ⚪ In range | **Confirm.** Matches Play Spotter. Consistency beats optimisation here. |
| **Rate validation band** | $5 – $5,000 | ⚪ In range | **Confirm.** Wide but finite, whole dollars, unknown keys rejected. Exactly right. |
| **Reader premium / ad-free** | Documented as candidate, not built | ⚪ In range | **Correct to defer.** When built, $2.99/mo — TropeLit has no non-ad premium value to bundle, so ad-free must stand alone. |

**Fix first — the duration ladder is inverted relative to the business.** Author campaigns are the most seasonal and most repeat-friendly of your three ad businesses. TropeLit is the app where a long commitment is worth most, and it has by far the weakest long-term incentive. Moving to 0/5/15/25% costs little and unlocks the term lengths you want.

**Risk — single revenue path, zero readers.** Ads are the only monetization and ads need an audience. Every other app has a second line. Until TropeLit has readers, treat ad revenue as a 2027 line item and price placements to build advertiser relationships, not margin.

**Confirmed.** Code constants as floor and fallback, with the database only ever overriding them, means a Mongo outage cannot make a placement free. Strongest pricing safety design in the portfolio — port it to the other two.

**Confirmed.** Computing the admin preview by calling the same `computePriceCents` that checkout calls, rather than reimplementing the maths for display, is exactly right — and `marketingClaimWarnings`, which flags when the "from $15" claim goes stale, is worth copying.

---

## PlayBound — live, 5 users

No ads, no ad removal — Premium is the whole business, sold through Discord entitlements and Stripe. The monthly price is well-benchmarked. Two things are wrong: the annual discount is too thin, and the most valuable features are server-level while the price is user-level.

| Ad / purchase type | List price | Verdict | Recommendation |
|---|---|---|---|
| **Premium — monthly** | $4.99/mo | ⚪ In range | **Confirm — sits exactly on the band.** Dyno $4.99, Carl-bot $5, PeakBot $8.25, MEE6 $11.95. If anything you're low for the feature depth. Hold at $4.99, then test $5.99. |
| **Premium — yearly** | $53.99/yr (9.8% off monthly) | 🔴 High | **$44.99/yr (25% off) or $49.99 (17% off).** A sub-10% annual discount moves nobody. The convention users recognise is "two months free" — $49.99 reads as exactly that. |
| **Server-tier Premium** | Does not exist | 🔴 High | **Add a per-server tier at $9.99–14.99/mo.** Largest single revenue gap in the portfolio. |
| **Ad campaigns** | No ad inventory | ⚪ In range | **Confirm — don't build ads here.** Ads in a Discord bot degrade the product and Discord's norms are against it. |
| **Ad removal** | Nothing to remove | ⚪ In range | **Correct by construction.** No action. |
| **Shop & cosmetics** | Points only | ⚪ In range | **Confirm; keep it points-only.** Your terms already state points have no real-world value. Selling currency pulls you into Discord's monetization rules and a refund surface you don't want. |

**Fix first — server-level value is being sold at an individual price.** Autopilot scheduling, Server Pro Shops, custom roles and badges, deeper mod tools all benefit an entire server, but Discord entitlements are user-scoped, so one owner pays $4.99 and everyone gets the benefit. MEE6, Dyno, and Carl-bot all charge *per server* for exactly this class of feature. Keep the $4.99 user tier for personal perks (2× points, cosmetics, daily bonus) and add a server tier at $9.99–14.99/mo for automation and shop tools.

**Note.** Confirm the $53.99 figure against Stripe — it's set by `STRIPE_PRICE_ID_YEARLY` and appears nowhere in the repo, so this audit takes it on your recollection. The recommendation holds for anything above ~$50.

**Confirmed.** Keeping official faction standings out of Premium is the right line to hold. Selling leaderboard placement would be the one change that breaks trust in a competitive community — the marketing copy is admirably explicit about it.

**Note.** At five users, pricing is not the constraint — distribution is. Nothing here is worth acting on before the bot is in 50+ servers, except the annual price, which is a one-line change.

---

## Where the four rate cards contradict each other

| Concept | What each app charges | Verdict | Recommendation |
|---|---|---|---|
| **7-day event spotlight** | Bar Snap $105 · Play Spotter $13 | 🔴 8× gap | Same placement concept, same reach profile, no justification. Set Bar Snap to $3–5/day ($21–35/week). |
| **Exclusive geographic sponsor** | Both $299/mo — but Bar Snap = a *state*, Play Spotter = a *metro* | 🔴 Inverted | A state contains 5–20 metros. Recommended: metro $99–149, state $199–299. |
| **Multi-month discount** | 0/5/15/25% — except TropeLit at 5%/addl month | 🔴 Odd one out | Move TropeLit onto the shared ladder. Three implementations of one idea shouldn't be two different ideas. |
| **Ad removal** | $4.99/mo sub · $9.99 one-time · none · none | 🔴 Incoherent | Defensible per-app once stated as a rule: bundle ad-free into a subscription where there's other premium value (Bar Snap), sell one-time where there isn't (Play Spotter, later TropeLit). Write the rule down. |
| **Founding / intro discount** | Play Spotter 60% · Bar Snap, TropeLit none | 🔴 Gap | The two pre-launch apps need it most and are the ones without it. Port the mechanism before either launches. |
| **Booking deposit** | 25% across all three ad apps | ⚪ Consistent | **Confirm.** Already aligned. Leave it alone. |
| **Quote-locking at checkout** | Consistent across all three | ⚪ Consistent | **Confirm.** Every app stores the quoted total and charges from it rather than recomputing. Most often gotten wrong; you have it right in three places. |

---

## What to change, in order

1. **Include the Play Spotter region sponsor in the beta discount — or cut it to $99–149.** The only revenue-earning ad slot in the portfolio, and the only placement quoted at full list to a founding advertiser. One conditional in `getPhasePrice`.
2. **Fix the Play Spotter website copy: "Starting at $99/mo" → "From $19/mo."** You're turning away inbound at a price you don't charge. Copy change, no code risk.
3. **Reprice PlayBound annual to $49.99.** One Stripe price ID. Confirm the current $53.99 first.
4. **Move TropeLit to the 0/5/15/25% duration ladder.** Do it before launch, while no campaign has been quoted under the old formula.
5. **Cut Bar Snap's event spotlight to $3–5/day and pull National and Global from self-serve.** Pre-launch, so this costs nothing today and prevents an 8× contradiction from ever being visible.
6. **Port the beta-discount mechanism into Bar Snap and TropeLit.** Largest piece of work here, and the one that determines whether either app can sign a first advertiser.
7. **Add a PlayBound server tier at $9.99–14.99/mo.** Highest revenue ceiling of anything listed, lowest urgency — wait until the bot is in 50+ servers.

---

## Method

List prices read from repo constants at HEAD (4 Aug 2026):

- **Bar Snap** — `backend/src/constants/adPricing.ts`, `stateSponsor.ts`, `eventSpotlightPricing.ts`, `config/index.ts`
- **Play Spotter** — `server/src/config/adPricingDefaults.js`, `services/pricingService.js`, `services/radiusTargetingService.js`, `website/app/lib/advertiseContent.js`
- **TropeLit** — `server/src/services/stripe.service.ts`, `adPricing.service.ts`, `adSlots.service.ts`
- **PlayBound** — prices are set by Stripe price IDs and the Discord SKU and are not in the repo; taken from the owner, pending confirmation.

Audience figures supplied by the owner: Play Spotter 600 users, PlayBound 5 users, Bar Snap and TropeLit pre-launch. Impression estimates are modelled, not measured — replace them with real `adAnalytics` impression counts before quoting any advertiser.

**Benchmarks:**
[Business of Apps — mobile app CPM rates](https://www.businessofapps.com/ads/research/mobile-app-advertising-cpm-rates/) ·
[Liftoff — in-app advertising 2026](https://liftoff.ai/blog/in-app-advertising-in-2026-a-complete-guide-for-mobile-marketers/) ·
[Digital Applied — display benchmarks 2026](https://www.digitalapplied.com/blog/display-advertising-benchmarks-2026-data-points) ·
[Paved — newsletter sponsorship rates](https://www.paved.com/blog/newsletter-sponsorship-rates/) ·
[beehiiv — newsletter ad cost](https://www.beehiiv.com/blog/newsletter-sponsorship-cost) ·
[David Gaughran — book promo site pricing](https://davidgaughran.com/best-promo-sites-books/) ·
[PeakBot — Discord bot pricing comparison 2026](https://peakbot.pro/blog/discord-bot-pricing-comparison-2026) ·
[Business of Apps — app pricing benchmarks](https://www.businessofapps.com/data/app-pricing/)

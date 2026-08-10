# Prompt 6 — Bar Snap: give the adjacency guard call sites

Paste everything below the line into a Bar Snap session.

---

You are working in `bar-snap`. One task: the no-stacking guard that PR #72 landed is never
called, so Rule 3 is not actually enforced anywhere. Wire it in.

**Verify every claim below before acting on it.** Line numbers were read against
`origin/main = 5134ac1` on 2026-08-10 and drift. Do not assert the result of a search you did
not run. If a premise here is wrong, say so and stop rather than building on it — a previous
prompt in this workstream shipped a false premise and the receiving session was right to
push back.

## What is true today

`website/src/lib/adAdjacency.ts` exports `resolveAdSlots`, `hasAdjacentAds`, and the
`AdSlotCandidate` type. `android/composeApp/src/commonMain/kotlin/com/barsnap/app/ads/AdAdjacency.kt`
exports the same three as a Kotlin mirror.

**Neither has a single production call site.** Verified by grepping all three symbol names
across `website/` and `android/`, excluding each guard's own file and its test:

```
grep -rn "AdSlotCandidate\|resolveAdSlots\|hasAdjacentAds" --include=*.kt android/ \
  | grep -v "ads/AdAdjacency.kt" | grep -v "AdAdjacencyTest.kt"
grep -rn "AdSlotCandidate\|resolveAdSlots\|hasAdjacentAds" --include=*.ts --include=*.tsx website/ \
  | grep -v node_modules | grep -v "lib/adAdjacency.ts" | grep -v "tests/ad-adjacency.spec.ts"
```

Both return nothing. The only importers are `website/tests/ad-adjacency.spec.ts` and
`AdAdjacencyTest.kt` — 122 and 137 lines of test around code the app never reaches. The test
suite is green and proves nothing about the running app.

Note this is *not* true of the neighbouring `ads/AdSlotResolution.kt`, which is correctly wired
(`RecipeListScreen.kt:52`, `MyBarScreen.kt:25`, `RecipeDetailScreen.kt:29`, `ShoppingScreen.kt:24`,
`IngredientListingAd.kt:11`). That file is ad-serving plumbing — cadence, region key, slot pick —
not an adjacency guard. Don't confuse the two.

## What I did NOT find, and you should not claim

**I found no live stacking violation on any screen.** I checked the three surfaces that carry
two or more in-flow ad slots:

| Surface | Slots | Why it is currently separated |
|---|---|---|
| `website/src/app/page.tsx` | `StateSponsorBanner` `:38`, `HomeFeaturedAd` `:58` | the action-button grid at `:41` is six static `<ActionButton>`s that always render |
| `website/src/app/recipes/page.tsx` | `StateSponsorStrip` `:547`, inline `PaidAdCard` `:744` | `shouldShowInlineAdAtIndex` starts at `INLINE_AD_FIRST_BREAK = 2`, so two `RecipeCard`s always precede the first inline ad |
| `RecipeListScreen.kt` | `StateSponsorCard` `:776`, `RecipeListRow.Ad` rows built at `:680`/`:696` | same cadence rule — two recipe rows precede the first ad row |

So the app complies today. **It complies by accident.** Every separator in that table is an
unrelated layout fact that nobody is protecting: the home page is safe because a static button
grid happens to sit between two ads, and both list screens are safe because the inline cadence
happens to start at 2 rather than 0. Change `INLINE_AD_FIRST_BREAK` to 0, or make the action
grid conditional on a signed-in user, and Rule 3 breaks silently with every test still green.

That is the reason to wire the guard, and it is the reason to state in the PR description.
**Do not write a PR that claims to fix a stacking bug** — there isn't one to fix. This is
converting an incidental property into an enforced one.

## What to do

Give the guard real call sites on those three surfaces, and only those three. Per-screen
coverage is the agreed standard — the product owner settled this on 2026-08-10. Screens with a
single ad slot (`MyBarScreen`, `RecipeDetailScreen`, `ShoppingScreen`) are out of scope; do not
touch them.

For each surface, build the ordered `AdSlotCandidate` list that describes what that container
actually renders, pass it through `resolveAdSlots`, and let the returned set decide which ad
slots render. The separators are the point: pass the non-ad content in alongside the ads, with
`isSeparator: true`, so one place states what can and cannot collapse. `wantsToRender` is
whatever the slot already decides for itself — ad-free state, a null fetch, geo unresolved.

Two constraints from the rules, both already documented in the guard's own header comment:

- **The presenting-sponsor header is exempt.** `PresentingSponsorLine` is persistent chrome
  outside the content flow, which is why the national partner was moved there. Do not pass it
  in as a candidate, and do not let it be dropped.
- **Adjacency is judged at runtime, signed-out, with optional sections empty.** That is the
  state where collapse actually happens. A check that only holds for a signed-in user with
  every section populated is not a check.

## Tests

`website/tests/ad-adjacency.spec.ts` and `AdAdjacencyTest.kt` already cover the helper's logic
well — do not duplicate that. What is missing is coverage that the call sites exist and hold.
Add, at the level the repo already tests at:

- A signed-out render of the home page and the recipes page with the optional sections empty,
  asserting no two ad units are adjacent. `website/tests/ad-adjacency.spec.ts` is a unit spec;
  the signed-out render check likely belongs alongside the existing Playwright specs instead —
  check what `website/tests/` already does before choosing.
- A regression test that would fail if `INLINE_AD_FIRST_BREAK` were set to 0. That is the
  specific silent break this work exists to prevent, and today nothing catches it.

## Out of scope

Everything else. Do not revisit the header format, the ad-free rule, the quote terminus, or the
exclusivity copy — all landed in #72 and were verified on 2026-08-10. Do not extend coverage to
TropeLit or Play Spotter; they guard one screen each by agreement.

## Report back

State plainly which surfaces you wired, which you found already compliant and why, and anything
in the table above that did not match what you found in the code. If you conclude the guard
should not be wired somewhere I listed, say so with the reason instead of wiring it anyway.

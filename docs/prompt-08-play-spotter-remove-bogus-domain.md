# Prompt 8 — Play Spotter: remove the playspotterapp.com domain from code

> ## ✅ LANDED — do not dispatch this prompt
>
> Merged 2026-08-16 as **play-place-finder PR #30**, squashed to `main` as `18d1331`.
> CI green: `server-tests`, `website-tests`, `website-e2e-mocked` all passed
> (`website-e2e-live` skipped by design).
>
> The coordinator session made this change directly rather than handing it off, so the prompt
> below was never dispatched. It is kept as the record of what was changed and why.
>
> **What shipped went further than this prompt asked.** The prompt said to leave
> `MarketingLinks.SUPPORT_EMAIL` alone. That was later reversed on the product owner's
> instruction, because the constant turned out to be interpolated into the in-app Privacy
> Policy contact section (`LegalDocumentMirror.kt:175`), the in-app Terms of Service contact
> section (`:254`), and the advertiser `TermsScreen.kt:158` — so the app and the website were
> publishing different contact addresses inside documents meant to be identical copies. It is
> now `support@play-spotter.com`, matching the server templates, `website/app/lib/contactEmails.js`,
> and the published legal docs. `grep -rn "playspotterapp"` returns no matches repo-wide.
>
> **Do not act on the "Do NOT change" section below — it is superseded.**

---

You are working in `play-place-finder`. One small task.

`playspotterapp.com` is **not a domain this project owns**. It was checked and confirmed: the
only thing that exists under that name is a Gmail account. The real marketing domain is
`play-spotter.com`, which is what every other file in the server already defaults to.

Two test fixtures still reference the domain we do not own.

**Verify before acting.** Read against `origin/main = 0de9b6a` on 2026-08-10; these line numbers
already moved once today (they were `:282`/`:291` a few hours earlier), so re-find them by
content rather than trusting the numbers.

## Change

`server/src/__tests__/userRoutes.test.js`, in the test
`GET /users/me/invite returns personal invite link and referral stats`:

- `:285` — the `inviteService.getInviteInfo.mockResolvedValue({ ... inviteUrl: ... })` fixture
- `:294` — the matching `expect(res.body.data).toEqual({ ... inviteUrl: ... })` assertion

Both currently read `'https://www.playspotterapp.com/invite/ABCD2345/'`. Change both to
`'https://www.play-spotter.com/invite/ABCD2345/'`.

**This does not change what the test proves.** `inviteService` is mocked here, so
`marketingSiteBaseUrl()` never runs and the URL is arbitrary pass-through data — the test asserts
the route returns what the service gave it, unchanged. Both occurrences must move together or
the assertion stops matching the mock.

Verified for context, no change needed: `server/src/services/inviteService.js:29` already falls
back to `https://www.play-spotter.com`, as do `advertiserTerms.js:17`,
`advertiserEmailService.js:50`, `regionSponsorMoveUpService.js:173`, `supportRoutes.js:11`, and
two spots in `adminRoutes.js`. The bogus domain never reaches production; it only sits in
this one test.

## Do NOT change

`android/composeApp/src/commonMain/kotlin/org/community/playgroundfinder/util/MarketingLinks.kt:50`
— `SUPPORT_EMAIL = "playspotterapp@gmail.com"`. That Gmail account is real and in use. Leave it
exactly as it is. It is the only other `playspotterapp` string in the repo, and deleting it would
break the app's support contact.

## Check your work

`grep -rn "playspotterapp" . | grep -v node_modules | grep -v "\.git/"` should afterwards return
exactly one line: the `MarketingLinks.kt` support email. Run the server test suite for
`userRoutes` and confirm it still passes.

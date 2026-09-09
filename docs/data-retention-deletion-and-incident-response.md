# Data retention, account deletion, and incident response

**Written:** 2026-09-09 · **Apps:** Bar Snap, TropeLit, Play Spotter (Lucht Applications LLC)

Two things in one document because they are one subject: what you hold, and what you do when you
either have to give it up (a deletion request) or lose control of it (a breach).

**I am not a lawyer. This is a read of the code against the common obligations, written so a lawyer
can correct it quickly.** Everything in §1 was verified against the repositories on 2026-09-09 and
cites where. §3 is a draft plan, not advice.

---

## 1. What is actually stored

This section exists because the working assumption was "we don't collect or store personal data
aside from age attestation and email." **That is not accurate**, and the gap matters for both
halves of this document.

Verified across the three repositories:

| Data | Where | Notes |
|---|---|---|
| Email address | all three | the account identifier |
| **IP address + user agent** | all three, on every consent row | `consent.service.ts` (Bar Snap), `policyConsent.service.ts` (TropeLit), `userRoutes.js:153-154` (Play Spotter) |
| Age attestation + version | all three | 21+ / 18+ / 18+ |
| App version, device type | Play Spotter | stored on each consent row |
| **Free-text a user wrote** | all three | TropeLit `BookRating.review` (2,000 chars); support-ticket messages everywhere |
| **Photographs users uploaded** | Bar Snap, Play Spotter | Play Spotter in Google Cloud Storage, `playground_app_bucket` |
| Behavioural profile | all three | favourites, saved searches, reading lists, cabinet contents |
| **Implied location** | Bar Snap, Play Spotter | a list of favourite playgrounds or bars is a home-neighbourhood inference |
| Display name / contributor profile | all three | shown publicly on contributions |
| Stripe customer ID | all three | the payment relationship. Card numbers are Stripe's, not yours |
| Date of birth | **none today** | TropeLit has a `dob` age-gate mode, but `ageGateMode` defaults to `checkbox` and a submitted DOB is explicitly discarded in that mode |

What you do **not** hold, which is the part that keeps this manageable: no card numbers, no
government identifiers, no financial account numbers, no health data, no passwords (Firebase holds
credentials).

---

## 2. Account deletion — the three apps do very different things

All three publish a delete-account route and all three delete every linked Firebase identity.
Below that, they diverge sharply.

| | Bar Snap | TropeLit | Play Spotter |
|---|---|---|---|
| Personal collections deleted | **20** | **0** | 5 |
| Contributions anonymised | 7 | 0 | 5 |
| Uploaded photo *files* removed | not in this path | no | **yes**, from storage |
| Firebase identities deleted | yes | yes | yes |
| Consent/attestation rows | left, real `userId` | left, real `userId` | **pseudonymised on purpose** |
| Deletion receipt emailed | no | no | **yes** |

### 2.1 TropeLit deletes almost nothing — highest priority

`server/src/routes/user.routes.ts:587` is the whole of it:

```ts
const user = await User.findByIdAndDelete(req.user!.userId);
// ...then delete the linked Firebase identities
```

**14 models reference `userId` and none of them are touched**: `Arc`, `AzChallengeEntry`,
`BingoCard`, `BookRating`, `BookTagVote`, `ChallengeCompletion`, `Contribution`, `FanCastVote`,
`FiveWordEntry`, `Notification`, `PolicyAcceptance`, `ReadingList`, `SupportTicket`,
`WantedTitle`.

So a TropeLit reader who deletes their account leaves behind their book reviews, their support
messages, their contributions and their reading lists — keyed to a user ID that no longer
resolves. Orphaned, not deleted. The app tells them the account is gone; most of what they wrote
is not.

This is the one item here I would not wait on a lawyer for. Bar Snap and Play Spotter show the
shape; TropeLit needs the same treatment.

### 2.2 Play Spotter already solved the tension you raised

The conflict — *"we have to keep attestations for legal proof but we also have to delete accounts
when requested"* — is resolved in `userRoutes.js:274-277`:

```js
await db.collection("user_consents").updateMany(
    { userId },
    { $set: { userId: anonymizedUserId } }
);
```

The consent row survives as evidence that *somebody* accepted version N on date D from IP X. The
link to a named person is severed. You keep the proof and honour the deletion, because the thing
you need for proof is the acceptance, not the identity.

**Recommendation: make this the rule in all three.** Bar Snap and TropeLit currently leave consent
rows with the real `userId` intact — which happens to retain the evidence, but by omission rather
than by design, and without severing the link. That is the worst of both: you have not deleted it
and you cannot say why you kept it.

### 2.3 Bar Snap deletes records you may be required to keep

`auth.routes.ts:307-329` deletes `PurchaseRecord`, `Subscription`, `SponsoredPlacement` and the
`Advertiser` row outright on request.

Those are **transaction records**. Tax and accounting retention generally runs several years and
does not bend to a deletion request; the usual answer is that financial records are retained under
a legal obligation while the *personal* fields on them are minimised. Worth putting in front of
the lawyer alongside §2.2 — it is the same question with the answer pointing the other way.

### 2.4 A deletion receipt should be standard

Play Spotter emails one (`accountDeletionEmail` in `templates/authEmails.js`) listing what was
removed. It is the only artifact the person keeps proving they asked and you complied. The other
two send nothing.

---

## 3. Do you need a breach notification plan?

**Yes — but a short one, and the reason is not the data, it is the deadline.**

The data you hold is not the worst case. No card numbers, no SSNs, no health records. Under most US
state statutes the notification trigger is a name combined with something like a government ID,
financial account or medical record, and you hold none of those.

But two things make a written plan worth having anyway:

1. **Email address combined with a credential or security question is a trigger in a number of
   states**, and Firebase holding the password does not obviously put you outside that.
2. **The deadlines are short and they start before you understand what happened.** Several states
   run "without unreasonable delay" with an outer bound (commonly 30–45 days); GDPR is **72 hours**
   to the supervisory authority if any EU/UK user exists. You will not design a process inside 72
   hours while also containing an incident. That is the entire argument for writing it down now.

There is also a version of this that is not legal at all: your Google Cloud Storage bucket holds
photographs the public uploaded. If that bucket is ever exposed, the question "who do we tell and
how fast" arrives regardless of what any statute requires.

### 3.1 The plan

Deliberately one page. A plan nobody can execute at 2am is not a plan.

**Roles.** With one person, the role is "you". Name a backup anyway — someone who can reach the
Stripe, Firebase, Vercel and Cloudflare consoles if you cannot.

**Step 1 — Contain (hour 0).** Stop the bleeding before understanding it. Rotate the credential
if one leaked. Revoke Firebase sessions. Take the affected surface down if that is what it takes.
Do not delete logs — they are the only record of scope.

**Step 2 — Preserve (hour 0-1).** Snapshot logs and the affected database before any cleanup.
Cleanup destroys the evidence you need for step 3, and step 3 decides whether the clock in step 5
has started.

**Step 3 — Scope (hour 1-24).** Answer three questions in writing:
- Which app, which collection or bucket?
- Which of the §1 data types, for how many people, in which jurisdictions?
- Was it accessed, or only exposed? (These have different consequences and the difference is
  usually in the logs you preserved in step 2.)

**Step 4 — Decide (by hour 48).** Notification is a legal judgement, not an engineering one. This
is the point where you call the lawyer, with step 3's written answer in hand. Do not wait until
step 3 is perfect.

**Step 5 — Notify (as counsel directs).** Draft text in §3.2. If EU/UK users are in scope, the
72-hour authority clock started when you became aware — at step 1, not step 4.

**Step 6 — Write it up (within two weeks).** What happened, what you changed, what would have
caught it. Attach it to the handoff, the way §6 of the sponsorship handoff records its own
failures.

### 3.2 Notification template

Fill the brackets. Send from the affected app's own domain through the same branded layout as
everything else — a security notice arriving from an unrecognised sender is a security notice
people delete.

> **Subject:** [App]: an incident affecting your account
>
> Hi [name],
>
> On [date] we discovered [plain description: "a misconfiguration that made uploaded photographs
> readable without a link"]. We are writing because your account was among those affected.
>
> **What was involved:** [the specific §1 data types. Name them. Do not write "certain
> information".]
>
> **What was not involved:** [name these too — "no payment card details, which we never store" is
> the most reassuring true sentence you have.]
>
> **What we have done:** [containment, in past tense, with the date.]
>
> **What you should do:** [only if there is genuinely something. If there is not, say "There is
> nothing you need to do" — inventing a task to look responsible wastes the reader's attention.]
>
> Questions go to [support@ for that app] and reach a person.
>
> Lucht Applications LLC
> [App name]
> 18053 Lillian St, Omaha, NE 68136, United States

### 3.3 Two things to raise with counsel

- **Do you have EU/UK users?** The apps do not appear to geo-restrict. If yes, the 72-hour clock
  and a much longer set of obligations apply, and that changes the plan above from "short" to "not
  short".
- **Is there a written retention schedule?** §2 shows three different de-facto answers. A schedule
  — this data, this long, this reason — is what makes both deletion and breach scoping mechanical
  instead of improvised.

---

## 4. Security headers — one of six is set

Verified in the repositories on 2026-09-09. All three sites set exactly one security header:
`Cross-Origin-Opener-Policy: same-origin-allow-popups`, added to stop Vercel's default breaking
the Firebase sign-in popup.

| Header | Bar Snap | TropeLit | Play Spotter |
|---|---|---|---|
| `Cross-Origin-Opener-Policy` | ✅ `next.config.js` | ✅ `next.config.ts` | ✅ `vercel.json` |
| `Strict-Transport-Security` | — | — | — |
| `Content-Security-Policy` | — | — | — |
| `X-Frame-Options` / `frame-ancestors` | — | — | — |
| `X-Content-Type-Options` | — | — | — |
| `Referrer-Policy` | — | — | — |
| `Permissions-Policy` | — | — | — |

**Not verified:** what the platform adds on top. Vercel sets some of these at the edge by default,
and I could not check the live responses — this sandbox's proxy returns 403 for outbound requests
to those domains, so the headers I got back were the proxy's and not yours. **Check the real
responses before acting on this table**, with `curl -I https://<domain>/` from your own machine or
any online header checker. It is a two-minute check and it decides how much of the below is
already handled.

Assuming the platform is not covering them, in priority order:

1. **`X-Frame-Options: DENY`** (or CSP `frame-ancestors 'none'`). All three sites have
   authenticated account pages and payment flows. This is the one whose absence has a concrete
   attack: framing `/account` or the advertiser dashboard and overlaying it. Cheap, no
   compatibility risk.
2. **`Strict-Transport-Security`** with a long max-age. Probably already set by Vercel — confirm
   rather than assume.
3. **`X-Content-Type-Options: nosniff`** and **`Referrer-Policy: strict-origin-when-cross-origin`**.
   One line each, no downside.
4. **`Permissions-Policy`** to switch off what you don't use. Play Spotter *does* use geolocation,
   so this one is not copy-paste across the three.
5. **`Content-Security-Policy`** last, and in `Report-Only` first. All three load Firebase, Stripe
   and Google Maps; a strict CSP written blind will break sign-in or checkout. It is the most
   valuable header here and the only one that can take a site down, which is why it goes last and
   goes in report mode.

Headers 1, 3 and 4 belong in each app's own config rather than a shared package — three repos,
three copies, one reviewable list. The same conclusion the email layout reached.

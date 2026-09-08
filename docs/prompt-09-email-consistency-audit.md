# Prompt 9 — Outbound email consistency across Bar Snap, TropeLit, and Play Spotter

**Written:** 2026-09-08 · **Repos:** `bar-snap`, `trope-lit`, `play-place-finder`
**Audited at:** freshly cloned `origin/main` of all three, 2026-09-08.

All three apps are published by **Lucht Applications LLC**. Today a recipient who receives mail
from two of them gets two different companies: different sign-offs, different reply behaviour,
different subject conventions, different promises about when we'll get back to them, and — in one
app — a support address that appears nowhere in that app's own config.

This document is the reconciled target. Each app section lists only what **that** app must change.

**How to read a citation:** every path and line number below was opened and read. Where I am
inferring rather than reporting, I say so in the line itself.

---

## 0. The agreed standard

Everything in §1–§3 is measured against this. It is derived from what the three apps already do,
picking the best-argued existing behaviour rather than inventing a fourth convention.

| Element | Standard | Where it already exists |
|---|---|---|
| **From** | `noreply@<app-domain>` | all three ✅ |
| **Reply-To — support/auth mail** | `support@<app-domain>` | Bar Snap ✅ |
| **Reply-To — advertiser/partner mail** | `advertise@<app-domain>` | Bar Snap ✅ |
| **Sign-off** | `Lucht Applications LLC` newline `<App Name>` | Play Spotter ✅ |
| **Subject** | `<App Name>: <Sentence case>` | TropeLit ✅ |
| **Greeting name** | contact person's name; fall back to business name; then `there` | TropeLit ✅ |
| **Footer — advertiser mail** | Agreement link + `advertise@` + dashboard/portal link | Play Spotter ✅ |
| **Response-time promise on inbound acks** | **2 business days** | TropeLit ✅ |
| **Unsubscribe** | one-click tokenised link on any waitlist/marketing mail | TropeLit, Play Spotter ✅ |
| **Auth mail (verify / reset / password-changed)** | sent by **our** server from our domain, branded | Play Spotter ✅ |
| **Branded HTML shell** | one shared wrapper, every customer-facing email | nobody ✅ |

Two notes on the choices:

- **2 business days over 48 hours.** They are not the same promise. "48 hours" written at 7pm Friday
  expires Sunday evening; nobody is reading partnership inquiries on Sunday. "2 business days" is
  the one we can actually keep, and it is the more conservative of the two we currently make.
- **Contact person over business name.** Play Spotter greets `businessName || contactName`, so the
  same submission produced "Hi Lucht Applications," from Play Spotter and "Hi Dani Lucht," from
  TropeLit. Greeting a company by its legal entity name reads like a form letter. Greeting the
  person who filled in the form does not.

---

## 1. Bar Snap — `bar-snap`

Bar Snap's reply routing is the best of the three and should not be touched. Its problem is that
**two entirely separate email systems live in one backend**, and the customer-facing partnership
track is the unbranded one.

### 1.1 The partnership track has no branding, no footer, and no company name — HIGH

`backend/src/services/supportEmailTriggers.ts:50-62` and
`backend/src/services/adCampaignEmailTriggers.ts:63-90` each define their **own private `wrap()`**
producing a branded shell: copper rule, `Bar Snap` wordmark, dashboard button, footer with
`advertise@` and the site link. Two near-identical copies of the same design, neither exported.

Every partnership email bypasses both and emits raw `<p>` tags:

- `backend/src/services/partnership.service.ts:266` — the lead confirmation
- `backend/src/services/partnershipNegotiation.service.ts:402` — the offer
- `…:443` — the deposit request
- `backend/src/services/offer.service.ts:66` — the waitlist acknowledgement
- `backend/src/services/partnershipLifecycle.service.ts:145, 184, 215`

This is why the two Bar Snap emails in hand look plain while other Bar Snap emails look designed.

**Change:** extract one `wrap()` into a shared module (`backend/src/services/emailLayout.ts`), delete
both private copies, and route every customer-facing send through it.

### 1.2 `agreementFooter()` exists twice, differs, and is missing from the offer — HIGH

- `backend/src/services/partnershipNegotiation.service.ts:32-38` — Agreement link + `advertise@`
- `backend/src/services/partnershipPayment.service.ts:39-47` — the same, **plus** the statement descriptor

Four sends use it. The **offer email** (`partnershipNegotiation.service.ts:402`) and the **lead
confirmation** (`partnership.service.ts:266`) do not — so the two emails that actually put a price
in front of a prospect are the two that never name the Advertiser Agreement.

**Change:** single exported `agreementFooter({ withDescriptor })`; apply to every advertiser-facing
send including the offer and the lead ack.

### 1.3 The lead acknowledgement is the thinnest of the three — MEDIUM

`partnership.service.ts:264-266` sends, in full:

> Hi {name},
> Thanks for your interest in a {tier} partnership with Bar Snap. We received your request and will be in touch shortly.
> The Bar Snap team

No timeframe, no link, no Agreement, no company name. Compare TropeLit (2 business days + private
portal link) and Play Spotter (48 hours + `/account` + entity signature).

**Change:** state 2 business days; link the negotiation portal; add the standard footer and sign-off.

### 1.4 The offer email states a price with no expiry — MEDIUM

`partnershipNegotiation.service.ts:386-400` sends the amount and a private link, and says nothing
about how long the price holds. TropeLit's equivalent says *"This price holds until {date}"* in the
offer itself, with a deliberate code comment at `trope-lit/server/src/controllers/partnership.controller.ts:590-592`
explaining why: a deadline first mentioned in the reminder is a deadline the prospect was never given.

Bar Snap **does** track expiry — `quoteReminderSentAt` / `quoteExpiredAt` at
`partnershipNegotiation.service.ts:364` — and sends a separate reminder. So the data exists and
the offer email simply omits it.

**Change:** state the expiry date in the offer body.

### 1.5 Sign-off carries no company name — MEDIUM

`The Bar Snap team` × 6 across the backend. `Lucht Applications LLC` appears exactly once
(`backend/src/services/dailyDigest.service.ts:231`), and that is an **internal** digest — no customer
has ever seen the operating company's name in a Bar Snap email.

**Change:** `Lucht Applications LLC` / `Bar Snap`.

### 1.6 The waitlist unsubscribe token is generated, stored, exposed — and never emailed — MEDIUM

`backend/src/services/interest.service.ts:67` mints an `unsubscribeToken`; `:92` implements
`withdrawInterestByToken`; `backend/src/routes/interest.routes.ts:125` returns it. The waitlist
acknowledgement at `backend/src/services/offer.service.ts:66-71` instead says:

> To come off the list at any time, reply to this email and ask.

The whole mechanism is built and the email doesn't use it. TropeLit
(`server/src/services/slotOffers.service.ts:276-277`) and Play Spotter
(`server/src/services/advertiserEmailService.js:752`) both send the link.

**Change:** put the tokenised link in the waitlist emails.

### 1.7 Auth email is Firebase's, not ours — MEDIUM

Verification and reset are sent by Firebase directly:

- `website/src/lib/auth-context.tsx:196, 268` — `sendEmailVerification`
- `website/src/app/account/page.tsx:113` — `sendPasswordResetEmail`
- `android/composeApp/src/androidMain/kotlin/com/barsnap/app/auth/FirebaseAuthManager.kt:27, 48`

So the **first email a new Bar Snap user ever receives** is a Firebase-templated message from a
Firebase sender. Play Spotter solved this: generate the link server-side
(`play-place-finder/server/src/routes/authRoutes.js:187-192`) and send our own branded email.

*Flagged, not specified:* this is a bigger change than the rest of §1 and touches the auth flow on
three clients. **Sequence it after §1.1–§1.6** — or tell me to leave Bar Snap on Firebase auth mail
and I'll take it off the list.

### 1.8 Subjects are Title Case; the standard is sentence case — LOW

`Bar Snap: Campaign Received, Under Review`, `Bar Snap: Your Campaign Is Live`. Two outliers break
even Bar Snap's own pattern: `We received your Bar Snap partnership request`
(`partnership.service.ts:266`, no prefix) and `Bar Snap: you're on the waitlist`
(`offer.service.ts:67`, lowercase).

**Change:** `Bar Snap: Sentence case` throughout.

---

## 2. TropeLit — `trope-lit`

TropeLit's copy is the best-written of the three. Its problem is routing: **the app tells advertisers
to reply, and their reply goes to the wrong inbox.**

### 2.1 Every email replies to `support@` — including advertiser mail — HIGH

`server/src/services/email.service.ts:238` hardcodes `replyTo: config.email.replyTo`, which resolves
to `support@tropelit.com` (`server/src/config/index.ts:91`). **There is no per-send override** — I
grepped the whole server for `replyTo` and it appears in exactly three places, all shown here.

`config.email.advertise` is defined at `server/src/config/index.ts:94` and **is never read anywhere
in the server.** That grep returned nothing.

The consequence, exactly: `server/src/services/adReceipt.service.ts:154` signs 18 advertiser emails
with

> Questions? Reply to this email or write to advertise@tropelit.com.

The Reply-To header on that same message is `support@tropelit.com`.

**Change:** add an optional `replyTo` to the `sendEmail` signature (Bar Snap's
`backend/src/services/email.service.ts:28` is the model, including the reasoning comment); pass
`advertise@` from every advertiser and partnership send.

### 2.2 The advertise address is hardcoded a second time — MEDIUM

`server/src/services/adReceipt.service.ts:12` declares `ADVERTISER_CONTACT_EMAIL = 'advertise@tropelit.com'`
as a literal, independent of `config.email.advertise`. Two sources of truth; changing the env var
silently updates one of them.

**Change:** delete the literal, import from config.

### 2.3 Sponsorship emails carry no Agreement link and no footer — HIGH

`advertiserSignoff()` (`server/src/services/adReceipt.service.ts:150-157`) attaches the Agreement
link and contact address. It is used in `ad.controller.ts`, `campaignLifecycle.service.ts`,
`adReceipt.service.ts`, and `stripe.webhook.ts`.

`server/src/controllers/partnership.controller.ts` — which sends the inquiry ack (`:194`), every
negotiation reply (`:596`), and the decline (`:682`) — calls it **zero times**. Verified by count.

So TropeLit's *self-serve* advertisers get the Agreement on every notice, and its *negotiated
sponsors* — the larger deals — never see it in email at all.

**Change:** apply `advertiserSignoff()` to all three partnership sends.

### 2.4 Sign-off is inconsistent within the app — MEDIUM

`TropeLit Team` × 87. `Lucht Applications LLC` appears three times, one of which is the customer-facing
decline at `server/src/controllers/partnership.controller.ts:682`. So the **only** sponsorship email
naming the operating company is the one telling someone no.

**Change:** `Lucht Applications LLC` / `TropeLit` everywhere.

### 2.5 Signup verification is Firebase's; the resend is ours — MEDIUM

- `website/src/components/auth/AuthForm.tsx:97` — signup fires Firebase's default `sendEmailVerification`
- `server/src/routes/user.routes.ts:141-156` — the resend sends our own branded email

Same action, two different emails, and the unbranded one is the one every new user gets first. The
same split applies to password reset: `website/src/components/auth/AuthForm.tsx:163` and
`android/app/src/main/kotlin/com/luchtapplications/tropelit/ui/auth/AuthViewModel.kt:80` both use
Firebase's default, and TropeLit has **no** server-side reset email at all.

**Change:** route signup verification through the existing server endpoint; add a server-side reset
email matching it.

### 2.6 The Android app publishes no contact address anywhere — MEDIUM

Grepping the entire `android/` tree for any `@…com` address returns **nothing**. `TropeLitMenu.kt:112`
opens `https://tropelit.com/privacy` in a browser instead. A user inside the TropeLit app has no way
to reach support without leaving it.

Bar Snap (`android/…/util/ContactEmails.kt`) and Play Spotter (`android/…/util/MarketingLinks.kt`)
both ship a contact constants file.

**Change:** add `ContactEmails.kt` mirroring Bar Snap's, and surface `support@` in the app.

### 2.7 The Android app has no email-address constants to keep in sync — LOW

Follows from §2.6. Bar Snap's file carries `// keep in sync with website/src/lib/contactEmails.ts`;
Play Spotter's carries a longer note naming both counterparts. TropeLit has neither file nor a
website-side module — its addresses are scattered across `website/src` as literals (6 × `advertise@`,
4 × `support@`, 2 × `privacy@`).

**Change:** add `website/src/lib/contactEmails.ts` and the Kotlin mirror.

---

## 3. Play Spotter — `play-place-finder`

Play Spotter has the best email *content* — entity signature, dashboard link, contact line, Agreement
URL, working unsubscribe. Two defects undercut all of it.

### 3.1 Advertiser emails say "reply to this email" and have no Reply-To header — HIGH

`server/src/services/notificationService.js:134-135`:

```js
const mailOptions = { from: EMAIL_FROM_ADDRESS, to, subject, text: textContent, html: htmlContent };
if (options.replyTo) mailOptions.replyTo = options.replyTo;
```

**No default.** Of 18 `sendEmail` call sites outside tests, exactly **one** passes `replyTo` —
`server/src/services/advertiserEmailService.js:305`, the admin free-text message.

Every templated advertiser email goes through `:280`, which passes nothing. The intake quote —
the "Your Play Spotter partnership quote is ready" message — goes through `:720`, which passes
nothing. Auth (`routes/authRoutes.js:192, 287, 305, 325`), support (`routes/supportRoutes.js:174`),
account deletion (`routes/userRoutes.js:295`) and ad-free (`services/adFreeEntitlementService.js:202`)
all pass nothing.

Meanwhile `CONTACT_LINE` (`services/advertiserEmailService.js:49`) is appended to all ~40 advertiser
templates via `EMAIL_SIGN_OFF` (`:203`):

> Questions? Reply to this email or contact advertise@play-spotter.com.

`EMAIL_FROM` is `noreply@play-spotter.com` (`server/src/config/contactEmails.js:9`). **Every one of
those replies is going to a no-reply mailbox.** Both other apps hit this and guarded against it —
Bar Snap at `backend/src/services/email.service.ts:200-203`, TropeLit at
`server/src/services/email.service.ts:235-238`, each with a comment saying precisely why.

**Change:** default `replyTo` to `SUPPORT_EMAIL` inside `sendEmail`; pass `ADVERTISE_EMAIL` from
every advertiser, intake, and waitlist send.

### 3.2 `admin@play-spotter.com` is published as the public support address — HIGH

Two user-facing surfaces:

- `website/app/support/page.js:142` — "Or email us directly at admin@play-spotter.com"
- `android/composeApp/src/commonMain/kotlin/org/community/playgroundfinder/ui/screens/SupportTicketScreen.kt:209` — "Or email us at admin@play-spotter.com"

`admin@play-spotter.com` **is not in `server/src/config/contactEmails.js`** — that file defines only
support / privacy / advertise / noreply. Elsewhere in the repo the same string is the seeded *admin
test account* (`website/tests/e2e/ads-admin-payments.spec.ts:10` and two others).

Both literals are hardcoded, bypassing `website/app/lib/contactEmails.js`, which exists for exactly
this and exports `mailtoSupport()`.

*I have not verified whether `admin@play-spotter.com` routes anywhere.* If it does not, support mail
from these two surfaces is being lost outright. Either way it should not be the published address.

**Change:** both surfaces to `SUPPORT_EMAIL`, sourced from the config module rather than retyped.

### 3.3 Advertiser emails greet people by their ad's headline — HIGH

`server/src/services/advertiserEmailService.js:319-320`:

```js
const adLine = data.adDisplayName != null ? String(data.adDisplayName).trim() : '';
const name = advertiser.businessName || adLine || 'Advertiser';
```

When an advertiser record has no `businessName`, all ~40 templates open with the **ad headline**:
"Hi Free Coffee Tuesdays,". The intake path at `:684` uses `businessName || contactName || 'there'` —
correct — so the two paths disagree, and the wrong one covers more emails.

**Change:** `contactName || businessName || 'there'` in both, per §0. Never `adDisplayName`.

### 3.4 Advertiser emails are plain text; auth emails are richly designed — MEDIUM

`server/src/templates/authEmails.js` is a full branded HTML template — teal palette, wordmark, rounded
card, CTA button, plain-text fallback. It is used **only** for verify / reset / deletion / password-changed.

Every advertiser email calls `sendEmail(to, subject, body)` with the 4th argument (`htmlContent`)
omitted — `:280`, `:720`, `:784`. So the ~40 emails carrying money, quotes and Agreement links are
plain text, while "Confirm your email" is the polished one.

**Change:** extend `authEmails.js`'s `wrapHtml` into a shared shell and give advertiser mail an HTML
part alongside the existing text.

### 3.5 Subjects omit the app name on three templates — MEDIUM

Play Spotter subjects embed the app name inline (`Your Play Spotter partnership quote is ready`).
The three waitlist templates don't name it at all: `You're on the Regional Partner waitlist`,
`A Regional Partner slot just opened up`, `The Regional Partner slot for your area is open`
(`services/advertiserEmailService.js:751, 755, 759`). Nothing in the subject says who is writing.

**Change:** `Play Spotter: <Sentence case>` throughout, per §0.

### 3.6 Android password reset bypasses the branded template — MEDIUM

`android/composeApp/src/androidMain/kotlin/org/community/playgroundfinder/App.kt:1650` calls
`FirebaseAuth.getInstance().sendPasswordResetEmail(em)` directly, while the website goes through
`server/src/routes/authRoutes.js:300-305` and gets the branded one. Same app, same action, two
different emails depending on platform.

**Change:** point the Android reset at the server endpoint.

### 3.7 The 48-hour promise — LOW

`services/advertiserEmailService.js:589` promises a quote "within 48 hours". Per §0 this becomes
2 business days.

---

## 4. Shared work — do this once, apply three times

### 4.1 CAN-SPAM: no physical postal address on any commercial email — HIGH, all three

I grepped all three backends for a postal address (`18053`, `Lillian`, `Omaha`) and for `unsubscribe`
in email bodies. **No app puts a mailing address in any email.**

15 U.S.C. 7704(a)(5) requires a valid physical postal address on any message whose primary purpose
is commercial. Transactional receipts are exempt; these are not clearly transactional:

- `play-place-finder/server/src/services/advertiserEmailService.js:554` — "You're halfway through your campaign: **20% off your next booking**"
- `…:564` — "Extend your regional sponsorship"
- `…:568` — "free Inline Listing month"
- `bar-snap/backend/src/services/adCampaignEmailTriggers.ts` — "{n}% Off Your Next Campaign"
- `bar-snap/backend/src/services/offer.service.ts:66` — waitlist mail
- `trope-lit/server/src/services/slotOffers.service.ts:276` — waitlist mail

The DMCA agent filing already publishes the address, so it is not a new disclosure:
`bar-snap/website/src/lib/contactEmails.ts:23-27` — 18053 Lillian St, Omaha, NE 68136.

**Change:** postal address in the footer of every promotional email in all three apps.

*I am not a lawyer and this is a read of the statute against the code, not legal advice.* The factual
half — that no email in any of the three carries a postal address — is verified.

### 4.2 One layout, three brand palettes

Four separate HTML shells exist today: two in Bar Snap (`supportEmailTriggers.ts:52-62`,
`adCampaignEmailTriggers.ts:74-88`), one in Play Spotter (`templates/authEmails.js:13-37`), none in
TropeLit. Same structure, different colours — Bar Snap copper `#b87333`, Play Spotter teal `#00838f`.

**Change:** identical markup per app, palette as the only difference. Not a shared package — three
repos, three copies, one reviewable design.

---

## 5. Suggested order

1. **§3.2** — `admin@` is published to users right now and may not route. One-line fix, two files.
2. **§3.1, §2.1** — reply-to. Both apps invite replies that reach nobody.
3. **§3.3** — greeting people by their ad headline.
4. **§1.2, §2.3** — Agreement link missing from the emails that quote a price.
5. **§4.1** — postal address.
6. Sign-offs, subjects, response times (§1.3, §1.5, §2.4, §3.5, §3.7) — one pass per app.
7. HTML shells (§1.1, §3.4, §4.2).
8. Auth email ownership (§1.7, §2.5, §3.6) — largest blast radius, least customer-visible payoff.

---

## 6. What I did not verify

Stated plainly so nobody builds on it:

- **Whether `admin@play-spotter.com` routes anywhere.** Mailbox config is outside the repos.
- **Whether `legal@bar-snap-app.com` has counterparts.** Bar Snap defines `LEGAL_EMAIL`
  (`website/src/lib/contactEmails.ts:5`); TropeLit and Play Spotter define no legal alias. Whether
  that is a gap or deliberate depends on the mailbox setup, which I can't see. Not listed as a change.
- **Rendering.** I read source, I did not send test mail. Claims about *what the code emits* are
  verified; claims about how a client renders it are not.
- **The CAN-SPAM conclusion in §4.1** — see the caveat there.
- **Marketing-site email capture / newsletter,** if any exists outside these code paths.
- **Bar Snap's gig-notification emails** (`backend/src/services/gigNotifications.ts:109, 140`) and
  ambassador mail. They have no counterpart in the other two apps, so they fall outside a
  cross-app consistency pass. They were not audited against §0.

# Prompt for the next session

> ## ⚠️ SUPERSEDED 2026-08-16 — do not paste this as-is
>
> This prompt describes the workstream as of 2026-08-10 and its central claim is now false.
> It says *"Five prompts were dispatched. Nothing has been confirmed as landed."*
> **All eight prompts have landed.** Prompts 1-5 had already landed when this was written;
> 6-8 landed since. See §6 and §7 of `sponsorship-workstream-handoff-2026-08.md` for the
> verified state.
>
> This file is left in place because it is a good record of *how* to work this workstream —
> the register for writing prompts, the rule against asserting unrun searches, the instruction
> to treat a session's pushback as right until disproven. Those all still hold. Only the
> status claims are stale.
>
> **It has now misled two consecutive sessions**, both of which read it, believed nothing had
> landed, and wrote work against code that was already fixed. If you are picking this up: read
> the handoff's §6 and §7 first, then re-verify against the repos before trusting either
> document. Status written down is stale the moment a branch merges; the repo is the only
> source of truth.
>
> There is one genuinely open question, in §7: prompt 6 named three surfaces and only the home
> page was wired. Whether the recipes page and the Kotlin `AdAdjacency` were left out
> deliberately needs the product owner to say.

---

Paste everything below the line into a new session.

---

You are picking up a coordination workstream across three app repos: **Bar Snap**, **TropeLit**,
and **Play Spotter** (`play-place-finder`). This session is the coordinator — it writes and
reconciles prompts, it does not edit the app repos. Those live in their own sessions.

**Read `docs/sponsorship-workstream-handoff-2026-08.md` in this repo first.** It carries the three
product rules, the reference implementation, what was dispatched to each app session, the open
pricing decision, and the verification failures that shaped how these prompts are written. Do not
start work before reading it.

## Where things stand

Five prompts were dispatched. Nothing has been confirmed as landed. Your job is to close the loop
on them, in this order:

1. **Collect what came back.** The product owner will relay each app session's report. Read each
   one for premise failures before reading it for progress — one prompt has already shipped with a
   false premise that a receiving session caught, and that catch was correct. If a session pushes
   back on a premise, **assume it is right until you have verified otherwise yourself**, then say
   so plainly and rewrite the prompt rather than defending the original.

2. **Reconcile across apps.** The three rules in the handoff are meant to be identical everywhere.
   If Bar Snap and TropeLit solve the same rule differently, that divergence is worth a sentence to
   the product owner, not a silent pick.

3. **Surface the pricing decision.** TropeLit's `home_featured` price disagrees three ways
   (Android 45 / web 99 / live $18 a month). No session should guess. This needs the product owner
   to name the authoritative source, and until they do, the TropeLit copy work is partly blocked.

4. **Check the test plans that come back for honesty, not completeness.** Each app session was
   asked to produce `docs/manual-test-plan.md` and to state explicitly what it could not verify and
   what needs a developer. A plan with no gaps listed is a plan that skipped the hard parts — Play
   Spotter in particular has no ad-free tier to test, so a Play Spotter plan claiming full ad-free
   coverage is wrong.

## How to work

Verify every file path and line number before you cite it. Line numbers in the handoff were correct
when written and have almost certainly drifted. **Do not assert the result of a search you did not
run** — this exact failure happened in the previous session, went into a dispatched prompt as a
stated fact with a confirming command attached, and was caught by the receiving session.

Match the previous session's register in prompts you write: name the file and line, state what the
code currently does, state what it should do, and say which parts you verified versus inferred.
Prompts go to sessions that will act on them literally, so an overstated claim becomes wrong work.

When you are unsure whether something is a defect or intended, ask the product owner rather than
specifying a fix. Two of the previous session's objections — one about ad-free gating, one about
impression liability — were retracted after the product owner explained the intent. Header
sponsors sell placement, not clicks; that is settled and should not be re-litigated.

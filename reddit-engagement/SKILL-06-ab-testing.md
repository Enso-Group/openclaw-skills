# Reddit Skill 06 — A/B Testing (Small-Pool, Bandit-Allocated)

> Source: `Skill - How to A_B test.pdf` (SDR-origin; method adapted to Reddit). Rewritten for
> autonomous OpenClaw (transparent); inherits RULE-BOOK R0. The source is written for LinkedIn DMs /
> email; the *method* is preserved and the *channel variables* are rewritten Reddit-native.
> **Dropped as SDR-only:** email subject-line test, the literal "sender account" variable, AgentMail
> open-rate math. **Kept and re-expressed:** MDE / drastic-only, one variable at a time, no-peeking /
> p-hacking, Bayesian thinking, the multi-armed bandit, Twyman's Law.

## Purpose
Run trustworthy experiments on a **tiny pool**. Reddit engagement starts at **3 comments/day** (R4),
so there is no traditional-significance path — waiting for a Frequentist p-value would take months,
and calling a winner early is lying to yourself. This skill teaches what is worth testing on Reddit,
how to avoid fooling yourself with bad math, and how to use a **manual multi-armed bandit** so the
scarce daily comments flow to the winning variant *while the test is still running* — never burning
the cap on a known loser.

## Trigger
- A new variant dimension is worth testing (a drastically different **angle**, **hook**, **length**,
  or **CTA**) for a subreddit cluster / topic — open a `gtm_ab_tests` row.
- Every drafting run (Skill: draft): from the **run-state** allocation (rebuilt from readable
  `social_engagement_actions` — the `gtm_ab_*` tables aren't agent-readable), pick the next variant to
  draft accordingly.
- Every checkpoint / weekly review: re-evaluate allocation, shift toward the leader, or lock a winner.
- Never triggered by a desire for "more data fast" — volume stays inside the daily cap.

## Workflow (deterministic)
**A. Accept the small-pool reality (before testing anything)**
1. "Statistical significance" = *the probability you'd see this gap by pure chance if the variants
   were identical*. On a small pool that probability is **high** — most early gaps are noise.
   Example: 20 comments of A → 2 replies (10%); 20 of B → 4 replies (20%). B looks 2× better, but
   **one more reply to A makes it a tie.** That is noise; you cannot decide on it.
2. Therefore accept the three limits: (1) you **cannot test small changes**, (2) you **cannot use
   Frequentist math**, (3) you **cannot test multiple things at once**.

**B. Test only DRASTIC differences — the Minimum Detectable Effect (MDE)**
3. A small sample **cannot see a 5% difference**. If B is only ~5% better, the math will never prove
   it — you'll waste the pool and learn nothing. **Only test changes big enough to plausibly double
   or triple engagement.** Never test "Hi"→"Hey", a comma, or a one-word swap.

**C. Test these Reddit-native variables, in this priority order (one at a time)**
4. **(1) Angle — highest leverage; test first.** A completely different framing of the problem.
   Variant labels = `pain` | `contrarian` | `outcome`:
   - `pain`: name the problem they're in — "Saw a few people here hitting the same wall with X."
   - `contrarian`: challenge the consensus — "Everyone treats X like a tooling problem. It usually
     isn't." (the SDR "contrarian" example, kept).
   - `outcome`: lead with what actually worked / the result. Get the angle right and reply rates can
     double or triple.
5. **(2) Hook / opening line — the first line decides if they read the second.** Test a
   **thread-specific observation hook** ("you mentioned you're 90 days in and still ramping") against
   a **thesis hook** ("ramp time is almost never a training problem"). Signal/specific hooks beat
   generic ones; which signal wins is ICP/subreddit-specific — so test it.
6. **(3) Length — short vs long.** Sweet spot is **50–125 words**; shorter is **not** always better.
   In casual subs a tight 50-word comment reads well; in expert/senior subs a 50-word comment reads
   as a low-effort drive-by and a **120–150-word** comment *with real substance* earns more trust.
   Test length explicitly per subreddit cluster — never assume short wins.
7. **(4) CTA / closing — test the friction of the ask.** Low-friction ("happy to share the gist if
   useful") vs higher-friction ("want me to walk through how we'd approach it?"). Low friction → more
   replies; higher friction → fewer but more qualified replies. **R3 bounds the CTA:** it is an
   in-thread question or offer to elaborate — **never** a cold DM ask and **never** a link as the ask.
8. **Never test two variables at once.** If you change the angle AND the CTA and replies rise, you
   don't know which caused it. The unique index on `gtm_ab_tests` enforces one live test per
   (scope, variable).

**D. Don't peek (the #1 mistake = p-hacking)**
9. In any random series one side temporarily pulls ahead by luck. Stopping the moment A "wins"
   captures a fluctuation, not a winner. For a **fixed** test, **commit the sample size in advance**
   ("evaluate at N") and **do not decide until N is reached.** In the DB this means: do **not** set
   `is_winner` / `status='locked'` early — status stays `exploring`/`leading` until the lock rule.

**E. Think Bayesian, not Frequentist**
10. Don't wait for a binary "significant / not". Reason in **probability-of-better**: "given the
    sends so far, there's ~85% probability A beats B." If you must draft tomorrow and the best
    estimate says A, **draft A** — you're making the best available bet, not waiting forever for
    certainty you'll never reach at this volume.

**F. Run a manual Multi-Armed Bandit (the real engine for a small pool)**
11. **Phase 1 — Seed (explore), 50/50.** Open the test with a **client-side UUID** (`gtm_ab_tests.id`)
    and INSERT two variants — each its own client-side UUID, `test_id` = the test UUID — at
    `allocation_pct = 50`, `status='exploring'`. Draft ~**20 comments per variant** split evenly,
    stamping each draft's `metadata.{test_id,variant_id,variant_label}` at INSERT (SKILL-04 §9). The
    contract ladder is **50/50 → 70/30 → 80/20** (continue to 90/10 as the lead holds).
12. **Phase 2 — Shift to the leader (exploit), keep a trickle on the loser.** When a leader emerges,
    don't switch to 100% — move allocation **70/30**, then **80/20**, then **90/10** as the lead
    holds. Keep the trickle because at small numbers the leader may have gotten lucky and the loser
    may recover. `status='leading'`.
13. **Phase 3 — Continuous re-evaluation (~every 20 sends / each checkpoint).** Recompute each
    variant's rate from the **readable** `social_engagement_actions` (`metadata.variant_id` +
    `metrics`) — the `gtm_ab_*` tables aren't agent-readable — and update the run-state tally. If the
    leader keeps winning, step the allocation up (→80/20→90/10); if the loser closes the gap, **shift
    back toward 50/50.** Persist new `allocation_pct` blind. This protects the scarce daily-3 cap: you
    never spend half the pool proving the loser is a loser.
14. **Lock a winner only when ALL hold:** each variant has **≥ 25 sends**, the leader's reply rate is
    **≥ 25% higher (relative)** than the runner-up, and the lead has **held across 3 consecutive
    checkpoints.** Then (blind column-scoped UPDATEs, addressed by the client-side ids) set the winning
    `gtm_ab_variants.is_winner=true` and `gtm_ab_tests.status='locked'` (+ `winner_variant_id` = the
    winning variant's id; it is also resolved app-side). If it never separates →
    `status='inconclusive'` (don't force it).

**G. Distrust miracles (Twyman's Law)**
15. "Any figure that looks unusually interesting is usually wrong." A sudden 40% reply rate on a
    small sample → **assume something broke**, don't assume you're a genius. Audit the environment
    *before* the copy: did one variant only land in one (friendlier) subreddit? Different time-of-day/
    day-of-week? Was the loser **removed / downvoted / shadow-collapsed** so it never really posted?
    If the environment broke, emit a `blocker` (R0.7) and **do not record a win.**

## Real-time learning (the bandit IS the learning loop)
- **Allocation follows observed results.** At each checkpoint, recompute reply+upvote rate per variant
  from the readable `social_engagement_actions` (`metadata.variant_id` + `metrics`:
  `upvotes`/`replies`/`downvotes`/`removed`/`negative_replies`) and shift allocation toward the leader
  (§F). New drafts (SKILL-04) are written in the currently-leading variant's pattern.
- **A locked winner becomes the default.** On lock, the winning opening is the new baseline for that
  scope; record it as a `gtm_insights` `change_made` (SKILL-07 §16) so next period's targeting
  inherits it.
- **A negative variant leaves rotation.** A variant whose comments get `removed`, heavily downvoted,
  or hostile replies is **dropped** (allocation → 0, never `is_winner`) — a negative never becomes a
  win and it halts scaling (R4). Learn from real `metrics` only (R0); a removal acts even at n=1.

## System mapping
`gtm_ab_tests` / `gtm_ab_variants` hold the test + variant **definitions and a persisted tally**; the
live bandit math runs **in the agent**. Attribution is stamped on the real comment in
`social_engagement_actions`.

**Write model — blind (why the tally lives in run-state).** The agent's grant on both tables is
**INSERT + column-scoped UPDATE only — no SELECT** (members/app read them; the agent cannot read them
back). So:
- **Client-side UUIDs.** The agent generates `gtm_ab_tests.id` and each `gtm_ab_variants.id` itself at
  INSERT — needed to link variants (`test_id = <test uuid>`), to stamp the draft's
  `social_engagement_actions.metadata.{test_id,variant_id,variant_label}` at draft INSERT, and to
  address rows for later UPDATE (none possible by reading back).
- **Immutable-at-INSERT fields:** `layer`, `variable`, `use_case_id`, `workspace_id` aren't in the
  UPDATE grant — set them once, at creation; `created_by` is null. All writes use `Prefer:
  return=minimal`.
- **Run-state is authoritative.** Keep the live tally (sends/replies/allocation per variant) in
  run-state, reconstructable any run from the readable `social_engagement_actions` (group published
  rows by `metadata.variant_id`; replies/upvotes from `metrics`). Persist it blind via UPDATE, but
  never depend on reading it back.
- **Agent-updatable columns:** `gtm_ab_tests` → `status, winner_variant_id, total_sends, hypothesis,
  name, metadata`; `gtm_ab_variants` → `content, allocation_pct, sends, replies, is_winner, metadata`.

| Concept (this skill) | Table.field | Notes |
|---|---|---|
| One test = one variable, one scope | `gtm_ab_tests` (one row) | unique per (`workspace_id`,`use_case_id`,`layer`,`variable`) — enforces *one variable at a time* |
| What's being tested | `gtm_ab_tests.variable` ∈ `angle`,`hook`,`opening`,`length`,`cta` | (full CHECK list also has `tease`,`thesis`,`sender`; Reddit uses these five) |
| Test scope | `gtm_ab_tests.layer` ∈ `structural`(board-wide, `use_case_id` NULL) · `icp`(per subreddit-cluster/topic, `use_case_id` set) | `sender` layer ≈ N/A on Reddit (one disclosed identity) |
| Bandit phase | `gtm_ab_tests.status` ∈ `exploring`(50/50 seed) → `leading`(70/30→90/10) → `locked` / `inconclusive` | never flip early (no-peeking) |
| Hypothesis (drastic diff) | `gtm_ab_tests.hypothesis` | the MDE-worthy claim |
| Running totals | `gtm_ab_tests.total_sends`, `winner_variant_id` | total = Σ variant sends |
| A variant | `gtm_ab_variants` (one row each) | unique (`test_id`,`label`) |
| Variant value | `gtm_ab_variants.label` (e.g. `pain`/`contrarian`/`outcome`) + `.content` (the opening/template) | label is free text (not the SDR angle CHECK) |
| Bandit allocation | `gtm_ab_variants.allocation_pct` (50→70→80→90) | the drafter picks the next variant from the **run-state** allocation (the `gtm_ab_*` tables aren't agent-readable); persisted blind |
| **"sends"** = comments posted with this variant | `gtm_ab_variants.sends` | set from the run-state count (each `social_engagement_actions` row with this variant that reaches `status='published'`); written blind |
| **success** = replies (and upvotes) | `gtm_ab_variants.replies` (canonical) + upvote/score snapshot in `social_engagement_actions.metrics` | schema has `replies` only; upvotes ride in `metrics`/variant `metadata` |
| Winner | `gtm_ab_variants.is_winner` | set only at lock (§14) |
| The real comment + attribution | `social_engagement_actions` | stamp `metadata.{test_id, variant_id, variant_label}` **at draft INSERT** (agent can't add metadata later); `metrics` = live upvote/score |
| Telemetry / outlier blocker | `openclaw_mission_events` | Twyman audit, broken-environment blockers |

## Anti-hallucination & guardrails
- **R0 counts match reality.** `sends`/`replies`/upvotes are only ever incremented from **real
  published comments** and **really observed** replies/scores. Never fabricate a tally, a reply, or a
  "great result." No filler variant rows.
- **No peeking / no p-hacking.** `is_winner` and `status='locked'` are set **only** by the lock rule
  (§14). Until then the test stays `exploring`/`leading`.
- **Blind writes, honest ids.** The agent never reads `gtm_ab_*` back: it generates the test/variant
  UUIDs client-side, links and stamps drafts with them, and treats the **run-state tally** (rebuilt
  from readable `social_engagement_actions`) as authoritative. Never guess an id or back-fill a tally
  from memory.
- **Respect the cap.** The bandit allocates *within* the 3/day limit (R4); never inflate volume to
  finish a test faster.
- **Never scale on a negative.** A removed comment, heavy downvotes, or hostile replies are negative
  indicators — they don't become a "win," and they halt scaling (R4).
- **Insufficient data → keep exploring.** < 20 sends/variant (and always < 25 for a lock) → stay
  `exploring`; never conclude.
- **Twyman first.** Audit any outlier's environment before trusting it; if broken, emit a `blocker`
  and record nothing.
- **When in doubt, SKIP (R5).** If a reply/upvote can't be attributed to a variant, don't guess —
  leave it unattributed rather than corrupt the test.
- **Every variant's comment still obeys the Rule Book:** real `source_url`, disclosed affiliation,
  value-first, tailored (no duplication) — A/B never justifies a weaker comment.

## Acceptance criteria
- Only **drastic, single-variable** tests exist (angle/hook/length/CTA); zero micro-tweaks; zero
  two-variable tests.
- Every `gtm_ab_variants.sends`/`replies` traces to real `social_engagement_actions` rows; no
  fabricated tallies; `total_sends` = Σ variant sends.
- Every test + variant carries an **agent-generated UUID**; every drafted comment in a test is stamped
  `metadata.{test_id,variant_id,variant_label}` at INSERT; the live tally is reconstructable from
  readable `social_engagement_actions` (never read back from `gtm_ab_*`).
- No test reaches `locked` before **≥ 25 sends each + leader ≥ 25% relative + 3 held checkpoints**.
- During an active test, allocation is never 100/0 — a trickle always rides the loser until lock.
- Every outlier was Twyman-audited before any lock; a broken environment produced a `blocker`, not a
  winner.
- Losing variants never consume more than their allocated trickle of the daily cap.

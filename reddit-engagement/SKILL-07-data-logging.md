# Reddit Skill 07 — Data Logging (Structure · Discipline · Insight)

> Source: `Skill- How to build a good database.pdf` ("How to Work With Data"). SDR-origin method, the
> *engineering principles preserved verbatim*, the *tables/fields re-expressed onto the live schema*.
> **Dropped as SDR-only:** AgentMail reply webhook, the `Prospects`/`Messages`/`linkedin_dm`/`email`
> names. **Kept:** decide-the-questions-first, 3NF, controlled vocabulary, referential integrity, real
> primary keys, log-the-moment, unknown→null/"unknown", dedup, mark-stale-not-delete, the four ways
> data lies, the observed/evidence/why/changed insight contract. Inherits RULE-BOOK R0.

## Purpose
Make the engagement data **queryable and trustworthy** — a daily operational discipline, not a
reporting afterthought: decide what you're learning, structure it right, **log every event the moment
it happens**, keep quality from rotting, query for patterns, promote patterns that hold into written
insights. Bad data work = a store of numbers you can't use, or worse, numbers that quietly lie.

## Live schema = what to log WHERE (do not invent columns)
The four operations this skill governs (LIVE CONTRACT — use these exact keys):

| Operation | Call | Shape |
|---|---|---|
| **Per-action metrics** | `PATCH social_engagement_actions.metrics` · `Prefer: return=minimal` | jsonb keys: `upvotes`, `downvotes`, `comments`, `shares`, `impressions`, `removed` (bool flag), `negative_replies` (count) |
| **Insight (knowledge)** | `POST gtm_insights` | `{ workspace_id, use_case_id?, title, observed, evidence, hypothesis, change_made, status:'active' }` |
| **Source yield** | `POST gtm_sources` | `{ workspace_id, platform:'reddit', query, prospects_found, qualified }` |
| **Telemetry** | `POST openclaw_mission_events` | `{ workspace_id, mission_id?, event_type:'progress'\|'blocker'\|'note', message, payload }` |

**Reply count → the `comments` key.** Reddit's `comments` = the number of replies on the expert's
comment/post (the source's "Replies" metric). **There is no `metrics.replies` key — never write one.**
"Comments posted" (volume vs `daily_cap`) is **not** a metric; it is `COUNT` of own
`status='published'` rows.

**Read vs blind-write (decides the tally + dedup):**
- **Readable:** `social_engagement_actions` — `GET ?status=eq.approved` (drafts to post) + **its own
  rows** (score metrics, dedup, rebuild the tally). This is the agent's only rich **DB** read surface.
  `openclaw_mission_events` is **write-blind** (no agent SELECT) — never GET it; cross-run recall =
  durable **Clawdi memory** (`memory_search`/`memory_add`).
- **Blind append-only** (`Prefer: return=minimal`; the write does not echo the row; **no UPDATE**):
  `gtm_insights` (contract: no UPDATE), `gtm_sources`. Never depend on reading them back.

## Trigger
Continuous — every run that writes or reads engagement data: drafting (append the row) · posting
(stamp the result) · observing a reply/upvote (PATCH `metrics`) · mining a subreddit (POST a
`gtm_sources` yield row) · weekly review (query patterns → promote a 2-period pattern to a
`gtm_insights` row).

## Workflow (deterministic)
**Step 1 — Decide the 5 questions FIRST (before any row).** Be specific. For Reddit:
1. Which **(subreddit cluster × angle × length)** earns the most **replies (`comments`)/upvotes** per post?
2. Does **`pain` vs `contrarian` vs `outcome`** win, and in **which** subreddits? (angle labels = SKILL-06.)
3. Which subreddits/topics yield the best **engaged-reply rate vs removal / zero-engagement noise**?
4. Of comments that **disclosed affiliation**, what share got positive vs negative reception?
5. Which **day/time** produces the most engagement?
> **Every tracked field must serve ≥1 question.** A field that answers none is noise — don't track it.

**Step 2 — Structure correctly (3NF, enforced by the live schema).**
- **1NF — one value per cell.** Never cram `pain+contrarian` into one field, or "replied, wants info"
  into `status`. Multiple values → a separate row.
- **2NF — every field depends on the row's key.** Brand voice / subreddit description / identity bio do
  **not** ride on each comment row — they belong to `gtm_brands`/`gtm_voice_rules`/
  `social_engagement_settings`. A `social_engagement_actions` row holds facts about **that comment only**.
- **3NF — no non-key field depends on another non-key field.** Don't denormalize a subreddit's display
  name onto the action; keep it lean and join. Redundancy → inconsistency → corrupted answers (the
  source's "company name in two tables" failure).
- **Referential integrity.** Honor the real relationships (`workspace_id`, `social_account_id`,
  `mission_id`). **Soft ref to enforce in-agent:** A/B attribution lives in
  `social_engagement_actions.metadata.{test_id,variant_id,variant_label}` (no DB FK) — keep it pointing
  at a real `gtm_ab_*` row (SKILL-06) or leave it null.
- **Real primary keys.** Every row is a `uuid` PK; the **natural dedup handle** is the thread's
  `source_url`/`permalink`/`external_id` — the Reddit analogue of the source's LinkedIn-URL key.
  **Never** identify a thread by a username or a subreddit *name* (not unique) — use the permalink.
- **Controlled vocabulary — fixed values, used VERBATIM.** Any field you filter by uses the schema's
  exact CHECK strings; deviate once ("Pain" vs "pain") and you silently break every filter on it.
  Confirmed values: `status` lifecycle `draft → approved (human) → published | failed` (human-reject →
  `rejected`); `platform:'reddit'`; `gtm_insights.status:'active'`; `event_type` ∈ `progress|blocker|note`;
  angle labels `pain|contrarian|outcome` (SKILL-06). **Read the live CHECK values and match them —
  never free-text a filterable field.**

**Step 3 — Log every event the moment it happens (late data is wrong data).**
- Draft → **INSERT** `social_engagement_actions` now: `status='draft'`, **real `source_url`** (R0.2 — the
  page you actually read this run), `content`, `created_by` null, and
  `metadata.{compliance,validation,variant_*}` stamped **at insert** (blind writes can't be enriched later).
- Human approves → app-side RPC sets `status='approved'`.
- Agent posts → `PATCH {status:'published', external_id, permalink, published_at}` (or `{status:'failed',
  error}`) at the moment of posting.
- Reply/upvote observed → `PATCH metrics` (the exact keys above) at the moment observed.
- Mine a subreddit → `POST gtm_sources` yield row.
- **Log every field; a known unknown beats a silent gap.** Unsure → `null`/`"unknown"`, never
  blank-guess. **R0 exception:** `source_url`/`content` must be **real** — no real source → you don't
  draft, you **SKIP** (R5). Never a placeholder to satisfy a NOT-NULL column.

**Step 4 — Keep the run-state tally (blind writes).** Because `gtm_insights`/`gtm_sources`/
`mission_events` writes are blind (return minimal, no read-back), hold this run's counters **in
memory** as you go, and rebuild cross-run state from the **readable** surfaces: `GET` your own
`social_engagement_actions` (+ `metrics`) and your own recent `mission_events`. Never assume a blind
write returned an id or a running total.

**Step 5 — Idempotency / no double-engage (R0.6).** Before drafting on a thread, `GET`
`social_engagement_actions` for that thread's `source_url`/`permalink`; if a row exists → already
engaged → **SKIP**. (`gtm_sources` can't dedup threads — it's blind/append-only; thread dedup uses the
readable actions table.)

**Step 6 — Maintain quality (it degrades if you don't).**
- **Inconsistency** ("VP of Sales"/"VP, Sales"/"VP Sales" = three values): prevented by controlled
  vocab; audit regularly; on a drift, fix it and note *why*.
- **Staleness — supersede, never delete.** `gtm_insights` is **append-only / no UPDATE**: when an
  insight stops being true, **POST a new insight** that records the change (the append-only log *is* the
  history of "what was true and when"). The agent never deletes; lifecycle/archival is app-side.
- **Dedup of yield:** you can't merge `gtm_sources` rows (blind) — append honestly; thread-level dedup
  is Step 5.

**Step 7 — Query for patterns (a habit, not on-request).** From the readable actions + metrics:
reply(`comments`)/upvote rate by **subreddit cluster × angle × length**; angle performance per
subreddit; source yield (`prospects_found`/`qualified`); timing by day/hour. Don't just read numbers —
ask *what does this tell me I didn't know, and what would I change if I took it seriously?*

**Step 8 — Turn data into insights (only when a pattern HOLDS).** When a pattern has held **≥2
consecutive periods**, `POST gtm_insights` with all four parts → four columns: **observed**, **evidence**,
**why** (`hypothesis`), **what changed** (`change_made`), plus `title`, `status:'active'`, `use_case_id?`.
- *Observation* (fact): "Last 30d, `contrarian` got 9% reply rate vs 3% for `pain` in the r/X cluster."
- *Insight* (knowledge): "In r/X (experienced practitioners) `contrarian` beats `pain` — they already
  feel the pain and resent being reminded; a specific counter-take earns engagement. **Changed default
  angle for r/X to `contrarian`.**" Capturing the *why* matters: if the pattern flips later, you can see
  what you believed and what changed. **One solid insight > twenty observations.**

**Step 9 — Know when the data is lying (note the bias BEFORE concluding).**
- **Sample too small** — 100% on 2 comments is noise; 12% on 200 is signal. A combo with **<20 sends** →
  mark insufficient data, keep sending before evaluating.
- **Confounding** — changed angle AND length the same week → you can't attribute it. One variable at a
  time (SKILL-06); when you can't isolate, note the confound.
- **Survivorship** — the comments with **no reply / removed / downvoted** are data too. Study the silence.
- **Temporal** — compare **like periods** (weekday vs weekend, season). A biased conclusion is worse than none.

## Real-time learning loop (every run)
1. **Read** recent own `social_engagement_actions` + `.metrics` and recent own `mission_events`
   (blockers/removals).
2. **Adapt this run** from real signal — lean into subreddits/angles that earned replies/upvotes; back
   off ones that got removed/downvoted/blocked last run (feeds SKILL-08 validation and SKILL-10 scaling).
3. **Persist** a `gtm_insights` row **only** when the pattern has held **≥2 periods** (Step 8). Emit a
   `progress` event summarizing the adaptation so the operator sees the learning live.

## System mapping
| Concept | Table.field | Notes |
|---|---|---|
| The event log (system of record) | `social_engagement_actions` | readable (approved + own rows); agent appends + PATCHes own status/metrics |
| Reality anchor | `.source_url`, `.content` | real, re-read this run (R0.2) — no placeholder, else SKIP |
| Lifecycle | `.status` (`draft→approved→published\|failed`/`rejected`) | approve is app-side; agent stamps `published`/`failed` |
| Dedup handle | `.external_id` / `.permalink` | Reddit analogue of the LinkedIn-URL PK; check before drafting |
| Live metrics | `.metrics` (jsonb) | `upvotes/downvotes/comments/shares/impressions` + `removed`/`negative_replies`; PATCH `return=minimal` |
| A/B attribution (soft, no FK) | `.metadata.{test_id,variant_id,variant_label}` | stamped at draft INSERT; tally lives in SKILL-06 (`gtm_ab_*`, write-blind) |
| Subreddit/topic yield | `gtm_sources` (`platform,query,prospects_found,qualified`) | append-only/blind; `query` = subreddit/search |
| Knowledge layer | `gtm_insights` (`title,observed,evidence,hypothesis,change_made,status,use_case_id?`) | append-only / **no UPDATE** → supersede by appending |
| Telemetry / failure | `openclaw_mission_events` (`progress\|blocker\|note`) | emit a `blocker` on any logging/query failure (R0.7, SKILL-09) |

## Anti-hallucination & guardrails
- **R0 — data mirrors reality.** Every row reflects something that happened; `source_url` is real and
  re-read this run; never fabricate a metric, reply, upvote, removal, or a "happy user." Counts match:
  cap 3 → ≤3 real drafts, no filler rows.
- **Exact keys only.** Metrics use the contract keys (no phantom `replies`/`score`); controlled fields
  use the live CHECK strings verbatim.
- **Unknown → `null`/`"unknown"`, never blank-guessed.** Real `source_url`/`content` or **SKIP**.
- **Idempotency (R0.6):** check the readable actions table before drafting; never double-engage.
- **Append-only respect:** never UPDATE `gtm_insights`; supersede by appending. Never delete. Keep
  blind-write tallies in-run.
- **Insights only on held patterns (≥2 periods);** note any small-sample/confound/survivorship/temporal
  bias in the row before concluding.
- **On any write/query failure → emit a `blocker` and move on (R0.7).** Never fabricate to fill a gap.

## Acceptance criteria
- The **5 questions** are written and **every tracked field maps to ≥1**.
- Metrics use the **exact contract keys**; reply count is `comments`, never a `metrics.replies`.
- Every row logged **at the moment of its event**; nothing backfilled from memory.
- No NOT-NULL field faked; unknowns are `null`/`"unknown"`.
- **No duplicate engagement**; dedup verified against the readable actions table before insert.
- Stale insights are **superseded by a new append**, never UPDATEd or deleted.
- Insights exist **only** for ≥2-period patterns and carry all four parts (observed/evidence/why/changed);
  relevant data-bias caveats noted.
- The run-state tally is rebuilt from readable surfaces + in-run memory; no reliance on blind writes echoing data.

# Reddit Skill 10 — Performance Tracking & Scaling (acceptance, not karma)

> Source: `OpenClaw Skill 7: Performance Tracking and Evaluation.md`. **Reddit-native** (no SDR
> adaptation). One change from the source: it speaks of a "persona campaign" — in v2 the actor is the
> **disclosed expert** (`IDENTITY.md`), so "is the persona welcomed" becomes "is the disclosed expert
> welcomed." Everything else (the metric set, the removal rule, the locked 3→5→8 ladder, the daily +
> weekly reviews) is preserved in substance. Inherits RULE-BOOK R0 + disclosure.

## Purpose
Close the feedback loop. Measure each engagement by **community acceptance and quality — not vanity
karma** — identify which subreddits welcome the disclosed expert versus reject it, and use that
evidence to govern **when (and only when) volume may scale**. This is the proof gate the Rule Book
points to: volume rises only on a sustained positive pattern, never on a negative.

## Trigger
- **Daily:** evaluate the comments/posts published on the **previous day** (one operational review per run).
- **Weekly:** a strategic review of the last 7 days.
- **Immediately:** whenever any published action shows a **removal** — halt that subreddit at once (don't
  wait for the daily pass).
- Before **any** scale-up decision, and before **adding any new subreddit**.

## Metrics = exact jsonb keys (LIVE CONTRACT — do not invent)
`PATCH social_engagement_actions.metrics` with `Prefer: return=minimal`. The reddit keys and how the
source's metric set maps onto them:

| Source metric (Skill 7) | Stored as | Indicates |
|---|---|---|
| **Comments/posts posted** | `COUNT` of own `status='published'` rows (**not** a metrics key) | adherence to `daily_cap` |
| **Upvotes** | `metrics.upvotes` | the community found it helpful |
| **Replies** (people conversing with the expert) | `metrics.comments` | engagement — **this is the reply count; there is no `metrics.replies` key** |
| **Downvotes** (if visible) | `metrics.downvotes` | off-topic / unhelpful / read as promotional |
| **Removed / deleted** | `metrics.removed` (bool flag) | severe warning: rules violated or read as spam |
| **Negative replies** | `metrics.negative_replies` (count) | qualitative rejection of tone/content |
| (also available) | `metrics.shares`, `metrics.impressions` | reach, if the platform exposes them |

Record **real, observed values only.** Downvotes not visible → `metrics.downvotes:"unknown"` — never
estimated.

## Workflow
1. **Daily acceptance + quality review (not karma alone).** For each action published yesterday, read
   the live result via Composio / the live page and PATCH the full metric set above. From it, classify
   each subreddit as **welcomed** (continue) or **should-cease** (stop). Off-tone / too-promotional /
   too-generic comments are quality failures even when not removed — note them and correct the drafting
   (SKILL-04 / `gtm_voice_rules`).
2. **Removal rule (hard).** A **removed** comment → immediately flag that subreddit **High risk** and
   **halt all activity there** until its rules are re-evaluated (re-run SKILL-01) or a human gives new
   guidance. Concretely: PATCH `metrics.removed=true` on the action; deactivate the target **yourself** —
   PATCH `social_engagement_settings.targets` setting that subreddit's `active=false` (you now have
   column-scoped `targets` write); `POST gtm_insights` (observed:
   removal; change_made: halt this subreddit). **Never keep posting into a subreddit that removed you.**
3. **Decide continue / reduce / stop per subreddit** from step 1. Keep engaging where the expert is
   welcomed; reduce or stop where it isn't.
4. **Gate scaling on a sustained positive pattern — never on karma.** Scaling is **forbidden** just
   because upvotes/karma rose. It is permitted **only** when the account shows a consistent positive
   pattern: comments get **replies** (`metrics.comments`), comments are **not removed**, the expert is
   recognized as **helpful**, and the **same subreddits respond well repeatedly**.
5. **Apply the locked Safe Scale-Up ladder** (`social_engagement_settings.daily_cap`, total across all
   subreddits — not per-subreddit):
   1. Start at **3/day**.
   2. After **≥2 weeks** sustained positive → **5/day**.
   3. After **another ≥2 weeks** sustained positive → **8/day** (ladder ceiling).
   - Never skip a rung; never scale on day-one upvotes (a sudden spike from a young account reads as
     coordinated manipulation — slow, deliberate growth proves a real, long-term human presence).
6. **Never scale on any negative indicator.** A removal, downvotes, negative replies, or unclear
   subreddit rules in the window → **do not scale** (hold or reduce). One negative resets the case for a step-up.
7. **Add new subreddits only after re-running SKILL-01.** Expansion requires a fresh mapping of the
   community's **rules and tone** first; record its yield with a `gtm_sources` row. Never bolt a new
   subreddit onto the cap without that mapping.
8. **Weekly strategic review.** Over 7 days, summarize: what worked, what failed, which subreddits are
   **safest**, which **topics produced the best replies**, which comments felt **too promotional**, and
   what to change next week. Recommend **keep / reduce / increase** volume. **Do not recommend scaling if
   there were any removals, negative replies, or unclear subreddit rules.** Write the conclusions to
   `gtm_insights`.
9. **Record lessons as insights (held ≥2 periods).** A pattern that holds across **2+ review periods** (a
   subreddit that always welcomes, a topic that always lands, an angle that always flops) → one
   `gtm_insights` row (observed / evidence / hypothesis / change_made). These feed SKILL-08 validation
   and next week's targeting.
10. **Autonomy is earned here.** Per R4, posting stays **approve-first** until performance is proven;
    only sustained positive performance justifies recommending a move toward autonomous posting.
    Flipping `posting_mode`/`autonomous` is a human/admin decision — the agent **recommends** it (an
    insight + a SKILL-09 "decide" escalation); it does not self-grant.

## Real-time learning loop (every run)
1. **Read** recent own `social_engagement_actions` + `.metrics` (removals/negatives) — the readable measure
   surface — and **recall** prior blockers via **Clawdi memory** (`memory_search`). `openclaw_mission_events`
   is write-blind (never GET it).
2. **Learn → act this run:** halt subreddits that removed you; reduce where downvotes/negative replies
   appeared; lean into subreddits/angles/topics that earned replies/upvotes. Adjust `daily_cap` **only**
   per the locked ladder, never on a single good day.
3. **Persist** a `gtm_insights` row when a pattern has held **≥2 periods**; emit a `progress` (or `note`)
   event summarizing the scale/hold/halt decision so the operator sees the loop live.

## System mapping (do not invent columns)
| Concept | Where it lives |
|---|---|
| Per-action metrics | `social_engagement_actions.metrics` (jsonb keys above) — readable + `PATCH ` (`return=minimal`) by the agent token |
| Which rows to score | `GET social_engagement_actions?status=eq.published` (own rows), per target = per subreddit/thread |
| Subreddit reception / yield | `POST gtm_sources { workspace_id, platform:'reddit', query, prospects_found, qualified }` (append-only/blind — append a fresh yield row; you cannot UPDATE it) |
| "Halt this subreddit" | the agent itself PATCHes `social_engagement_settings.targets` → that subreddit `active=false` (column-scoped `targets` write) + appends a `gtm_insights` halt row + holds High-risk in Clawdi memory — **not** an UPDATE to `gtm_sources` |
| Scaling volume (3→5→8) | `social_engagement_settings.daily_cap` — read every run at STEP 1; honored, ceiling 8, never exceeded |
| Lessons / weekly review / scale rec | `POST gtm_insights { workspace_id, use_case_id?, title, observed, evidence, hypothesis, change_made, status:'active' }` (append-only; no UPDATE → supersede by appending) |
| New-subreddit expansion gate | re-run **SKILL-01** (live rules + tone) → the agent adds the new (Low/Med-risk) subreddit to `social_engagement_settings.targets` itself (humans can too); High-risk stays inactive |
| Review telemetry | `POST openclaw_mission_events { event_type:'progress'\|'note', payload:{stage:'engagement'} }` |
| Global STOP / proof-gated autonomy (recommend, human sets) | `gtm_pipeline_settings` (`paused`, `autonomous`) · `social_engagement_settings.posting_mode` |

**Flagged mappings:**
- **Who writes `daily_cap`.** Settings are strategy/config; depending on role/autonomy the agent may
  update it directly **or** must recommend it. Default: if config is agent-writable under the workspace's
  autonomy settings, apply the ladder; otherwise escalate the cap change as a SKILL-09 "decide" request.
  Either way the cap is read each run and never exceeded.
- **`targets[].active`** deactivation assumes that sub-field exists on the settings shape; if not
  agent-writable, escalate the halt as a "decide"/"build" blocker (the in-run High-risk memory + the
  `gtm_insights` halt row still prevent re-engagement this run).

## Anti-hallucination & guardrails
- **R0 metrics:** record only real, observed numbers from the live platform/Composio reads. Never invent
  an upvote count, a reply, a removal, or an outcome. Use the **exact metric keys** (reply count is
  `metrics.comments`, not `replies`). Downvotes not visible → `"unknown"`. Counts match reality (R0.3).
- **Acceptance over vanity:** karma alone is never the success metric; quality + community acceptance
  govern (Prime Directive — value supersedes every metric).
- **Removal = halt, always.** A single removal flags the subreddit High and stops drafting there until
  SKILL-01 re-eval / human guidance. No exceptions.
- **Never scale on a negative** (removal, downvotes, negative replies, unclear rules) — R4. Negatives
  hold or reduce volume.
- **Ladder discipline:** 3 → (≥2 wks positive) 5 → (≥2 wks positive) 8; total daily, not per-subreddit;
  never on day-one upvotes; ceiling 8.
- **New subreddits only after SKILL-01.** No expansion without a fresh live rules + tone mapping.
- **Append-only respect:** never UPDATE `gtm_insights`/`gtm_sources`; supersede/append. Keep the
  cross-run tally from the readable actions table + in-run memory.
- **Doubt → don't scale** (R5). Autonomy is recommended, not self-granted (R4); config the agent can't
  write → escalate via SKILL-09.

## Acceptance criteria
- Each published action has the full metric set (acceptance + quality, not karma alone) in
  `social_engagement_actions.metrics` with **real values and the exact keys**; reply count is
  `comments`; unseen downvotes are `"unknown"`.
- Any removed comment flags its subreddit High and **halts drafting there** (target deactivated /
  escalated, `gtm_insights` halt row), and `metrics.removed=true` is set.
- `daily_cap` only rises on the **3→5→8** ladder after **≥2 weeks** sustained positive with **zero**
  negatives; it is honored each run and **never exceeded** (ceiling 8).
- No scale-up ever rides on day-one upvotes or vanity karma.
- New subreddits enter the cap only after a fresh **SKILL-01** mapping.
- The weekly review produces a `gtm_insights` row (worked/failed, safest subreddits, best topics,
  keep/reduce/increase) and **never recommends scaling when removals, negative replies, or unclear rules
  occurred**.
- The run measured from real readable signal (own metrics + blockers) and adapted; any move toward
  autonomous posting is recommended on proven performance, not self-granted.

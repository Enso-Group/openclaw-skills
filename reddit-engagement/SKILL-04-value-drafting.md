# Reddit Skill 04 — Value-Driven Comment Drafting

> Source: "OpenClaw Skill 5: Value-Driven Comment Drafting" — its volume cap, the four drafting
> rules, the human-review payload, the execution prompt, and both Q&As are all preserved below.
> Rewritten for autonomous OpenClaw (transparent model); inherits RULE-BOOK R0 + disclosure; voice
> from `IDENTITY.md`.

## Purpose
Govern **how** the agent writes a Reddit reply (or post): genuinely helpful, correctly formatted,
volume-capped, and **draft-only**. Every output must deliver **standalone value** — useful even if
the reader never opens the profile or touches the product. The agent drafts; a human approves; then
the agent posts (R4). This skill produces exactly one `social_engagement_actions` draft row per
eligible thread and never publishes.

## Trigger
A real, recent thread has passed the upstream monitor + compliance checks (relevant to `topics`,
recent, live subreddit rules re-read this run, safe to participate, **not already engaged**) **and**
the daily cap is not yet spent. Gate, all must hold: `gtm_pipeline_settings` → `paused=false`,
`autonomous=true`, `stages.engagement=true`; `social_engagement_settings.enabled=true`. Any gate
false → do not draft.

## Workflow
1. **Re-check STOP.** Re-GET `gtm_pipeline_settings`; if `paused=true` or `autonomous=false` → stop
   cleanly, draft nothing.
2. **Enforce the volume cap (3/day TOTAL).** The cap is **3 comments per day across ALL
   subreddits/targets — not 3 per subreddit.** *(Why 3, Skill 5 Q1: a young account suddenly posting
   many detailed comments trips spam filters and human suspicion; 3/day is a safe, organic volume
   that builds karma + trust slowly.)* Start value = `social_engagement_settings.daily_cap` (=3),
   bounded by `gtm_pipeline_settings.daily_caps.comments`; effective cap = `min(...)`. GET today's own
   `social_engagement_actions` (agent-readable via SELECT) and count rows with `status in
   (draft,approved,published)`. If count ≥ effective cap → SKIP (stop drafting this run).
3. **Idempotency (R0.6).** Confirm no existing action row for this thread's `target_url`. If one
   exists → SKIP the thread.
4. **Load voice + claims + history.** Read `gtm_voice_rules` (say / never-say / tone), `gtm_brands`
   positioning, and citable `gtm_proof_points` (`is_citable=true`) — cite ONLY these (one read via the
   `gtm_brand_book` view). Then run **Real-time learning** (below) to bias this draft toward what has
   actually worked.
5. **Draft the answer.** Write a concrete, practical reply to the user's **actual question**.
   Constraints (Skill 5 §2):
   - **Standalone value** — actionable and useful on its own (see the checklist below).
   - **No unprompted product mention** — only if the user explicitly asked for tools/recommendations
     AND the subreddit allows it (defer to SKILL-05).
   - **No unprompted links** — same gate as SKILL-05.
   - **Tailored** — written for this thread only; never reuse/duplicate a prior comment (R1).
6. **Shape it like a real comment.** Open by addressing their **actual situation** (no preamble, no
   "Great question"); give the **one specific, actionable answer** (the *how*); add at most **one**
   concrete real detail or example; end naturally — **no** salesy CTA, **no** summarizing-conclusion
   paragraph. One short paragraph or a few tight lines.
7. **Apply the voice** (pulled from `gtm_voice_rules`, not hardcoded). Casual, clear, short — "a smart
   person typing from their phone" (IDENTITY §4). **Banned** (`gtm_voice_rules` never-say + IDENTITY
   §4): hype words (`unlock`, `leverage`, `revolutionize`, `game changer`), corporate tone, em-dashes,
   "ChatGPT voice," multi-paragraph essays with a summarizing conclusion.
8. **Build the human-review payload** (Skill 5 §3 — 4 parts, all required): (a) **subreddit-rules
   summary** quoted from the page re-read this run; (b) **why this thread is relevant**; (c) **risk
   level**; (d) the **final draft comment**.
9. **Write ONE draft row** to `social_engagement_actions` (see System mapping): `status='draft'`,
   `source_url` REQUIRED (the exact page read), `created_by=null`; **never** set `external_id` /
   `permalink` / `published_at` / `approved_by`. Carry the payload as `content` (draft) + `evidence`
   (why-relevant) + `metadata` (rules summary + risk + any SKILL-05/SKILL-06 stamp). Send `Prefer:
   return=minimal`. **Stamp `metadata` NOW** — the agent's column grant can't add it after insert.
10. **Log telemetry.** POST a `progress` event to `openclaw_mission_events` with real counts
    (`created_by` null). Loop to the next eligible thread until the cap is reached.
11. **Do NOT post.** Publishing is a separate, post-approval step (approve_first loop). This skill
    ends at `status='draft'`.

## Standalone-value checklist (a draft must pass ALL — else rewrite or SKIP)
- It answers the **literal** question the user asked.
- It hands over a **specific, actionable** step they could act on today — not generic platitudes.
- It still reads as useful with **any product mention/link deleted** (R3).
- It would plausibly earn an upvote **with the brand removed** — it helps the community, it doesn't sell.

## Real-time learning (read prior outcomes → bias this draft)
The agent can SELECT `social_engagement_actions` (its `gtm_ab_*` tables are write-blind), so it
reconstructs what worked from there and steers the new draft accordingly:
- **Read the metrics.** GET `social_engagement_actions?platform=eq.reddit&status=eq.published&
  select=target_url,content,metadata,metrics&order=published_at.desc`. Each `metrics` carries
  `upvotes`, `replies`, `downvotes`, `removed`, `negative_replies` (real values only; unseen →
  `"unknown"`).
- **Bias toward what worked.** Prefer the **angle / length / hook / opening** patterns that earned
  **upvotes + replies**. The live A/B **winner** for this subreddit cluster is the
  `metadata.variant_label` with the best reply+upvote rate across published rows (group by label — see
  SKILL-06 for the bandit math); draft in that winning pattern.
- **Stop what failed (hard).** A comment that was **`removed=true`** → **stop that pattern now** (that
  angle/topic, and that subreddit) — never repeat it; a removal is a hard negative even at n=1 (R4
  "never scale on a negative"; SKILL-10 halts the subreddit). Heavy `downvotes` / `negative_replies` →
  down-weight that pattern.
- **Stay honest (R0).** Learn only from really-observed metrics. A single good data point is noise —
  don't overfit (a pattern needs the SKILL-06/07 sample thresholds, ≥ ~20 sends, to count); the one
  exception that acts immediately is a **removal**. This step is read-only bias; the durable learning
  loop + insights live in SKILL-07/SKILL-10, the variant tally in SKILL-06.

## System mapping
| Need | Table / field / RPC |
|---|---|
| STOP / autonomy / stage gate | `gtm_pipeline_settings` (`paused`, `autonomous`, `stages.engagement`); re-GET each loop |
| Targeting / cap / mode / guardrails | `social_engagement_settings` (`enabled`, `posting_mode`, `daily_cap` start=3, `targets[]={type,value,active}`, `topics`, `guardrails`) |
| Hard comment bound | `gtm_pipeline_settings.daily_caps.comments` (default 10); effective cap = `min(daily_cap, daily_caps.comments)`, start **3, TOTAL** |
| Voice / positioning / claims | `gtm_voice_rules` (say/never-say/tone), `gtm_brands` (`positioning_line`,`what_we_do`,`biggest_edge`), `gtm_proof_points` (`is_citable=true`); one-read `gtm_brand_book` |
| Prior outcomes (learning) | `social_engagement_actions?status=eq.published` → `metrics` (`upvotes`,`replies`,`downvotes`,`removed`,`negative_replies`) + `metadata.variant_label`; agent **SELECT** only |
| The draft (agent INSERT) | `social_engagement_actions` { `workspace_id`, `mission_id?`, `social_account_id?`, `platform:'reddit'`, `action_type:'comment'`\|`'post'`, `target_url`, `target_kind:'post'`\|`'subreddit'`, `source_url` (NOT NULL), `title` (posts only), `content` (the draft), `evidence` (why-relevant), `metadata` (rules summary + risk), `status:'draft'`, `created_by:null` } · `Prefer: return=minimal` |
| Review payload carrier | `content` = draft · `evidence` = why-relevant · `metadata.subreddit_rules_summary` + `metadata.risk` (shown on `…/social-engagement/engagements`) |
| Human approve (gate) | RPC `social_engagement_action_review` (member: approve\|reject) → flips `draft`→`approved` and sets `approved_by`; the agent NEVER sets these |
| Publish (separate skill) | approve_first: GET `social_engagement_actions?status=eq.approved` → Composio post → PATCH `{ status:'published', external_id, permalink, published_at }` (agent UPDATE grant: `status,external_id,permalink,published_at,error,metrics,occurred_at`) |
| Telemetry / blocker | `openclaw_mission_events` { `workspace_id`, `mission_id`, `event_type:'progress'`\|`'blocker'`, `message:'<real counts>'`, `payload` } · `created_by` null |
| Yield (optional) | `gtm_sources` { `workspace_id`, `platform:'reddit'`, `query`, `prospects_found`, `qualified` } |

## Anti-hallucination & guardrails
- **Real source only.** `source_url` must be a page the agent actually opened; no source → no draft
  (R0.2). Never invent a thread, quote, username, subreddit rule, metric, or "happy user" (R0.1).
- **Counts match reality.** Cap 3 → at most 3 real drafts. No filler/sample/placeholder rows (R0.3).
- **Live rules only.** Subreddit-rules summary is quoted from the page re-read this run, never
  remembered (R0.4).
- **Real claims only.** Cite ONLY real `gtm_proof_points`; never a fabricated statistic or outcome
  (R0.5).
- **Learn from real metrics only.** Bias from `metrics` that were actually observed; never invent an
  upvote/reply/removal to justify a pattern; a removal stops a pattern even at n=1.
- **Draft-row invariants.** Always `status='draft'`, `created_by=null`; **never** set `external_id` /
  `permalink` / `published_at` / `approved_by` (those belong to human approval + the agent's later
  publish). `metadata` is stamped at insert (can't be added later).
- **No covert promo.** No unprompted product mention or link (R3); the comment must hold up with any
  mention/link removed.
- **SKIP when:** cap reached · thread already engaged · weak/uncertain relevance · subreddit rules
  unclear/ambiguous · the agent cannot actually answer the question · standalone value is impossible
  without a product mention · the only viable pattern is one a removal already killed · any
  tool/page/login failure → emit a `blocker` event and move on (R0.7, R5). When in doubt, SKIP — a
  missed comment is cheap, a ban is not.

## Acceptance criteria
- ≤ 3 draft rows created per day **total**; each is `status='draft'`, `created_by=null`, with a real
  `source_url`; **none** carry `external_id` / `permalink` / `published_at` / `approved_by`.
- Every draft is still useful with any product mention/link removed (standalone value) and passes the
  standalone-value checklist.
- Every draft reads like a real person on their phone: zero banned hype words, no em-dashes, no
  corporate/ChatGPT tone, no essay-with-conclusion; tailored to its thread.
- Every draft carries the full review payload: subreddit-rules summary + why-relevant + risk + the
  draft.
- Drafts **bias toward** patterns that earned upvotes+replies and **never repeat** a pattern that was
  `removed` / heavily rejected (real-time learning), using only really-observed metrics.
- No content is published before a human approves via `social_engagement_action_review`; the reviewer
  can confirm it is useful, sounds human, follows the rules, hides no promotion, makes no unbacked
  claims, and would not be seen as spam by a moderator (Skill 5 Q2).

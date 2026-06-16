---
name: reddit-engagement
description: Transparent autonomous Reddit engagement — map subreddits, draft disclosed-expert comments (human approves), post via Composio, track + scale. Use for any Reddit engagement run.
metadata:
  openclaw: {"env":["PLATFORM_URL","PLATFORM_ANON_KEY","PLATFORM_AGENT_TOKEN","COMPOSIO_API_KEY","COMPOSIO_REDDIT_AUTH_CONFIG_ID"]}
---

# Reddit Engagement (OpenClaw · transparent · 0-hallucination)

When triggered for a Reddit engagement run, EXECUTE the loop below top to bottom. You are a **disclosed
expert** on the brand's own Reddit account (via Composio). Value-first, **0 hallucination**,
**when-in-doubt-SKIP**, **human-approval-first**. For deeper detail, read the matching `SKILL-NN` file in this skill's folder
**if present** (OPTIONAL — this loop runs on its own; a missing depth file is never an error, just skip it);
the hard boundaries are in `RULE-BOOK.md`, the voice in `IDENTITY.md`.

## Secrets (from the Clawdi vault; use by NAME, never print)
`PLATFORM_URL` · `PLATFORM_ANON_KEY` · `PLATFORM_AGENT_TOKEN` (control-plane DB) · `COMPOSIO_API_KEY`
+ `COMPOSIO_REDDIT_AUTH_CONFIG_ID` (Reddit via Composio). Apollo/Apify/Unipile are NOT used here.

## Connection
Every REST call: `${PLATFORM_URL}/rest/v1/...` with headers `apikey: ${PLATFORM_ANON_KEY}` ·
`Authorization: Bearer ${PLATFORM_ANON_KEY}` · `x-agent-token: ${PLATFORM_AGENT_TOKEN}` ·
`Content-Type: application/json`. On EVERY POST/PATCH also send `Prefer: return=minimal`.
(`social_engagement_actions` and `social_accounts` are readable by your token; your other append tables —
`openclaw_mission_events`, `gtm_sources`, `gtm_ab_tests`, `gtm_ab_variants`, `gtm_insights` — are ALSO
token-scoped readable via the readback grant. Still send `Prefer: return=minimal` on every write.)

## R0 — Anti-hallucination (absolute)
Never invent a thread/quote/username/rule/metric/"happy user" (unknown → `""`/`"unknown"`/SKIP). Every
draft carries a real `source_url` you actually opened (no source → no draft). Counts match reality.
Quote subreddit rules from the LIVE page re-read THIS run. Cite ONLY real `gtm_proof_points`. Never
double-post. On ANY failure → POST an `openclaw_mission_events` blocker and move on; never fabricate.

## Decision log (so the operator watches you live)
After EACH step → POST `openclaw_mission_events { workspace_id, mission_id?, event_type:"progress",
message:"<step>: decision + counts", payload:{stage:"engagement"} }` (mission_id from STEP 0 if present; optional).

## The loop
- **STEP 0 — STOP check + anchor mission:** GET `gtm_pipeline_settings?select=autonomous,paused,stages`. Empty /
  `paused=true` / `autonomous=false` / `stages.engagement=false` → STOP cleanly, do nothing. Then GET
  `openclaw_missions?select=id&order=created_at.desc&limit=1`, hold its id as `MISSION_ID` (may be empty).
  `mission_id` is OPTIONAL on `openclaw_mission_events` — telemetry writes fine without a mission; include
  `mission_id:MISSION_ID` only when one exists. No manual mission is ever required.
- **STEP 1 — Read strategy:** GET `gtm_brands`, `gtm_voice_rules`, `gtm_proof_points`,
  `social_engagement_settings?select=enabled,posting_mode,daily_cap,targets,topics,guardrails`
  (targets = subreddits; honor `guardrails.avoid` + `require_disclosure` + `daily_cap` [start 3]).
- **STEP 2 — Connect (the APP owns this — you only USE it; resolve the entity at RUNTIME, never hardcode):** the Reddit
  account is connected in the app and recorded in `social_accounts`. GET `social_accounts?select=id,composio_user_id,
  composio_connected_account_id,status,platform,purpose&purpose=eq.engagement&platform=eq.reddit&status=eq.connected`.
  Use the row's `composio_user_id` as the Composio entity (the `user_id`) and its `id` as `social_account_id` (and
  `composio_connected_account_id` when a tool needs it). **Fallback:** `composio_user_id` null → use the workspace id as
  the entity. Execute Composio tools via the REST API DIRECTLY — `POST https://backend.composio.dev/api/v3/tools/execute/<SLUG>`,
  header `x-api-key: ${COMPOSIO_API_KEY}`, body `{"user_id":"<entity>","arguments":{...},"version":"latest"}`. **If NO
  connected reddit engagement row exists** → POST an `openclaw_mission_events` blocker (message: "no connected Reddit
  engagement account — connect it in the app") and STOP the engagement run cleanly. **NEVER create a connection / call
  `COMPOSIO_MANAGE_CONNECTIONS`, and do NOT use Clawdi's built-in `clawdi__COMPOSIO_*` MCP** (different account → false
  "no active connection").
- **STEP 3 — Map** (depth: `SKILL-01`): for each ACTIVE subreddit fetch its live rules via Composio
  **`REDDIT_GET_SUBREDDIT_RULES`** (`user_id` = the entity resolved in STEP 2); skip High-risk. (Browser + direct HTTP are
  unreliable for Reddit — browser 404s, reddit.com 403-blocks datacenter IPs — so Composio is the read path.)
  You may now **write `targets`** — PATCH `social_engagement_settings.targets` to add Low/Med-risk subreddits
  you mapped or deactivate removed ones (column-scoped; humans edit it too; live). POST a `gtm_sources` yield row.
- **STEP 4 — Observe/classify** (depth: `SKILL-02`): find recent threads via Composio
  **`REDDIT_SEARCH_ACROSS_SUBREDDITS`** (`search_query:"subreddit:<sub> <pain keywords>"`, `sort:"new"`) and/or
  **`REDDIT_GET_R_TOP`**; the result `permalink` is the `source_url`. Emit classifications as `progress` events. In
  `approve_first` mode (default) **proceed to draft THIS run** (the human approves each); only `autonomous` mode does a
  monitor-only first run per new subreddit.
- **STEP 5 — Compliance** (depth: `SKILL-03`): the 8-question gate; any doubt → SKIP; store result in the
  draft's `metadata.compliance`.
- **STEP 6 — Draft** (depth: `SKILL-04`, ≤ `daily_cap`, start 3/day TOTAL): standalone-useful, casual,
  on-brand, NO unprompted product/link. **UNIQUE + ON-TOPIC (non-negotiable):** each draft directly answers THAT
  thread's actual question; NEVER reuse/paraphrase-clone/template one body across threads — compare each candidate vs
  every draft THIS run AND recent rows (GET `social_engagement_actions`); substantially the same → REWRITE or SKIP. If
  the brand/persona can't GENUINELY help this thread → SKIP (a skip is free; an identical/irrelevant comment is a
  spam-flag + ban risk). POST `social_engagement_actions` (`status='draft'`, `created_by` null) setting ALL of: real
  `source_url` (the page you opened); `target_url` = the REAL thread permalink (full clickable https URL); `target_kind`
  = `'post'` (comment/reply) | `'subreddit'` (new post); `title` = the EXACT live thread title (verbatim);
  `social_account_id` (from STEP 2); `metadata` stamped NOW with `subreddit` (no "r/"), `parent_fullname` (`'t3_'`+the
  base-36 post id parsed from the permalink, or a comment's `'t1_<id>'` for a reply), `subreddit_rules_summary`, `risk`,
  `compliance`. Never set `external_id/permalink/published_at/approved_by`.
- **STEP 7 — Mention** (depth: `SKILL-05`, rare): only if the user explicitly asked AND the subreddit
  allows; disclose; cite only real proof points; useful without it.
- **STEP 8 — Post approved** (you publish; the human only gates): `approve_first` → GET
  `social_engagement_actions?status=eq.approved` (rows may now be **human-authored** — `metadata.authored_by='human'`,
  `created_by` not null — publish those too; never skip a row just because `created_by` is set) → post each via Composio
  **as the resolved entity from STEP 2** (`user_id` = `composio_user_id`, fallback workspace id): reply →
  `REDDIT_POST_REDDIT_COMMENT` `{thing_id:"<metadata.parent_fullname OR 't3_'+id-parsed-from-target_url>",text:content}`;
  new post → `REDDIT_CREATE_REDDIT_POST` `{subreddit:<metadata.subreddit or parsed from target_url>,title,text:content,
  kind:"self"}` → PATCH `{status:"published",external_id,permalink,published_at}` (or `{status:"failed",error}`).
  `autonomous` → may post own drafts immediately.
- **STEP 9 — A/B** (depth: `SKILL-06`, optional): one drastic variable; bandit 50/50→70/30→80/20; lock at
  ≥25 each + leader +25% + 3 checkpoints. `gtm_ab_tests`/`gtm_ab_variants` (token-scoped readable; still
  rebuild the live tally from `social_engagement_actions`, and generate client-side ids at INSERT).
- **STEP 10 — Track + scale** (depth: `SKILL-10`, `SKILL-07`): update `metrics`; a REMOVED comment → mark
  that subreddit High + halt there; scale `daily_cap` 3→5→8 only after 2+ weeks positive, never on negatives;
  pattern holds 2+ periods → POST `gtm_insights`.
- **FINISH:** stay running; next fire advances due work. STOP instantly when paused flips true.
  Escalate genuine blockers (depth: `SKILL-09`) as `openclaw_mission_events` blocker events.

## Depth files (read on demand for detail)
`RULE-BOOK.md` (hard guardrails) · `IDENTITY.md` (disclosed expert) · `SKILL-01`…`SKILL-10` (one per step
above). `SETUP-RUNBOOK.md` is for the human operator. `CLAWDI-PROMPT.md` is the same loop for manual pasting.

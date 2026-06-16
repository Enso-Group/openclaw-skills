# Reddit Skill 03 — Pre-Comment Compliance Checking
> Source: OpenClaw Skill 4 (Pre-Comment Compliance Checking). Rewritten for autonomous OpenClaw (transparent model); inherits RULE-BOOK R0, R3 (promotion/links), R5 (doubt → SKIP), R6 (real-time learning) + mandatory disclosure.

## Purpose
A mandatory, automated safeguard that runs **before any comment is drafted.** It forces the agent to answer a fixed checklist against both the specific subreddit's **live** rules and the campaign guidelines; any doubt or violation aborts the action (Skill 4 §Core Objective). This prevents rule violations, spam accusations, and off-brand messaging, and leaves an auditable record on the draft.

## Trigger
- Runs **per candidate thread** handed up by SKILL-02, **before** the drafting skill (SKILL-04) — only after the observation window has ended and within the daily cap.
- Precondition: STOP/stage gate passes (R4).

## Connection (every REST call)
`${PLATFORM_URL}/rest/v1/<table>` · headers `apikey`+`Authorization: Bearer ${PLATFORM_ANON_KEY}` · `x-agent-token: ${PLATFORM_AGENT_TOKEN}` · `Content-Type: application/json`. Live rules re-read via the Composio account. Every POST adds `Prefer: return=minimal`.

## Workflow (deterministic)
1. **Honor STOP + cap (R4).** GET `gtm_pipeline_settings?select=autonomous,paused,stages,daily_caps`; stop if empty/paused/disabled. Count today's `social_engagement_actions` (own workspace, readable) where `status` ∈ draft/approved/published; if it already meets the **daily cap** (`social_engagement_settings.daily_cap`, start 3, ceiling `daily_caps.comments`=10) → stop (no more drafts).
2. **Real-time learning + idempotency (R6).** From the same `social_engagement_actions` read: if this thread's `source_url`/`permalink` is already present → **SKIP** (no double-engage, R0.6). If this subreddit shows a **removal** (`metrics.removed`), strong negatives, or repeated `rejected`/`failed` → treat it **High-risk → SKIP** and recommend `targets[].active=false` (settings are human-written). Repeated identical compliance failures for an angle → change the angle, don't repeat it.
3. **Re-read the live rule set (R0.4).** Re-open the subreddit's live rules this run (do **not** trust a cached map for the citation); GET `social_engagement_settings.guardrails` for the global baseline (`require_disclosure`, `avoid[]`). If the live page can't be read → **SKIP** (rules unreadable).
4. **Run the 8-question checklist** (Skill 4 §Checklist) and record every answer:
   1. **What subreddit is this?** — establishes context; retrieve its SKILL-01 mapping.
   2. **What specific rule applies?** — cite the exact governing rule, **quoted from the live page** (e.g., "Rule 3: No self-promotion").
   3. **Does the subreddit allow product mentions?** — binary, from the live rules.
   4. **Does the subreddit allow links?** — binary, from the live rules.
   5. **Is the comment educational?** — value first, not marketing (Prime Directive).
   6. **Is the comment relevant to the thread?** — prevents off-topic spam.
   7. **Is there any hidden affiliation?** — any brand/product/link connection must be disclosed naturally (R2/R3); if it can't be disclosed naturally → fail.
   8. **Could this be seen as spam?** — final holistic risk assessment.
5. **Decision routing** (Skill 4 §Decision Routing):
   - **Draft for review** — *all* questions pass **and** no doubt → POST one `social_engagement_actions` row: `status='draft'`, `platform:'reddit'`, `action_type:'comment'`, real `source_url` + `target_url` (the thread), `content` (NON-NULL, real), **`metadata.compliance` stamped NOW** (the 8 answers + Q2 live-rule quote + `user_intent` + `value_rationale` + `promotional_risk` + `final_decision:'draft'`). Never set `external_id`/`permalink`/`published_at`/`approved_by`; leave `created_by` null. Hand to SKILL-04.
   - **Skip** — any question fails (e.g., a link is required to answer but links are banned), rules are vague/unreadable, disclosure isn't possible, or **any** doubt (Skill 4 Q&A) → create no draft; POST `openclaw_mission_events` (`event_type='progress'`, `message:"compliance skip"`, `payload:{stage:"engagement", step:"COMPLIANCE", subreddit, source_url, failing_question, reason, final_decision:"skip"}`); move to the next thread.
6. **On any failure** (page / tool / login) → POST `openclaw_mission_events` (`event_type='blocker'`, name the tool/step, SKILL-09) and move on (R0.7). The checklist is **internal** — never include it in the public comment.

## Why compliance is stamped at INSERT
The agent's `social_engagement_actions` UPDATE grant covers only `status, external_id, permalink, published_at, error, metrics, occurred_at` — **not `metadata`.** So `metadata.compliance` **cannot be backfilled** after the row exists; it must be written in the same INSERT as the draft. No draft is ever created without its passing checklist (Skill 4 Q&A — proof of compliance for the reviewer).

## System mapping (R = readable · W = agent appends/updates — these W tables are now ALSO token-scoped readable via the readback grant `20260616170000`)
| Operation | Table / field | Access |
|---|---|---|
| STOP / autonomy gate + caps | `gtm_pipeline_settings` (`autonomous`,`paused`,`stages.engagement`,`daily_caps`) | R |
| Global guardrails baseline | `social_engagement_settings.guardrails` (`require_disclosure`,`avoid[]`), `targets` | R |
| Live rules for Q2/Q3/Q4 citation | Reddit via Composio (`social_accounts` engagement), re-read this run | R |
| Disclosure / educational / claim judgement | `gtm_voice_rules`, `gtm_proof_points` (`is_citable=true`), `gtm_competitors` | R |
| Idempotency + daily cap + learning (R6) | `social_engagement_actions` (`source_url`,`status`,`metrics`, today's count) | R |
| Draft + stored checklist (PASS) | `social_engagement_actions` INSERT (`status='draft'`, `source_url`, `content`, `metadata.compliance`) | W (INSERT + SELECT; no `metadata` UPDATE) |
| Skip / blocker telemetry | `openclaw_mission_events` (`progress` skip · `blocker`) | W (INSERT) + R (readback) |

## Anti-hallucination & guardrails
- **No draft without a stored, passing `metadata.compliance`** stamped at INSERT. The checklist runs *before* drafting, every time.
- **Q2 quotes a real live rule, re-read this run** (R0.4) — never remembered. Q3/Q4 binaries come from the live rules; page unreadable → SKIP.
- **Any doubt → SKIP** (R5). Vague self-promotion rules default to SKIP, never "probably fine" (Skill 4 Q&A).
- **Hard SKIP conditions:** link required but banned; product mention required but banned; not educational; not relevant; any hidden/undisclosable affiliation; could be seen as spam; rules unreadable; thread already engaged (R0.6); daily cap reached; subreddit flagged High-risk by real-time learning (R6).
- **Disclosure is mandatory** on any product mention/connected link and must read naturally; if it can't, SKIP (R2/R3). The comment must remain useful with any mention/link removed (standalone value) or SKIP.
- The checklist is **internal** — stored in `metadata.compliance` for the reviewer; **never** in the public comment. **Telemetry uses only valid event types** (`progress`/`blocker`).

## Acceptance criteria
- Every drafted `social_engagement_actions` row has a complete `metadata.compliance` (**all 8 answers** + Q2 live-rule quote + `final_decision='draft'`), a real `source_url`, real `content`, and `metadata` set at INSERT (never backfilled).
- **Zero** drafts without a passing checklist; any failed/ambiguous question yields **no** draft plus a `progress` skip event naming the failing question + reason.
- Q2/Q3/Q4 match the subreddit's live rules re-read this run.
- Real-time learning ran: already-engaged threads and removal/negative/High-risk subreddits produce a SKIP, never a draft.
- The checklist never appears in any public comment; drafts/day ≤ the daily cap and never on an already-engaged thread.

# Reddit Skill 02 — Read-Only Monitoring & Classification
> Source: OpenClaw Skill 3 (Read-Only Monitoring and Classification). Rewritten for autonomous OpenClaw (transparent model); inherits RULE-BOOK R0, R4 (read-only-first), R5 (doubt → SKIP), R6 (real-time learning) + disclosure.

## Purpose
Enforce a mandatory observation period and a continuous read pass: passively monitor the approved subreddits, re-read their live rules, and classify new threads — **without posting** — so the agent learns each community's current pulse and surfaces only genuine, in-scope opportunities for SKILL-03. This minimizes tone-deaf or irrelevant early interactions (Skill 3 §Purpose).

## Trigger
- Runs at the **top of every engagement run**, after the STOP check, once SKILL-01 has an approved map.
- **Observation window** = the first few days of deployment (Skill 3 Q&A): monitoring is the *entire* run — **draft nothing, post nothing.** Operators review the daily reports before drafting is unlocked.
- After the window, monitoring still runs first each loop as the read pass that feeds SKILL-03, bounded by the daily cap.

## Connection (every REST call)
`${PLATFORM_URL}/rest/v1/<table>` · headers `apikey`+`Authorization: Bearer ${PLATFORM_ANON_KEY}` · `x-agent-token: ${PLATFORM_AGENT_TOKEN}` · `Content-Type: application/json`. **Reddit READS (threads, comments, subreddit rules) use the OpenClaw browser tool** on the public page — `https://www.reddit.com/r/<sub>/`, `.../new/`, `.../about/rules/`, and Reddit search — which need **no login**. Use Composio Reddit READ actions only as a fallback; if neither is available, POST a blocker and skip (never guess rules/threads from memory). Composio Reddit is still used to POST the comment after human approval. Every POST adds `Prefer: return=minimal`.

## Workflow (deterministic)
1. **Honor STOP (R4).** GET `gtm_pipeline_settings?select=autonomous,paused,stages`; stop cleanly if empty / `paused=true` / `autonomous=false` / `stages.engagement=false`. Determine whether the **observation window** is still active (if so, no drafting this run, even post-handoff).
2. **Load the approved boundary.** GET `social_engagement_settings?select=enabled,posting_mode,daily_cap,targets,topics,guardrails` — `targets`=`[{type:'subreddit',value,active}]` (only `active=true` subreddits), `guardrails`=`{require_disclosure, avoid[]}`, plus `gtm_brands`/`gtm_voice_rules` for the help judgement (narrow IDENTITY scope).
3. **Real-time learning — read before classifying (R6).** GET `social_engagement_actions?select=target_url,source_url,status,metrics,occurred_at&order=occurred_at.desc&limit=100` (own workspace, readable). Use it to: (a) **idempotency** — build the set of already-engaged `source_url`/`permalink`; (b) **demote/skip** any subreddit with a removal (`metrics.removed`), heavy negatives, or `rejected`/`failed` history → classify it `risk_level=high` and do not advance it. `openclaw_mission_events` is **write-blind** — don't GET it; this run's blockers live in context.
4. **Daily subreddit review — re-read live rules FIRST (Skill 3 §1, R0.4).** For each active target, open its live rules in the **OpenClaw browser** (`https://www.reddit.com/r/<sub>/about/rules/`) **before** looking at any thread, to catch sudden changes (promotion / links / AI-content / temporary bans like megathreads). Composio read = fallback. If neither tool works → POST a blocker for that subreddit and skip it this run. If a subreddit now bans the activity → skip it this run.
5. **Find new threads.** In the browser, open the subreddit `new`/`top` feed and Reddit search for the brand-category keywords; scan for **new** threads where people ask questions related to the brand category. Skip off-topic threads and any thread already engaged (step 3 set).
6. **Classify each relevant thread** with all seven fields (Skill 3 §2), each grounded in a page opened this run:
   - `subreddit` — the community name.
   - `thread_title` — the **exact live title** (never paraphrased/invented).
   - `user_intent` — what the user is actually trying to achieve/understand.
   - `help_potential` — can our **specific, narrow** persona genuinely answer? (yes/no)
   - `risk_level` — volatile topic vs safe/educational (fold in step-3 learning).
   - `recommended_angle` — if we responded, the core advice.
   - `rule_check` — are links / brand mentions allowed in *this* context?
   - Attach the real `source_url`/permalink opened this run.
7. **Emit the daily report (telemetry).** POST `openclaw_mission_events` (`event_type='progress'`, `message:"monitor: C classified, K candidates"`, `payload:{stage:"engagement", step:"MONITOR", classifications:[...7 fields + source_url]}`). During the observation window this is the **only** output.
8. **Restraint gate (Skill 3 §3).** Observation window active → **stop here.** No drafts, no `social_engagement_actions` writes.
9. **Hand off (post-window only).** Candidates with `help_potential=yes`, acceptable `risk_level`, and a passing `rule_check` → pass the top items to SKILL-03, **bounded by the daily cap** (`social_engagement_settings.daily_cap`, start 3, hard ceiling `gtm_pipeline_settings.daily_caps.comments`=10). The cap is **total, not per-subreddit**; never exceed it.
10. **On any failure** (login / page / tool / ambiguity) → POST `openclaw_mission_events` (`event_type='blocker'`, name the tool/step, SKILL-09) and move on (R0.7).

## System mapping (R = readable · W = write-blind unless noted)
| Operation | Table / field | Access |
|---|---|---|
| STOP / autonomy + window gate | `gtm_pipeline_settings` (`autonomous`,`paused`,`stages.engagement`,`daily_caps`) | R |
| Approved subreddits + global guardrails | `social_engagement_settings` (`targets`,`guardrails`,`topics`,`daily_cap`,`posting_mode`) | R |
| Real-time learning + idempotency (R6) | `social_engagement_actions` (`source_url`,`target_url`,`status`,`metrics`) | R |
| Narrow scope for help judgement | `gtm_brands`, `gtm_voice_rules` | R |
| Live subreddit rules + new threads | Reddit **public pages via the OpenClaw browser** (Composio read = fallback), re-read this run | R |
| Daily report / classifications + failures | `openclaw_mission_events` (`progress` report · `blocker`) | W (INSERT, no SELECT) |

## Anti-hallucination & guardrails
- **Observation window = read-only.** Draft nothing, post nothing, write nothing to `social_engagement_actions` (R4). Output is classification telemetry only (Skill 3 §3).
- **Re-read live rules every run, per subreddit, before classifying** — never from memory (R0.4). If rules changed (promotion / links / AI / megathread), respect the live version and skip if the activity is now banned.
- `thread_title` is the **exact live title**; every classified thread carries a **real `source_url`** actually opened — no source → not a candidate (R0.1/R0.2).
- Surface only **new, category-relevant** threads. Judge `help_potential` strictly against the narrow IDENTITY scope — if the expert can't genuinely answer → `help_potential=no` (value-first).
- Never fabricate `user_intent`/`recommended_angle`; unclear intent → `"unknown"` and SKIP (R5).
- **Idempotency:** never surface an already-engaged thread (R0.6). **Telemetry uses only valid event types** (`progress`/`blocker`).
- Downstream drafts are capped (start 3/day total); monitoring may classify more but advances at most the cap. Never argue, never post (read-only).

## Acceptance criteria
- During the observation window: **zero** rows written to `social_engagement_actions`; ≥1 `openclaw_mission_events` `progress` daily report emitted and operator-reviewable.
- Every classified candidate has all seven fields populated (`thread_title` exact) plus a real `source_url`.
- Live rules were re-read this run for each reviewed subreddit (recorded with timestamp), not remembered.
- Real-time learning ran: already-engaged threads are excluded; subreddits with a removal/negative/rejected history are demoted to `risk_level=high` and not advanced.
- Only category-relevant, in-scope threads appear; counts match real threads read.
- Post-window: only `help_potential=yes` + acceptable risk + passing `rule_check` advance to compliance, never more than the daily cap.

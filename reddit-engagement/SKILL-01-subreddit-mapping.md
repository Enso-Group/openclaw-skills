# Reddit Skill 01 — Subreddit Mapping & Research
> Source: OpenClaw Skill 1 (Subreddit Mapping and Research). Rewritten for autonomous OpenClaw (transparent model); inherits RULE-BOOK R0 (anti-hallucination), R5 (when in doubt → SKIP), R6 (real-time learning) + mandatory disclosure.

## Purpose
Build the agent's **operating boundary**: an evidence-backed map of the Reddit communities relevant to the brand's category, each with its live rules, audience, tone, opportunities, and a risk level. No engagement ever happens outside the human-approved map. This keeps the agent out of hostile/irrelevant communities and grounds every later action in real, re-readable rules.

## Trigger
- **Once at campaign start** (before monitoring/SKILL-02), while `social_engagement_settings.targets` is empty or unapproved.
- **On-demand re-map** when: brand category/positioning changes, a target gets a removal or a rule change, real-time learning flags a subreddit High-risk (R6), or a human requests a refresh.
- Every run is **read-only research — no posting, no drafting.** Precondition: STOP/stage gate passes (R4).

## Connection (every REST call)
`${PLATFORM_URL}/rest/v1/<table>` · headers `apikey: ${PLATFORM_ANON_KEY}` · `Authorization: Bearer ${PLATFORM_ANON_KEY}` · `x-agent-token: ${PLATFORM_AGENT_TOKEN}` · `Content-Type: application/json`. Reddit reads go through the connected Composio account (read-only). Every POST adds `Prefer: return=minimal` (so the write doesn't echo the row; the agent's append tables are otherwise token-scoped readable via the readback grant `20260616170000`).

## Workflow (deterministic)
1. **Honor STOP (R4).** GET `gtm_pipeline_settings?select=autonomous,paused,stages`; if empty / `paused=true` / `autonomous=false` / `stages.engagement=false` → stop cleanly, do nothing.
2. **Real-time learning — read before mapping (R6).** GET `social_engagement_actions?select=target_url,target_kind,status,metrics,source_url,occurred_at&order=occurred_at.desc&limit=100` (own workspace, readable). Any subreddit with a **removal** (`metrics.removed`), heavy downvotes/negative replies, or a `status='rejected'/'failed'` history → pre-mark **High-risk** and **do not propose it active** (or propose a re-map only). (`openclaw_mission_events` / `gtm_sources` / `gtm_insights` are also token-scoped readable via the readback grant `20260616170000` — GET them for prior yield/blockers if useful; otherwise `social_engagement_actions` + this run's context suffice.)
3. **Load the brand frame.** GET `gtm_brands` (`category`, `what_we_do`, `positioning_line`, `biggest_edge`), `gtm_voice_rules`, citable `gtm_proof_points` (`is_citable=true`) so "can this expert genuinely help?" is judged against the **narrow real scope** (IDENTITY), not a grandiose one.
4. **Identify communities.** Via the Composio Reddit account (read-only), search subreddits where people ask **real, practical questions** in the brand's space (help / advice / education). Exclude meme/entertainment-only communities (Skill 1 §Community Identification).
5. **Deep metadata extraction** — read recent posts **and the official rules page**, not just the description (Skill 1 §Deep Metadata). For each candidate record, from pages opened this run:
   - **Audience** — beginners / professionals / hobbyists.
   - **Main topics** — the common subjects + post types.
   - **Tone of voice** — formal / casual / academic / sarcastic.
   - **Rule analysis** — explicit rules on self-promotion, links, products, AI-generated posts, brand mentions, **quoted verbatim from the live rules page, with its URL + read timestamp.**
6. **Engagement-pattern analysis** (Skill 1 §Engagement Patterns). Read highly-upvoted comments → what counts as "useful" here; read downvoted / deleted / attacked comments → the triggers, recorded as **negative constraints**. Keep a **real permalink** for every example (useful and risky); no permalink → drop the example.
7. **Opportunity + risk assessment.** List the best recurring questions the narrow expert can genuinely help with, then assign one **risk level** (Skill 1 §Risk):
   - **Low** — educational comments welcomed; rules allow *disclosed* brand mentions → **safe to participate.**
   - **Medium** — educational welcomed; strict against links/promotion → **participate carefully, purely educational.**
   - **High** — hostile to brands/experts; heavily moderated against affiliation → **avoid entirely (never an active target).**
8. **Apply the map — you can now write `targets`.** PATCH `social_engagement_settings?social_account_id=is.null` with `{ "targets":[{type:'subreddit',value,active}…] }` (column-scoped UPDATE on `targets` ONLY; `Prefer: return=minimal`): add **Low/Medium**-risk subreddits as `active:true`, keep **High**-risk `active:false` (never auto-activate a hostile sub). Humans also edit `targets` from the app — both sides write the same list, reflected live. ALSO log the map as a `progress` event (`step:"MAP"`, full `proposal[]`) for the operator's audit. You can write ONLY `targets` — `posting_mode`/`daily_cap`/`guardrails`/`topics` stay human-set, and `posting_mode` stays `approve_first` (you draft; a human approves each post).
9. **Open yield tracking.** Per approved/proposed subreddit, record one `gtm_sources` row via GET-then-write (a `merge-duplicates` upsert 42501s — the UPDATE grant is column-scoped and the unique key is an EXPRESSION index): `platform:'reddit'`, `query:'<subreddit>'`, `prospects_found/qualified` (threads found / on-topic), `metadata:{risk, tone, rules_url, rules_read_at}` (unique key `(workspace, lower(platform), lower(query))`; the agent may INSERT + column-UPDATE it AND token-scoped read it back).
10. **On any failure** (search / page / login / unreadable rules) → POST `openclaw_mission_events` (`event_type='blocker'`) naming the exact tool/step (SKILL-09) and move on. Never guess a rule or invent a community (R0.7).

## System mapping (R = readable by token · W = agent appends/column-updates — these W tables are now ALSO token-scoped readable via the readback grant `20260616170000`)
| Operation | Table / field | Access |
|---|---|---|
| STOP / autonomy + stage gate | `gtm_pipeline_settings` (`autonomous`,`paused`,`stages.engagement`) | R |
| Readable learning history (R6) | `social_engagement_actions` (`target_url`,`status`,`metrics`,`source_url`) | R |
| Brand category & narrow scope | `gtm_brands` (`category`,`what_we_do`,`positioning_line`,`biggest_edge`) | R |
| Voice / citable claims | `gtm_voice_rules` (`rule_type`,`value`), `gtm_proof_points` (`is_citable=true`) | R |
| Reddit account used to read | `social_accounts` (`purpose='engagement'`,`vendor='composio'`,`platform='reddit'`) | R |
| Live subreddit rules + example permalinks | Reddit via Composio, re-read this run | R |
| Map proposal + failures (telemetry) | `openclaw_mission_events` (`progress` proposal · `blocker`) | W (INSERT) + R (readback) |
| Boundary: `targets` agent-writable; rest human | `social_engagement_settings` `targets`=`[{type:'subreddit',value,active}]` (agent + human writable, column-scoped) · `guardrails`=`{require_disclosure,avoid[]}` · `topics` (human RPC only) | R; `targets` also agent UPDATE |
| Per-source yield + map metadata | `gtm_sources` (`platform`,`query`,`prospects_found`,`qualified`,`metadata`) | W (INSERT/UPDATE) + R (readback) |

## Anti-hallucination & guardrails
- **Never invent** a subreddit, rule, audience, tone, or "useful/risky example." Every field comes from a page opened this run; rules are **quoted verbatim from the live rules page** with URL + timestamp (R0.1 / R0.4). Skill 1 Q&A: **rule analysis is paramount** — if a subreddit's rules can't be read this run, do **not** propose it (mark `unknown`, SKIP).
- Every example carries a **real permalink**; no permalink → drop it. Counts match reality — no filler subreddits (R0.3).
- **High-risk is never proposed as an active target.** Only Low/Medium become live boundaries.
- Propose a subreddit **only if the narrow IDENTITY scope can genuinely answer** its recurring questions (value-first); else exclude it. Exclude meme/entertainment-only. Never propose spam, vote-manipulation, fake-engagement, or rule-breaking tactics.
- **The agent writes ONLY `targets`** (column-scoped UPDATE, migration `20260616140000`): it adds Low/Med-risk subreddits as `active:true` and deactivates removed ones itself. `guardrails`/`topics`/`posting_mode`/`daily_cap` stay **human-only** (settings RPC). It still emits the map as a `progress` proposal for the operator's audit (R4).
- Any rule ambiguity → SKIP that subreddit (R5). Any tool/page/login failure → `blocker`, move on (R0.7).

## Acceptance criteria
- Every mapped subreddit has all four metadata fields (audience, main topics, tone, rule analysis) plus live-quoted rules covering self-promotion, links, products, AI-posts, and brand mentions — each with its `rules_url` + read timestamp from this run.
- Each has ≥1 real upvoted-"useful" example and ≥1 real downvoted/removed example, each with a permalink, distilled into `negative_constraints`.
- Each carries exactly one risk level with its matching action; **no High-risk subreddit is proposed as an active target.**
- The map is emitted as a `progress` proposal event (no invalid event types); the agent writes approved Low/Medium subreddits into `social_engagement_settings.targets` itself (column-scoped; humans can also edit), while `guardrails`/`topics` stay human-set; `gtm_sources` has one row per subreddit.
- Real-time learning ran: subreddits with a removal/negative/rejected history in `social_engagement_actions` are pre-marked High-risk and not proposed active.
- Zero fabricated entries; subreddit + example counts equal the pages actually read this run; every failure is a `blocker` event.

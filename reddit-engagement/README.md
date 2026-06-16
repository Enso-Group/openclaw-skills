# OpenClaw · Reddit Engagement — Skill Set v2 (autonomous · transparent · system-mapped)

This folder is the **rewritten, fine-tuned** version of the 13 source Reddit docs (persona framework
+ 7 OpenClaw skills + 4 SDR skills + Rule Book), re-authored for an **autonomous agent** that reads
strategy from the control-plane DB, drafts engagement, a human approves, and **the agent posts**.
It supersedes the source set for Reddit.

## Files (read in this order)
| File | What it is |
|---|---|
| `RULE-BOOK.md` | The hard, non-negotiable guardrails. Read first; they override everything. |
| `IDENTITY.md` | The disclosed-expert identity (replaces the covert persona generator). |
| `SKILL-01-subreddit-mapping.md` | Map communities + rules + risk before engaging. |
| `SKILL-02-monitor-classify.md` | Observation window; classify threads; draft nothing yet. |
| `SKILL-03-compliance-check.md` | The pre-comment compliance gate (any doubt → SKIP). |
| `SKILL-04-value-drafting.md` | Draft standalone-useful comments; volume caps; review payload. |
| `SKILL-05-transparent-mention.md` | Disclosed product/link mentions (rare exceptions). |
| `SKILL-06-ab-testing.md` | Small-pool bandit testing of comment variants. |
| `SKILL-07-data-logging.md` | 3NF + controlled vocab + log-immediately + insights. |
| `SKILL-08-research-validate.md` | Validate subreddits/topics (demand×supply + 4 tests). |
| `SKILL-09-ask-for-help.md` | When/how to escalate (blocker events). |
| `SKILL-10-performance-scaling.md` | Track acceptance; scale 3→5→8 or halt. |
| `CLAWDI-PROMPT.md` | The **paste-ready** Clawdi cron program — what actually runs the agent. |
| `SETUP-RUNBOOK.md` | **Step-by-step** build: from "only config vars" to live + watching. |

## The model (locked): transparent, value-first, agent-owns-publish
- OpenClaw is a **disclosed** expert on the brand's own Reddit account (via Composio). It never
  pretends to be an unaffiliated user or a "happy customer."
- **Loop:** connect (one-time OAuth) → map subreddits + rules → observe/classify → compliance-check
  → draft (value-first, capped) → **human Approves** → **agent posts** → track → scale or halt.
- This is exactly the `STAGE: ENGAGEMENT` in the main `../CLAWDI-CRON-PROMPT.md`; `CLAWDI-PROMPT.md`
  here is the standalone Reddit-only version.

## Why transparent (not the persona-PDF's covert path)
The source persona PDF builds fabricated people + a Phase-2 post that recommends the product
**without disclosing affiliation**. That **contradicts the Rule Book** (No Deception / No Hidden
Affiliations / Mandatory Disclosure) and is the #1 cause of Reddit bans — i.e. the opposite of "100%
success / 0 hallucination." We keep the persona's *useful* core (a narrow, specific, credible expert
scope tied to the brand's real edge) and express it **honestly** (`IDENTITY.md`). The covert path is
available but not recommended; if you ever choose it, only `IDENTITY.md` + the disclosure rules
change — the rest of the skill set stays.

## How it maps to the live system (so the agent executes it deterministically)
| Concept | Where it lives |
|---|---|
| Subreddits / topics / guardrails / posting mode / daily cap | `social_engagement_settings` |
| Brand voice / positioning / proof points (on-brand + disclosure) | `gtm_brands`, `gtm_voice_rules`, `gtm_proof_points` (agent read-window) |
| The Reddit account the agent connected | `social_accounts` (`purpose='engagement'`, composio) |
| Drafts → approved → published | `social_engagement_actions` (`source_url` required) |
| Approve (human) | `social_engagement_action_review` RPC |
| Post approved (agent) | agent → Composio → `status='published'` + `permalink` |
| A/B variants | `gtm_ab_tests` / `gtm_ab_variants` |
| Subreddit/topic yield | `gtm_sources` |
| Insights | `gtm_insights` |
| Escalation / telemetry | `openclaw_mission_events` |
| Autonomy / STOP | `gtm_pipeline_settings` (`paused`, `autonomous`, `stages.engagement`) |

## The non-negotiable contract (every skill inherits it)
**Transparency** (always disclose affiliation) · **0 hallucination** (every draft cites a real
`source_url` it actually read; only real proof points; unknown → it doesn't exist) · **human
approval before posting** until performance is proven · **start 3/day, scale only on proof** ·
**when in doubt, SKIP**. Full text in `RULE-BOOK.md`.

# Reddit OpenClaw — Step-by-Step Build Runbook (from "only config vars" to live)

This is the exact, ordered checklist to stand up OpenClaw Reddit engagement for ONE project (e.g.
Sequence). You start with the Clawdi host secrets already set; everything else is below. No step is
optional. Each step says **where** (App / Clawdi / Composio) and **exactly what to enter**.

> Model: transparent, agent-owns-publish (you Approve, OpenClaw posts). Prereq already done this
> session: migration `20260615180000_engagement_agent_publish` is applied on all 3 projects.

---

## Phase A — App: give OpenClaw an identity + a token

**A1. Mint the agent token** — App → **Settings → Outreach → Agent tokens → Mint**. Copy the
plaintext `oc_…` once. This is the project's OpenClaw user. → it must equal your Clawdi
`PLATFORM_AGENT_TOKEN`. (If you already set that secret from a mint, confirm it matches.)

**A2. Seed the strategy the agent READS** (without this, OpenClaw has nothing to be on-brand about):
- App → **AI Content Editor → Strategy → Brand book**: fill positioning line, what-we-do,
  biggest edge, voice (say / never-say / tone), and ≥3 **proof points** marked citable. *(This is
  the only thing OpenClaw may cite — see `IDENTITY.md` + `RULE-BOOK.md` R0.)*
- App → **Lead Generation → Use Cases & ICPs**: create ≥1 use-case + persona (the narrow problem
  you help with). This anchors the expert scope + topics.

---

## Phase B — App: configure Reddit targeting (NO connect button — OpenClaw connects)

**B1.** App → **Settings → Social Engagement → Settings tab**, set:
- **Tracked subreddits** (your starter list) → these become `targets`.
- **Topics** → what's on-topic for the brand.
- **Guardrails** → `require_disclosure = ON`; `avoid = [competitor bashing, unverified claims]`.
- **Posting mode** = **`approve_first`** (you review every draft before it posts).
- **Daily cap** = **3** (the safe start; you scale later — `SKILL-10`).

*(There is intentionally no "Connect Reddit" button here anymore — OpenClaw owns the connection,
see Phase D. `Connected Users` is read-only and will show the account once OpenClaw registers it.)*

---

## Phase C — App: turn the agent ON

**C1.** App → **Settings → Lead Generation → "Automation" card → toggle Autonomous ON** (admin).
This creates the `gtm_pipeline_settings` row the agent checks first; `stages.engagement` defaults
**on**. **STOP** lives in this same card (any member can hit it; Resume is admin).

---

## Phase D — Clawdi: create the cron (the program that runs OpenClaw)

**D1.** Confirm host secrets exist (you have these): `PLATFORM_URL`, `PLATFORM_ANON_KEY`,
`PLATFORM_AGENT_TOKEN` (= A1), `COMPOSIO_API_KEY`, `COMPOSIO_REDDIT_AUTH_CONFIG_ID`. *(For a
Reddit-only run you do NOT need `UNIPILE_DSN`/Apollo/Apify.)*

**D2.** Create **one cron** for this OpenClaw user and paste the program from
**`CLAWDI-PROMPT.md`** (the whole block between the lines, verbatim).

**D3.** Cron settings: schedule **`*/30 * * * *`** · session target **isolated** · delivery
**None** (or exactly one channel) · timeout **`1800`** · payload **empty**.

---

## Phase E — First run: authorize Reddit once

**E1.** On its first fire, OpenClaw initiates the Composio Reddit connection and **surfaces a
one-time OAuth URL** in the Clawdi run output. **Open it once and authorize** the brand's Reddit
account. *(This is the only manual touch — "connect from OpenClaw" still needs one human OAuth
click; it's just initiated by the agent, not an app button.)*

**E2.** OpenClaw then registers the account → it appears (read-only) under **App → Social
Engagement → Connected Users**.

---

## Phase F — Watch it work + verify (acceptance criteria)

**F1.** App → **Social Engagement → Engagements**: drafts appear (`status='draft'`), each tied to a
real **source thread** (`source_url`). The feed polls live.

**F2.** Click **Approve** on a good draft → status flips to `approved` → on its next fire OpenClaw
**posts it** → status flips to `published` with a **permalink** ("View on Reddit").

**F3.** Test **STOP**: App → Settings → Lead Generation → Automation → **Stop now** → confirm the
next fire does nothing ("pipeline paused by operator"); **Resume** restarts it.

**Done when:** drafts stream in (each with a real source_url), Approve → posts → permalink, Connected
Users shows the account, and STOP halts it. If a comment is ever removed by a subreddit, OpenClaw
flags that subreddit High and halts there (`SKILL-10`).

---

## Quick troubleshooting
| Symptom | Cause → fix |
|---|---|
| No drafts at all | No active subreddits/topics (B1), or autonomy off / paused (C1), or still in the observation window (`SKILL-02`). |
| "no pipeline settings" stop | You didn't flip Autonomous ON (C1) — the settings row doesn't exist yet. |
| Reddit OAuth never appears | Missing `COMPOSIO_API_KEY` / `COMPOSIO_REDDIT_AUTH_CONFIG_ID` (D1). |
| Draft posts but never publishes | You haven't Approved it (approve_first), or the Composio connection lapsed. |
| Agent cites a metric you don't recognize | Should be impossible (R0) — only `gtm_proof_points` are citable; report it. |

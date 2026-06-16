# Reddit Skill 09 — Know When to Ask for Help (escalate as a blocker)

> Source: `Skill - Know when to ask for help .pdf` ("How to Know When You Need Help and How to Ask for
> It"). **SDR-origin — ADAPTED.** The *fixable test, the three help types, and the four-part
> documentation are preserved exactly*; the *channel* moves to the live system: every escalation is an
> `openclaw_mission_events` row with `event_type='blocker'` that **names the exact tool, step, and
> stage** (R0.7). The Help-Log surface and Calendly link are SDR-origin artifacts with no Reddit-only
> home — flagged below. Inherits RULE-BOOK R0 + disclosure.

## Purpose
Tell the difference between a wall the agent can clear itself and one that genuinely needs a human —
and, for the ones that need a human, communicate them so clearly the person can resolve them fast. Most
obstacles are self-fixable (re-read the rules, validate, adjust, SKIP). Escalation is reserved for real
walls, because **every unnecessary escalation buries the ones that matter.**

## Trigger
- The agent hit something it cannot get past **after** it searched, tried, and thought (the test below).
- A required capability is missing: Composio Reddit not authorized, a `REDDIT_*` tool failing, a config
  field empty.
- A **decision only a human can make** is blocking engagement (new subreddit class, change
  `posting_mode`, allow a link, flip `autonomous`).
- Any R0.7 failure (tool/login/page/ambiguity) — a blocker is **mandatory**, not optional.
- Do **not** trigger on things SKILL-03 / SKILL-08 / R5 already solve (ambiguous rule → SKIP; weak
  relevance → validate or SKIP).

## Workflow
1. **Apply the fixable test before anything.** A problem is **fixable by you** if you have — or can get
   — what you need: research it, try another approach, re-read a skill / the live rules, adjust strategy,
   or SKIP (R5). It is **not fixable** if it needs something only a person can provide: a tool/
   integration to be built or connected, a skill/config that must be authored, a decision only the user
   can make, data only the operator/client has, or credentials you don't hold.
   - **The test:** if you **searched, tried, and thought** and still can't move forward → you need help.
     If you have **not** yet searched/tried/thought → do the work first. Never escalate a problem you
     could fix; never escalate something you haven't attempted.
2. **Classify which of the three help types it is** (this sets what you ask for):
   - **Build / connect** — a tool, integration, connection, or config field must be created. *"I need X
     built/connected so I can do Y."* (Reddit: Composio OAuth not authorized; missing
     `COMPOSIO_REDDIT_AUTH_CONFIG_ID`; `social_engagement_settings.targets`/`topics` empty.)
   - **Decide** — an unanswered strategic question needs human judgment. *"I need you to decide X so I
     can proceed."* (Reddit: enter a new/edgy subreddit class; switch `posting_mode`; allow a link;
     flip `autonomous`.)
   - **Thinking partner** — something is broken and you've exhausted your own diagnosis. *"I tried X, Y,
     Z; here's what I found; I'm stuck — can we think through this together?"* (Reddit: replies/upvotes
     dropped and SKILL-08 + SKILL-10 didn't explain it; a once-good subreddit went cold.)
3. **Document the four parts BEFORE escalating** (a poorly documented request wastes everyone's time; a
   well-documented one resolves fast):
   1. **What you were doing** — the precise task, stage/step, target. Not "engaging Reddit" but
      "drafting a comment for thread <url> in r/X at STEP 6 DRAFT."
   2. **What you tried** — every self-fix already attempted: the SKILL-08 validation/searches, the
      live-rule re-read (SKILL-03), the alternate threads, the voice adjustments. Tried nothing → you're
      not ready to ask.
   3. **What you found** — specifics, not "it didn't work": "`REDDIT_POST_REDDIT_COMMENT` returned 403
      on all attempts," "`targets` is empty," "removed in r/X within 5 min despite passing compliance."
   4. **What you need** — the exact unblock: a connection, a config field, a decision, a piece of data, a
      thinking partner. "I need the Composio Reddit account authorized" is specific; "I need help with
      Reddit" is not.
4. **Emit the escalation as a `blocker` that names the exact tool / step / stage.** One blocker per real
   wall; the four parts go in `message` + `payload` (schema below).
5. **Honor blocking vs non-blocking.**
   - **Blocking the engagement objective and can't wait** → emit the blocker **and surface it in the
     run's output** as a 2-minute briefing (what's blocked, what you tried, what you found, what you
     need — a clean yes / no / direction). If a human channel/booking link is configured, surface it too.
   - **Not immediately blocking** → log the blocker and **continue to the next thread/target/stage**;
     raise it at the next natural checkpoint. Not everything needs a meeting; some things resolve with a
     message.
6. **Move on cleanly.** After emitting, do not improvise around the missing capability and do not
   fabricate progress to avoid asking (R0.7). Continue other stages/threads; STOP only on the global
   STOP (R4).
7. **Treat recurrence as a systemic signal.** Your Help Log is your **durable Clawdi memory**: `memory_add`
   a one-line note for every blocker you emit, and `memory_search` it at run start (you **cannot** GET
   `openclaw_mission_events` — it is write-blind; the operator reviews the full blocker history in the app).
   If the **same** need recurs (same missing tool, config gap, decision class) across **2+ runs**, `POST
   gtm_insights` flagging it as a **structural** issue, not another one-off. A need that recurs *after*
   being resolved is a process failure; a need documented, resolved, and never recurring is the system
   getting smarter.

## Real-time learning loop (every run)
1. **Recall** your durable Clawdi memory (the Help Log) at run start via `memory_search` — NOT a GET on
   `openclaw_mission_events` (write-blind, no agent SELECT). `memory_add` each new blocker as you emit it.
2. **Adapt:** don't re-attempt a capability that hard-blocked last run (e.g., an unauthorized tool) —
   surface it as still-blocking instead of burning the run rediscovering it; route around non-blocking
   gaps to productive work.
3. **Detect recurrence:** the same need across **2+ runs** → `POST gtm_insights` (status `active`)
   naming the structural gap, so it's elevated from "repeated blocker" to "thing to build/decide."

## System mapping (do not invent columns)
| Artifact | Where it lives |
|---|---|
| Every escalation (all three help types) | `POST openclaw_mission_events { workspace_id, mission_id?, event_type:'blocker', message, payload }` |
| The four parts | `message:"doing: … \| tried: … \| found: … \| need: …"` and `payload:{ stage:'engagement', step:'<e.g. STEP 8 POST>', tool:'<e.g. REDDIT_POST_REDDIT_COMMENT>', status:'blocked' }` |
| Common Reddit blockers to name | Composio OAuth not authorized (`COMPOSIO_REDDIT_AUTH_CONFIG_ID`); `REDDIT_CREATE_REDDIT_POST` / `REDDIT_POST_REDDIT_COMMENT` 4xx/permission; a write missing `Prefer: return=minimal`; `social_engagement_settings` empty / `targets` undefined; an ambiguous live subreddit rule |
| Help Log (agent's own recall) | `memory_add` a note per blocker → `memory_search` at run start (`openclaw_mission_events` is write-blind to the agent; the operator reviews the full history in the app) |
| Recurring need → systemic flag | `POST gtm_insights { workspace_id, use_case_id?, title, observed, evidence, hypothesis, change_made, status:'active' }` (append-only; no UPDATE) |
| "Decide" requests target these (agent requests; human/admin sets) | `gtm_pipeline_settings` (`autonomous`, `paused`, `stages.engagement`) · `social_engagement_settings` (`posting_mode`, `targets`, `guardrails`) |
| Human approval of drafts (not an escalation — the normal gate) | app-side `social_engagement_action_review` (sets `status='approved'`) |

**Flagged mappings:**
- **No dedicated Help-Log surface**, and the agent **cannot read `openclaw_mission_events`** (write-blind).
  The agent's own recall lives in **Clawdi memory**; the operator reviews the raw blocker history in the app.
  Track resolution by a follow-up `note` event or a `gtm_insights` row. (Agent `event_type` ∈
  `status_change|progress|blocker|result|note`.)
- **No `{CALENDLY_LINK}` in the Reddit-only env.** The source's "book a 10-minute meeting" maps to:
  surface the blocking briefing in the run output (and a configured operator channel/booking link if one
  exists). Kept as concept, flagged as not-configured here.
- Whether the agent may **self-write** `social_engagement_settings`/`gtm_pipeline_settings` for a
  "decide" item, or must only request it, depends on autonomy/role config — default to **request via
  blocker**; do not flip config the agent isn't authorized to write.

## Anti-hallucination & guardrails
- **Earn the escalation:** never escalate before search + try + think (R5 first — SKIP is free). Never
  escalate what you can fix by re-reading rules (SKILL-03) or validating (SKILL-08).
- **State only real facts** in the blocker — real tool names, real step, real counts of what completed.
  Unknown cause → `"unknown"`, never a guessed reason (R0.1).
- **Never fabricate progress** to avoid asking, and never improvise around a missing capability to "keep
  going" (R0.7).
- **Never argue** with mods/users as a substitute for escalating — disengage and log the blocker (R1).
- **One blocker per real wall**, naming exact tool/step/stage; then continue other work (don't halt the
  whole run unless STOP, R4).
- **Don't bury the signal:** only true walls become blockers, so the important ones stay visible.

## Acceptance criteria
- Every escalation passes the fixable test (searched + tried + thought) and carries all **four parts**
  (doing / tried / found / need).
- Every escalation is an `openclaw_mission_events` `blocker` that **names the exact tool, step, and
  stage**; no vague "I'm stuck."
- The help type (build/connect · decide · thinking-partner) is identifiable from the request, and the
  ask is specific.
- No fabricated cause; unknown is `"unknown"`; the agent moved on cleanly and did not improvise around
  the gap.
- Blocking items are surfaced in the run output; non-blocking items are logged and deferred to the next
  checkpoint.
- A need recurring across 2+ runs produces a `gtm_insights` row flagging the systemic gap.
- Nothing the agent could have fixed itself (ambiguous rule → SKIP, weak relevance → validate) was escalated.

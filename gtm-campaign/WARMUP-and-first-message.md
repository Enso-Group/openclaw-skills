# WARMUP + First Message (depth)

Depth file for the **WARMUP** and **FIRST MESSAGE** stages referenced by `gtm-campaign/SKILL.md`.
This is the **Clawdi hosted-runtime** methodology (what the agent does each fire), NOT app/Lovable
code. It faithfully encodes the *"LinkedIn Connection Approval Playbook"* (the 10-touch warmup +
invite note) and the post-acceptance first-message method, reconciled with the live contract (real
tables/tools only — no invented columns).

**Provenance legend:**
- `[Warmup PDF]` — `Downloads/2.List building/2. Warmup + send linkedin message.pdf` (the 10-touch
  playbook, invite note, timing, safety rules — primary source for **Part A**).
- `[Contract]` — `gtm-campaign/SKILL.md` + `CLAWDI-CRON-PROMPT.md` (live tables/tools/secrets; the
  post-acceptance first-DM rules — primary source for **Part B**).
- `[First-message skill]` — `03-SKILL-LIBRARY.md` §"SKILL 4" (the repo's faithful extraction of the
  first-LinkedIn-message methodology; deepens **Part B**, marked as EXAMPLE where niche).
- `[Lead Gen PRD]` — `Lead_Generation_-_Component.pdf` (the APP boundary: it sends follow-ups +
  ingests replies; the agent owns only the first touch).

> **The principle `[Warmup PDF]`:** by the time the invite arrives, the target has seen the account's
> name **9 times across different contexts** over ~2 business days. The invite is the **10th touch** —
> so it feels like the obvious next step, not a cold approach. Every action is automated.

---

## 0. Gates — BEFORE any touch or message `[Contract]`

1. **STOP check first (re-check between stages):** `GET
   gtm_pipeline_settings?select=autonomous,paused,...,min_score,stages,daily_caps,target_geography`.
   Empty / `paused=true` / `autonomous=false` → STOP cleanly; if `paused` flips true mid-run, stop.
2. **Stage toggles:** WARMUP runs only if `stages.warmup`; FIRST MESSAGE only if `stages.message`.
3. **Daily caps (per sending account/day) from `daily_caps`** — defaults reconcile exactly with
   `[Warmup PDF]`'s safety table:

   | Cap | Default | Counts which touches |
   |---|---|---|
   | `invites` | **20** | touch 10 |
   | `messages` | **25** | the first DM (Part B) `[Contract]` |
   | `likes` | **40** | touches 2, 3, 5, and the company-post like in 7 |
   | `comments` | **10** | touches 4 and 6 |
   | `endorsements` | **5** | touch 8 |

4. **NO actions on Saturday or Sunday `[Warmup PDF]`.** Randomize all timing (fixed intervals are a
   bot signal).
5. **R0 anti-hallucination `[Contract]`:** every warmup action and every message carries a **real
   `source_url` you opened**; counts match reality; **never repeat an `action_no` for a lead**; never
   message the same prospect twice. On ANY failure → blocker event + continue.

Connection/headers as in `FIND-list-building.md` §0; **every POST sends `Prefer: return=minimal`.**

---

# Part A — WARMUP: the 10-touch approval playbook `[Warmup PDF]`

**Who to warm `[Contract]`:** leads at `pipeline_stage` `sourced`/`queued`
(`GET lead_generation_leads?select=id,linkedin,pipeline_stage,...` where readable; if not visible,
warm by the **staged `linkedin_url`** and set `recipient_linkedin_url`).

**One due touch per fire `[Contract]`.** The source runs all 10 across 2 days; the live runtime fires
incrementally, so each fire advances the **next due** touch for a lead per the schedule below (don't
batch all 10 in one fire). Touches **1–9 via Composio** (the user's own LinkedIn — Composio cannot
cold-invite/DM); **touch 10 via Unipile**.

## The 10 actions

| # | Action | `action_type` | Day / time (±30 min) | Tool | Notification & why `[Warmup PDF]` |
|---|---|---|---|---|---|
| 1 | View their profile | `view` | D1 ~9:00 | Composio | Profile-view ping — they click through to see who looked. |
| 2 | Like their **most recent** post | `like` | D1 ~9:15 | Composio | Second ping in quick succession — curiosity compounds. |
| 3 | Like their **2nd most recent** post | `like` | D1 ~11:00 | Composio | Two likes same day = genuine interest, not a random click. |
| 4 | **Unique LLM comment** on their most recent post | `comment` | D1 ~14:00 | Composio | The most powerful touch — specific, shows you read it. **Never a template, never "Great post."** |
| 5 | Like a comment they left on **someone else's** post | `like_comment` | D1 ~15:30 | Composio | "I noticed what you *said*, not just what you posted." Almost nobody does this. |
| 6 | Comment on a post they **recently commented on** | `comment_shared` | D1 ~17:00 | Composio | You appear in a shared conversation — looks organic, same circles. |
| 7 | **Follow their company page** + like a recent company post | `follow_company` | D2 ~9:00 | Composio | Interest in the business, not just the person — founders/leaders watch the company page. |
| 8 | **Endorse one of their top skills** | `endorse` | D2 ~10:00 | Composio | Small, personal, 5 seconds — almost nobody does it in outreach. |
| 9 | **Share their best recent post + tag them** | `share` | D2 ~11:30 | Composio | One of LinkedIn's strongest notifications. **LLM evaluates post quality first — only share if it's genuinely strong.** |
| 10 | **Send the connection invite** + personalized note | `invite` | D2 **14:00–16:00 local** | **Unipile** | The 10th touch — after 9 contexts it feels expected, not cold. |

**Timing law `[Warmup PDF]`:** all times randomized within a **±30-minute** window; **2–8 min**
randomized delay between actions on the same person; the **invite send window is 2pm–4pm local
time**. No fixed intervals.

## The invite note (touch 10) `[Warmup PDF]`

LLM-generated **per person**, **under 200 characters (strict)**. It explains who's sending + why and
references something specific from the target's recent activity, so it never reads like a template.
No two targets get the same text. **Generation prompt:**

```text
Write a LinkedIn connection invite message.
Sender's name: [SENDER NAME]
Sender's role: [SENDER ROLE / WHAT THEY DO]
Target's name: [FIRST NAME]
Target's job title: [JOB TITLE]
Their most recent post topic: [POST TOPIC OR SUMMARY]
The message must:
- Be under 200 characters (strict limit)
- Briefly explain who the sender is and what they do — one short phrase
- Reference something specific from the target's recent post or activity
- Sound warm and human — not a pitch, not a template
- Not ask for a meeting or a call
- Make the target feel seen, not targeted
Output only the message text.
```

Example output *style* (the LLM generates it, not a fixed template) `[Warmup PDF]`:
> "[Sender] here — I work with [what sender does]. Saw your post on [topic] and it hit close to home.
> Would love to connect."

**Send the invite via Unipile (JSON) `[Contract]`:** `POST {UNIPILE_DSN}/api/v1/users/invite`
`{ account_id, provider_id, message? }`, header `X-API-KEY`. **`UNIPILE_DSN` must be the FULL base URL incl.
scheme + port** — e.g. `https://api8.unipile.com:13443` (so `{UNIPILE_DSN}/api/v1/...` resolves; a bare host with
no `https://` fails). The `account_id` is a **LinkedIn account connected INSIDE Unipile** (Unipile ≠ Composio —
connect one in the Unipile dashboard; the Composio LinkedIn does NOT work here). First call `GET
{UNIPILE_DSN}/api/v1/accounts` (header `X-API-KEY`) to get the `account_id`. **If `UNIPILE_DSN` is
absent/misformatted or no Unipile LinkedIn account exists → POST a blocker for touch 10 and continue** (touches
1–9 still ran via Composio).

## What the agent needs per person — already in the scrape `[Warmup PDF]`

| Input | Source |
|---|---|
| LinkedIn profile URL | the scored lead list (FIND) |
| Most recent post content | activity feed |
| A post they recently commented on | comment activity |
| Their top skill | profile |
| Their company LinkedIn URL | profile |
| Job title + company stage | the scored lead list |

> All of this is available from the **Apify Full-mode scrape already in the pipeline** — *no
> additional data collection needed* (see `FIND-list-building.md` §3b).

## Record EVERY action `[Contract]`

```json
POST /rest/v1/gtm_warmup_actions
{
  "workspace_id": "<ws>",
  "lead_id": "<if readable, else omit>",
  "recipient_linkedin_url": "<the prospect profile URL>",
  "mission_id": "<if present>",
  "action_no": 4,
  "action_type": "comment",
  "status": "done",
  "executed_at": "<iso8601>",
  "source_url": "<the exact page you acted on>",
  "evidence": "<what you did, e.g. the comment text / which skill you endorsed>"
}
```

`action_type` ∈ `view | like | comment | like_comment | comment_shared | follow_company | endorse |
share | invite`. **Idempotency:** never repeat an `action_no` for the same lead; check caps before
each action and stop at the cap. **On the invite (touch 10):** also open the SDR thread you'll
message into once it's accepted (Part B).

## Full flow `[Warmup PDF]`

```text
DAY 1   9:00 view · 9:15 like post1 · 11:00 like post2 · 14:00 LLM comment
        15:30 like-their-comment · 17:00 comment-on-shared-thread
DAY 2   9:00 follow company + like post · 10:00 endorse skill · 11:30 share + tag
        14:00–16:00 SEND INVITE (personalized note)
→ target has seen the name 10 times across 10 contexts. The invite is expected, not cold.
```

---

# Part B — FIRST MESSAGE: after the connection is ACCEPTED `[Contract]` + `[First-message skill]`

Scope: the **first DM to a new connection** — get the first reply. (It does **not** run follow-ups,
handle objections, or book meetings — see the boundary note.)

## Trigger + the 30-minute rule
When a connection is **ACCEPTED**, **wait 30 minutes**, then send `[Contract]`. A message that
arrives the instant someone accepts looks automated and kills trust `[First-message skill]`. If it
was accepted > 30 min ago → send now; if < 30 min → queue to the 30-minute mark. No exceptions.

## Build the message (in order)
1. **Read both profiles `[First-message skill]`:** the **sender** (the connecting account — find the
   one credibility hook; write *like them*) and the **prospect** (recent posts, company openings,
   company news).
2. **Form the THESIS `[Contract]`:** the prospect's one *"11pm question"* — the thing a real person
   in their role types into Google at 11pm (persona pain × their recent activity × a
   `gtm_geo_questions` seed).
3. **Find the matching blog article `[Contract]`:** search the client blog with the use-case
   `gtm_keywords` — **try ≥ 3 angles**, pick the **closest educational** article (a guide / breakdown
   / case study). Never a homepage, pricing, or product page; always pick the closest available —
   never stop because the match isn't perfect. **The article goes in message 2, never message 1.**
4. **Write 3 lines / ≤ 300 chars `[Contract]`** — it must read like a text message, not an email:

   | Line | Job `[First-message skill]` |
   |---|---|
   | 1 — **Trigger** | one specific observation from their profile/activity (or a direct pain hypothesis). Not a compliment, not a pleasantry. |
   | 2 — **Tease** | reference the article's value **without sending it** — the "most important line": specificity + a hint of surprise = an information gap. |
   | 3 — **CTA** | ask them to reply to receive it (low-friction), e.g. *"Want me to send it over?"* |

5. **Voice + truth `[Contract]`:** on-brand — obey `gtm_voice_rules` (say / never-say / tone) +
   `gtm_brands` positioning, in the **sender's** real voice; **cite ONLY real `gtm_proof_points`**
   (never an invented metric — R0). Pick one **angle**:
   `pain | ambition | identity | contrarian | raw_question`.

**Never in the first message `[First-message skill]`:** a link (of any kind); "Hope you're doing
well"; "Thanks for connecting"; a meeting/call ask; a feature pitch; paragraphs; banned words; the
article itself before they reply.

**Example first DMs** (same use-case, two senders — *the message must sound like the sender*;
these are EXAMPLES, the LLM writes fresh) `[First-message skill]`:
> "Saw you've posted three SDR job listings in two weeks — big push. We found the one thing that
> actually cuts ramp time in half, and it's not what the playbooks say. Want me to send it over?"
>
> "Scaled an SDR team 4→22 reps; the ramp-time thing nearly broke us. Found one thing that actually
> fixed it — pretty different from the usual advice. Want me to send it over?"

## When nothing replies — the angle ladder `[First-message skill]`
If ~20 sends get no replies, fix in this order: (1) **tease** — does it create curiosity or just
state a fact? (2) **line-1 specificity** — go from "could be sent to a thousand people" to "could
only be sent to this one"; (3) **angle ladder**: pain (crowded) → **ambition** → **identity** →
**contrarian** → (4) **strip to a raw question** (no trigger/tease/CTA); (5) **swap the sender** —
same message, different account. Every attempt is a test: beats the standard by **≥ 25% over ~20
sends** → it earns a place; otherwise drop it (variant tracking lives in the A/B `test` stage —
tag the `variant_id` you sent).

## Send + record `[Contract]`
**Send the DM via Unipile (multipart/form-data, NOT JSON):** start a chat =
`POST {UNIPILE_DSN}/api/v1/chats` (form fields `account_id`, `attendees_ids` (repeat per attendee),
`text`); header `X-API-KEY`. **Missing `UNIPILE_DSN` → blocker + continue.** Then record the thread
+ message — **generate the UUIDs client-side** (writes are minimal/no read-back) so the message links
to the thread:

```json
POST /rest/v1/sdr_threads
{
  "id": "<thread_uuid you generate>",
  "workspace_id": "<ws>",
  "lead_id": "<if known>",
  "mission_id": "<if present>",
  "platform": "linkedin",
  "mode": "invite_then_message",
  "recipient_linkedin_url": "...",
  "recipient_name": "...",
  "status": "active",
  "cadence_days": [3, 5],
  "follow_up_step": 0,
  "follow_up_at": "<now + 3 days, iso>",
  "last_outbound_at": "<iso>"
}
```

```json
POST /rest/v1/sdr_messages
{
  "workspace_id": "<ws>",
  "thread_id": "<the thread_uuid above>",
  "direction": "outbound",
  "kind": "message",
  "status": "sent",
  "sent_at": "<iso>",
  "body_text": "<the 3-line message>",
  "thesis": "<the 11pm question>",
  "article_url": "<chosen article — for message 2, not sent yet>",
  "article_title": "...",
  "angle": "pain",
  "variant_id": "<the A/B variant you sent>",
  "sender_account": "<sender name>"
}
```

> **Boundary `[Contract]` + `[Lead Gen PRD]`:** **YOU send only the FIRST message.** The APP's
> `sdr-cadence-tick` sends follow-ups (`follow_up_step ≥ 1` on `cadence_days`); the app's
> `webhook-unipile` ingests replies (and suppresses echoes of your own send). Caps: the first DM
> counts toward `daily_caps.messages` (25); no weekends; randomized timing. The PRD confirms a
> LinkedIn reply requires an **accepted invite + a prior outbound DM** — which is exactly this flow.

---

## Real-time learning (tune every fire) `[Contract]` + `[First-message skill]`

- **Reply rate by angle/variant → what to write next.** Track replies ÷ sends per
  `angle` / `variant_id` (and per **sender** — sender is a *measured* variable, not one you change).
  Lean toward the leading angle/tease/CTA for that use-case; when one stalls, climb the angle ladder.
  The A/B tables are token-scoped readable, but rebuild the live tally from your own sends/replies (re-post
  counts in your decision-log events) — never trust a winner on `< 20` sends (small-sample noise).
- **Copy has a shelf life `[First-message skill]`.** When a winner's reply rate decays, decide *did
  the world change* (new angle — re-read `gtm_geo_questions` + the ICP's recent posts) *or did the
  copy go stale* (same angle/thesis/article, fresh trigger + tease). A fresh sender can reset a
  decayed relationship.
- **Blockers → don't repeat them.** If `UNIPILE_DSN` was missing last fire, the invite/DM stays
  blocked — keep advancing warmup touches 1–9 (Composio) and re-attempt sending only once Unipile is
  configured. GET your own recent `openclaw_mission_events` (token-scoped readable via the readback grant
  `20260616170000`) to avoid re-failing the same way (Clawdi memory is an optional extra).
- **Warmup pacing.** `gtm_warmup_actions` is token-scoped readable (readback grant `20260616170000`) — GET
  your own rows to compute each lead's next due `action_no` + cap headroom (no reliance on Clawdi memory,
  which is OFF without an embedding key); if a cap is hit, defer the touch to the next eligible (weekday)
  fire rather than exceeding it.

---

## SKIP / blocker quick-reference

| Situation | Action |
|---|---|
| `paused` / `autonomous=false` | STOP cleanly |
| `stages.warmup` / `stages.message` false | skip that part |
| Weekend (Sat/Sun) | no actions |
| A daily cap reached | stop that action type; defer to next weekday fire |
| `UNIPILE_DSN` missing | blocker for touch 10 + first DM; touches 1–9 continue via Composio |
| Composio LinkedIn URN unavailable (post/comment) | blocker for that touch; continue |
| Touch 9 post not genuinely strong | skip the share (quality gate); proceed to invite |
| Connection not yet accepted | no first message yet (Part B trigger unmet) |
| Accepted < 30 min ago | queue the DM to the 30-minute mark |
| No real `source_url` / can't verify activity | skip the touch/message (no source → no row) |
| Any tool/login/page failure | blocker event + continue to next lead/stage |

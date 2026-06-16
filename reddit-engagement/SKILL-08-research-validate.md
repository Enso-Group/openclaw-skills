# Reddit Skill 08 — Research & Validate (is this subreddit/topic/thread worth engaging?)

> Source: `Skill - How to learn to solve complex situation.pdf` ("How to Find Answers and Validate
> Decisions"). **SDR-origin — ADAPTED.** The *method is preserved exactly*; the *application* moves to
> Reddit: validate a **subreddit**, a **topic**, or a single **"is this thread worth engaging?"**
> decision. The one bend: the source's "small cheap test = send a handful of messages" — cold outbound
> is banned (R1), so the test becomes **one capped, approved, disclosed, value-first comment** measured
> by SKILL-10 (flagged below). Inherits RULE-BOOK R0 + disclosure.

## Purpose
Stop guessing. When relevance / compliance / value is not obvious — a new subreddit, a fuzzy topic, a
thread that *might* fit, a strategy that stopped working — find the answer in the conversation that
already exists, then apply a rigorous standard **before** spending a draft (and a human's approval) on
it. Output is a decision: **engage / refine / drop**, with real evidence behind it.

## Trigger
Run before committing a draft whenever any holds:
- A target in `social_engagement_settings.targets` is **new/unproven** (no yield history).
- A thread passes compliance (SKILL-03) but value/relevance is **uncertain** — "should we even be here?"
- Engagement that used to land has **stopped working** (replies/upvotes fell — SKILL-10).
- A `topics` entry is ambiguous, or you can't tell who actually has the problem.
- Any moment the honest answer is "I'm not sure this is worth it." (R5: doubt is the trigger, not a
  license to wing it.)

## Workflow
1. **Reframe to the frustrated, human form.** Turn the polished question ("is r/X worth engaging?") into
   what a real person types when annoyed ("why is everyone in r/X so done with [tool]", "what do people
   here actually want help with"). Frustrated phrasing surfaces real threads. Generate several phrasings.
2. **Search where the conversation already happens.** On Reddit the conversation *is* the subreddit/
   thread: use the connected Composio Reddit account (read) and **open and read the live pages this
   run** (R0.4). If broader search tooling is configured in the host env (Apify RAG / Reddit Scraper /
   web search), run it in **parallel** for outside corroboration; if it is **not** configured
   (Reddit-only program ships Composio only), **do not pretend you searched it** — rely on live Reddit
   pages and **SKIP** when you can't gather enough (R5).
3. **Read the answers as carefully as the questions.** Who replies — practitioners who lived it, or
   speculators? Are top answers upvoted/accepted, or challenged/contradicted? The quality of the
   **supply** (answers) tells you as much as the **demand** (questions). Pull full post + comment
   bodies; the honest signal is often three levels deep.
4. **Score DEMAND × SUPPLY.**
   - **Demand** = the problem is real and actively felt: questions, complaints, threads started, upvotes
     on posts that name the pain.
   - **Supply** = others already solving it: answers given, tools recommended, people tagged, products
     mentioned.
   - **Strongest signal = high DEMAND + SUPPLY that falls short** (people asking, existing answers
     incomplete/outdated/wrong). That gap is where genuine value — and a worthwhile engagement — lives.
   - Demand but **no supply** → a genuine gap **or** a problem with no solution (dead end). Find out
     which **before** engaging.
   - Supply but **no demand** → saturated or the pain isn't acute → weak premise → don't force it.
5. **Apply the FOUR tests (all four must pass to engage):**
   1. **Independence** — evidence from sources that don't know each other. One 200-comment thread = *one*
      source; a thread + a different subreddit + an outside post each describing it = three.
   2. **Consistency** — different people, similar language, no contact. Stretching to connect them = not
      confirmed.
   3. **Volume & engagement** — one highly engaged post (200 comments) outweighs 50 posts with 2 upvotes.
      Many people caring beats one caring deeply.
   4. **Surprise** — real research surfaces something you didn't expect. If everything matched your
      assumption, you confirmed your bias and haven't dug deep enough.
   - **All four → engage** (draft, SKILL-04). **Any fail → refine the question and re-search, or drop**
     (R5 SKIP). Never act on a premise because you want it to be true.
6. **Resolve contradictory evidence — never average it.** Half the threads say one thing, half the
   opposite — that's signal, exactly one of:
   - **True for a subset** → narrow the target (this subreddit/segment yes, that one no) — don't abandon.
   - **Market shifted** → check the **timeline**: loud old posts + silent recent posts = the pain got solved.
   - **Two different problems** wearing the same words → separate them, validate each independently.
   - "Never average it out and call it uncertain. That is lazy. Dig into why it splits." The reason is
     usually more useful than the original question.
7. **Can't find an answer at all → the question is the problem.** Try a different frame ("what would
   someone have to believe for this to be the wrong question?"); **search the outcome, not the problem**
   ("how do I get X to work" finds different threads than "X problem"); follow the **adjacent questions**
   the community asks constantly as your new entry point.
8. **Acting before full validation → document three things FIRST** (`POST gtm_insights`, status `active`):
   - **What you know** — the evidence + the real `source_url`s + why it convinced you (`observed`/`evidence`).
   - **What you don't know** — open questions, hypothetical parts, what you searched for and couldn't
     find (`hypothesis`).
   - **What would change your mind** — the specific signal (a reply/upvote pattern, a removal, missing
     evidence) that would say "stop" (`change_made` = the planned response). If you can't state this,
     you're not testing a premise — you're executing a belief.
9. **Still unsure → run a small, cheap test (Reddit-adapted).** NOT a batch of cold messages (R1 bans
   cold outbound/DMs). The test is **one** standalone-useful, disclosed, approved comment within
   `daily_cap` on the single most representative thread. Read the result via SKILL-10: real fit →
   replies (`metrics.comments`)/upvotes; a weak premise → silence, downvotes, or removal. Record the
   outcome (a `gtm_sources` yield row + a `gtm_insights` row when a pattern holds), refine, repeat.
   Research → validate → test → learn → refine never ends; it just gets faster.

## Real-time learning loop (every run, before validating a known target)
1. **Read** the readable surface first: recent own `social_engagement_actions` + `.metrics` for that
   subreddit/topic (a removal / heavy-negative = already a "drop" — don't re-validate it into a re-engage).
   For "did it block last run?", GET your own recent `openclaw_mission_events` (token-scoped readable via
  the readback grant `20260616170000`; Clawdi memory is an optional extra, and OFF without an embedding key).
2. **Fold that real signal into demand×supply** — your own past replies/upvotes/removals are first-party
   evidence, weighted alongside the live threads.
3. **Persist** a `gtm_insights` row when a validate/disprove pattern has held **≥2 periods** (a subreddit
   that always welcomes, a topic that always flops). Emit a `progress` event with the engage/refine/drop
   verdict so the operator sees the reasoning live.

## System mapping (do not invent columns)
| Step / artifact | Where it lives |
|---|---|
| Targets/topics/guardrails being validated | `social_engagement_settings` (`targets`, `topics`, `guardrails.avoid`, `guardrails.require_disclosure`) — read |
| Live threads read for evidence (demand/supply) | Composio Reddit (connected `social_accounts`) + the **live page opened this run**; every cited thread = a real `source_url` |
| Subreddit/topic yield (accumulated demand/supply + cheap-test result) | `POST gtm_sources { workspace_id, platform:'reddit', query:'<subreddit/topic>', prospects_found, qualified }` (append-only/blind) |
| Four-test verdict + demand×supply gate for a specific draft | `social_engagement_actions.metadata.validation` (beside `metadata.compliance`; engage only if verdict=pass) |
| Documented uncertainty + validated/disproven premises | `POST gtm_insights { workspace_id, use_case_id?, title, observed, evidence, hypothesis, change_made, status:'active' }` (append-only; no UPDATE → supersede by appending) |
| Notes/verdicts emitted while researching | `POST openclaw_mission_events { event_type:'progress', payload:{stage:'engagement'} }` |
| Any research/read tool failure | `POST openclaw_mission_events { event_type:'blocker' }` (SKILL-09) |

**Flagged:** there is no dedicated "knowledge base" surface for accumulated premises — `gtm_insights`
is the closest home and is used here. If a workspace later configures one, learnings move there.

## Anti-hallucination & guardrails
- **R0:** every piece of demand/supply evidence is a **real thread the agent actually opened**, captured
  as a `source_url`. Never invent a thread, quote, username, upvote count, or a "happy user." Unknown →
  `""`/`"unknown"` or SKIP.
- Subreddit signals/rules are **re-read from the live page this run** (R0.4), never remembered.
- **The four tests are a gate, not a vibe:** all four → engage; any fail → refine or **drop** (R5).
  Don't average contradictions; explain the split.
- The "cheap test" obeys every rule: ≤ `daily_cap` (start 3/day **total**), approve-first, disclosed,
  standalone-useful, **one** engagement per thread, **no cold outbound/DMs** (R1/R3/R4). One capped
  comment, never a blast. (SDR→Reddit adaptation — flagged.)
- Don't act on a premise because you want it true; record **what would change your mind** before any
  act-anyway (falsifiability).
- On any tool/page/ambiguity failure → emit a `blocker` and move on; never improvise (R0.7).
- When in doubt → **SKIP** (R5). A missed thread is cheap; a low-value or non-compliant post is not.

## Acceptance criteria
- Every "engage vs skip" decision is backed by ≥1 real, opened thread (`source_url`); zero invented
  evidence or metrics.
- Demand × supply is assessed and the **four tests explicitly evaluated**; the agent engages only when
  all four pass — otherwise it refined the question or dropped the target.
- Contradictory evidence is explained as subset / market-shifted / two-problems — **never averaged into
  "uncertain."**
- Any act-before-full-validation has a `gtm_insights` row (status `active`) with know / don't-know /
  **what-would-change-my-mind** written *before* acting.
- The cheap test, when used, is a single capped, disclosed, approved comment — never cold outbound,
  never over `daily_cap`.
- The run consulted real first-party signal (own recent metrics + blockers) before re-validating a
  known target; a removed/halted subreddit was not re-engaged.

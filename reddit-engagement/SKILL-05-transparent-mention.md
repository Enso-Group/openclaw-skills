# Reddit Skill 05 — Transparent Product & Link Mentioning

> Source: "OpenClaw Skill 6: Transparent Product and Link Mentioning" — its pre-condition check, the
> Bad/Good disclosure table, the five URL conditions, the rules-supersede-the-user conflict rule, and
> both Q&As are all preserved below. Rewritten for autonomous OpenClaw (transparent model); inherits
> RULE-BOOK R0 + disclosure; voice from `IDENTITY.md`.

## Purpose
Govern the single most ban-prone action: naming the product or dropping a URL. Mentions and links are **rare exceptions, not the rule.** They are allowed only when a user **explicitly asked**, the **subreddit allows** it, and the comment carries a **natural, transparent disclosure** of affiliation. Done wrong, it flags the account as a marketer and gets it banned — the opposite of the value-first, transparent model.

## Trigger
While drafting (SKILL-04) for an already-eligible thread, EITHER: the user **explicitly asked** for tools / workflows / examples / product recommendations, OR a candidate draft would include a URL. Any product mention or link decision routes through this skill first.

## Workflow
1. **Pre-condition gate — BOTH must hold** (if either fails → educational-only answer, no mention, no link; continue under SKILL-04):
   1. **User Request:** the thread's user **explicitly** asked for tools, workflows, examples, or product recommendations. Quote the exact ask. A vague or adjacent thread does NOT count.
   2. **Subreddit Permission:** the **live subreddit rules, re-read this run** (R0.4/R1), explicitly allow brand mentions and/or links — consistent with `social_engagement_settings.guardrails` (`avoid`, link policy). Remembered rules do NOT count.
2. **Disclosure is mandatory.** If `social_engagement_settings.guardrails.require_disclosure=true` (default in this model), every mention/link MUST disclose affiliation. There is no undisclosed-mention path.
3. **Craft the transparent mention.** The product is **context for the advice, not the whole comment.** Disclose the bias in the **first clause**, then pivot to educational value (IDENTITY §2):

| Bad (never) | Good (use this) | Why |
|---|---|---|
| "You should try [PRODUCT]." | "I work on [PRODUCT], so I'm biased, but the pattern we see a lot is…" | Discloses bias immediately, then returns to teaching. |
| "[PRODUCT] is the best tool for this." | "Full disclosure, I'm connected to [PRODUCT]. The general thing I'd look for is…" | Builds trust through honesty before any recommendation. |

4. **Standalone-value test.** Remove the mention — is the comment still useful on its own? If **no**, it is an ad → rewrite to teach first, or SKIP the mention (R3).
5. **URL handling (exceedingly rare).** Add a URL only if **ALL** are true:
   - the subreddit allows links, **and**
   - the user asked for a link, **and**
   - the link directly answers the question, **and**
   - affiliation is disclosed if the link is connected to the brand, **and**
   - the comment stays useful even if the user never clicks it.
   If any condition fails → no link. **Never drop a link as the main point of the comment.**
6. **Conflict rule — subreddit rules supersede the user request.** If the user asked for a link but the subreddit bans them: do **not** link. Tell the user links aren't allowed in this subreddit and give the info/steps in **plain text** instead.
7. **Hand off to the draft write.** The resulting comment is persisted by SKILL-04 as ONE `social_engagement_actions` row: `status='draft'`, `source_url` REQUIRED, `created_by=null`; never `external_id`/`permalink`/`published_at`/`approved_by`. Flag the review payload in `metadata`: `mention_used`, `link_used`, `disclosure_included` — **stamped at INSERT** (the agent's column grant can't add `metadata` after insert).
8. **Ambiguity → SKIP the mention/link.** Any doubt about permission or whether the ask was explicit → keep the educational answer, drop the mention/link (R5).

## Real-time learning (let prior reception gate the next mention)
Mentions are the most ban-prone action, so weight them by what actually happened (read your own
published rows; `gtm_ab_*` are token-scoped readable too, but reconstruct the live tally from `social_engagement_actions`):
- **A disclosed mention/link that was `removed` or drew `negative_replies`/heavy `downvotes` in a
  subreddit → stop mentioning there.** Default that subreddit to educational-only until SKILL-01 +
  SKILL-10 re-clear it. A removal is a hard stop even at n=1 (R4 — never scale on a negative).
- **A disclosure phrasing that earned a positive reception** (upvotes/replies, no removal) → reuse
  that natural framing. Read `metrics` (`upvotes`,`replies`,`downvotes`,`removed`,`negative_replies`)
  on rows where `metadata.mention_used=true`; learn only from really-observed numbers (R0).
- This never loosens the gates — both pre-conditions and disclosure still apply every time; learning
  only makes mentions **rarer** where they were rejected. The durable loop lives in SKILL-10.

## System mapping
| Need | Table / field / RPC |
|---|---|
| Pre-condition A — user asked | the live thread itself (`target_url`/`source_url`) — quote the explicit ask; no field is invented for this |
| Pre-condition B — subreddit allows | **live subreddit rules re-read this run** (authoritative; R0.4/R1), overlaid by `social_engagement_settings.guardrails` (`avoid` list, link policy) |
| Disclosure requirement | `social_engagement_settings.guardrails.require_disclosure` (true → disclosure mandatory on every mention/link) |
| Voice / positioning / claims | `gtm_voice_rules`, `gtm_brands` (positioning), `gtm_proof_points` (`is_citable=true`) — cite ONLY real ones; one-read `gtm_brand_book` |
| Where the mention/link lands | `social_engagement_actions` { …, `content` (mention as context / plain-text fallback), `source_url` (NOT NULL), `status:'draft'`, `created_by:null`, `metadata:{ mention_used, link_used, disclosure_included }` (stamped at INSERT) } — never `external_id`/`permalink`/`published_at`/`approved_by` |
| Prior reception (learning) | `social_engagement_actions?status=eq.published` where `metadata.mention_used=true` → `metrics` (`upvotes`,`replies`,`downvotes`,`removed`,`negative_replies`); agent **SELECT** only |
| Human approve (gate) | RPC `social_engagement_action_review` (member: approve\|reject); the agent never self-approves |
| STOP / stage gate | `gtm_pipeline_settings` (`paused`, `autonomous`, `stages.engagement`) |
| Telemetry / blocker | `openclaw_mission_events` { `event_type:'progress'`\|`'blocker'`, `message`, `payload` } · `created_by` null |

## Anti-hallucination & guardrails
- **Never fabricate permission.** Never claim "the user asked" unless they literally did (quote it); never claim "the subreddit allows it" from memory — re-read the live rules this run (R0.1/R0.4).
- **No unprompted promo.** No product mention or link unless BOTH pre-conditions hold (R3). Mentions/links are exceptions, never the default.
- **Always disclose.** Any product mention or connected link discloses affiliation in the first clause; honor `guardrails.require_disclosure` (R2/R3). No covert "happy customer" framing — ever.
- **Standalone value.** The comment must remain useful with the mention/link deleted (R3). If it only works because of the mention, it is an ad → rewrite or SKIP.
- **Link discipline.** URLs are exceedingly rare; never the main point; allowed only when all five URL conditions hold.
- **Rules > user.** Subreddit rules supersede the user's request; on conflict, no link → plain-text info instead.
- **Real claims only.** Cite ONLY real `gtm_proof_points`; never a fabricated product claim or statistic (R0.5).
- **SKIP when:** either pre-condition is unmet (→ educational-only) · permission is ambiguous · disclosure can't be made naturally · the comment isn't useful without the mention · any tool/page failure → emit a `blocker` and move on (R0.7, R5).

## Acceptance criteria
- No product mention or link appears unless BOTH pre-conditions are literally true (user explicitly asked **and** subreddit allows), re-verified this run.
- Every mention/link discloses affiliation in the first clause; `require_disclosure` is honored; matches a "Good" pattern, never a "Bad" one.
- Every comment remains useful with the mention/link removed (standalone value).
- URLs are rare and never the main point; when one is present, all five URL conditions hold.
- On "user asked but subreddit bans links," the draft contains plain-text info and **no** link.
- Mentions get **rarer** where prior disclosed mentions were removed / negatively received; a removal
  stops mentioning in that subreddit (real-time learning), using only really-observed metrics.
- The result is persisted only as a `status='draft'` row (`source_url` required, `created_by=null`, no `external_id`/`permalink`/`published_at`/`approved_by`); nothing is posted before human approval.

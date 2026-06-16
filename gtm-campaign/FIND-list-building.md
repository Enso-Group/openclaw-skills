# FIND — List-Building (depth)

Depth file for the **FIND** stage referenced by `gtm-campaign/SKILL.md`. This is the **Clawdi
hosted-runtime** methodology (what the agent does each fire), NOT app/Lovable code. It faithfully
encodes the *"Finding People from a Use Case"* build-list method and reconciles it with the live
contract (real tables/tools only — no invented columns). Read this when FIND needs detail.

**Provenance legend** (each rule is tagged with where it came from):
- `[Build list]` — `Downloads/2.List building/1.Build list.docx` + `1.Build list.pdf` (identical; the
  build-list methodology — the primary source for this file).
- `[Lead Gen PRD]` — `Lead_Generation_-_Component.pdf` (the APP's system-of-record behaviour; used
  only to fix the FIND→leads boundary).
- `[Contract]` — `gtm-campaign/SKILL.md` + `CLAWDI-CRON-PROMPT.md` (the live tables/tools/secrets).

> **One-line summary `[Build list]`:** *Scrape big, filter hard, keep only high-ranking profiles.*
> Scrape ~4,000 → hard-filter to ~2,400 → score → keep only ≥ `min_score` → ~1,200 high-ranking
> people. The agent stages those people; the **APP** auto-imports them into companies + leads.

---

## 0. Gates — run these BEFORE any hunting `[Contract]`

1. **STOP check (always first; re-check between use-cases).** `GET
   gtm_pipeline_settings?select=autonomous,paused,auto_import,min_score,stages,daily_caps,target_geography`.
   Empty / `paused=true` / `autonomous=false` → STOP cleanly. If `paused` flips true mid-run, stop
   immediately. Keep `min_score` (**default 16**, matches `[Build list]`'s "keep 16+") and `stages`.
2. **`stages.find` must be true** — else skip FIND, run the other stages.
3. **Anchor a mission per use-case (AUTO-CREATE if missing).** `GET openclaw_missions?select=id,kind,use_case_id,status&order=created_at.desc`.
   `mission_id` is **REQUIRED for `openclaw_results_staging`**. A use-case with **no mission → CREATE one yourself**
   (you own the work; no human mission needed): `POST /rest/v1/openclaw_missions
   {workspace_id, kind:"build_list", status:"dispatched", use_case_id, title:"FIND — <use-case>", brief:{"auto":true}}`
   with `Prefer: return=representation` to read back its `id`. Reuse it on later fires — one build_list mission per
   use-case, never create a second (idempotent). Then proceed with FIND. **Never skip FIND for "no mission."**
4. **Read the compass (once):** `gtm_use_cases` (active queues = one per ICP), `gtm_personas`
   (titles, stage→size, pain), `gtm_keywords`, `gtm_geo_questions` (the 11pm questions),
   `gtm_brands`, `target_geography`, `openclaw_playbooks`.
5. **R0 anti-hallucination (absolute) `[Contract]`:** every staged person carries a **real
   `source_url` you actually opened**; counts match reality (say 12 → write exactly 12 rows); never
   re-stage a `linkedin_url`; unknown → `""`. On ANY failure → POST a blocker event for FIND and move
   on; never fabricate to keep going.

Connection for every call: `${PLATFORM_URL}/rest/v1/...` with headers `apikey` /
`Authorization: Bearer ${PLATFORM_ANON_KEY}`, `x-agent-token: ${PLATFORM_AGENT_TOKEN}`,
`Content-Type: application/json`. **Every POST/PATCH also sends `Prefer: return=minimal`** (the
append tables have no anon SELECT; a 42501 *with* this header = a real permission blocker).

---

## 1. The funnel `[Build list]`

| Stage | Action | Survival | Volume (source model) |
|---|---|---|---|
| Search (FREE) | Apollo `api_search` by **title + size + location** → **obfuscated** candidates (`id`+title+org) | 100% | 4,000 |
| Filter Pass 1 | Remove hard disqualifiers using the free search fields | ~60% survive | 2,400 |
| **Enrich (1 credit/person)** | **Apollo `bulk_match` by `id` → reveals real `name` + `linkedin_url` + `email`** | — | credit-budgeted |
| Scoring pass | Score every **enriched** profile | — | 2,400 |
| Keep ≥ `min_score` | Discard everything below the threshold | ~50% survive | 1,200 |
| Final | High-ranking people, ready to use | — | 1,200 |

Source planning model `[Build list]`: **1,200 high-ranking profiles/month**, across **5 use-cases**
(~**240 per use-case/month**), historically split across 5 people (~240/person). *Scrape 4,000 to
get 1,200.*

> **Reconciliation `[Contract]`:** the live runtime is **incremental** — each fire advances the next
> due FIND work per active use-case rather than running a single monthly batch. The 4,000→1,200
> shape, the 240/use-case target, and the threshold are the **goal**; §7 maps the source's
> cold-start + daily-cron model onto incremental top-ups. The per-person "Assigned To" split is a
> human-team concept with **no agent field** — the app/queue owns distribution; the agent's only
> hand-off is the staging row (§6).

---

## 2. Step 1 — Extract the targeting parameters (per use-case) `[Build list]`

Pull these four from the use-case ICP block (`gtm_use_cases` / `gtm_personas`) before querying:

| Pull | Where it comes from | How it's used |
|---|---|---|
| **Job titles** | ICP Title field (`gtm_personas.titles`) | the query's title filter (REQUIRED) |
| **Company-size range** | ICP **Stage** → convert via the table below | size filter (Apify) / Filter Pass 1 + scoring (Apollo) |
| **Location** | brand target geography (`target_geography`) | location filter (OPTIONAL — never a stopper) |
| **Pain keywords** | ICP pain paragraph — key nouns & verbs (`gtm_keywords`) | the scoring pass (§5) |

**Funding stage → company size `[Build list]`:**

| Funding Stage | Company Size Range |
|---|---|
| Pre-Seed | 1–10 |
| Seed | 1–50 |
| Series A | 10–200 |
| Series B | 50–500 |
| Series C | 200–2,000 |

**Headcount caveat `[Build list]`:** LinkedIn headcount is an estimate and is often wrong. *LinkedIn
is used to find the person; Crunchbase (or equivalent) is used to confirm the company.* When ICP
stage is a critical filter, confirm actual stage with a funding source before the person enters the
list.

**Funding-confirmation tools `[Build list]`** (use the page you actually open as the `source_url`):

| Tool | How to use it | Apify actor |
|---|---|---|
| Crunchbase | company page → funding rounds, stage, headcount | `epctex/crunchbase-scraper` |
| Wellfound (AngelList) | company profile shows funding stage + team size | `curious_coder/angel-list-scraper` |
| LinkedIn company page | "About" → employee count, sometimes stage | already in the scrape via company URL |
| PitchBook / Dealroom | most accurate funding data | manual lookup (high-priority only) |

**When to confirm funding (NOT every profile — that does not scale) `[Build list]`:** (1) when a
company's LinkedIn size looks wrong for the ICP stage (e.g. a "Series A" showing 500 employees);
(2) as a **scoring signal** — a recent round matching the ICP stage scores **+3** (§5).

---

## 3. Step 2 — Query for REAL people (Apollo and/or Apify) `[Contract]` + `[Build list]`

**Apollo FIND is a TWO-STEP pipeline: SEARCH (free, obfuscated) → ENRICH (1 credit/person, reveals `name` +
`linkedin_url`).** The search endpoint deliberately returns LOCKED records — `name:null`,
`last_name_obfuscated:"Po***r"`, `linkedin_url:null`, only `id` + `first_name` + `title` + `organization` +
`has_*` flags. You CANNOT stage those (R0: no real name, no source_url) — that is exactly why a search-only run
stages 0. To get a stageable lead you MUST enrich the filtered candidates by their `id` (§3a step 2). Use either
Apollo (search→enrich) or Apify; if one is unavailable, run the other — **never stop for "no geography."**

### 3a. Apollo (`APOLLO_API_KEY`) — the default, runs now `[Contract]`
**STEP 1 — SEARCH (free, returns obfuscated candidates):**
- **Endpoint — use EXACTLY this** (the old `mixed_people/search` is DEPRECATED/403 on lower plans → HTTP 422/403):
  `POST https://api.apollo.io/api/v1/mixed_people/api_search`, header `X-Api-Key: ${APOLLO_API_KEY}`
  (the key MUST be in this header, never in the body) + `Content-Type: application/json`. Body:
  `{ "person_titles":[...], "person_locations":[...](optional), "page":1, "per_page":25 }`.
  Response: `total_entries` + `people[]`, where each person is **LOCKED**: `id`, `first_name`,
  `last_name_obfuscated` ("Po***r"), `title`, `organization{name,has_*}`, `has_email` — **`name` and
  `linkedin_url` are `null` by design.** This step is FREE and is ONLY a candidate list — never stage from it.
- `person_titles` — **REQUIRED** (all title variants from the ICP Title field).
- `person_locations` — **OPTIONAL** (use `target_geography` / persona geography if present, else
  **OMIT** — never stop for missing geography).

**STEP 1.5 — FILTER (free, before spending credits):** apply Filter Pass 1 (§4) on what search returned
(`title`, `organization.name`, `has_email`) so you only enrich people worth a credit. Drop obvious mismatches now.

**STEP 2 — ENRICH (1 Apollo credit per person; reveals the real `name` + `linkedin_url` + `email`):**
- `POST https://api.apollo.io/api/v1/people/bulk_match`, header `X-Api-Key: ${APOLLO_API_KEY}` +
  `Content-Type: application/json`. Body: `{ "details": [ {"id":"<search id>"}, ... up to 10 per call ] }`.
  Batch the filtered candidates in groups of ≤10. Response `matches[]` carries the **real** `name`,
  `first_name`, `last_name`, `linkedin_url`, `title`, `email`, `organization{name,website_url,...}`.
- **Budget the credits.** Enrich only Filter-Pass-1 survivors, and cap per fire at the use-case top-up target
  (§7) — do NOT blindly enrich all 25×6. Prefer `has_email:true` candidates first (higher match rate).
- **Now stage from the ENRICHED record** (§6): the enriched `linkedin_url` is the `source_url`. If enrichment
  returns a real `name` but `linkedin_url` is still null, you MAY stage using `name` + `organization` (the
  autoimport dedups on `name|domain`) — but only if you also have a real company domain; otherwise SKIP (R0:
  no person source_url). If enrichment returns a still-locked/obfuscated `name` → your Apollo plan is out of
  credits or lacks reveal: POST a FIND blocker ("Apollo enrich returned locked data — check plan credits") and
  fall back to Apify (§3b).
- Company-size (stage→size) and pain-keyword targeting apply as **Filter Pass 1 + scoring** (§4–§5).

### 3b. Apify LinkedIn scrape (`APIFY_TOKEN`) — needs a LinkedIn cookie + actor `[Build list]`+`[Contract]`
- **Missing the cookie or actor → POST a FIND blocker and continue** with Apollo.
- Actor: `harvestapi/linkedin-profile-search`. **One saved configuration per use-case.**
  `profileScraperMode = Full`. **One CSV per use-case — never combined.**

| Apify field | What to enter |
|---|---|
| `currentJobTitles` | all title variants from the ICP Title field |
| `searchQuery` | ICP titles + stage keywords + industry, as a keyword string |
| `locations` | target geography |
| `industryIds` | the LinkedIn industry ID for this client's sector |
| `profileScraperMode` | `Full` |
| `maxItems` | 800 per use-case (cold start) — see §7 for top-up volumes |

Source cost model `[Build list]`: 800/use-case ≈ **$11.20**; all 5 = 4,000 ≈ **$56/month**.

> **Why `Full` mode matters `[Build list]`:** the Full scrape already returns everything WARMUP needs
> per person (recent posts, a post they commented on, top skill, company URL, title/stage) — **no
> additional data collection** downstream. See `WARMUP-and-first-message.md` §A.

---

## 4. Step 3 — Filter Pass 1: hard disqualifiers (binary) `[Build list]`

Remove any person that fails **any** check (fail one → drop the row). Expect ~2,400 to survive.

| Check | Remove if |
|---|---|
| Title | does not **exactly** match any ICP title |
| Location | outside the target geography |
| Company size | outside the ICP size range (use the stage→size table) |
| Blank profile | summary **and** experience are both empty |
| Non-commercial | non-profit, government, educational institution, or personal project |
| Duplicate | LinkedIn URL already staged before (idempotency — R0; never re-stage a `linkedin_url`) |

---

## 5. Step 4 — Score every survivor `[Build list]` (reconciled to `[Contract]`'s `score`)

Every profile **starts at 10** — title match (**+5**) and company-size match (**+5**) are guaranteed
by the query. Add signal points on top; subtract negatives; **keep only `score ≥ min_score`**
(default **16**, i.e. the source's "16–20 enter the list, below 16 discard").

**Positive signals (add):**

| Signal | Points | Where to find it |
|---|---|---|
| Pain keywords in the LinkedIn summary | **+4** | summary field in the scrape |
| Company recently raised a round matching the ICP stage | **+3** | Crunchbase / Wellfound / LinkedIn company page |
| Company actively hiring for a role that signals the pain | **+3** | LinkedIn Jobs / Wellfound |
| Person posted publicly about the pain topic in the last 90 days | **+4** | LinkedIn activity tab / Twitter-X |
| Person is a member of a community relevant to the pain space | **+2** | cross-reference community member lists |

**Negative signals (subtract):**

| Signal | Points |
|---|---|
| Profile completely blank | **−3** |
| Title matches but is clearly a **non-buying** role | **−5** |
| Company is in a sector the product does **not** serve | **−5** |

Record the components you applied in `signals` and one observed fact in `evidence_of_fit` (§6).

---

## 6. Step 5 — Stage the keepers + record yield (the REAL outputs) `[Contract]`

The source's "sort by score, randomise within a score, assign 240/person, write the 8-column list"
maps to **two real writes** (the app/queue handles ordering + distribution; there is **no agent
field** for "Assigned To"). Map the source's columns to the payload fields that exist (Full Name,
Job Title, Company, Location, LinkedIn URL, Score); "Use Case" = the mission / use-case scope.

**(a) Stage each kept person** (only `score ≥ min_score`; status defaults to `staged`; never set
`imported_at`). Send `Prefer: return=minimal`. **You CAN now GET `openclaw_results_staging` (token-scoped SELECT)**
to (i) confirm your finds landed and (ii) build the already-staged `linkedin_url` set for idempotency. A 42501 on
that GET means the read-back grant migration isn't applied yet — fall back to your own run tally and continue.

```json
POST /rest/v1/openclaw_results_staging
{
  "workspace_id": "<ws>",
  "mission_id": "<REQUIRED — the use-case mission from §0.3>",
  "row_kind": "person",
  "source_url": "<the Apollo-returned linkedin_url for this person (a real URL) — or a page you opened>",
  "payload": {
    "full_name": "...",
    "job_title": "...",
    "company_name": "...",
    "company_domain": "...",
    "linkedin_url": "...",
    "email": "",
    "score": 17,
    "signals": { "title_match": 5, "size_match": 5, "pain_in_summary": 4, "hiring_for_pain": 3 },
    "evidence_of_fit": "one observed fact, e.g. 'posted about SDR ramp 11 days ago'",
    "location": "..."
  }
}
```

> **Boundary `[Lead Gen PRD]` + `[Contract]`:** the **APP** auto-imports every staged row with
> `score ≥ min_score` into `lead_generation_companies` + `lead_generation_leads`
> (`pipeline-autoimport-tick`). **The agent never writes lead/company tables** — staging is the only
> hand-off. (PRD lifecycles confirm the receiving side: companies `new → leads_found → exhausted →
> excluded`; leads `ready → engaging → …`.)

**(b) Record yield per source — GET-then-write** (your `gtm_sources` UPDATE grant is COLUMN-scoped and the unique
key is an EXPRESSION index, so a `Prefer: resolution=merge-duplicates` upsert fails with `42501 permission denied`).
Use the read-then-write pattern (you now hold token-scoped SELECT):
1. `GET /rest/v1/gtm_sources?workspace_id=eq.<ws>&platform=eq.<p>&query=eq.<q>&select=id`.
2. **Found →** `PATCH /rest/v1/gtm_sources?id=eq.<id>` with ONLY `{prospects_found, qualified, last_used_at}`
   (exactly the column-scoped grant) + `Prefer: return=minimal`.
3. **Not found →** `POST /rest/v1/gtm_sources` (plain INSERT) + `Prefer: return=minimal`.
This non-critical yield log lets the next fire learn where to spend hunting time (§8) — **never let it block FIND**
(on any gtm_sources error, log nothing and continue staging):

```json
// POST body (insert); the PATCH sends only prospects_found / qualified / last_used_at
{
  "workspace_id": "<ws>",
  "platform": "apollo",            // or "apify" | "linkedin"
  "query": "<the exact title/keyword/location query you ran>",
  "prospects_found": 120,          // people returned by search
  "qualified": 41,                 // how many enriched + scored ≥ min_score
  "last_used_at": "<iso now>"
}
```

**(c) Decision log** (so the operator watches live): `POST openclaw_mission_events { workspace_id,
mission_id?, event_type:"progress", message:"find: <real counts>", payload:{stage:"find"} }`.

---

## 7. Cadence — cold start vs incremental top-up `[Build list]` (reconciled to `[Contract]`)

**Cold start (first run for a use-case)** — seed the queue once: scrape 800 → Filter Pass 1 →
score → keep ≥ `min_score` → stage the keepers. Source target ≈ 240 kept per use-case. After the
queue exists, the incremental fires maintain it.

**Incremental top-up (every fire, the live equivalent of `[Build list]`'s `0 6 * * 1-5` Mon–Fri
6am cron):**
1. Gauge queue depth for each use-case (how many fresh, not-yet-contacted people are available —
   `GET lead_generation_leads?select=id,pipeline_stage,...` where your token can read it; if not
   readable, fall back to your own run-to-run tally + `gtm_sources`).
2. **Below the buffer threshold → trigger a top-up scrape for that use-case only.**
3. Filter Pass 1 → score → keep ≥ `min_score` → stage, **removing anyone already contacted/staged**.

**Buffer threshold (cover ≥ 5 working days) `[Build list]`:**

| Accounts / use-case | Requests / account / day | Consumed / day | Trigger top-up when queue drops below |
|---|---|---|---|
| 1 | 20 | 20 | **100** (5-day buffer) |
| 2 | 20 | 40 | **200** (5-day buffer) |

**Top-up scrape volume `[Build list]`:** below buffer → top up ~2 weeks → `maxItems = 300`;
critically low (< 2 days of supply) → emergency → `maxItems = 500`.

**When to scale up `[Build list]`:** consistently triggering top-ups **> twice a week** → increase
default `maxItems` per top-up. **Fewer than the monthly target reach ≥ `min_score`** → **expand the
title list** (add one or two adjacent titles to the ICP). *(No weekends — honour the STOP/weekend
laws.)*

---

## 8. Real-time learning (tune every fire) `[Contract]` + `[Build list]`

Before querying, read what prior fires learned and adapt; after, leave signal for the next fire:

- **Source yield → where to spend time.** Read prior `gtm_sources` (per `platform` + `query`:
  `prospects_found` vs `qualified` = the qualify rate). Lean into high-qualify queries; **drop a
  query/platform that keeps producing low-`qualified` rows** ("a source that stops producing → find
  a new hunting ground"). Where `gtm_sources` is not readable by your token, keep your own running
  tally and re-post counts so the trend survives across fires.
- **Threshold miss → widen the net the right way `[Build list]`.** If a use-case keeps landing
  **fewer keepers than target**, expand titles / add pain-keyword phrasings / loosen geography
  (never tighten below the contract) — *more of the right people, not lower standards.*
- **Blockers → don't repeat them.** Read recent `openclaw_mission_events` blockers; if Apify cookie/
  actor was missing last fire, go straight to Apollo this fire instead of re-failing.
- **Scoring drift.** If a signal (e.g. "+4 posted in 90d") rarely appears for a use-case, prefer the
  structural signals (hiring/funding) for that ICP so the score still clears `min_score`.
- **Idempotency feeds learning.** Never re-stage a `linkedin_url`; the dedupe set is also your "queue
  depth" estimate for §7.

---

## 9. SKIP / blocker quick-reference

| Situation | Action |
|---|---|
| `paused=true` / `autonomous=false` / no settings | STOP cleanly |
| `stages.find` false | skip FIND, run other stages |
| Use-case has no mission | CREATE a build_list mission (§0.3) then run FIND — never skip for this |
| Apify cookie/actor missing | blocker; continue with Apollo |
| A person has no real name OR no `linkedin_url` | skip just that person (no verifiable source → no row); keep the rest |
| `score < min_score` | discard (do not stage) |
| 42501 on a write **with** `Prefer: return=minimal` | real permission problem → blocker |
| Any tool/login/page failure | blocker event for FIND, move to next use-case/stage |

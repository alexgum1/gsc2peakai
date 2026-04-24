---
name: gsc-peec-ai-visibility
description: >
  Connects Google Search Console with Peec.ai to build and maintain an AI-search
  visibility tracking system. Use this skill whenever the user wants to:
  - Find which GSC queries should be tracked as Peec.ai prompts
  - Set up W-question or conversational prompts from Search Console data in Peec.ai
  - Analyse current Peec.ai prompts and replace low-performers with better GSC candidates
  - Build a live dashboard showing GSC × Peec.ai KI-visibility with SISTRIX volume
  - Run the full GSC → Peec.ai onboarding workflow for AI search visibility monitoring
  - Regularly refresh the Peec.ai prompt set based on new GSC data
  Trigger on phrases like: "peec + search console", "GSC prompts anlegen", "KI-Sichtbarkeit tracken",
  "welche prompts peec", "ai visibility gsc", "low performer peec archivieren", "prompt set auffrischen"
---

# GSC × Peec.ai – AI Visibility Tracker Skill

This skill orchestrates a three-phase workflow that bridges traditional SEO data (Google Search Console) with AI-search visibility tracking (Peec.ai). The core insight: search queries people type into Google are the same questions they ask AI engines — tracking them in Peec.ai reveals how visible your brand is in AI-generated answers.

> **Required connectors:** Peec.ai MCP (`https://api.peec.ai/mcp`), Google Search Console MCP, SISTRIX MCP  
> **Reference:** See `references/filter-criteria.md` for exact W-question filter logic  
> **Reference:** See `references/low-performer-criteria.md` for archiving rules

---

## Phase 1 — Discover & Create Prompts

**Goal:** Pull high-potential queries from GSC and create them as tracked prompts in Peec.ai.

### Step 1.1 — Get project context

```
list_projects()                    → get project_id
list_brands(project_id)            → find own brand (is_own=true) → brand_id
list_topics(project_id)            → check if "GSC AI-Anfragen" topic exists
list_prompts(project_id)           → count active prompts (hard limit: 50)
```

If the "GSC AI-Anfragen" topic doesn't exist, create it:
```
create_topic(project_id, name="GSC AI-Anfragen", country_code="DE")
```

### Step 1.2 — Fetch GSC data

Call `gsc_search_analytics` for the last 90 days:
- `siteUrl`: ask user or infer from GSC site list
- `dimensions: ["query"]`
- `rowLimit: 5000`
- `startDate`: 90 days ago, `endDate`: today

### Step 1.3 — Filter candidates

Apply the GSC filter (details in `references/filter-criteria.md`):

```python
W_WORDS = {
  'was','wie','welche','welcher','welches','welchen','welchem',
  'warum','wo','wann','wer','wen','wem','wessen','wohin','woher',
  'womit','wozu','wofür','wobei','wodurch','weshalb','weswegen',
  'wieso','inwieweit','worin','worauf','worüber','woran','wovor','wonach'
}

def qualifies(query, impressions):
    words = query.lower().split()
    if len(words) < 5 or impressions < 100:
        return False
    return any(w in W_WORDS for w in words) or len(words) >= 10
```

Sort candidates by impressions descending. Deduplicate against existing Peec prompts.

### Step 1.4 — Handle the 50-prompt limit

If `active_prompts >= 50`, run Phase 3 (low-performer archiving) **before** creating new prompts. After archiving, proceed with creation.

If `50 - active_prompts < len(candidates)`, trim candidates to fit the available slots.

### Step 1.5 — Create prompts in Peec.ai

Create prompts in batches of 5 to avoid connection timeouts:

```
for batch in chunks(candidates, 5):
    for query in batch:
        create_prompt(project_id, text=query, country_code="DE", topic_id=gsc_topic_id)
    sleep(1s)
```

Report to the user: how many candidates found, how many created, how many skipped (duplicates or limit).

---

## Phase 2 — Build the Live Dashboard

**Goal:** Create a persistent artifact showing GSC × SISTRIX × Peec.ai data side by side.

### Data columns

| Source | Columns | Colour |
|--------|---------|--------|
| 🔵 Google Search Console | Impressionen (30d), Klicks (30d) | Blue (#1d4ed8) |
| 🟠 SISTRIX | Suchvolumen/Mo | Amber (#b45309) |
| 🟣 Peec.ai | KI-Sichtbarkeit %, Erwähnung/Zitierung, Prompt-Vol., Gemini, AI Mode, AI Overview | Violet (#6d28d9) |

**Important:** Remove the average position column. Visual colour separation between data sources is required.

#### Erwähnung vs. Zitierung column

This column distinguishes how the brand appears in AI responses:

| Value | Meaning |
|-------|---------|
| **Beides** | Brand is mentioned in response text AND its URL is cited as a source |
| **Erwähnung** | Brand name appears in response text only (parametric or inline mention, no URL) |
| **Zitierung** | Brand URL is cited as a source but brand name is not mentioned in text |
| **–** | No data or brand not visible for this prompt |

Fetch citation data (parallel with the brand reports in Phase 1):
```
get_domain_report(project_id,
  start_date, end_date,
  dimensions: ["prompt_id"],
  filters: [
    {field: "domain", operator: "in", values: [<own_brand_domains>]},
    {field: "prompt_id", operator: "in", values: [<prompt_ids>]}
  ],
  limit: 100
)
```

Cross-reference with `mention_count` from `get_brand_report`:
- `mention_count > 0` AND `citation_count > 0` → **Beides**
- `mention_count > 0` AND `citation_count = 0` → **Erwähnung**
- `mention_count = 0` AND `citation_count > 0` → **Zitierung**
- Otherwise → **–**

### Fetching strategy

Load data in three phases to avoid overloading the proxy:

**Phase 1** (parallel, 2 calls): Peec.ai brand reports
- `get_brand_report(dimensions=["prompt_id"], filters=[topic, brand, prompts])` → visibility + visibility_total per prompt
- `get_brand_report(dimensions=["model_id","prompt_id"], filters=[topic, brand, 3 active models, prompts])` → per-model visibility

Active models to track: `gemini-scraper`, `google-ai-mode-scraper`, `google-ai-overview-scraper`

**Important — Modell-Report Vollständigkeitsprüfung:**
Wrap the model report call with `.catch(()=>null)`. If it returns null (exception), retry once after 1.5s. If it still fails after retry, set `mFailed=true` and show all model cells as `⚠` with a warning banner — never silently show `–`.

Peec.ai's `get_brand_report` only returns rows where the brand actually appeared (visibility > 0). A missing row for a prompt/model combination does NOT mean "no data" — it means the brand was not visible (0%). Apply this mapping when building model values:
- `mFailed=true` → `'err'` (API-Fehler, zeige ⚠)
- `isNew=true` + no row → `null` (neu, noch keine Daten, zeige `–`)
- `isNew=false` + no row → `0` (Modell hat ausgeführt, Marke nicht sichtbar, zeige `✗`)
- Row vorhanden → Wert × 100 für % (zeige `✓ X%`)

**Phase 2** (two-stage): GSC calls — never trust 0 rows from a filtered call

The GSC MCP's `query:` filter parameter may do substring matching, full-text matching, or no filtering at all depending on the MCP version. **0 rows from a per-prompt call ≠ 0 impressions.** Use the following two-stage strategy:

**Stage 1** — Per-prompt filtered calls (up to MAX_PASSES=3, RETRY_DELAYS=[0, 2500, 5000]ms):
```
gsc_search_analytics(siteUrl, startDate, endDate, query=prompt_text, dimensions=["query"], rowLimit=10)
```
Match logic (in order): exact string → case-insensitive → single-row fallback.
- Match found → `{imp, clk}` ✓ done
- Exception / unwrap fails → `null` → retry on next pass
- 0 rows returned OR rows present but no match → `'needsBatch'` (do NOT assume 0)

After all passes: any remaining `null` also becomes `'needsBatch'`.

**Stage 2** — Batch fallback for all `'needsBatch'` prompts:
```
gsc_search_analytics(siteUrl, startDate, endDate, dimensions=["query"], rowLimit=500)
// NO query filter — returns top 500 queries, client-side matching via Map
```
Retry this batch call once (2s delay) if it fails.
- Match in batch → `{imp, clk}`
- No match in batch → `{imp:0, clk:0}` — now trustworthy (not in top 500 = genuinely no impressions)
- Batch call itself fails → `{gscErr:true}` — show ⚠ warning banner, never show `–` silently

**After loading:** Count gscErrCount. If > 0, show a visible blue warning banner naming the affected queries. Do **not** silently show `–` or `0` for API failures.

**Phase 3** (batched, 5 at a time with 400ms delay): SISTRIX
- `keyword_seo_traffic(kw=prompt_text, country="de")`
- Hard-code last-known values in the artifact as fallback; live calls refresh them
- **Important:** Track errors separately from "no data". A SISTRIX call can fail (API error / timeout) OR legitimately return null (keyword has no indexed traffic data). Treat these differently:
  - `catch` → `err: true` — API-Fehler, show ⚠ in cell
  - resolved but `traffic == null` → no data for this keyword, show `–` in cell
- After the first pass, collect all indices where `err: true` AND no fallback value exists. Run a single retry pass on those (wait 1.5s, then re-fetch in batches of 5, 600ms apart).
- After loading, count `errCount`. If `errCount > 0`, show a visible warning banner naming the affected queries. Do **not** silently show `–` for API failures — the user cannot tell the difference from genuine no-volume keywords.

### Unwrap helper

Peec.ai MCP responses can arrive either as direct JSON or wrapped in `{content: [{type:"text", text:"...JSON..."}]}`. Always unwrap before parsing:

```javascript
function unwrap(raw) {
  if (raw == null) return null;
  if (typeof raw === 'string') { try { return JSON.parse(raw); } catch { return null; } }
  if (raw.content && Array.isArray(raw.content)) {
    const txt = raw.content.filter(c => c?.type === 'text').map(c => c.text).join('');
    try { return JSON.parse(txt); } catch { return null; }
  }
  return raw;
}
```

### Peec columnar response parser

```javascript
function c2o(raw) {
  const d = unwrap(raw);
  if (!d || !d.columns || !d.rows) return [];
  return d.rows.map(r => Object.fromEntries(d.columns.map((c, i) => [c, r[i]])));
}
```

### Visibility badge thresholds

| Value | Badge colour | Class |
|-------|-------------|-------|
| ≥ 30% | Green | `b-high` |
| 10–29% | Yellow | `b-mid` |
| 1–9%  | Red | `b-low` |
| 0%    | Grey | `b-zero` |
| null  | Blue "Neu" | `b-new` |

### Debug bar

After loading, show a yellow debug bar: `🟣 Peec: N/20  🔵 GSC: N/20  🟠 SISTRIX: N/20`

Use `callMcpTool` via `window.cowork.callMcpTool(toolName, args)` for live data.

---

## Phase 3 — Low-Performer Archiving & Refresh

**Goal:** Regularly replace prompts that generate little AI visibility with fresh GSC candidates.

Run this phase:
- When the 50-prompt limit is reached (triggered automatically in Phase 1)
- When the user explicitly asks for a "prompt refresh" or "low-performer check"
- Recommended cadence: monthly

### Step 3.1 — Identify low-performers

Fetch `get_brand_report` for ALL prompts (not just the GSC topic), last 30 days:

```
filters: [brand_id=own_brand]
dimensions: [prompt_id]
```

A prompt is a **low-performer** if ALL of the following are true:
- `visibility < 20%` (visibility ratio < 0.20)
- `mention_count < 110`
- The prompt is **not** in the GSC AI-Anfragen topic (don't archive what you just created)

Also consider archiving prompts in the GSC topic if they have been running ≥ 14 days and still show `visibility_total < 10` (prompt has barely been run — something may be wrong).

### Step 3.2 — Present low-performers to user

Before archiving, show the user a summary table:

```
| Prompt text | Visibility | Mentions | Reason |
|-------------|-----------|---------|--------|
| ...         | 5%        | 42      | Low vis + low mentions |
```

Ask for confirmation before proceeding: "Ich möchte diese N Prompts archivieren. Soll ich fortfahren?"

### Step 3.3 — Archive low-performers

Delete in batches of 5 (parallel deletes can cause connection drops):

```
for batch in chunks(low_performers, 5):
    delete_prompt(project_id, prompt_id) for each in batch
    sleep(1s)
```

Note: Peec uses **soft-delete** — it is **not idempotent**. A second delete call returns "not found". If you get "not found" errors, the first batch likely succeeded — verify with list_prompts rather than retrying.

### Step 3.4 — Find replacement candidates

Run Phase 1 steps 1.2–1.3 with the current GSC data. Exclude:
- Queries already active in Peec.ai (dedup by text)
- Queries that were previously archived this session

Create the replacements following Phase 1 Step 1.5.

---

## Important Peec.ai API behaviours

- **Scale:** `visibility` and `share_of_voice` are 0–1 ratios (multiply × 100 for %). `sentiment` is already 0–100 (neutral = 50). `position` is rank among tracked brands (lower = better).
- **Response format:** Always columnar JSON `{columns: [...], rows: [[...]], rowCount: N}`. Use `c2o()` to parse.
- **ID prefixes:** `or_` project, `kw_` brand, `pr_` prompt, `tp_` topic, `tg_` tag, `ch_` chat.
- **Prompt text is immutable** after creation — delete + recreate to change text.
- **`update_prompt`** only changes `topic_id` and `tag_ids`, not text.
- **Parallel calls:** Batch writes in groups of 5 with a short delay. Parallel batches of 20+ cause connection drops.

---

## Communication guidelines

- After Phase 1: report `N prompts created in topic "GSC AI-Anfragen"` with a breakdown by filter reason (W-Frage / langer Query).
- After Phase 3: report `N low-performers archiviert, N neue Prompts angelegt`.
- If the GSC query returns 0 W-questions: expand to all 5+-word queries with ≥ 100 impressions and explain why.
- Always confirm with the user before bulk-deleting prompts.
- Daten-Stand (timestamp) is important — always show when the data was last fetched.

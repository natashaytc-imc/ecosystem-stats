# PRD — AppWorks Ecosystem Stats Dashboard

## Overview

A single-file static HTML dashboard (`dashboard.html`) that connects to Airtable via REST API and displays portfolio metrics for AppWorks' startup ecosystem. Deployed on GitHub Pages for internal team access.

**Live URL:** https://natashaytc-imc.github.io/ecosystem-stats/  
**Repo:** https://github.com/natashaytc-imc/ecosystem-stats

---

## Architecture

- **Single file:** All HTML, CSS, and JS are inline in `dashboard.html`. No build tools, no dependencies, no backend.
- **Data source:** Airtable REST API, authenticated at runtime by each user entering their own Personal Access Token (PAT).
- **AI chatbot:** Anthropic Claude API (claude-sonnet-4-20250514), called directly from the browser via `anthropic-dangerous-direct-browser-access` header. API key is embedded in the source code.
- **Hosting:** GitHub Pages (static), auto-deploys on `git push` to `main`.

---

## Airtable Configuration

| Item | Value |
|------|-------|
| Base ID | `appKg1PG4KurhvyIK` |
| Table 1 | `Companies` |
| Table 2 | `Ecosystem Stats` |

### Table: Companies (minimal fetch)

Fetched fields: `Company Name`, `HQ`

Used only for two purposes:
1. Building a `recordId → companyName` map (to resolve linked record IDs from Ecosystem Stats)
2. Extracting HQ market codes (not available in Ecosystem Stats)

### Table: Ecosystem Stats (primary data source)

Fetched fields: `Company Name`, `Period`, `Total Raised`, `Valuation`, `Revenue`, `Staff`, `Has Update`, `AI Label Reasoning`, `AI Label`, `AI-Label (from 26H1)`, `On (from Company Name)`, `Batch (from Company Name)`, `Company Verticals (from Company Name)`

- `Company Name` — Linked Record to Companies table (returns record IDs via REST API)
- `Period` — Single Select (`24H2`, `25H1`, `25H2`, `26H1`)
- `Total Raised`, `Valuation`, `Revenue`, `Staff` — Number (metrics)
- `Has Update` — Checkbox. When `true`, the record's numeric data is confirmed fresh for that period. When `false`, data may be carried over from an earlier period (stale).
- `AI Label Reasoning` — Formula. Explains why a company is classified as AI-native, AI-enabled, or neither.
- `AI Label` — Formula. Used for 26H1 AI classification (`AI-native`, `AI-enabled`, or empty).
- `AI-Label (from 26H1)` — Lookup. Used for 25H1 records to determine AI classification (references the 26H1 record's AI label).
- `On (from Company Name)` — Lookup. Pulls the `On` Single Select value from Companies. Used for active-company filtering (26H1 only).
- `Batch (from Company Name)` — Lookup. Pulls the `Batch` Multi Select values from Companies. Used for portfolio filtering (both periods).
- `Company Verticals (from Company Name)` — Lookup. Pulls the `Company Verticals` Multi Select values from Companies. Used for Web3 filtering (both periods).

**All company metadata, filtering criteria, and numeric metrics are sourced from this table.** Both 26H1 and 25H1 records are built into independent company lists for accurate period-specific counts.

---

## Data Pipeline

### 1. Pagination
Airtable returns max 100 records per request. The dashboard paginates using the `offset` field until all records are fetched. A 250ms delay between pages avoids rate limiting. Retries with exponential backoff on 401/403/429/500/502/503.

### 2. Field Format Handling
Lookup fields from Ecosystem Stats return arrays of strings (not Select objects). The parsing code handles both formats defensively:
- Lookup of Single Select: returns `["value"]` → extract first element
- Lookup of Multi Select: returns `["val1", "val2"]` → use as flat array
- Formula fields: return strings directly
- All filter functions use `typeof v === 'object' && v.name ? v.name : String(v)` to handle both object and string formats.

### 3. Build Company Objects from Stats
`buildCompaniesFromStats(statsRecords, idToName, hqMap, period)` creates company objects from Ecosystem Stats records for a given period. Each company object includes: resolved company name, On/Batch/Company Verticals from lookup fields, HQ from the Companies hqMap, and metrics directly from the same record.

- For **26H1** records: `AI Label` is read from the `AI Label` formula field.
- For **25H1** records: `AI Label` is read from the `AI-Label (from 26H1)` lookup field (references the corresponding 26H1 record's classification).

Both 26H1 and 25H1 company lists are built independently, enabling accurate per-period counts for Active Startups.

### 4. Filtering
After building company objects, filter client-side using `filterCompanies()`:
- **26H1:** `On === '1'` AND `Batch` has any of: `Portfolio`, `AW#1`–`AW#32` (case-insensitive, any leading-zero format)
- **25H1:** `Batch` filter only (no `On` filter — `skipOnFilter: true`)

### 5. Period Lookups
From Ecosystem Stats, build two maps using `buildPeriodLookup(statsRecords, idToName, period)`:
- **26H1 lookup (`gLatestLookup`):** `companyName → { raised, valuation, revenue, staff, hasUpdate, aiReasoning }` — used by chatbot for data freshness and AI reasoning context
- **25H1 lookup (`gH1Lookup`):** same structure — used for YoY change calculations

Since the REST API returns linked record IDs (not names) for the `Company Name` linked record field, a `recordId → companyName` map is built from the Companies table to resolve them.

### 6. Change Calculation
For each company: `change = 26H1 value − 25H1 value`

**Percentage in Change ranked lists:** Each company's change is divided by the **net total change** across all companies: `totalDenom = Σ(all companies' 26H1 values) − Σ(matched companies' 25H1 values)`. This denominator matches the YoY absolute change shown in the Snapshot cards. Companies with negative changes show negative percentages.

### 7. Active Startups Count
All counts are derived from independently built and filtered company lists per period:
- **26H1 General:** filtered 26H1 companies (On=1 + Batch)
- **26H1 AI:** 26H1 filtered companies where `AI Label` is `AI-native` or `AI-enabled`
- **26H1 Web3:** 26H1 filtered companies where `Company Verticals` includes `Web3`
- **25H1 General:** filtered 25H1 companies (Batch only, no On filter)
- **25H1 AI:** 25H1 filtered companies where `AI-Label (from 26H1)` is `AI-native` or `AI-enabled`
- **25H1 Web3:** 25H1 filtered companies where `Company Verticals (from Company Name)` includes `Web3`

Global state `gData25` stores the 25H1 filtered company lists for chatbot diff calculations.

---

## Dashboard Structure

Four tabbed pages:

### Tab 1: General
All filtered portfolio companies. Contains Snapshot + Top 10 Latest + Top 10 Change + Company Lookup.

### Tab 2: AI Companies
Filtered to `AI Label === 'AI-native'` OR `AI Label === 'AI-enabled'`. Same section layout as General.

### Tab 3: Web3
Filtered to `Company Verticals` includes `'Web3'`. Same section layout as General.

### Tab 4: Presentation
An internal presentation view with 7 slide-like sections designed for screen sharing. Uses larger titles and narrative text. Metric cards and ranked lists are **not recalculated** — they are copied directly from the already-rendered General, AI, and Web3 tabs via `innerHTML` to guarantee identical numbers. Presentation-specific metrics (e.g. excl. Kraken breakdowns, AI/Web3 shares) are pre-computed once in `loadDashboard()` and stored in a global `gPresData` object.

See **Presentation Tab** section below for slide details.

### Sections per Tab (General / AI / Web3)

| # | Section | Description |
|---|---------|-------------|
| 01 | Snapshot | 5 metric cards: **Active Startups**, Total Raised, Total Valuation, Total Revenue, Total Staff. All metric values sourced from Ecosystem Stats table (`26H1` period). The 4 numeric metric cards show a YoY badge (↑/↓ % + absolute change vs 25H1). Active Startups card shows a `26H1: X / 25H1: Y` subtitle line. 25H1 counts are independently computed from 25H1 Ecosystem Stats records: General = Batch filter only (no On filter); AI = 25H1 records where `AI-Label (from 26H1)` matches; Web3 = 25H1 records where `Company Verticals` includes Web3. |
| 02 | Top 10 by Latest Metrics | 4 ranked lists (Raised, Valuation, Revenue, Staff), sorted descending by latest value. Each row shows: rank number, batch badge (`AW#xx` or `Portfolio`), company name, individual percentage of total, formatted value. Footer shows the Top 10 combined share of all portfolio companies (e.g. `Top 10 / Total — 68.3%`). |
| 03 | Top 10 Absolute Change — 26H1 vs 25H1 | 4 ranked lists sorted by absolute change. Each row shows rank, batch badge, company name, individual % of **net total change**, +/- formatted change value (green positive, red negative). Same footer showing Top 10 share. The denominator for per-company percentages is the net total change (= Σ all companies' 26H1 values − Σ matched companies' 25H1 values), matching the snapshot YoY absolute change. Only companies that have both a latest value and a 25H1 value appear in the ranked rows. |
| 04 | Company Lookup | Text search input with autocomplete dropdown (scoped to that tab's companies, shows batch badge in dropdown). Selecting a company displays a comparison table: Metric / 25H1 / 26H1 (Latest) / Change (green/red). |

### Presentation Tab

7 slide-like sections with large titles (26px DM Sans bold), accent-colored slide numbers (DM Mono), and narrative text auto-generated from computed data.

**Data strategy:** Slides 1, 2, 4, 5 copy rendered HTML directly from other tabs (`innerHTML`). Slide 3 uses `renderMetricCards()` with Kraken-excluded data. Narrative-only metrics are pre-computed once in `loadDashboard()` and stored in `gPresData`.

| Slide | Title | Content |
|-------|-------|---------|
| 01 | Ecosystem at a Glance: Positive Growth Across the Board | Copied from General tab snapshot cards. Narrative: total startups 26H1 vs 25H1. |
| 02 | Kraken Driving Headline Numbers | Copied from General tab Top 10 Absolute Change ranked lists. Narrative highlights Kraken's share of total Raised growth (% of net total change). |
| 03 | The Rest of the Ecosystem: Steady Growth | Snapshot cards computed for all companies **excluding Kraken**. Narrative shows Revenue YoY and Staff YoY excl. Kraken. |
| 04 | {X}% of Ecosystem Are Now AI | Copied from AI tab snapshot cards. Title dynamically set to actual AI %. Subtitle: "Includes AI-native + AI-enabled companies". Narrative shows AI count, +/- vs 25H1, AI share of Raised and Valuation. **Collapsible section** below with AI classification definitions (AI-Native, AI-Enabled, Not Yet AI) and data source note. |
| 05 | Web3: Kraken's Breakout Masks a Quieter Ecosystem | Copied from Web3 tab snapshot cards. Narrative shows Web3 count, +/- vs 25H1, Raised YoY with/without Kraken. **Additional section below:** "Web3 excl. Kraken" snapshot cards showing metrics with Kraken removed. |
| 06 | Looking Ahead to 26H2 | Three forward-looking bullet points (AI count trajectory, Web3 next growth driver, Revenue sustainability). |
| 07 | Data Collection Methodology | Data sources, update coverage stats (confirmed vs carried forward), period definition. |

**AI Classification Definitions (Slide 04 collapsible):**
- **AI-Native:** AI is the core product; the startup could not exist without AI. e.g. trains its own models, product IS the model (LLM, CV, agent), moat depends on proprietary data or pipelines.
- **AI-Enabled:** AI is used in the product, but is not the core value proposition. e.g. uses AI to automate certain workflows, product could still function without AI, integrates off-the-shelf APIs like GPT or Claude.
- **Not Yet AI:** No meaningful AI/ML used in the product.
- Data source: company URL + web search

### AI Chatbot

A floating chatbot powered by Anthropic Claude that can answer natural-language questions about the loaded portfolio data.

**Architecture:**
- Uses already-loaded in-memory data (`gData`, `gData25`, `gH1Lookup`, `gLatestLookup`, `gBatchMap`) — no additional Airtable API calls.
- Calls the Anthropic Messages API directly from the browser using `anthropic-dangerous-direct-browser-access` header.
- API key is embedded as a constant (`ANTHROPIC_KEY`) in the source code. Repo should be private.
- Model: `claude-sonnet-4-20250514`. Responses are streamed via SSE.

**Context passed to Claude (system prompt):**
- Role instructions (AppWorks data analyst assistant)
- Response style: prioritize data and facts first. When answering about changes (e.g. count decreases), list specific companies added/removed between periods before offering analysis. Always include batch tag (e.g. `AW#09`, `Portfolio`) when mentioning any company name.
- Data freshness rules: each company has a `Has Update` flag — `hasUpdate=true` means 26H1 data is confirmed fresh, `false` means it may be stale. Claude is instructed to flag stale data and ask users which period they mean if unspecified.
- Aggregate stats for General / AI / Web3 (total raised, valuation, revenue, staff, company count, update rate)
- Period diff for General / AI / Web3: lists companies added in 26H1 and removed from 25H1 (with batch tags), including net change counts
- Market breakdown (company count by HQ)
- Per-company data: name, batch, AI label + `AI Label Reasoning`, `Has Update` flag, 26H1 metrics, 25H1 metrics

**UI:**
- Floating orange "Ask AI" button (bottom-right corner), disabled until data loads
- Chat panel: fixed-position 420×560px drawer with header ("AppWorks Data Assistant"), scrollable message area, and input bar
- User messages: orange bubbles (right-aligned). Assistant messages: light grey bubbles (left-aligned) with basic markdown rendering (bold, code, lists)
- Streamed responses render token-by-token. Chat history maintained (last 20 messages for context).
- Responsive: full-width panel on mobile, FAB label hidden

---

## Number Formatting

| Type | Examples |
|------|----------|
| Raised / Valuation / Revenue | `$6.38B`, `$17.4M`, `$500K`, `$0` |
| Staff | `28.3K`, `450` (raw if < 1000) |
| Change values | `+$690M`, `-$2.1M`, `+37` |
| Negative currency | `-$5M` (sign before dollar sign) |

---

## UI / Design

### Theme
- Light mode, cream/米色 background (`#f5f0e8`)
- White card surfaces, warm grey borders
- AppWorks orange accent (`#ee7a2f`)
- Green (`#16a34a`) for positive changes, red (`#dc2626`) for negative

### Typography
| Usage | Font |
|-------|------|
| Body text | DM Sans |
| Large numbers (card values) | DM Serif Display |
| Labels, values, monospace elements | DM Mono |

### Components
- **Header:** `AppWorks Ecosystem Stats 26H1 (AW#32)` title + password input for Airtable PAT + Load button (also triggered by Enter key)
- **Status bar:** Monospace text with colored dot indicator (loading pulse / success green / error red); shows real-time per-page fetch progress during load
- **Tab bar:** Four tabs with orange underline indicator — General / AI Companies / Web3 / Presentation
- **Metric cards:** 5-column grid per tab. Each card: DM Mono label, DM Serif Display large value, optional subtitle (`26H1: X / 25H1: Y` on Active Startups), optional YoY badge (arrow + %, absolute change vs 25H1)
- **Ranked lists:** 4-column grid per tab. Each list: DM Mono title, 10 rows (rank · batch badge · company name · individual % of total · value), orange `Top 10 / Total — X%` footer
- **Batch badge:** Small grey DM Mono pill shown on ranked list rows and lookup dropdown entries. Prefers `AW#xx` over `Portfolio` when a company has both
- **Company lookup:** Per-tab scoped text search with keyboard-navigable dropdown (↑↓ Enter Escape). Result shows a 4-column table: Metric / 25H1 / 26H1 (Latest) / Change
- **Presentation slides:** Large title format (26px DM Sans bold) with accent-colored slide number (14px DM Mono). Optional subtitle (14px DM Sans, muted). Narrative text (14px DM Sans, muted) below cards. Collapsible sections use CSS `max-height` transition with a toggle button.
- **Chat FAB:** Floating orange button (bottom-right), shows "Ask AI" label on desktop, icon-only on mobile. Disabled (greyed out) until dashboard data loads.
- **Chat panel:** Fixed-position drawer (420×560px) with title bar, scrollable message history, and text input + send button. Opens when FAB is clicked; FAB hides while panel is open.
- **Skeleton loading:** Animated shimmer placeholders for all sections shown immediately while data is being fetched

### Responsive
- Desktop: 5-col cards, 4-col ranked lists
- Tablet (≤1024px): 2-col
- Mobile (≤600px): 1-col, stacked header

---

## Deployment

| Item | Detail |
|------|--------|
| Hosting | GitHub Pages |
| Repo | `natashaytc-imc/ecosystem-stats` |
| Branch | `main` |
| Build | None (static HTML) |
| Deploy trigger | `git push` to `main`, auto-deploys in ~1 min |
| Auth | None (API key entered at runtime by each user) |

### Update Process
```bash
cd "/Users/natashachou_aw/Ecosystem Stats"
# edit dashboard.html
git add -A && git commit -m "description of changes" && git push
```

---

## Security Considerations

- **Airtable PAT:** Entered by each user at runtime via a password input field. Never stored, committed, or transmitted anywhere except directly to Airtable's API.
- **Anthropic API key:** Embedded as a constant in the source code for convenience. The repo **should be set to private** to prevent exposure. Consider setting a usage limit on the Anthropic dashboard.
- **GitHub token:** Used only for initial deployment; removed from git remote after push.

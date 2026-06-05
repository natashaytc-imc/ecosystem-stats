# Chatbot Log — System Prompt & Data Flow

## Overview

The chatbot (AppWorks Data Assistant) uses **Claude Sonnet 4.6** (`claude-sonnet-4-6`) via the Anthropic Messages API with SSE streaming. It does NOT make any additional Airtable API calls — all data comes from in-memory global variables populated during `loadDashboard()`.

---

## Data Sources (Global Variables)

| Variable | Content | Built By |
|----------|---------|----------|
| `gData` | 26H1 filtered companies: `{ general, ai, web3 }` | `buildCompaniesFromStats()` → `filterCompanies()` → `filterAI()` / `filterWeb3()` |
| `gData25` | 25H1 filtered companies: `{ general, ai, web3 }` | Same pipeline with `period='25H1'` and `skipOnFilter: true` |
| `gH1Lookup` | `companyName → { raised, valuation, revenue, staff, hasUpdate, aiReasoning }` for 25H1 | `buildPeriodLookup(statsRecords, idToName, '25H1')` |
| `gLatestLookup` | Same structure for 26H1 | `buildPeriodLookup(statsRecords, idToName, '26H1')` |
| `gBatchMap` | `companyName → batchLabel` (e.g. "AW#09", "Portfolio", "AW#15 · Portfolio") | `buildBatchMap(companies)` |

---

## System Prompt Structure

The system prompt is built by `buildChatContext()` and consists of these sections, concatenated with newlines:

### Section 1: Role & Response Style

```
You are the AppWorks Ecosystem Stats data assistant...

RESPONSE STYLE:
1. LISTING REQUESTS: If the user simply asks to list or enumerate items
   (e.g. "列出 AI 公司", "list Web3 companies"), just provide the list
   WITHOUT extra analysis. End with: "需要更深入的分析嗎？"
2. ANALYTICAL QUESTIONS: If the user asks "why", "how", "compare",
   then provide data-driven analysis.
3. Keep initial answers concise and data-driven.

Rules:
- Always include batch tag when mentioning companies
- Flag stale data (hasUpdate=false)
- Ask which period if user doesn't specify
```

### Section 2: Aggregate Stats (per tab)

For each of `[general, ai, web3]`:

```
## All Portfolio — Aggregates (X companies, 26H1)
26H1 confirmed updated: Y/X companies (Z%).
Total Raised: $XXM | Valuation: $XXM | Revenue: $XXM | Staff: XXX
25H1 comparison (N companies had data): Raised: $XXM | ...
```

**Data source:** Iterates over `gData[tabKey]`, summing each company's `Latest Raised / Valuation / Revenue / Staff`. Cross-references `gLatestLookup` for `hasUpdate` count and `gH1Lookup` for 25H1 comparison sums.

### Section 3: Period Diffs (per tab)

For each of `[general, ai, web3]`, compares company name sets between `gData[tabKey]` (26H1) and `gData25[tabKey]` (25H1):

```
## All Portfolio — Changes (25H1 → 26H1)
25H1: X companies | 26H1: Y companies (net +Z)
Added in 26H1 (N): CompanyA (AW#12), CompanyB (Portfolio), ...
Not in 26H1 / removed (M): CompanyC (AW#05), ...
```

**Data source:** Set difference between `gData[tabKey]` names and `gData25[tabKey]` names. Batch tags from `gBatchMap`.

### Section 4: Market Breakdown

```
## Market Breakdown (by HQ)
TW: 45, SG: 12, HK: 8, ...
```

**Data source:** Iterates `gData.general`, calls `getHQ(company)` which reads `company.fields['HQ']`.

### Section 5: Per-Company Data

One line per company in `gData.general`:

```
## Per-Company Data
CompanyName | AW#09 | AI:AI-native (reasoning text) | upd:Y | R:5000000 V:20000000 Rev:1000000 S:50 | 25H1 R:3000000 V:15000000 Rev:800000 S:40
```

**Data source per company:**
| Field | Source |
|-------|--------|
| Name | `company.fields['Company Name']` |
| Batch | `gBatchMap[name]` |
| AI Label | `company.fields['AI Label']` |
| AI Reasoning | `gLatestLookup[name].aiReasoning` |
| Updated flag | `gLatestLookup[name].hasUpdate` |
| 26H1 metrics | `company.fields['Latest Raised/Valuation/Revenue/Staff']` |
| 25H1 metrics | `gH1Lookup[name].raised/valuation/revenue/staff` |

---

## Filtering Logic

### How companies get into `gData`

```
Airtable (Ecosystem Stats table, all periods)
  │
  ├─ buildCompaniesFromStats(stats, idToName, hqMap, '26H1')
  │    → creates company objects from 26H1 records
  │    → resolves linked record IDs to names via idToName map
  │    → attaches On, Batch, AI Label, Company Verticals, HQ, metrics
  │
  ├─ filterCompanies(allCompanies26)
  │    → On === '1' (active)
  │    → Batch includes Portfolio or AW#1–AW#32
  │    → Result: gData.general
  │
  ├─ filterAI(gData.general)
  │    → AI Label === 'AI-native' OR 'AI-enabled'
  │    → Result: gData.ai
  │
  └─ filterWeb3(gData.general)
       → Company Verticals includes 'Web3'
       → Result: gData.web3
```

### How 25H1 companies get into `gData25`

Same pipeline but with `period='25H1'` and `skipOnFilter: true` (no On filter).
AI labels for 25H1 use `AI-Label (from 26H1)` lookup field (retroactive classification).

---

## Chat Flow

```
User types message
  → appendMessage('user', msg)
  → push to chatHistory[]
  → buildChatContext() generates system prompt from global vars
  → POST to Anthropic API:
      {
        model: 'claude-sonnet-4-6',
        max_tokens: 1024,
        system: <system prompt>,
        messages: chatHistory.slice(-20),  // last 20 messages for context
        stream: true
      }
  → SSE stream response token-by-token
  → appendMessage('assistant', fullText)
  → push to chatHistory[]
```

### Key behaviors:
- **System prompt is rebuilt on every message** — always reflects current in-memory data
- **Context window:** last 20 messages (user + assistant) are sent
- **No tool use / function calling** — Claude answers purely from the data in the system prompt
- **Streaming:** responses render token-by-token via SSE

---

## Example: "列出 AI-native 的公司"

1. `buildChatContext()` runs, includes per-company data with AI labels
2. Claude sees the RESPONSE STYLE rule: this is a LISTING REQUEST
3. Claude filters companies where `AI:AI-native` in the per-company data
4. Response: lists company names with batch tags, no analysis
5. Ends with: "需要更深入的分析嗎？"

## Example: "為什麼 Web3 公司數量減少了？"

1. `buildChatContext()` runs, includes period diff section showing added/removed Web3 companies
2. Claude sees the RESPONSE STYLE rule: this is an ANALYTICAL QUESTION ("why")
3. Response: FIRST lists specific companies removed/added, THEN provides analysis

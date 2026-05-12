# 🏭 PE Firm Portfolio Scraper — Manufacturing & APAC Targeting

> An automated n8n workflow that classifies Private Equity firms, scrapes their full portfolio company lists, and filters for manufacturing, industrials, logistics, healthcare, and APAC-connected investments.

---

## 📋 Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [6-Stage Classification Framework](#6-stage-classification-framework)
- [Workflow Architecture](#workflow-architecture)
- [Setup & Installation](#setup--installation)
- [Configuration](#configuration)
- [Output](#output)
- [Targeting Logic](#targeting-logic)
- [Troubleshooting](#troubleshooting)

---

## Overview

This system automates the end-to-end process of:

1. **Qualifying** PE firms using 4 gates (APAC mandate, physical sectors, size, deal activity)
2. **Classifying** each firm into one of 4 types (Centralized / Hybrid / Network / Decentralized)
3. **Scraping** their full portfolio company list (150–200+ companies per firm)
4. **Filtering** for manufacturing, logistics, energy, infrastructure, healthcare with physical ops
5. **Outputting** a clean prospect list to Google Sheets

---

## How It Works

```
Google Sheet (Classification)
        ↓
Filter: Hybrid + Decentralized only
        ↓
Loop — one firm at a time
        ↓
  Build Portfolio URLs (smart fallback)
        ↓
  ┌─────────────────────────────────────┐
  │  7 Parallel Data Sources            │
  │  • Tavily Extract — Main page       │
  │  • Tavily Extract — Alt URL         │
  │  • Tavily Extract — Page 2 & 3      │
  │  • Search — Full portfolio list     │
  │  • Search — Manufacturing & APAC    │
  │  • Search — Healthcare & Logistics  │
  └─────────────────────────────────────┘
        ↓
  Smart Data Cleaner
  (strips base64, extracts company name patterns)
        ↓
  AI Step 1 — Census (GPT-4o)
  "List every portfolio company you can find"
        ↓
  Census Parser (merges AI + pattern-extracted names)
        ↓
  AI Step 2 — Filter & Classify (GPT-4o)
  "Keep only physical ops / APAC companies"
        ↓
  Result Builder (deduplicate + validate)
        ↓
Google Sheet (Portfolio Companies)
```

---

## 6-Stage Classification Framework

### Stage 1 — Firm Qualification (4 Gates)

Every PE firm must pass **all 4 gates** to enter the workflow:

| Gate | Check | Pass Condition |
|------|-------|---------------|
| **Gate 1** | APAC Investment Mandate | Office in SG, HK, Mumbai, Jakarta, Sydney, or Tokyo — OR portfolio companies have APAC operations |
| **Gate 2** | Physical Operations Sectors | 30%+ of portfolio in manufacturing, industrials, infrastructure, logistics, energy, consumer goods, healthcare (physical ops), or real assets |
| **Gate 3** | Right Size | AUM $500M–$15B, team 20–500 people. Sovereign wealth funds (GIC, Temasek, INA) bypass this gate |
| **Gate 4** | Active APAC Deal Flow | At least 1 APAC deal closed in the last 24 months |

### Stage 2 — Firm Classification (7 Weighted Checks)

| Weight | Check | Centralized Signal | Decentralized Signal |
|--------|-------|-------------------|---------------------|
| 25% | Website team page | Dedicated ops team page with named members | No separate ops page |
| 20% | Language analysis | "team of X professionals", "full-time" | "network of X advisors" |
| 20% | LinkedIn headcount | 50+ ops/value-creation titles | Fewer than 5 results |
| 15% | Job postings | Full-time ops roles posted | Only deal team roles |
| 10% | Annual report | Named ops team with headcount | — |
| 5% | Board composition | Ops members as employees | Ops as independent affiliates |
| 5% | Umbrex cross-reference | Confirmed centralized profile | Confirmed decentralized |

> Confidence score below 60 → flagged for manual review

### Stage 3 — Classification Output

| Type | Examples |
|------|---------|
| **Centralized** | KKR Capstone, Bain Capital Portfolio Group |
| **Hybrid** | Carlyle GPS, TPG, Warburg Pincus, General Atlantic |
| **Network-based** | EQT Advisors, Partners Group, Advent |
| **Decentralized** | Apollo Global, Affinity, Navis, Granite Asia |

### Stage 4 — Targeting Paths

| Path | Description | Triggered By |
|------|------------|-------------|
| **Path A** | Scrape ops team from website | Centralized + Hybrid (internal team) |
| **Path B** | Portfolio company operators | Decentralized, Network, Hybrid |
| **Path C** | LinkedIn external advisors | Network-based + Hybrid |
| **Path D** | PE firm deal partners & associates | All firm types |

**Path combinations:**
- Centralized = A + D
- Hybrid = A + B + D
- Network = B + C + D
- Decentralized = B + D

### Stage 5 — Title Targeting

| Level | Titles | Role |
|-------|--------|------|
| **L1** Associates | Investment Associate, Analyst, Senior Associate | Own vendor evaluation workflow |
| **L2** VPs | Vice President, Director, Investment Manager | $20–50K budget authority, no partner sign-off |
| **L3** Partners | Partner, MD, Managing Partner | Deal decision authority |
| **L4** Ops Team *(centralized only)* | Operating Partner, VP Value Creation, Capstone Associate | Direct BI buyer |
| **L5** Portfolio Co *(Path B only)* | CEO, COO, VP Supply Chain, VP Corp Dev, Head of M&A | Highest conversion, APAC expansion budget |

> **Outreach sequence:** L1–L2 → L5 → L3–L4

### Stage 6 — Prioritization

| Priority | Trigger | Action |
|----------|---------|--------|
| **P1** — Within 48h | Changed jobs in 30 days + ICP match; OR firm just announced APAC deal; OR portfolio co posted APAC roles | Send connection request immediately |
| **P2** — Within 1 week | Posted about APAC in 14 days; OR headcount growing 10%+; OR Tier 1 firm, no prior outreach | Engage with content first, then connect |
| **P3** — Nurture | All other ICP matches | Like/comment 1–2 weeks before connecting |
| **Discard** | No APAC signal; pure fintech/software PE; AUM under $500M | Remove from list |

---

## Workflow Architecture

### Nodes (n8n)

| Node | Type | Purpose |
|------|------|---------|
| Manual Trigger | Trigger | Start workflow |
| Read Classification Sheet | Google Sheets | Read firm list |
| Filter Decentralized & Hybrid | Code | Keep only Hybrid + Decentralized firms |
| Loop Over Firms | Split In Batches | Process one firm at a time |
| Wait 3s | Wait | Rate limit protection |
| Build Portfolio URLs | Code | Clean junk URLs, generate fallback variants |
| Extract — Main/Alt/Page 2/Page 3 | HTTP Request | Tavily Extract (JS-rendered pages + pagination) |
| Search 1/2/3 | HTTP Request | Tavily Search (portfolio, manufacturing, APAC) |
| Merge All 7 Sources | Merge | Combine all data |
| Smart Data Cleaner | Code | Strip base64, extract company name patterns |
| AI Step 1 — Census | HTTP Request | GPT-4o: list all company names |
| Census Parser | Code | Merge AI + pattern-extracted names |
| AI Step 2 — Filter & Classify | HTTP Request | GPT-4o: filter physical ops / APAC |
| Portfolio Result Builder | Code | Deduplicate, validate, flatten |
| Wait 1s | Wait | Google Sheets rate limit protection |
| Write to Portfolio Sheet | Google Sheets | Append results |
| Loop Back | Merge | Return to loop for next firm |

---

## Setup & Installation

### Prerequisites

- [n8n](https://n8n.io/) (self-hosted or cloud)
- Tavily API account — [tavily.com](https://tavily.com)
- OpenAI API key (GPT-4o access)
- Google account with Sheets API enabled

### Steps

1. **Clone or download** this repository
2. **Import the workflow** into n8n:
   - Open n8n → Workflows → Import
   - Upload `Portfolio_Scraper_V6.json`
3. **Set up credentials** in n8n:
   - `OpenAI API` — add your OpenAI key
   - `Google Sheets OAuth2` — connect your Google account
4. **Prepare your Google Sheet** with two tabs:

#### Tab 1: `W2_Classification`
| Column | Description |
|--------|-------------|
| `pe_firm_name` | Full firm name |
| `classification` | `HYBRID` or `DECENTRALIZED` |
| `website_url` | Firm homepage URL |
| `portfolio_page_url` | Direct portfolio page URL (optional) |

#### Tab 2: `Portfolio Companies` *(output — create with headers)*
| `pe_firm_name` | `classification` | `company_name` | `industry` | `region` | `hq_country` | `website` | `total_found` | `run_date` |

---

## Configuration

### API Keys (in n8n credentials)

```
OpenAI API Key     →  OpenAI account credential
Tavily API Key     →  Hardcoded in HTTP Request nodes (update if key changes)
Google Sheets      →  OAuth2 credential
```

### Google Sheet ID

Update `GOOGLE_SHEET_DOC_ID` in the workflow if you use a different sheet.

### Filter Settings

The AI filter keeps companies matching **any** of these:

**Sectors (Condition A):**
- Manufacturing, Industrials, Packaging, Chemicals
- Logistics / Supply Chain / Freight / Warehousing
- Energy (with physical assets)
- Infrastructure — roads, utilities, ports, construction
- Healthcare with physical ops — hospitals, clinics, pharma mfg, medical devices
- Consumer Goods with own production/distribution
- Real Assets — property, farmland
- Agriculture / Agritech with physical farms
- Mining, Defense manufacturing, Waste management

**APAC Presence (Condition B):**
- Office or operations in: Singapore, Hong Kong, Mumbai, Jakarta, Sydney, Tokyo
- Any APAC country: India, China, Japan, Korea, Australia, Indonesia, Vietnam, Malaysia, Thailand, Philippines

**Hard Excludes:**
- Pure SaaS / cloud software
- Pure fintech / digital payments
- Online education
- Pure digital media
- AdTech / MarTech

---

## Output

Each row in the `Portfolio Companies` sheet:

| Field | Description |
|-------|-------------|
| `pe_firm_name` | Source PE firm |
| `classification` | HYBRID or DECENTRALIZED |
| `company_name` | Portfolio company name |
| `industry` | One of: Manufacturing, Logistics & Supply Chain, Energy, Infrastructure, Healthcare Physical, Consumer Goods Physical, Real Assets, Agribusiness, Industrial Tech, Other Physical |
| `region` | Southeast Asia, South Asia, East Asia, APAC, Europe, North America, MENA, Africa, Latin America, Global |
| `hq_country` | HQ country |
| `website` | Company website |
| `total_found` | Total qualifying companies found for this firm |
| `run_date` | Date workflow ran |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `cleaned_text` is empty | Tavily Extract returning base64 only | Check `portfolio_page_url` in sheet — must be a valid URL starting with `https://` |
| `portfolio_page_url` shows `["advisor network"]` | Bad data in sheet | Manually enter the correct portfolio URL, e.g. `https://firmname.com/portfolio` |
| Only 1–3 companies returned | AI hallucinating with empty input | Check Census Parser output — `census_count` should be 10+ |
| Tavily 432 error | API usage limit exceeded | Upgrade Tavily plan or wait for quota reset |
| `?` icon on AI nodes | LangChain node not installed in your n8n version | The workflow uses plain HTTP Request nodes to call OpenAI directly — no LangChain needed |
| Fake company names (hallucination) | AI received empty company list | Ensure `extracted_names` in cleaner output is non-empty |
| Google Sheets write fails | Rate limit | The `Wait 1s` node handles this — if still failing, increase wait to 2s |

---

## Tech Stack

- **n8n** — workflow automation
- **Tavily API** — web extraction and search (JS-rendered pages)
- **OpenAI GPT-4o** — company name extraction and classification
- **Google Sheets** — input (firm list) and output (portfolio companies)

---

## License

MIT License — free to use and modify.

---

*Built for PE firm prospect targeting — manufacturing, industrials, logistics, and APAC-connected investments.*

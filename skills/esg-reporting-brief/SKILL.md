---
name: esg-reporting-brief
description: Use when a user asks to draft an ESG performance narrative, board pack, investor update, or annual report section from their KEY ESG data.
compatibility: Requires KEY ESG MCP server (https://api.keyesg.com/mcp) connected as a custom connector or via Claude Desktop config.
---

## Step 1 — Establish Scope

Confirm with the user:

- **Reporting year** (required, e.g. `2024`).
- **Audience**: `board`, `investor`, or `annual_report`.

Determine session type from your tool list:

- **`list_portfolio_metrics` available** → Fund Manager session
- **`list_metrics` available** → company session (standalone or portfolio company)

Tone by audience:

- `board` — concise, strategic, narrative-first, limited data tables
- `investor` — data-heavy, credible, transparent on coverage gaps
- `annual_report` — formal, comprehensive, third-party-ready

---

## Step 2 — Portfolio / Company Overview (Fund Manager only)

Skip for standalone/portfolio company users.

```
get_portfolio_reporting_summary
  year: "<year>"
```

Returns: portfolio company details (name, ID, country of HQ, primary operating country, sector, industry), company count, reporting entity count, fund list, assigned frameworks, organisation currency, sector. Use this in the "Portfolio Overview" section — the per-company geographic and sector data enriches breakdowns.

---

## Step 3 — Metrics by ESG Pillar

Fetch all metrics in a single call — each result includes `category`, so the LLM can bucket them by pillar without separate requests.

**Fund Manager path:**

```
list_portfolio_metrics
  year: "<year>"
```

**Self-assessment path:**

```
list_metrics
  year: "<year>"
```

For each pillar, select the most meaningful metrics (prioritise those with `hasValue: true`). Fetch their values:

**Fund Manager — batch up to 50 IDs per call:**

```
get_portfolio_metric_value
  metricIds: ["<uuid>", ...]
  year: "<year>"
  fundId: "<uuid>"   # optional
```

Returns `companiesIncluded` and `companiesMissingData` per metric. `companiesIncluded` counts companies with real reported values; `companiesMissingData` counts companies on the data request without data. Their sum is the total in-scope companies (those asked for this metric, not the whole portfolio). Use for coverage: e.g. "15 of 22 in-scope companies (7 missing data)".

**Fund Manager — single portfolio company drilldown (optional):**

To include per-company breakdowns in the report, use `get_metric_value` with `companyId`:

```
get_metric_value
  metricIds: ["<uuid>", ...]
  year: "<year>"
  companyId: "<portfolio-company-uuid>"   # from get_portfolio_reporting_summary
```

Only built-in and fund-manager-owned custom metrics are returned.

**Self-assessment — batch up to 50 IDs:**

```
get_metric_value
  metricIds: ["<uuid>", ...]
  year: "<year>"
```

Note any `isFresh: false` values — flag in the Data Notes section.

---

## Step 4 — Target Progress

```
list_targets
```

Group targets by status:

- **Achieved** — highlight as wins
- **On track** — positive forward momentum
- **At risk / Missed** — acknowledge transparently; do not omit
- **Not started** — include if audience context warrants it

---

## Step 5 — Forward-Looking Content

```
list_action_plans
  status: "in_progress"   # optional — filter to active initiatives
list_policies
  category: "<environment|social|governance|general>"   # optional
```

Use action plans for the "Looking Ahead" section (initiatives underway, investment in ESG improvement).
Use policies for governance narrative — include `documentUrl` as an evidence link where relevant.

---

## Step 6 — Draft Document

Use the structure below. Omit sections where no data is available.

```markdown
# ESG Performance [Brief/Report/Summary] — [Year]

[Organisation name] | [Audience] | [Date]

## Executive Summary

[3–5 sentences. For board: highlight 2–3 key achievements and 1–2 risks.
For investor: open with headline metric, then target progress.
For annual report: formal statement of ESG strategy and year's outcomes.]

## [Portfolio Overview / Company Profile]

[Fund Manager: company count, fund names, frameworks in use, sector, currency.
Standalone: company description, frameworks, reporting period.]

## Environmental Performance

[Narrative paragraph followed by a data table of key E metrics.]

| Metric           | Value  | Unit  | Coverage        |
| ---------------- | ------ | ----- | --------------- |
| GHG Scope 1      | 12,450 | tCO2e | 18/22 companies |
| GHG Scope 2      | 8,730  | tCO2e | 15/22 companies |
| Renewable energy | 41%    | %     | 12/22 companies |

## Social Performance

[Narrative + data table of key S metrics.]

## Governance

[Narrative + data table of key G metrics. Include policy highlights if list_policies available.]

## Target Progress

| Target                         | Status   | Progress | Target Year |
| ------------------------------ | -------- | -------- | ----------- |
| Net zero Scope 1+2 by 2030     | On track | 34%      | 2030        |
| 50% renewable energy by 2026   | At risk  | 41%      | 2026        |
| Zero waste to landfill by 2025 | Achieved | 100%     | 2025        |

## Looking Ahead

[Active initiatives from list_action_plans. Forward commitments from list_policies.
For board/investor: strategic framing. For annual report: programme details.]

## Data Notes

[Always include this section. List every metric with hasValue: false or isFresh: false.
Example: "Scope 3 emissions: no portfolio companies reported data for 2024."
Do not omit data gaps — flag them transparently.]
```

---

## Common Mistakes

- **Omitting the Data Notes section** — This section is mandatory for all audiences. Sophisticated readers (investors, auditors) will notice absent metrics; transparency builds credibility.
- **Omitting at-risk or missed targets** — These must be acknowledged transparently, not omitted. Credibility depends on honest reporting.
- **Fabricating or estimating values** — If data is absent, state it in Data Notes. Never interpolate.

---

## Notes

- For `investor` audience, include absolute numbers and percentage coverage (e.g. "15 of 22 companies"). Credibility depends on transparency about what is and isn't measured.
- For `board` audience, translate metrics into business/risk language where possible (e.g. "carbon intensity improved 12%").
- For `annual_report`, use formal language, avoid first-person, and include all standard ESG disclosures supported by the data.
- The Data Notes section is mandatory — do not remove it for any audience. Sophisticated readers (investors, auditors) will notice absent metrics; transparency builds credibility.
- Never fabricate or estimate values. If data is absent, say so in Data Notes.

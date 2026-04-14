---
name: esg-questionnaire
description: Use when answering LP or vendor ESG due diligence questionnaires using KEY ESG data, or when a user needs to populate a questionnaire from ESG metrics, targets, policies, or action plans.
compatibility: Requires KEY ESG MCP server (https://api.keyesg.com/mcp) connected as a custom connector or via Claude Desktop config.
---

## Step 1 — Establish Context

Determine session type from the tools visible in your tool list:

- **`list_portfolio_metrics` is available** → Fund Manager session — use portfolio tools (`list_portfolio_metrics`, `get_portfolio_metric_value`)
- **`list_metrics` is available** → Company session (standalone or portfolio company) — use self-assessment tools (`list_metrics`, `get_metric_value`)

Ask the user for:

- **Reporting year** (required, e.g. `2026`). Do not proceed without this.
- **Fund scope** (Fund Manager only, optional) — answers scoped to a specific fund, or the whole portfolio? Only ask if you already have fundIds from a previous tool call otherwise skip this.

---

## Step 2 — Load Portfolio Context (Fund Manager only)

Skip this step for standalone/portfolio company users.

```
get_portfolio_reporting_summary
  year: "<year>"
  fundId: "<uuid>"   # optional — omit for whole portfolio or if calling tool for the first time
```

Returns: portfolio company details (name, ID, country of HQ, primary operating country, sector, industry), company count, reporting entity count, fund list, assigned frameworks, organisation currency, sector. Use this to contextualise answers (e.g. "across 18 portfolio companies in Fund A").

---

## Step 3 — Discover Available Metrics

Fetch the full metric catalogue. This tells you what data exists and whether values are available for the reporting year.

**Fund Manager path:**

```
list_portfolio_metrics
  year: "<year>"
  fundId: "<uuid>"         # optional
  category: "<environment|social|governance|general>"   # optional — use per questionnaire section
```

**Self-assessment/standalone company path:**

```
list_metrics
  year: "<year>"
  scope: "<scope-1|scope-2|scope-3>"   # optional
  category: "<environment|social|governance|general>"   # optional
```

Both return: `metricId`, `metricTitle`, `unitOfMeasure`, `hasValue`, `category`, `ghgScope`.

Save this list. You will match questionnaire questions against `metricTitle` in the next step.

---

## Step 4 — Answer Each Question

Work through each questionnaire question using the classification logic below.

### 4a. Metric / Performance Questions

Questions about quantities, measurements, or reported data (e.g. "Total Scope 1 emissions?", "Employee headcount?", "Renewable energy %?").

1. Find the closest `metricTitle` match from Step 3 (semantic match — partial/synonymous matches are fine).
2. If `hasValue: false` → do not fetch; apply the low-confidence flag from Step 5.
3. If `hasValue: true` → fetch the value.

**Fund Manager path — batch up to 50 IDs per call:**

```
get_portfolio_metric_value
  metricIds: ["<uuid>", "<uuid>", ...]
  year: "<year>"
  fundId: "<uuid>"   # optional
```

**Fund Manager — single portfolio company (optional):**

If the questionnaire asks about a specific portfolio company rather than the aggregated portfolio, use `get_metric_value` with `companyId`:

```
get_metric_value
  metricIds: ["<uuid>", "<uuid>", ...]
  year: "<year>"
  companyId: "<portfolio-company-uuid>"   # from get_portfolio_reporting_summary
```

Only built-in metrics and fund-manager-owned custom metrics are returned; portfolio-company custom metrics are excluded.

**Standalone/Self-assessment path — batch up to 50 IDs per call:**

```
get_metric_value
  metricIds: ["<uuid>", "<uuid>", ...]
  year: "<year>"
```

For the portfolio path, `companiesIncluded` counts companies with real reported values; `companiesMissingData` counts companies on the data request without data. Their sum is the total in-scope companies (those asked for this metric, not the whole portfolio). Use for coverage: e.g. "12,450 tCO2e across 18 of 22 in-scope companies (4 missing data)".

Draft answer: **"[value] [unitOfMeasure] ([metricTitle], [year])"**

### 4b. Target / Commitment Questions

Questions about goals, net-zero commitments, or reduction pledges (e.g. "Do you have a net-zero target?", "What are your emissions reduction commitments?").

```
list_targets
  category: "<environment|social|governance|general>"   # optional
```

Each target includes a `description` field — a generated human-readable sentence covering type, value, unit, baseline year, and target year. Use `description` directly in the answer. Also note `status` (achieved / on_track / at_risk / not_started / missed) and `progressPercent`.

### 4c. Policy Questions

Questions about policies in place (e.g. "Environmental policy?", "Modern slavery policy?").

```
list_policies
  category: "<environment|social|governance|general>"   # optional
```

Returns: `id`, `policyType`, `title`, `description`, `category`, `reviewedAt`, `documentUrl`.

Match the question to a policy using `policyType` first (e.g. `modern_slavery`, `climate`, `dei`) then `description`. Always include `documentUrl` in the answer — it is the evidence link for the policy.

### 4d. Action Plan / Initiative Questions

Questions about ongoing ESG improvement programmes.

```
list_action_plans
  category: "<environment|social|governance|general>"   # optional
  status: "in_progress"   # optional — filter to active initiatives
```

Returns: `id`, `title`, `description`, `category`, `owner`, `status`, `dueDate`.

### 4e. Qualitative / Narrative Questions

Questions that cannot be answered from platform data (e.g. "Describe your ESG governance structure"). Draft a placeholder and flag for human review.

---

## Step 5 — Flag Low-Confidence Items

Be explicit when data quality is uncertain. Use these standard flags:

| Situation                                     | Flag                                                                |
| --------------------------------------------- | ------------------------------------------------------------------- |
| `value: null`                                 | ⚠️ No data available for [year]                                     |
| `isFresh: false`                              | ⚠️ Value may be stale — recalculation pending                       |
| `hasValue: false` for all portfolio companies | ⚠️ Metric exists but no companies have reported data for [year]     |
| `companiesMissingData > 0`                    | ⚠️ [n] in-scope companies have not reported data for this metric    |
| No matching `metricTitle` found               | ⚠️ No matching metric — human review required                       |
| Tool not yet available                        | ⚠️ Data not accessible via this integration — human review required |
| Qualitative / narrative question              | ⚠️ Requires human input — cannot be answered from platform data     |

---

## Step 6 — Present for user review

Present the completed questionnaire as a table, followed by a collected "Review Required" list.

```
## ESG Questionnaire — [Year]
Data source: KEY ESG [portfolio data / self-assessment] ([Fund name] if fund-scoped)

| # | Question | Answer | Source | Notes |
|---|----------|--------|--------|-------|
| 1 | Total Scope 1 emissions | 12,450 tCO2e | GHG Scope 1 (2024) | |
| 2 | Net-zero commitment | Reduce absolute Scope 1+2 by 50% by 2030 vs 2020 baseline (on track, 34% progress) | Target | |
| 3 | Modern slavery policy | In place — [documentUrl] | list_policies (modern_slavery) | |

## Review Required
- Q3: Modern slavery policy — policy data not accessible via this integration
- Q7: Employee wellbeing programmes — no matching metric found
```

---

## Step 7 - Fill questionnaire

Offer to create a filled in version of the questionnaire in the same format as the input. Present as a table as a fallback, if
you're not able to create an excel sheet or pdf file.

---

## Common Mistakes

- **Fetching values when `hasValue: false`** — The value will be null. Skip the fetch and flag it instead.
- **Fabricating or estimating missing values** — If data is absent, flag it. Never interpolate or estimate.

---

## Notes

- Batch `get_metric_value` and `get_portfolio_metric_value` calls (up to 50 metric IDs per call) to reduce round-trips.
- If a portfolio tool returns an authorisation error, the session lacks Fund Manager admin rights — note the scope limitation and do not fabricate data.
- If the questionnaire has sections (E / S / G), use the `category` filter in Step 3 per section to keep discovery focused.
- Do not estimate or interpolate missing values. If data is absent, flag it.

---
name: portfolio-data-query
description: Use when a Fund Manager user asks a plain-language question about portfolio ESG data, requests metric values across funds or years, or wants to compare ESG performance across their portfolio.
compatibility: Requires the KEY ESG MCP server connected as a custom connector or via Claude Desktop config — either the legacy endpoint (https://api.keyesg.com/mcp) or the fund-manager endpoint (https://api.keyesg.com/pe/mcp). Uses portfolio tools, so a fund-manager account is required; the standalone endpoint does not serve them. The connector may appear under any name (typically "key-esg" or "KEY ESG") — detect the connection by tool availability (e.g. who_am_i), not by server name or URL.
---

## Step 1 — Confirm Session and Parse the Question

Call `who_am_i` first (no arguments). Confirm `organisationType: "fund_manager"` — if it returns `"company"`, this skill does not apply; the user has a company account with no portfolio tools. `who_am_i` also returns `portfolioCompanies` (`companyId` + `companyName`) — use these ids directly for single-company drilldowns instead of an extra `get_portfolio_reporting_summary` call when you only need the id.

Extract from the user's question:

- **Year** — required. Ask if not given. Do not assume the current year without confirmation.
- **Fund name** — if a specific fund is mentioned, resolve it to a `fundId` in Step 2.
- **ESG category** — environment, social, governance, or general. Optional — use to narrow metric discovery.
- **GHG scope** — scope-1, scope-2, or scope-3. Optional — relevant for emissions questions.
- **Comparison intent** — is the user comparing funds, years, or metrics?

If the year is ambiguous (e.g. "latest", "current year"), ask before proceeding.

---

## Step 2 — Resolve Fund ID and Portfolio Context

```
list_funds
  year: "<year>"
```

Returns: `fundId`, `fundName`, `vintageYear`, `portfolioCompanyCount` for each fund.

Match the user's fund name against `fundName` (case-insensitive, partial match). Use the matched `fundId` in subsequent calls.

If no match is found, show the list of available fund names and ask the user to confirm which one they mean.

For geographic or sector-specific questions, also call:

```
get_portfolio_reporting_summary
  year: "<year>"
  fundId: "<uuid>"   # optional
```

Returns per-company details including `countryOfHq`, `primaryCountry`, `sector`, `industry` — useful for answering questions like "Which companies in Europe have the highest emissions?" or "What's the sector breakdown of our portfolio?".

---

## Step 3 — Discover Relevant Metrics

```
list_portfolio_metrics
  year: "<year>"
  fundId: "<uuid>"         # optional — omit for whole-portfolio scope
  category: "<environment|social|governance|general>"   # optional
```

Returns: `metricId`, `metricTitle`, `ghgScope`, `category`, `unitOfMeasure`, `hasValue`.

Filter by relevance to the user's question using `metricTitle` (semantic match). Identify the 1–5 most relevant metrics.

---

## Step 4 — Fetch Values

For each relevant metric where `hasValue: true` (batch up to 50 metricIds in a single call):

```
get_portfolio_metric_value
  metricIds: ["<uuid>", "<uuid>", ...]
  year: "<year>"
  fundId: "<uuid>"   # optional
```

Returns `value`, `unitOfMeasure`, `companiesIncluded`, `companiesMissingData`, `isFresh` per metric. `companiesIncluded` counts companies with real reported values; `companiesMissingData` counts companies on the data request without data. Their sum is the total in-scope companies (those asked for this metric, not the whole portfolio). Use for coverage: e.g. "12,450 tCO2e across 18 of 22 in-scope companies (4 missing data)".

For metrics where `hasValue: false`: do not fetch — note the coverage gap in the answer.

---

### Single Company Drilldown

If the user asks about a specific portfolio company (e.g. "What are Company X's Scope 1 emissions?"):

```
get_metric_value
  metricIds: ["<uuid>", ...]
  year: "<year>"
  companyId: "<portfolio-company-uuid>"   # from who_am_i portfolioCompanies or get_portfolio_reporting_summary
```

Match the company name against `who_am_i`'s `portfolioCompanies` (case-insensitive, partial match). This returns that company's own metric values rather than the portfolio aggregate. Only built-in and fund-manager-owned custom metrics are returned.

---

## Step 5 — Answer

**Single metric:**

```
Scope 1 emissions (2024): 12,450 tCO2e
Across 18 portfolio companies.
```

**Multiple metrics or fund comparison — use a table:**

```
| Metric | Value | Unit | Year |
|--------|-------|------|------|
| GHG Scope 1 | 12,450 | tCO2e | 2024 |
| GHG Scope 2 | 8,730 | tCO2e | 2024 |
| GHG Scope 3 | — | tCO2e | No data |
```

**Coverage gaps** — always state explicitly; never silently omit:

- `hasValue: false` → "No portfolio companies have reported data for [metricTitle] in [year]."

---

## Comparison Queries

### Fund vs fund

1. `list_funds(year)` to get all fund IDs.
2. For each fund: `get_portfolio_metric_value(metricIds: [id], year, fundId)`.
3. Present as a table sorted by value.

### Year-over-year

Repeat Steps 3–4 for each year in scope. Present with year columns.

### "Which fund has the highest / lowest X?"

Call `get_portfolio_metric_value(metricIds: [id], year, fundId)` for each fund and rank the results.

---

## Common Mistakes

- **Assuming the current year** — Always ask if the year is not specified. Do not default silently.
- **Silently omitting `hasValue: false` metrics** — Always state explicitly when data is unavailable; never drop it from the answer without comment.
- **Calling portfolio tools without a fund-manager role** — Portfolio tools require a **Fund Manager Admin** or **Fund Manager** KEY ESG account (these are different roles; both can use portfolio MCP tools). If an auth error is returned, note the role requirement rather than retrying.

---

## Error Handling

| Situation                           | Action                                                                                                               |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Year not given                      | Ask before proceeding                                                                                                |
| Fund name not found in `list_funds` | Show available fund names, ask user to confirm                                                                       |
| `hasValue: false` for all funds     | Report "No data available for [metricTitle] in [year]"                                                               |
| Auth error on portfolio tool        | Note "Fund Manager Admin or Fund Manager role required for portfolio data"                                           |
| No matching metric for question     | Say "No matching metric found for '[question]'" — suggest closest alternatives from `list_portfolio_metrics` results |

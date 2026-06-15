---
name: metric-breakdown-analysis
description: Use when a user wants to break a KEY ESG metric down over time (monthly or quarterly) or across entities, sites, or portfolio companies — for example "how did our Scope 1 emissions trend month by month?", "which sites drove our water use?", or "break this metric down by portfolio company and entity".
compatibility: Requires the KEY ESG MCP server connected as a custom connector or via Claude Desktop config. Any endpoint works — the legacy endpoint (https://api.keyesg.com/mcp), the fund-manager endpoint (https://api.keyesg.com/pe/mcp), or the standalone-company endpoint (https://api.keyesg.com/standalone/mcp). The connector may appear under any name (typically "key-esg" or "KEY ESG") — detect the connection by tool availability (e.g. who_am_i), not by server name or URL.
---

## Step 1 — Establish Scope

Call `who_am_i` first (no arguments). `organisationType` determines which metric tools apply in Step 2 (`"fund_manager"` → `list_portfolio_metrics`; `"company"` → `list_metrics`), and `companyName` names the organisation in output headers. If `who_am_i` is unavailable, fall back to the tool list: `list_portfolio_metrics` present → Fund Manager session; `list_metrics` present → company session.

Confirm with the user:

- **Reporting year** (required, e.g. `2024`).
- **Metric(s)** of interest — a name or theme you will resolve to metric IDs in Step 2.
- **Breakdown dimension**:
  - **Time** — monthly (`m1`–`m12`) or quarterly (`q1`–`q4`).
  - **Entity** — by site / subsidiary within one company.
  - **Portfolio** (fund managers only) — by portfolio company, then by entity.

Session type from `who_am_i`:

- **`organisationType: "fund_manager"`** → Fund Manager session (`portfolioCompanies` gives accessible company ids)
- **`organisationType: "company"`** → company session (standalone or portfolio company)

---

## Step 2 — Resolve Metric IDs

Find the metric IDs to break down. Each result includes `category` and (where available) whether it has a value, so you can pick meaningful metrics without extra calls.

**`organisationType: "company"` path:**

```
list_metrics
  year: "<year>"
```

**`organisationType: "fund_manager"` path:**

```
list_portfolio_metrics
  year: "<year>"
  fundId: "<uuid>"   # optional — scope to one fund
```

Prefer metrics with reported data. Collect up to 50 metric IDs to break down.

---

## Step 3 — Time Breakdown (monthly / quarterly)

Use `get_metric_breakdown` to get the org-level period series plus the yearly total.

```
get_metric_breakdown
  metricIds: ["<uuid>", ...]
  year: "<year>"
  period: "monthly"      # or "quarterly"
  companyId: "<uuid>"    # fund managers only — a portfolio company you can access
```

Each metric returns:

- `total` — the yearly roll-up (same value as `get_metric_value`).
- `periods` — an ordered series: 12 entries for monthly, 4 for quarterly. Missing periods are `null` (not omitted).
- `baselinePeriod` — the periodicity the data was actually collected at (`Yearly`, `Quarterly`, or `Monthly`). **If `baselinePeriod` is coarser than the period you requested** (e.g. yearly data shown monthly), the finer values are pro-rated estimates, not directly entered figures — say so.
- `isFresh` — flag `false` values in your data notes.

---

## Step 4 — Entity Breakdown (one company)

Use `get_metric_entity_breakdown` to split a metric across a company's entities (sites / subsidiaries). Use `list_entities` first if you need the entity IDs.

```
get_metric_entity_breakdown
  metricIds: ["<uuid>", ...]
  year: "<year>"
  period: "yearly"        # or "monthly" / "quarterly" for a per-entity series
  entityIds: ["<uuid>"]   # optional — restrict to specific entities
  companyId: "<uuid>"     # fund managers only
```

Each metric returns `total` (org roll-up across all entities) and `entities[]`, each with `entityId`, `entityName`, `value`, and (for monthly/quarterly) a `periods` series.

---

## Step 5 — Portfolio Breakdown (Fund Manager only)

To break a metric down across the whole portfolio and, within each company, by entity:

```
get_portfolio_metric_entity_breakdown
  metricIds: ["<uuid>", ...]
  year: "<year>"
  period: "yearly"     # or "monthly" / "quarterly"
  fundId: "<uuid>"     # optional — scope to one fund
```

Returns one entry per metric with `companies[]`, each carrying `companyName`, `total`, and an `entities[]` breakdown. Only companies in your accessible portfolio with a calculated value appear. **Only built-in and fund-manager-owned metrics are returned** — portfolio companies' own custom metrics are never included.

---

## Step 6 — Present the Breakdown

- For **time** breakdowns, present a simple period table or describe the trend (peak/trough months, quarter-on-quarter change). Confirm the periods sum to (or reconcile with) `total`.
- For **entity / portfolio** breakdowns, rank entities or companies by value and call out the largest contributors.
- Always reconcile against the yearly `total` and note coverage gaps (`null` values).

```markdown
## [Metric title] — [Year] ([monthly|quarterly|by entity|by company])

| Period / Entity | Value | Unit |
| --------------- | ----- | ---- |
| m1              | 220   | kWh  |
| m2              | 180   | kWh  |
| ...             | ...   | ...  |
| **Total**       | 2,630 | kWh  |

_Baseline periodicity: Monthly. Data is up to date (isFresh: true)._
```

---

## Common Mistakes

- **Treating pro-rated values as reported** — when `baselinePeriod` is coarser than the requested period, the finer numbers are estimates. State this.
- **Dropping null periods** — keep them visible so coverage gaps are obvious; do not omit or interpolate.
- **Expecting a portfolio company's custom metrics** — fund managers only see built-in and fund-manager-owned metrics across the portfolio.
- **Fabricating values** — if a period or entity has no data, report it as missing.

---

## Notes

- `get_metric_breakdown` is org-level only; use `get_metric_entity_breakdown` for per-entity detail of the same metric.
- Fund managers can break a single portfolio company down with `companyId` on `get_metric_breakdown` / `get_metric_entity_breakdown`, or the whole portfolio with `get_portfolio_metric_entity_breakdown`.
- All breakdown tools accept up to 50 metric IDs per call — batch related metrics to reduce round-trips.
- Never fabricate or estimate values beyond what the platform returns. If data is absent, say so.

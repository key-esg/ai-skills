---
name: carbon-data-audit
description: Use when a user wants to audit their carbon data, find gaps or anomalies in GHG activity entries, break down emissions activity by site/entity or period, or verify what sits behind their Scope 1/2/3 numbers before reporting.
compatibility: Requires the KEY ESG MCP server connected as a custom connector or via Claude Desktop config. Any endpoint works — the legacy endpoint (https://api.keyesg.com/mcp), the fund-manager endpoint (https://api.keyesg.com/pe/mcp), or the standalone-company endpoint (https://api.keyesg.com/standalone/mcp). The connector may appear under any name (typically "key-esg" or "KEY ESG") — detect the connection by tool availability (e.g. who_am_i), not by server name or URL.
---

## Step 1 — Establish Context

Call `who_am_i` (no arguments).

- **`organisationType: "company"`** → proceed with the full audit. Use `companyName` in the audit header.
- **`organisationType: "fund_manager"`** → proceed with a **self-assessment audit** of the fund manager organisation's own carbon data. All three carbon data tools (`list_entities`, `list_datapoint_groups`, `list_raw_carbon_entries`) return the FM org's own data. To audit a **portfolio company's** carbon data, the user must run this skill from that company's own account.

Ask the user for:

- **Reporting year** (required, e.g. `2025`). Do not assume the current year — but do not ask either when `who_am_i` returned a single entry in `availableReportingYears`: use that year and say which one you used. Reporting years are calendar years (no fiscal-year offset).
- **Scope focus** (optional) — a single GHG scope (`scope-1`, `scope-2`, `scope-3`) or all scopes.

---

## Step 2 — Map What Should Exist

Build the expected coverage picture:

```
list_entities
```

Returns every entity (site, subsidiary): `entityId`, `name`, `country`, `isMain`, `tags`, plus a `tags` summary listing every tag in use across the organisation with its entity count. Every active entity is expected to have carbon entries.

Entities are flat — there is no parent-child relationship, and tags are the grouping mechanism. When you roll entities up into a group, use a tag from that summary, the `country`, or a grouping the user has named, and say which one you used. Never infer a group from entity names: two similarly named entities are often deliberately separate.

```
list_datapoint_groups
  year: "<year>"
  isCarbon: true
  ghgScope: "<scope>"   # optional — narrows to one scope
  limit: 200            # optional — raise the page size
```

Returns the carbon data point groups the organisation is collecting this year: `dataPointGroupId`, `title`, `ghgScope`. These are the categories data should be arriving under (e.g. stationary combustion, purchased electricity).

The audit's coverage picture is only as good as this list, so it has to be the whole list: follow `pagination.nextCursor` until `hasMore` is false, and check that the groups you hold match `totalMatching` before you call any category silent. Do not pass `includeSections` — the column schema is not needed for an audit and is the bulk of the response.

---

## Step 3 — Pull the Raw Entries

```
list_raw_carbon_entries
  year: "<year>"
  ghgScope: "<scope>"          # optional — repeat per scope when auditing all scopes
  entityIds: ["<uuid>", ...]   # optional — narrow to specific sites
  cursor: "<nextCursor>"       # follow pagination until hasMore is false
```

Each entry: `entityId`, `entityName`, `dataPointGroupId`, `dataPointGroupTitle`, `ghgScope`, `subCategory`, `activityDescription`, `customDescription`, `dataEntryMethod`, `quantity`, `unit`, `period`, `updatedAt`.

`period` is an object: `{ token, label, periodStart, periodEnd }` — e.g. `{ token: "Q2", label: "Q2 2026", periodStart: "2026-04-01", periodEnd: "2026-06-30" }`. Periods are calendar periods of the reporting year (no fiscal-year offset), so use `periodStart` / `periodEnd` to answer date-range questions rather than working the months out yourself, and pass `token` back in the `periods` filter to narrow a follow-up query.

**Always paginate to the end** — follow `pagination.nextCursor` until `hasMore: false`. A partial read produces a false gap analysis. For large datasets, audit scope by scope using the `ghgScope` filter rather than pulling everything at once.

---

## Step 4 — Analyse

### 4a. Coverage gaps

Cross-reference entries against Step 2:

- **Silent entities** — entities from `list_entities` with no entries at all for the year.
- **Silent categories** — data point groups from `list_datapoint_groups` with no entries.
- **Period gaps** — entities or groups that report some periods but not others (e.g. M1–M9 present, M10–M12 missing). Use the `periods` filter to verify a suspected gap.

### 4b. Anomalies

- **Quantity outliers** — within one data point group + unit, entries far outside the typical range across periods or entities (e.g. one month of diesel 10× the others).
- **Unit inconsistencies** — the same data point group reported in mixed units across entities (e.g. kWh at one site, MWh at another). Legitimate but worth confirming.
- **Possible duplicates** — same entity, data point group, activity description, period, and quantity appearing more than once.
- **Null quantities** — entries with `quantity: null` are placeholders without data; count them separately, not as zero.

### 4c. Cross-check the totals

Compare the audited activity data against reported emissions metrics:

**Company accounts:**

```
list_metrics
  year: "<year>"
get_metric_value
  metricIds: ["<uuid>", ...]   # batch up to 50
  year: "<year>"
```

**Fund-manager accounts (self-assessment):**

```
list_portfolio_metrics
  year: "<year>"
get_metric_value
  metricIds: ["<uuid>", ...]   # batch up to 50
  year: "<year>"
```

If a scope total has `hasValue: false` while raw entries exist, the figure may be pending recalculation — flag it. Never recalculate emissions yourself from quantities; entry quantities are activity inputs (litres, kWh), not emissions.

---

## Step 5 — Output

```
## Carbon Data Audit — [Organisation] — [Year]

### Coverage
| Entity | Scope 1 | Scope 2 | Scope 3 | Last entry |
|--------|---------|---------|---------|------------|
| HQ (main) | ✅ M1–M12 | ✅ M1–M12 | ⚠️ Q1–Q2 only | 2025-11-02 |
| Warsaw plant | ⚠️ no entries after Q2 | ✅ | ❌ none | 2025-07-15 |

### Anomalies for review
- [Entity] — [data point group]: [description of outlier/duplicate/unit issue, with period and values]

### Reported totals vs activity data
| Metric | Value | Notes |
|--------|-------|-------|
| GHG Scope 1 | 12,450 tCO2e | consistent with entry coverage |
| GHG Scope 2 | — | raw entries exist but total not yet calculated |

### Recommended actions
1. [Concrete, ordered chase list — who/what/which period]
```

---

## Common Mistakes

- **Stopping at the first page** — `list_raw_carbon_entries` is paginated. An audit on a partial read invents gaps. Always follow `pagination.nextCursor` until `hasMore: false`.
- **Recalculating emissions from quantities** — `quantity` is the activity input (litres, kWh), not tCO2e. Emission factors live inside KEY ESG; use `get_metric_value` for emissions figures.
- **Treating `quantity: null` as zero** — null means no data was entered, not zero consumption.
- **Flagging opted-out items as gaps** — entries for opted-out groups/entities are hidden by default (`hideOptedOutItems: true`). Absence there is intentional, not a gap.

---

## Notes

- `subCategory` groups entries within a scope (e.g. "Stationary", "Mobile", "3.1") — useful for structuring the anomalies section.
- `dataEntryMethod` shows how each figure was produced — a heavy reliance on estimates vs measured data is itself an audit finding worth reporting.
- `activityDescriptionSearch` supports case-insensitive substring search — handy when the user asks about a specific fuel or activity ("show me everything diesel").
- Scope 4 (`scope-4`) holds avoided emissions — exclude it from footprint coverage analysis unless the user asks.

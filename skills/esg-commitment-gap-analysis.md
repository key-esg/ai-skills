---
name: esg-commitment-gap-analysis
description: Use when a user wants to identify ESG commitment gaps, understand which targets are off-track or unmitigated, audit action plan coverage, or prepare a gap analysis for board or programme review.
compatibility: Requires KEY ESG MCP server (https://api.keyesg.com/mcp) connected as a custom connector or via Claude Desktop config.
---

## Step 1 — Load All Targets

Fetch the complete target set with no filters. This gives the full picture.

```
list_targets
```

Returns for each target: `id`, `metricId`, `metricTitle`, `category`, `description`, `targetType`, `currentValue`, `unitOfMeasure`, `baselineYear`, `baselineValue`, `targetYear`, `progressPercent`, `status`.

Status values: `achieved`, `on_track`, `at_risk`, `not_started`, `missed`.

---

## Step 2 — Isolate Problem Targets

From these filter out targets that have status "at_risk" or "missed".

---

## Step 3 — Cross-Reference Action Plans

```
list_action_plans
```

Returns: `id`, `title`, `description`, `category`, `owner`, `status`, `dueDate`. Action plans have no `metricId` — match to targets by `category` and semantic similarity between the action plan `description` and the target `metricTitle`. A target whose `category` has no active action plan is an unmitigated gap.

---

## Step 4 — Check Data Coverage for At-Risk Targets

For each at-risk or missed target, determine whether the status is driven by actual performance or missing data.

Check your tool list to determine session type, then call the appropriate tool:

**`list_portfolio_metrics` available (Fund Manager session):**

```
list_portfolio_metrics
  year: "<targetYear or current year>"
```

**`list_metrics` available (company session):**

```
list_metrics
  year: "<targetYear or current year>"
```

Find the target's `metricId` in the results. If `hasValue: false`, the target is at-risk/not_started due to absent data rather than underperformance — flag this distinctly.

---

## Step 5 — Classify Each Gap

Assign one or more of these gap types to each problem target:

| Gap type             | Condition                                                        |
| -------------------- | ---------------------------------------------------------------- |
| **Missed**           | `status: missed` — target year passed, target not achieved       |
| **At risk**          | `status: at_risk` — current trend falls short of target          |
| **No progress data** | `progressPercent: null` — cannot compute progress                |
| **No action plan**   | No active action plan in the same `category` as the target       |
| **No metric data**   | `hasValue: false` — metric exists but no data to verify progress |

A single target can have multiple gap types (e.g. at-risk AND stale data).

---

## Step 6 — Output

Present a gap table followed by a data freshness caveat.

```
## ESG Commitment Gap Analysis — [Year]

| Metric | Target | Status | Current | Target Value | Action Plan | Gap Types |
|--------|--------|--------|---------|--------------|-------------|-----------|
| GHG Scope 1 | Reduce 50% by 2030 vs 2020 | at_risk | 12,450 tCO2e | 7,500 tCO2e | None | At risk, No action plan |
| Water consumption | Reduce 20% by 2025 | missed | 85,000 m³ | 68,000 m³ | Water efficiency programme | Missed |
| Renewable energy | 100% by 2027 | on_track | 74% | 100% | Solar installation project | — |

## Targets with No Data
The following targets cannot be assessed — no metric data exists for [year]:
- [metricTitle] (target year: [targetYear])

## Data Freshness Note
Target statuses are derived from the latest available metric values inside KEY ESG. If the underlying metric data has not been recalculated recently, status assessments may be outdated.

## Summary
- Total targets: [n]
- Achieved: [n] | On track: [n] | At risk: [n] | Missed: [n] | Not started: [n]
- Targets with no action plan: [n]
- Targets with no metric data: [n]
```

---

## Common Mistakes

- **Treating `at_risk` as missed** — A target is only `missed` once the target year has passed without achievement. `at_risk` means current trend falls short but the deadline hasn't passed.
- **Ignoring data absence as a gap type** — When `hasValue: false`, the at-risk status may reflect missing data rather than underperformance. Always classify these separately.
- **Interpreting `progressPercent: null` as zero progress** — Null progress often means the target is boolean type or baseline data is absent. Check `targetType` before drawing conclusions.

---

## Notes

- Always show the full target set, not just gaps — context helps the user understand the ratio of gaps to total commitments.
- For board reporting, lead with the Summary block.
- `progressPercent: null` often means the target is `boolean` type or baseline data is absent — check `targetType` and `baselineValue` before drawing conclusions.

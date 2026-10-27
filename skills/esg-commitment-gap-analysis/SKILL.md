---
name: esg-commitment-gap-analysis
description: Use when a user wants to identify ESG commitment gaps, understand which targets are off-track or unmitigated, audit action plan coverage, or prepare a gap analysis for board or programme review.
compatibility: Requires the KEY ESG MCP server connected as a custom connector or via Claude Desktop config. Any endpoint works — the legacy endpoint (https://api.keyesg.com/mcp), the fund-manager endpoint (https://api.keyesg.com/pe/mcp), or the standalone-company endpoint (https://api.keyesg.com/standalone/mcp) — pick the one matching your account type. The connector may appear under any name (typically "key-esg" or "KEY ESG") — detect the connection by tool availability (e.g. who_am_i), not by server name or URL.
---

## Step 1 — Establish Context, Then Load All Targets

Call `who_am_i` first (no arguments). This workflow audits the authenticated organisation’s own targets and action plans for both account types. Use `list_metrics` and own-organisation values throughout; never compare these targets with portfolio metrics. `companyName` names the organisation in the output header.

Fetch the complete target set with no filters. This gives the full picture.

```
list_targets
```

Returns for each target: `id`, `metricId`, `metricTitle`, `category`, `description`, `targetType`, `coverage`, `entities`, `currentValue`, `currentValueYear`, `currentValueScope`, `unitOfMeasure`, `baselineYear`, `baselineValue`, `baselineValueYear`, `targetYear`, `progressPercent`, `status`.

`currentValue` is calculated from submitted data for `currentValueYear` at `currentValueScope` (`organisation`, `entity` or `multiple-entities`) — it is never typed in against the target. Quote the year and scope whenever you report a target's progress, and if a value looks wrong for the metric, compare those two rather than suggesting it was entered by hand.

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

Ask which reporting year to check if the user has not specified one. Use the same own-organisation scope for discovery and evidence, including for fund managers:

```
list_metrics
  year: "<confirmed reporting year>"
```

Find the target's `metricId` in the results. If `hasValue: false`, the target is at-risk/not_started due to absent data rather than underperformance — flag this distinctly.

**Locate the missing carbon data (optional):**

When an emissions-related target has no data, pinpoint where collection stalled. The carbon data tools work in both session types — for fund managers they return the FM organisation's own (self-assessment) data, which matches the self-assessment targets this skill audits:

```
list_entities
list_raw_carbon_entries
  year: "<year>"
  ghgScope: "<scope of the target metric>"
```

Compare which entities and periods have entries against the full entity list from `list_entities`. Report the gap concretely (e.g. "No Scope 1 entries for the Warsaw site after Q2") so the user knows exactly what to chase. Follow `pagination.nextCursor` to read all pages.

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

| Metric | Target | Status | Current (year, scope) | Target Value | Action Plan | Gap Types |
|--------|--------|--------|-----------------------|--------------|-------------|-----------|
| GHG Scope 1 | Reduce 50% by 2030 vs 2020 | at_risk | 12,450 tCO2e (2024, organisation) | 7,500 tCO2e | None | At risk, No action plan |
| Water consumption | Reduce 20% by 2025 | missed | 85,000 m³ (2025, organisation) | 68,000 m³ | Water efficiency programme | Missed |
| Renewable energy | 100% by 2027 | on_track | 74% (2024, entity: Warsaw site) | 100% | Solar installation project | — |

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
- **Reporting a current value without its year and scope** — `currentValue` is the latest year with data at or before `targetYear`, at the target's own scope. Quote `currentValueYear` and `currentValueScope` every time, or the reader cannot tell a stale year or an entity-scoped target from a wrong number.
- **Speculating that a surprising value was entered by hand** — Target values are always calculated. When `currentValue` disagrees with the metric the user has in mind, the explanation is `currentValueYear` or `currentValueScope`, not data entry. Name the mismatch; never diagnose it as a typo.

---

## Notes

- Always show the full target set, not just gaps — context helps the user understand the ratio of gaps to total commitments.
- For board reporting, lead with the Summary block.
- `progressPercent: null` often means the target is `boolean` type or baseline data is absent — check `targetType` and `baselineValue` before drawing conclusions.

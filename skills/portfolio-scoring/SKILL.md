---
name: portfolio-scoring-methodology
description: Helps a fund manager build and save their own customized ESG portfolio scoring methodology, then applies it consistently against live KEY ESG MCP data every year after. This skill does NOT impose a fixed KEY ESG scoring standard — it elicits each user's own choices (which dimensions, which metrics, which weights, which scale) and saves the result as the source of truth for that fund, so scores stay comparable year over year instead of silently drifting. Triggers on phrases like "score our portfolio", "build an ESG scoring methodology", "help us design a scoring system", "rank portfolio companies on ESG", "how do our portfolio companies compare on ESG", or any request to weight or aggregate ESG metrics into a score. If a methodology was already defined and saved for this fund, reuses it automatically instead of re-asking or re-deriving.
compatibility: Requires the KEY ESG MCP (Pilot) connector and the verify-esg-account skill
---

# Portfolio Scoring Methodology

Helps a fund manager design their own ESG portfolio scoring methodology through a guided set of questions, then applies that methodology consistently against live KEY ESG data every year after. The methodology is theirs, not a fixed KEY ESG standard — this skill's job is to elicit good decisions, surface tradeoffs they might not think to ask about themselves, and then hold the line on consistency once it's set.

Before producing any output, read the company context reference file:
`../../../context/company-context.md`

---

## Step 0: Verify account and entity scope

Run `verify-esg-account` first if it has not already run earlier in this conversation. Do not proceed until the authenticated account and fund in scope are both confirmed.

---

## Step 1: Check for an existing saved methodology

- Look for a Confluence page titled **"[Fund Name] Portfolio Scoring Methodology"** under the analytics output folder specified in `../../context/team-context.md`.
- If found, fetch it, summarise it back to the user in a few lines, and ask whether to reuse it as-is or revise it.
- If reusing, skip to **Step 6** (apply and present results as normal). Since nothing changed, Step 9's Confluence write is unnecessary unless the user asks to revise via Step 8 — reusing an unmodified methodology doesn't need to re-save it.
- If revising or no methodology exists yet, continue to Step 2.

Never proceed on a methodology only discussed in conversation but never saved. Conversation memory is not the source of truth — the saved Confluence page is.

---

## Step 2: Lay out the data landscape

Before asking any scoring questions, show the user what they're actually working with:

- The portfolio companies in scope (from `list_entities` / `list_portfolio_metrics`)
- What each one currently reports: which metrics are populated, for which years, and any that are entirely absent

This grounds every question that follows in what data actually exists, rather than designing a methodology around a metric nobody has populated yet.

---

## Step 3: Core questions (always asked)

**Unit of scoring — read this before asking anything below.** The methodology (weights, dimensions, rules) is defined once at the fund level and saved as one set of rules. But the *score itself* is always computed per portfolio company — the fund itself is never assigned a single score. Phrase every question accordingly: "what should count toward each portco's Reporting Maturity score," not "toward the fund's Reporting Maturity score." Getting this wrong is an easy mistake to make out loud even when the underlying logic is right — watch for it.

Ask using the interactive multiple-choice interface where options are enumerable. **Batch aggressively: if multiple sub-questions use the same mechanism (e.g. the E/S/G "all vs. specify" pattern), ask all of them together in a single tool call as multiple questions, never one pillar per exchange.** A general instruction to minimize screens is not enough to prevent drifting into one-at-a-time — check explicitly, before sending, whether any other pending question could be asked in the same batch. **Never bundle a required free-text answer (e.g. numeric weights) into the same turn as a button/multiple-choice tool call.** The buttons visually dominate the response and the free-text ask gets silently skipped even when it's asked in the same message — this has happened repeatedly. Weight numbers and other free-text-only questions get their own turn, with nothing else competing for the user's response. This skill does not prescribe answers — every question below is asked of the user, not decided for them.

**Progress indicator.** Before each batch of questions from Step 3 onward, show a short progress line: which batch this is, a rough estimate of how many more remain, and an approximate completion percentage — e.g. "Batch 4 of ~7 · ~55% through methodology setup." Base the initial total on which dimensions were selected in 3a (more dimensions and any overlap/redundancy checks found along the way mean more batches — this is why the total is an estimate, not a fixed count) and revise the estimate honestly as new questions surface mid-flow (e.g. an unexpected overlap check) rather than quietly keeping a stale number. Do not give a false-precision time estimate (e.g. "4.5 minutes") — a qualitative range (e.g. "a few more minutes, depending on how quickly you answer") is honest about what can actually be predicted. Skip the indicator for single stand-alone follow-up questions outside the main batch sequence (e.g. a one-off clarification) — it's meant to track the elicitation phase as a whole, not every individual exchange.

**3a. Which dimensions do you want?** (multi-select, any combination)
- **Reporting Maturity** — how much is being reported, not how well the company performs (entity coverage %, metric population %, years of history, assurance)
- **Qualitative Indicators** — policy/practice presence, tiered as Foundational / Expected / Additional, scored by count-present
- **Objective Performance** — absolute metrics normalized by size, benchmarked against a peer set
- **Year-over-Year Progress** — trend/direction, tracked separately from absolute standing

**3b. What scale?** Settle this before any floor/ceiling or scoring-range questions below — a floor of "20" means something different on a 1–100 scale than a 1–5 scale, so scale must be fixed first.
- 1–100 throughout
- 1–5 band labels (e.g. Foundational / Developing / Leading)
- 1–100 internally, mapped to 1–5 bands for presentation

**3c. For each dimension selected:**

- *Reporting Maturity:* What counts toward it? Common building blocks (any combination): **% of entities reporting** (used in KEY ESG's own fund manager benchmark tool), % of chosen metrics populated, years of reporting history, third-party assurance. Does a low score here **reduce** the Objective Performance score, or sit alongside it as a confidence flag only that doesn't affect the number?
- *Qualitative Indicators:* Use a default indicator tiering, or define your own Foundational/Expected/Additional list? Auto-generate a prioritized action plan from gaps — yes or no?
- *Objective Performance:*
  - Which pillars (E/S/G, any subset) are in scope, and what weight does each carry (must sum to 100%)?
  - For each pillar, which metric(s) represent it? **Users choose from ALL available populated metrics for that pillar — never a curated or "headline" subset.** This is a hard requirement, not a default that can be quietly narrowed for convenience.
    - **Mechanism note:** interactive button/multiple-choice interfaces typically cap out around 4 options per question. Environment in particular will regularly exceed this once Scope 3 sub-categories are included. When the full candidate list exceeds what buttons can hold: print the complete list as plain text in the question itself (grouped clearly, e.g. by GHG scope), then offer a simple 2-option button choice — **"Include all metrics listed"** vs. **"I'll specify which to include/exclude"** — dropping to free text only if the user picks the second option. Do not shrink the printed list to fit the UI; only the *response mechanism* changes, never the list itself. Reserve plain multi-select buttons (no all/specify step needed) for pillars where the full candidate list is short enough to fit directly (commonly Governance) — but verify the actual count first; do not assume a pillar is short.
    - **Overlap/double-counting check — check every layer, not just the first one found:** candidate lists routinely contain the same underlying value more than once. Watch for (a) **parent/child roll-ups**, which can nest more than one level deep (Scope 1+2+3 contains Scope 1+2, which contains Scope 1 and Scope 2; Total Scope 3 contains its individual categories like Purchased Goods, Business Travel, etc. — resolving the outermost layer does not resolve inner layers, check each one), (b) **methodology-duplicate variants**, where the same physical value is reported twice under different accounting conventions (e.g. Scope 2 and Scope 3 each commonly have both a market-based and a location-based figure for the same emissions), and (c) **a raw count alongside its own derived rate or percentage** (e.g. "Total women board members" + "Total board members" + "% women board members" is one fact stated three ways, not three data points; "Work-related injuries" alongside "Injuries per 1,000 FTEs" and "Injuries per 30k hours worked" and LTIFR is the same injury count normalized three different ways). This pattern (c) is common across all three pillars, not just Environment — check Social and Governance candidate lists for it just as carefully. For any overlap found, ask explicitly how to resolve it — keep the raw count only, keep one normalized/derived version only, or keep multiple with near-zero weight on the redundant ones. Do not let "include all" silently double- or triple-count the same value through any of these paths.
    - If more than one metric per pillar, what weight does each carry **within** that pillar (must sum to 100%)?
  - Is a pillar's score a straight weighted average of its metrics, or can one metric act as a gate/cap on the whole pillar regardless of the others?
  - **Normalization basis — ask this as its own separate question, after metric selection, never bundled into it.** Size-normalize by revenue, FTE, both, or not at all (absolute value)? Does this choice vary by metric? Do not offer pre-normalized variants (e.g. "Scope 1 intensity vs FTE") as if they were metric *choices* — select the raw metric first, then ask how (or whether) to normalize it as an independent decision. **Scope this question to the metrics that actually need it.** Raw absolute-value metrics (e.g. total emissions in Kg CO₂e) scale with company size and need normalization to be comparable. Metrics that are already rate-normalized (e.g. LTIFR, injury gravity rate — already per hours worked) or already scale-invariant (e.g. board composition percentages) do not need a second normalization pass, and applying one anyway produces a meaningless unit (e.g. injuries/hour/£). Ask "does this metric need normalizing at all" per metric, not "what should the normalization basis be" as a single blanket setting across every pillar.
  - Peer set: this portfolio only, the whole fund, or KEY ESG's broader benchmark dataset?
  - Floor and/or ceiling values, if any? (Reference the scale fixed in 3b — e.g. "floor of 20" only makes sense once the scale is known to be 1–100.)
- *Year-over-Year Progress:* How should it be handled — tracked as its own visible signal (separate from the absolute score), integrated directly into the composite score itself, or not tracked at all? How should Year 1 (no baseline) be handled — must be explicitly flagged as "no comparison yet," never defaulted to a placeholder score, regardless of which handling option is chosen.

**3d. How should dimensions be presented?**
- Fully separate scorecard (no single number)
- Separate dimensions shown, plus one optional blended headline number
- Fully blended into a single composite (if chosen, ask the weight each dimension carries)

**3e. Should weights or metric relevance vary by company type?**
E.g., a software portco isn't materially scored on Scope 1/2 the same way a logistics company is. If yes, capture the adjustment rule itself (not an ad hoc per-company judgment) so it's applied consistently in future years too.

Do not proceed until every selected dimension has complete answers and all weights sum correctly. Flag specific gaps and re-ask rather than filling with an assumed default.

---

## Step 4: Advanced questions (only if the user opts in)

Ask once, after Step 3: **"Want to configure advanced settings (overrides, grace periods, governance), or use sensible defaults for these?"**

If they opt in, ask:
- **Normalization method:** percentile rank within the peer set, z-score, min-max scaling, or user-defined absolute bands?
- **Red flags/overrides:** does a confirmed severe controversy, compliance breach, or exclusion-list exposure cap or override the computed score regardless of the weighted math?
- **Grace period for newly acquired portcos:** excluded from scoring in year one, scored on a lighter subset (e.g. qualitative indicators only), or scored fully immediately? (Pairs naturally with `key-esg-ma-rebaselining` for funds doing buy-and-build.)
- **Metric substitution/proxies:** is there an approved fallback metric when the chosen one is unavailable for a specific company, or does it fall under the missing-data rule from Step 3b?
- **Governance:** who is authorized to approve a future revision? Do revisions append to a visible history (old weights, what changed, when, why) rather than overwrite?

If the user declines advanced setup, apply sensible defaults — but write those defaults explicitly into the saved methodology page in **Step 9**, labeled as defaults, not as choices the user made. Nothing gets silently assumed.

---

## Step 5: Concentration check (automatic, not a question)

Before applying the methodology to live data, check whether any single metric ends up driving a disproportionate share of the total score once pillar weight × metric weight × structure compound through. If one metric exceeds roughly 40–50% of the effective total, flag this to the user explicitly and ask if that's intended before proceeding.

---

## Step 6: Apply the methodology to live data

Using the verified account/entity context from Step 0 and the methodology just built (or loaded from Step 1):

- Pull each required metric per portfolio company via `get_portfolio_metric_value` / `get_metric_entity_breakdown` for the requested year(s)
- Normalize per the confirmed rules (peer set, method, floor/ceiling, materiality adjustments)
- Compute each selected dimension per the confirmed structure
- Apply any confirmed overrides, grace periods, or proxy rules

**Nothing gets saved to Confluence yet.** This is a trial run against real data, not a commitment.

---

## Step 7: Present results

Show results per the confirmed structure (separate scorecard, hybrid, or blended). Include an assumptions section with confidence levels:

| Assumption | Basis | Confidence |
|---|---|---|
| [e.g. Peer set = portfolio companies only] | Per methodology as built | High |
| [e.g. FY2024 governance gap-filled, not measured] | Data absent for that period | Medium |

**Explicitly flag** anywhere a score moved mainly because data *became available* rather than because of *operational performance*. Never blur that distinction.

---

## Step 8: Confirm or revise before saving anything

**Always ask this before proceeding to Step 9 — never move straight from results to a save.** Seeing the actual computed scores is the real test of whether the methodology behaves as intended; a fund manager may only notice a problem (an unintended metric dominating, a company scoring oddly, a weight that "sounded right" but produces a strange result in practice) once they see it applied to real companies. Ask directly: **"Does this match what you expected, or is there anything you'd like to adjust before this becomes the saved methodology?"**

- If the user wants changes, return to the relevant part of Step 3/4, update the answers, and re-run Steps 5–7 with the revised methodology. Do not save anything until the user confirms the results look right.
- If confirmed as-is, proceed to Step 9.

---

## Step 9: Save the methodology

Write the confirmed methodology to a Confluence page titled **"[Fund Name] Portfolio Scoring Methodology"** (create if new, update in place if revising — never a second page for the same fund). Include:

- Every answer from Steps 3 and 4 (or defaults used, clearly labeled as defaults)
- Date defined or revised, and who confirmed it
- If revising: what changed from the prior version and why, appended to a visible history rather than overwriting it

State clearly to the user that this page is now authoritative, and that future runs of this skill will load it automatically rather than re-asking these questions.

---

## Step 10: Prompt for next steps

End with something like: **"Saved. Want this as a client-facing case study, saved elsewhere as an internal read-out, or anything else?"**

Client-facing output always hands off to `key-esg-client-report` rather than being produced directly here.

---

## Notes for maintainers

- **Sensitive data in scope:** portfolio ESG and financial data. Handle per the Data Sensitivity Policy in `../../context/team-context.md`.
- **External output:** writes to Confluence only (internal). Customer-facing output always routes through `key-esg-client-report`'s human review gate instead.
- **Depends on:** `verify-esg-account`. Run it first, every time.
- **Core design constraint:** this skill elicits a methodology; it never supplies one. If a user asks "what should I choose," offer tradeoffs and real-world precedent (e.g. reporting-maturity-based approaches, indicator-tier approaches) rather than a default recommendation — the point is their own defensible choice, not a KEY ESG house view.
- **Consistency constraint:** treat any "just try different weights this once" request as a Step 3/4 revision (update the saved page, append to history) rather than an undocumented one-off. If the user explicitly wants an exploratory recalculation that should NOT overwrite the saved methodology, label the output as exploratory and do not touch the Confluence page.

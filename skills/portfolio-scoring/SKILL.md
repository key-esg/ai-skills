---
name: portfolio-scoring-methodology
description: Helps a fund manager build and save their own customized ESG portfolio scoring methodology, then applies it consistently against live KEY ESG MCP data every year after. This skill does NOT impose a fixed KEY ESG scoring standard — it elicits each user's own choices (which dimensions, which metrics, which weights, which scale) and saves the result as the source of truth for that fund, so scores stay comparable year over year instead of silently drifting. Triggers on phrases like "score our portfolio", "build an ESG scoring methodology", "help us design a scoring system", "rank portfolio companies on ESG", "how do our portfolio companies compare on ESG", or any request to weight or aggregate ESG metrics into a score. If a methodology was already defined and saved for this fund, reuses it automatically instead of re-asking or re-deriving.
compatibility: Requires the KEY ESG MCP (Pilot) connector
---

# Portfolio Scoring Methodology

Helps a fund manager design their own ESG portfolio scoring methodology through a guided set of questions, then applies that methodology consistently against live KEY ESG data every year after. The methodology is theirs, not a fixed KEY ESG standard — this skill's job is to elicit good decisions, surface tradeoffs they might not think to ask about themselves, and then hold the line on consistency once it's set.

Before producing any output, read the company context reference file: `../../../context/company-context.md`

---

## Step 0: Verify account and entity scope (inlined — no external dependency)

Before pulling any data through the KEY ESG MCP connector, confirm which account is authenticated and which fund/entity is in scope. Do this every time this skill runs, and again immediately after any reconnect — never assume a prior verification in the conversation still holds after a reconnect.

1. Call `who_am_i` first, always, before any other KEY ESG MCP tool call. If the returned organisation doesn't match what the user described or expected, **stop** and tell them which account is actually authenticated (switching requires reconnecting via Settings → Integrations).
2. Resolve the fund/entity explicitly via `list_funds` / `list_entities` — never fabricate a `fundId` or `companyId`. If ambiguous or unresolvable, stop and ask rather than guessing.
3. Show a short verification report before proceeding:
   ```
   Confirmed account: [organisation name]
   Fund in scope: [name] ([id])
   Portfolio companies: [list]
   Reporting period: [year(s)]
   ```
   If anything in steps 1–2 was ambiguous or needed a fallback, say so here explicitly rather than silently proceeding.
4. Treat any metric that comes back absent as **not reported**, never coerced to zero — this matters throughout the rest of this skill (missing data must never be silently zeroed in any downstream calculation).

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

Ask using the interactive multiple-choice interface where options are enumerable. **Batch aggressively: if multiple sub-questions use the same mechanism, ask all of them together in a single tool call as multiple questions, never one pillar per exchange.** A general instruction to minimize screens is not enough to prevent drifting into one-at-a-time — check explicitly, before sending, whether any other pending question could be asked in the same batch. **Never bundle a required free-text answer (e.g. custom numeric weights) into the same turn as a button/multiple-choice tool call.** The buttons visually dominate the response and the free-text ask gets silently skipped even when it's asked in the same message — this has happened repeatedly. Free-text-only questions get their own turn, with nothing else competing for the user's response. This skill does not prescribe answers — every question below is asked of the user, not decided for them.

**Progress indicator.** Before each batch of questions from Step 3 onward, show a short progress line: which batch this is, a rough estimate of how many more remain, and an approximate completion percentage — e.g. **"Batch of questions 4 of ~7 · roughly 55% completion."** Base the initial total on which dimensions were selected (more dimensions and any overlap/redundancy checks found along the way mean more batches — this is why the total is an estimate, not a fixed count) and revise the estimate honestly as new questions surface mid-flow (e.g. an unexpected overlap check) rather than quietly keeping a stale number. Do not give a false-precision time estimate (e.g. "4.5 minutes") — a qualitative range is honest about what can actually be predicted. Skip the indicator for single stand-alone follow-up questions outside the main batch sequence — it's meant to track the elicitation phase as a whole, not every individual exchange.

**Weight questions — always offer a shortcut first.** For any question asking how weight should be split across multiple items (pillar weights, in-pillar metric weights), do not go straight to free text. Offer two buttons first: **"Equal weight"** (auto-splits evenly across however many items are in scope) or **"Specify custom breakdown"** — only if the user picks the second option does it drop to a free-text numeric turn (still its own turn, per the rule above). This removes typing burden for the common case where equal weighting is what's wanted.

**3a. Pillar scope, asked first, before anything else in Step 3.** Ask **"Which ESG pillars should Objective Performance cover?"** (Environment / Social / Governance / all three) as the very first question of the elicitation flow, before dimension selection, before scale, before Reporting Maturity. Ask this regardless of whether Objective Performance ends up being a selected dimension — if it isn't selected in 3b, this answer is simply discarded. That's an acceptable, low-cost outcome since it's one lightweight question, not the full weight-setting effort.

Do **not** bundle the "straight weighted average vs. gate/cap" structural question into this first batch — that question is more substantial and should only be asked once Objective Performance is confirmed as a selected dimension in 3b. Ask it together with pillar weights once that's confirmed (see 3c).

**3b. Which dimensions do you want?** (multi-select, any combination)

- **Reporting Maturity** — how much is being reported, not how well the company performs (entity coverage %, metric population %, years of history, reporting-method variety)
- **Objective Performance** — absolute metrics normalized by size, benchmarked against a peer set
- **Year-over-Year Progress** — trend/direction, tracked separately from absolute standing

**What scale?** Ask alongside dimensions. Settle this before any floor/ceiling or scoring-range questions later — a floor of "20" means something different on a 1–100 scale than a 1–5 scale, so scale must be fixed first.

- 1–100 throughout
- 1–5 band labels (e.g. Foundational / Developing / Leading)
- 1–100 internally, mapped to 1–5 bands for presentation

**3c. If Objective Performance was selected in 3b:** now ask the gate/cap structural question, paired with pillar weights (using the equal/custom-breakdown button pattern from above):

- Is a pillar's score a straight weighted average of its metrics, or can one metric act as a gate/cap on the whole pillar regardless of the others?
- What weight does each pillar (from the 3a answer) carry — must sum to 100%?

**3d. For each other dimension selected:**

- *Reporting Maturity:* What counts toward it? Common building blocks (any combination): **% of entities reporting** (used in KEY ESG's own fund manager benchmark tool), % of chosen metrics populated, years of reporting history, **variety of reporting methods (i.e. activity-based vs spend-based vs direct entry)**. Does a low score here **reduce** the Objective Performance score, or sit alongside it as a confidence flag only that doesn't affect the number?
- *Year-over-Year Progress:* How should it be handled — tracked as its own visible signal (separate from the absolute score), integrated directly into the composite score itself, or not tracked at all? How should Year 1 (no baseline) be handled — must be explicitly flagged as "no comparison yet," never defaulted to a placeholder score, regardless of which handling option is chosen.

**3e. How should dimensions be presented?**

- Fully separate scorecard (no single number)
- Separate dimensions shown, plus one optional blended headline number
- Fully blended into a single composite (if chosen, ask the weight each dimension carries)

**3f. If Objective Performance was selected — metric selection per pillar:**

For each pillar in scope, which metric(s) represent it? **Users choose from ALL available populated metrics for that pillar — never a curated or "headline" subset.** This is a hard requirement, not a default that can be quietly narrowed for convenience.

- **Mechanism note:** interactive button/multiple-choice interfaces typically cap out around 4 options per question. Environment in particular will regularly exceed this once Scope 3 sub-categories are included. When the full candidate list exceeds what buttons can hold: print the complete list as plain text in the question itself (grouped clearly, e.g. by GHG scope), then offer a simple 2-option button choice — **"Include all metrics listed"** vs. **"I'll specify which to include/exclude"** — dropping to free text only if the user picks the second option. Do not shrink the printed list to fit the UI; only the *response mechanism* changes, never the list itself.
- **Overlap/double-counting check — ask ONE overarching question per pillar, not one per family.** Candidate lists routinely contain the same underlying value more than once: (a) **parent/child roll-ups**, which can nest more than one level deep (Scope 1+2+3 contains Scope 1+2, which contains Scope 1 and Scope 2; Total Scope 3 contains its individual categories), (b) **methodology-duplicate variants**, where the same physical value is reported twice under different accounting conventions (e.g. Scope 2 and Scope 3 each commonly have both a market-based and a location-based figure), and (c) **a raw count alongside its own derived rate or percentage** — this pattern is common across all three pillars, not just Environment. For pattern (c) specifically: rather than asking a separate single-select question per family (e.g. one for injuries, one for injury severity, one for fatalities), consolidate into **one overarching question per pillar**: "This pillar has N metric families, each with a raw count and a normalized rate/percentage version — which form should count across all of them?" with the two options being the raw-count set and the normalized-rate/percentage set, each listing the specific metrics that fall under it. This cuts the number of questions the user has to answer without losing the resolution. For pattern (a)/(b), roll-up level and accounting convention still need their own explicit questions (they aren't a raw/rate choice), but batch them together with pattern (c)'s single question wherever they apply to the same pillar. When asking the accounting-convention question, include a third option alongside "Market-based only" and "Location-based only": **"Both market-based and location-based"** (keeping both as separate metrics, each carrying its own weight).
- If more than one metric per pillar, ask what weight each carries **within** that pillar — using the equal/custom-breakdown button pattern.

**3g. Normalization basis — ask this as its own separate question, after metric selection, never bundled into it.** Size-normalize by revenue, FTE, both, or not at all (absolute value)? Does this choice vary by metric? Do not offer pre-normalized variants (e.g. "Scope 1 intensity vs FTE") as if they were metric *choices* — select the raw metric first, then ask how (or whether) to normalize it as an independent decision. **Scope this question to the metrics that actually need it.** Raw absolute-value metrics (e.g. total emissions in Kg CO₂e) scale with company size and need normalization to be comparable. Metrics that are already rate-normalized (e.g. injury gravity rate — already per hours worked) or already scale-invariant (e.g. board composition percentages) do not need a second normalization pass, and applying one anyway produces a meaningless unit. If "both revenue and FTE" is chosen, note explicitly to the user (and later in the saved methodology) how the two normalized values are combined into one score — this is not self-evident and should never be silently assumed without flagging it as an assumption.

**3h. Peer set:** ask with only two options — **"A select set of the whole fund's portfolio"** or **"The whole fund."** (Do not offer a "KEY ESG's broader benchmark dataset" option — this has been removed.)

**3i. Floor and/or ceiling values, if any?** Always include a concrete example when asking this, so the user understands what a floor/ceiling means in practice before answering — e.g. *"For example, you might want the lowest possible score to be 20 rather than 0 (so no company looks like a total failure), or cap the highest score below 100 (so no one looks 'done'). Do you want either of those?"* Reference the scale fixed earlier (e.g. "floor of 20" only makes sense once the scale is known to be 1–100).

**3j. Should weights or metric relevance vary by company type?** Always include a concrete example when asking this too — e.g. *"A software company might reasonably be scored less on Scope 1/2 emissions (minimal direct operational footprint) while a logistics or manufacturing company would be scored heavily on it, since that's more central to their impact. Do you want a rule like that?"* If yes, capture the adjustment rule itself (not an ad hoc per-company judgment) so it's applied consistently in future years too. If the current portfolio is homogeneous by whatever classification is chosen (e.g. all one sector), flag this explicitly to the user — the rule may be entirely inert for the current portfolio, and it's worth confirming whether they still want it captured forward-looking, or want to skip it and revisit only if/when the portfolio diversifies.

Do not proceed until every selected dimension has complete answers and all weights sum correctly. Flag specific gaps and re-ask rather than filling with an assumed default.

---

## Step 4: Advanced questions (only if the user opts in)

Ask once, after Step 3: **"Want to configure advanced settings (overrides, grace periods, governance), or use sensible defaults for these?"**

If they opt in, ask:

- **Normalization method:** percentile rank within the peer set, z-score, min-max scaling, or user-defined absolute bands?
- **Red flags/overrides:** does a confirmed severe controversy, compliance breach, or exclusion-list exposure cap or override the computed score regardless of the weighted math?
- **Grace period for newly acquired portcos:** excluded from scoring in year one, scored on a lighter subset (e.g. Reporting Maturity only), or scored fully immediately? (Pairs naturally with `key-esg-ma-rebaselining` for funds doing buy-and-build.)
- **Metric substitution/proxies:** is there an approved fallback metric when the chosen one is unavailable for a specific company, or does it fall under the missing-data rule?
- **Governance:** who is authorized to approve a future revision? Do revisions append to a visible history (old weights, what changed, when, why) rather than overwrite?

If the user declines advanced setup, apply sensible defaults — but write those defaults explicitly into the saved methodology page in **Step 9**, labeled as defaults, not as choices the user made. Nothing gets silently assumed.

---

## Step 5: Concentration check (automatic, not a question)

Before applying the methodology to live data, check whether any single metric ends up driving a disproportionate share of the total score once pillar weight × metric weight × structure compound through. If one metric exceeds roughly 40–50% of the effective total, flag this to the user explicitly and ask if that's intended before proceeding.

---

## Step 6: Confirmation Gate (a) — confirm the methodology BEFORE calculating anything

**This is a distinct, mandatory gate — do not skip or merge it with the results confirmation in Step 8.** Before running any numbers against live data, summarize the complete methodology exactly as confirmed through Steps 3–5: dimensions, scale, all weights (pillar and in-pillar), metric selections and how overlaps were resolved, normalization basis, peer set, floor/ceiling, materiality adjustment (if any), and defaults applied in Step 4. Ask directly: **"Does this methodology look right to calculate against live data?"**

- If the user wants changes, return to the relevant part of Step 3/4 and re-summarize before asking again.
- Only once confirmed, proceed to Step 7.

---

## Step 7: Apply the methodology to live data

Using the verified account/entity context from Step 0 and the methodology just confirmed (or loaded from Step 1):

- Pull each required metric per portfolio company via `get_portfolio_metric_value` / `get_metric_entity_breakdown` for the requested year(s)
- Normalize per the confirmed rules (peer set, method, floor/ceiling, materiality adjustments)
- Compute each selected dimension per the confirmed structure
- Apply any confirmed overrides, grace periods, or proxy rules

**Nothing gets saved to Confluence yet.** This is a trial run against real data, not a commitment.

---

## Step 8: Present results, then Confirmation Gate (b)

Show results per the confirmed structure (separate scorecard, hybrid, or blended). Include an assumptions section with confidence levels:

| Assumption                                        | Basis                       | Confidence |
| ------------------------------------------------- | --------------------------- | ---------- |
| [e.g. Peer set = portfolio companies only]        | Per methodology as built    | High       |
| [e.g. FY2024 governance gap-filled, not measured] | Data absent for that period | Medium     |

**Explicitly flag** anywhere a score moved mainly because data *became available* rather than because of *operational performance*. Never blur that distinction.

**Then ask — this is Confirmation Gate (b), distinct from Gate (a) above and never collapsed into it:** **"Does this match what you expected, or is there anything you'd like to adjust before this becomes the saved methodology?"** Seeing the actual computed scores is a real test of whether the methodology behaves as intended — a fund manager may only notice a problem (an unintended metric dominating, a company scoring oddly) once they see it applied to real companies, even after confirming the methodology description itself in Gate (a).

- If the user wants changes, return to the relevant part of Step 3/4, update the answers, and re-run Steps 5–8 with the revised methodology. Do not save anything until the user confirms the results look right.
- If confirmed as-is, proceed to Step 9.

---

## Step 9: Confirmation Gate (c) — save the methodology

Only after both Gate (a) and Gate (b) are confirmed: write the confirmed methodology to a Confluence page titled **"[Fund Name] Portfolio Scoring Methodology"** (create if new, update in place if revising — never a second page for the same fund). Include:

- Every answer from Steps 3 and 4 (or defaults used, clearly labeled as defaults)
- Date defined or revised, and who confirmed it
- If revising: what changed from the prior version and why, appended to a visible history rather than overwriting it

State clearly to the user that this page is now authoritative, and that future runs of this skill will load it automatically rather than re-asking these questions.

If the user declines to actually persist the page (e.g. this was a test run), treat that as a normal, valid outcome — not an error — and don't retry or push back on the decision.

---

## Step 10: Prompt for next steps

End with something like: **"Saved. Want this as a client-facing case study, saved elsewhere as an internal read-out, or anything else?"**

Client-facing output always hands off to `key-esg-client-report` rather than being produced directly here.

---

## Notes for maintainers

- **Sensitive data in scope:** portfolio ESG and financial data. Handle per the Data Sensitivity Policy in `../../context/team-context.md`.
- **External output:** writes to Confluence only (internal). Customer-facing output always routes through `key-esg-client-report`'s human review gate instead.
- **No external dependency:** account/entity verification is now inlined in Step 0 — this skill no longer depends on a separate `verify-esg-account` skill.
- **Core design constraint:** this skill elicits a methodology; it never supplies one. If a user asks "what should I choose," offer tradeoffs and real-world precedent (e.g. reporting-maturity-based approaches, indicator-tier approaches) rather than a default recommendation — the point is their own defensible choice, not a KEY ESG house view.
- **Consistency constraint:** treat any "just try different weights this once" request as a Step 3/4 revision (update the saved page, append to history) rather than an undocumented one-off. If the user explicitly wants an exploratory recalculation that should NOT overwrite the saved methodology, label the output as exploratory and do not touch the Confluence page.
- **Three-gate discipline:** Gate (a) (methodology), Gate (b) (results), and Gate (c) (persistence) are each independent checkpoints. Live testing surfaced that collapsing (a) and (b) into a single confirmation misses real problems — a methodology can look right in the abstract and still produce an unexpected result once run against actual portfolio data.

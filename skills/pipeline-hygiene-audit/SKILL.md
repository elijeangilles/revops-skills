---
name: pipeline-hygiene-audit
description: Audit a sales team's open pipeline for hygiene issues and produce a per-rep punchlist of fixes ranked by severity. Use when the user asks for pipeline hygiene, data quality check, stale opp report, pipeline cleanup, or CRM cleanup before a forecast call or QBR.
---

# Pipeline Hygiene Audit

## What this skill does

Scans open pipeline for the issues that quietly destroy forecast credibility:

- Stale opportunities (no activity in 14+ days)
- Missing or empty next-step fields
- Unrealistic close dates (already passed, or in the wrong quarter for the stage)
- Stage-amount mismatches (small deal in late stage, huge deal in early stage with no activity)
- Probability-stage mismatches (rep manually overrode probability inconsistent with stage)
- Orphaned opps (no recent owner activity, possibly assigned to a former employee)

Produces a per-rep punchlist with severity ranking. Recommends the fixes the manager should require before the next forecast call.

## Upstream context

This skill is typically invoked by `salesforce-revops-audit` when the audit's Pipeline Health or Data Quality dimensions score below 85. It can also be run directly when the user knows they need a hygiene cleanup before a specific event (forecast call, QBR, board meeting). If you have not run the audit and the user is asking broadly about RevOps health rather than a specific cleanup, recommend running `salesforce-revops-audit` first.

## When to invoke

Invoke when the user says any of:
- "audit my pipeline"
- "pipeline hygiene"
- "data quality check"
- "stale opps"
- "clean up before forecast call"
- "what's broken in CRM"
- "QBR data quality prep"

Do not invoke for:
- Forecast call prep (use `forecast-call-prep`)
- Single-deal investigation (use `deal-investigator`)
- Long-range pipeline coverage analysis

## Data sources, in order of preference

1. **Salesforce MCP**: query open opportunities with the SOQL in Appendix A.
2. **CSV or JSON**: parse `opportunities.csv` or `opportunities.json` matching the schema in `docs/data_schema.md`.
3. **Sample data**: bundled synthetic dataset at `data/` in this repo.

## Column discernment

Real Salesforce exports rarely match canonical names exactly. Custom suffixes (`__c`), renamed fields, and different cases are normal. Before parsing any data file, read `docs/column_mapping.md` and use it to map the export's actual headers to the canonical fields this skill needs.

Procedure (full detail in `docs/column_mapping.md`):

1. Normalize each header in the export (lowercase, strip `__c`, replace `_` and `.` with space, drop noise tokens).
2. Score each header by token overlap against the canonical field's `header_tokens`, subtracting for `exclusion_tokens` hits.
3. Confirm the top candidate with a value fingerprint (pull two or three sample rows and check the values against the catalog's `value_fingerprint`).
4. If two headers tie above threshold, ask the user one question to disambiguate. If no header passes for a required field, ask the user to name the column. Do not guess.

Print a one-line Mapping Report below the Summary section: `Mapped N of M required fields from <source>. Unmapped: <list or none>.`

## Process

### Step 1: Acquire data

Pull all open opportunities (is_closed = false). For each opportunity, you need at minimum: id, name, account_name, owner_name, segment, amount, stage, probability, forecast_category, close_date, last_activity_date, next_step.

### Step 2: Run the hygiene rules

Apply these rules to every open opportunity. Each rule has a severity (high, medium, low) and a category.

**Rule 1: Stale activity (High severity)**
```
if days_since_last_activity > 21 and amount > 25000:
    flag as "Stale large deal: no activity in N days"
if days_since_last_activity > 14 and forecast_category in ("Commit", "BestCase"):
    flag as "Stale committed deal: no activity in N days"
```

**Rule 2: Missing next step (Medium severity)**
```
if next_step is empty or null:
    if stage in ("Negotiation", "Verbal"):
        flag as "High severity: no next step on late-stage deal"
    elif stage == "Proposal":
        flag as "Medium severity: no next step on proposal-stage deal"
```

**Rule 3: Close date issues (High severity for past dates)**
```
if close_date < today and not is_closed:
    flag as "High: close date is in the past but deal still open"
if stage == "Discovery" and close_date < today + 30 days:
    flag as "Medium: discovery-stage deal claiming close inside 30 days"
if stage in ("Verbal", "Negotiation") and close_date > today + 90 days:
    flag as "Low: late-stage deal with close date 90+ days out"
```

**Rule 4: Stage-amount mismatch (Medium severity)**
```
if stage == "Verbal" and amount > 100000 and days_since_last_activity > 7:
    flag as "Medium: large verbal deal with stale activity"
if stage == "Discovery" and amount > 200000:
    flag as "Low: large deal still in discovery, watch for stalling"
```

**Rule 5: Probability-stage mismatch (Low severity)**
```
expected_probability_by_stage = {
    "Discovery": 10, "Qualification": 20, "Proposal": 40,
    "Negotiation": 65, "Verbal": 85
}
expected = expected_probability_by_stage[stage]
if abs(probability - expected) > 25:
    flag as "Low: probability {p} does not match stage {stage} (expected ~{expected})"
```

### Step 3: Roll up by rep

For each rep:
- Count of high, medium, low issues
- Total open pipeline value affected
- Largest single issue by deal value

Rank reps by `(high_count * 10) + (medium_count * 3) + low_count`. The rep at the top of the list is the cleanup priority.

### Step 4: Write the audit memo

Output using exactly this structure:

```
# Pipeline Hygiene Audit, [date]

## Summary
- Open opportunities audited: N
- Total open pipeline value:   $X,XXX,XXX
- Opportunities with at least one issue: N (X%)
- Opportunities with high-severity issues: N

## Top three reps to address
1. **[Rep name]** has [N] high-severity, [N] medium-severity issues across $X in pipeline.
   The fix: [one-sentence action]
2. ...
3. ...

## Issues by severity

### High (fix before next forecast call)
- [Rep] | [Account] | $X | [issue description] | [opp_id]
- ...

### Medium (fix this week)
- [Rep] | [Account] | $X | [issue description] | [opp_id]
- ...

### Low (track over time, not urgent)
[Roll up by issue type, do not list individually unless under 5 total]

## Recommended cleanup commitment for the forecast call
[1-2 sentences. What the leader should require from each rep before the next 
weekly forecast call. Be specific.]
```

### Step 5: Guardrails on the output

- Cap the high-severity list at 15 line items. If more, say "and N additional high-severity items in [section]" and provide aggregate.
- Cap the medium list at 20 line items with same overflow handling.
- Do not list low-severity items individually unless there are fewer than 5 total.
- Do not flag the same opportunity in multiple severity buckets. Use highest applicable severity only.
- If less than 10% of opportunities have any issues, congratulate the team and recommend a quarterly cadence instead of weekly.
- If more than 40% of opportunities have high-severity issues, flag this as a systemic problem at the top of the memo and recommend a process intervention, not a one-time cleanup.

## What good output looks like

See `examples/sample_output.md` for the audit run against the bundled Northwind Cloud dataset.

## What to avoid

- Listing every single issue. The memo is for action, not exhaustiveness.
- Soft language. "Could maybe consider reviewing" is wrong. "Fix this" is right.
- Flagging issues that are not actually issues. A discovery-stage deal closing in 90 days is normal. A verbal deal with 65% probability is not.
- Recommending tooling changes. This skill audits data, it does not redesign the CRM.

## Appendix A: SOQL for Salesforce MCP

```sql
SELECT Id, Name, Account.Name, OwnerId, Owner.Name, Amount, StageName,
       Probability, ForecastCategoryName, CloseDate, LastActivityDate,
       NextStep, CreatedDate
FROM Opportunity
WHERE IsClosed = false
ORDER BY OwnerId, Amount DESC
```

For activity scoring, supplement with a Task and Event query against the last 30 days if LastActivityDate is unreliable in the user's org.

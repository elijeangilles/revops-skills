---
name: forecast-call-prep
description: Prepare the executive summary for a weekly forecast call by reconciling rep commits against historical close-rate models and surfacing material variance. Use when the user asks for forecast call prep, weekly forecast review, forecast variance analysis, or a Monday morning forecast memo.
---

# Forecast Call Preparation

## What this skill does

Produces a one-page executive summary for a weekly forecast call. Reconciles three views of the quarter:

1. Rep commit (what the reps say they will close)
2. Pipeline-weighted forecast (sum of open opportunity amount times stage probability)
3. Historical model forecast (rep commit adjusted by that rep's calibration over the last 12 weeks)

Surfaces the deltas that materially move the quarter. Names the deals worth discussing on the call. Recommends a position for the leader to take into the meeting.

## Upstream context

This skill is typically invoked by `salesforce-revops-audit` when the audit's Forecast Hygiene dimension scores below 85. It can also be run directly when the user knows they need a forecast call memo. If you have not run the audit and the user is asking broadly about RevOps health rather than a specific forecast call, recommend running `salesforce-revops-audit` first.

## When to invoke

Invoke when the user says any of:
- "prep me for the forecast call"
- "weekly forecast variance"
- "forecast call memo"
- "where are we vs commit"
- "what's at risk this quarter"
- "Monday morning forecast prep"

Do not invoke for:
- Long-range planning (next quarter, next year)
- Comp plan analysis
- Individual deal forensics (use `deal-investigator` instead)
- Pipeline hygiene scanning (use `pipeline-hygiene-audit` instead)

## Data sources, in order of preference

The skill works against any of these sources. Pick the first one available:

1. **Salesforce MCP**: if a Salesforce MCP server is connected, query opportunities with the SOQL in Appendix A and forecast history from the `Forecast__c` custom object or equivalent.

2. **CSV files**: if the user has `opportunities.csv` and `forecast_history.csv` (matching the schema in `examples/data_schema.md`), parse those directly.

3. **Sample data**: if neither of the above, use the bundled synthetic dataset at `data/` in this repo. Tell the user you are using sample data and what they would need to provide for a real run.

## Column discernment

Real Salesforce exports rarely match canonical names exactly. Custom suffixes (`__c`), renamed fields, and different cases are normal. Before parsing any data file, read `docs/column_mapping.md` and use it to map the export's actual headers to the canonical fields this skill needs.

Procedure (full detail in `docs/column_mapping.md`):

1. Normalize each header in the export (lowercase, strip `__c`, replace `_` and `.` with space, drop noise tokens).
2. Score each header by token overlap against the canonical field's `header_tokens`, subtracting for `exclusion_tokens` hits.
3. Confirm the top candidate with a value fingerprint (pull two or three sample rows and check the values against the catalog's `value_fingerprint`).
4. If two headers tie above threshold, ask the user one question to disambiguate. If no header passes for a required field, ask the user to name the column. Do not guess.

Print a one-line Mapping Report under the memo's "The number" section: `Mapped N of M required fields from <source>. Unmapped: <list or none>.`

## Process

### Step 1: Acquire data

Get three things:
- All open opportunities with close date on or before the current quarter end
- Each rep's last 12 weeks of forecast submissions (commit, best case, pipeline)
- Each rep's quarter-to-date closed-won total

### Step 2: Compute three forecast views

For each rep, calculate:

**Rep commit (qtd + open commit)**
```
rep_commit = sum(closed_won_qtd) + sum(open_opps where forecast_category == "Commit").amount
```

**Pipeline-weighted forecast**
```
weighted = sum(closed_won_qtd) + sum(open_opps where !is_closed)(amount * probability / 100)
```

**Historical model forecast**
Compute each rep's calibration ratio over the last 12 weeks:
```
calibration_ratio = average(actual_closed_won / commit_amount) for weeks where commit > 0
```
Then:
```
model_forecast = sum(closed_won_qtd) + (open_commit_amount * calibration_ratio)
```

If a rep has less than 8 weeks of history, flag them as "insufficient data" and use 1.0 as their ratio with a confidence warning.

### Step 3: Identify material variance

A rep's forecast has material variance if any of these are true:
- Delta between rep commit and model forecast exceeds 15%
- Rep commit exceeds best case from prior week by more than 20% (sandbagging release)
- Rep commit dropped from prior week by more than 20% (commit pull)
- Pipeline-weighted forecast is less than 70% of rep commit (thin coverage)

For each variance, identify the top 2-3 opportunities driving it.

### Step 4: Write the memo

Output the memo using exactly this structure:

```
# Forecast Call Memo, Week of [date]

## The number
- Team commit:           $X,XXX,XXX  ([change] vs last week)
- Pipeline-weighted:     $X,XXX,XXX
- Historical model:      $X,XXX,XXX
- Confidence:            [High | Medium | Low] based on [reason]

## What to discuss on the call

### Risks (forecast is at risk of missing)
1. **[Rep name]**: [one sentence on what's off]. Top deals to pressure-test: [opp name 1, opp name 2].
2. ...

### Upsides (forecast may be conservative)
1. **[Rep name]**: [one sentence]. Top deals worth pulling forward: [opp name 1, opp name 2].
2. ...

### Coverage concerns
1. **[Rep name]** has pipeline-weighted of $X vs commit of $Y. Coverage ratio [N]x. 
   This is the deal-by-deal review needed.

## Recommended position for the call
[2-3 sentence recommendation for the leader. Should this team be calling commit, 
calling best case, or signaling risk to the CRO. Be direct.]

## Appendix: rep calibration over 12 weeks
[Table showing each rep's average commit accuracy, with their profile if known: 
"runs hot," "runs cold," "calibrated," "wildcard"]
```

### Step 5: Guardrails on the output

- Do not list every flagged opp. Cap at 3 risks, 3 upsides, 3 coverage concerns.
- Do not editorialize beyond what the data supports. If a rep looks bad, say so factually. Do not soften.
- Do not recommend specific actions for individual reps. That is the manager's job.
- If data quality is poor (more than 30% of open opps have stale activity or missing fields), flag this at the top of the memo and recommend `pipeline-hygiene-audit` before the next forecast call.

## What good output looks like

See `examples/sample_output.md` for a worked example using the bundled synthetic data.

## What to avoid

- Padding the memo with executive-speak. The CRO can read.
- Hedging recommendations. Take a position.
- Listing every deal. Surface the material ones.
- Treating the historical model as ground truth. It is one of three views, not the answer.
- Recommending the user do `pipeline-hygiene-audit` if data quality is fine. Only flag this if data quality is genuinely poor.

## Appendix A: SOQL for Salesforce MCP

If using the Salesforce MCP, the queries are:

```sql
-- Open opportunities in current quarter
SELECT Id, Name, Account.Name, OwnerId, Owner.Name, Amount, StageName,
       Probability, ForecastCategoryName, CloseDate, LastActivityDate,
       NextStep, CreatedDate, IsClosed, IsWon
FROM Opportunity
WHERE IsClosed = false
  AND CloseDate <= LAST_DAY_OF_QUARTER
ORDER BY OwnerId, Amount DESC

-- Closed-won this quarter
SELECT OwnerId, Owner.Name, SUM(Amount) total
FROM Opportunity
WHERE IsWon = true AND CloseDate = THIS_QUARTER
GROUP BY OwnerId, Owner.Name

-- Forecast history (if Forecast__c object exists, or equivalent)
SELECT Rep__c, Week_Of__c, Commit_Amount__c, BestCase_Amount__c, Pipeline_Amount__c
FROM Forecast__c
WHERE Week_Of__c = LAST_N_WEEKS:12
ORDER BY Rep__c, Week_Of__c DESC
```

If the Forecast__c object does not exist in the user's org, ask the user where forecast history is stored or fall back to using current data only with a confidence warning.

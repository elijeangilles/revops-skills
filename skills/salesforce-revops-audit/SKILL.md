---
name: salesforce-revops-audit
description: Run a comprehensive Salesforce RevOps health audit across five dimensions (pipeline health, data quality, forecast hygiene, deal integrity, process integrity) and produce a graded report (A through F) with a prioritized remediation queue. Use as the entry point for any new Salesforce org assessment, before a new RevOps leader's first forecast call, before a QBR cycle, or quarterly as a recurring health check. The audit recommends which remediation skills to run next.
---

# Salesforce RevOps Audit

## What this skill does

Diagnoses a Salesforce org's RevOps health across five dimensions, grades each one A through F, and outputs a prioritized remediation queue mapped to the specific skills in this pack that fix each issue.

Designed as the entry point for the rest of the pack. Most users should run this audit first, then execute the recommended remediation skills in the order the audit specifies.

This is the skill a Director of Revenue Operations would run on day one of a new role, or quarterly as a recurring health check, or before any major sales planning cycle.

## When to invoke

Invoke when the user says any of:
- "run the audit"
- "audit my Salesforce org"
- "RevOps health check"
- "where do we stand on Salesforce"
- "new role, where do I start"
- "quarterly health check"
- "is our pipeline in good shape"
- "assess the state of our RevOps"

Do not invoke for:
- Single-deal investigation (use `deal-investigator`)
- Targeted hygiene cleanup (use `pipeline-hygiene-audit`)
- Forecast call preparation (use `forecast-call-prep`)
- Anything narrower than a full-org assessment

If the user wants something narrower, recommend the specific remediation skill and skip the audit.

## Data sources, in order of preference

1. **Salesforce MCP**: query opportunities, accounts, and forecast history with the SOQL in Appendix A.
2. **CSV or JSON**: parse `opportunities.csv`, `forecast_history.csv`, and `quarterly_actuals.json` matching the schema in `docs/data_schema.md`.
3. **Sample data**: bundled synthetic dataset at `data/` in this repo.

## Column discernment

Real Salesforce exports rarely match canonical names exactly. Custom suffixes (`__c`), renamed fields, and different cases are normal. Before parsing any data file, read `docs/column_mapping.md` and use it to map the export's actual headers to the canonical fields this skill needs.

Procedure (full detail in `docs/column_mapping.md`):

1. Normalize each header in the export (lowercase, strip `__c`, replace `_` and `.` with space, drop noise tokens).
2. Score each header by token overlap against the canonical field's `header_tokens`, subtracting for `exclusion_tokens` hits.
3. Confirm the top candidate with a value fingerprint (pull two or three sample rows and check the values against the catalog's `value_fingerprint`).
4. If two headers tie above threshold, ask the user one question to disambiguate. If no header passes for a required field, ask the user to name the column. Do not guess.

Print a Mapping Report at the very top of the audit output. List each canonical field, the chosen export header, and the match outcome (matched, ambiguous, unmapped). The audit's grades depend on the mapping, so the mapping must be auditable.

## Process

### Step 1: Acquire data

Pull all opportunities (open and closed-this-quarter), forecast history (12 weeks), and rep metadata.

### Step 2: Score the five dimensions

Each dimension is scored 0 to 100, then converted to a letter grade: A (90-100), B (80-89), C (70-79), D (60-69), F (below 60).

**Dimension 1: Pipeline Health (weight: 25%)**

Starts at 100. Subtract:
- 15 points if open pipeline coverage is less than 2.5x committed forecast for the quarter
- 10 points if more than 30% of open pipeline is in early-stage (Discovery + Qualification)
- 10 points if more than 25% of pipeline sits in a single rep's book
- 5 points if average opportunity age exceeds 90 days
- 5 points if win rate over last quarter is below 20%

Recommends: `pipeline-hygiene-audit` if score is below 80.

**Dimension 2: Data Quality (weight: 20%)**

Starts at 100. Subtract:
- 20 points if more than 25% of open opportunities have stale activity (>14 days)
- 10 points if more than 15% of open opportunities have stale activity (>14 days) but less than 25%
- 15 points if more than 20% of late-stage opportunities (Negotiation, Verbal) lack a next step
- 10 points if more than 5% of opportunities have close dates in the past
- 10 points if more than 10% have probability values that mismatch their stage by 25+ points

Recommends: `pipeline-hygiene-audit` if score is below 85.

**Dimension 3: Forecast Hygiene (weight: 25%)**

Starts at 100. Subtract:
- 20 points if team commit varies more than 15% from historical model forecast
- 15 points if more than half of reps are uncalibrated (variance >20% from their personal historical pattern)
- 10 points if any single rep accounts for more than 40% of committed forecast
- 10 points if pipeline-weighted forecast is less than 70% of committed (thin coverage)

Recommends: `forecast-call-prep` if score is below 85.

**Dimension 4: Deal Integrity (weight: 20%)**

Starts at 100. Subtract:
- 5 points for each individual opportunity over $100K that has 3+ risk flags (per `deal-investigator` rubric), capped at 30 points total
- 15 points if more than 20% of late-stage pipeline value is held by a single rep
- 10 points if median time-in-stage for Verbal exceeds 30 days
- 10 points if any opportunity in Commit category has stale activity beyond 21 days

Recommends: `deal-investigator` for each flagged opportunity, listed by ID.

**Dimension 5: Process Integrity (weight: 10%)**

Starts at 100. Subtract:
- 10 points if more than 5% of opportunities show stage progression without supporting activity
- 10 points if probability-stage mismatch rate exceeds 15%
- 10 points if multiple owners are still listed as active on inactive opportunities
- 10 points if forecast category and stage are misaligned for more than 10% of open opps

Recommends: a v0.2 skill (process-integrity-audit, planned) if score is below 80. For v0.1, surface the findings but note the dedicated remediation skill is forthcoming.

### Step 3: Compute the overall grade

Weighted average across the five dimensions:

```
overall_score = (pipeline_health * 0.25) +
                (data_quality * 0.20) +
                (forecast_hygiene * 0.25) +
                (deal_integrity * 0.20) +
                (process_integrity * 0.10)
```

Convert to letter grade using the same scale (A: 90+, B: 80-89, C: 70-79, D: 60-69, F: below 60).

### Step 4: Build the remediation queue

Order the recommended skills by impact, with the most impactful first. Impact is determined by which dimension lost the most points relative to its weight. The skill that addresses the worst-scoring dimension goes first.

If multiple skills are tied, order them by ease of execution: `pipeline-hygiene-audit` first (lowest cost to fix), then `forecast-call-prep`, then individual `deal-investigator` runs.

### Step 5: Write the audit report

Output using exactly this structure:

```
# Salesforce RevOps Audit, [date]

Run against [data source: Salesforce MCP / synthetic Northwind Cloud dataset / uploaded data].

## Overall Health: [Letter Grade] ([Score]/100)

[1-2 sentences. State the headline. Example: "C+ overall. Pipeline coverage is 
adequate but data quality and forecast hygiene are the limiting factors. The 
forecast call this Monday should be approached with low confidence until cleanup 
is done."]

## Dimensional Scores

| Dimension | Grade | Score | Weight |
|---|---|---:|---:|
| Pipeline Health | [Grade] | [Score]/100 | 25% |
| Data Quality | [Grade] | [Score]/100 | 20% |
| Forecast Hygiene | [Grade] | [Score]/100 | 25% |
| Deal Integrity | [Grade] | [Score]/100 | 20% |
| Process Integrity | [Grade] | [Score]/100 | 10% |

## What's driving the grade

### Pipeline Health: [Grade]
[2-3 bullets. What's working, what's not. Specific numbers.]

### Data Quality: [Grade]
[2-3 bullets.]

### Forecast Hygiene: [Grade]
[2-3 bullets.]

### Deal Integrity: [Grade]
[2-3 bullets, including any specific deals flagged.]

### Process Integrity: [Grade]
[2-3 bullets.]

## Remediation queue, in order

The audit recommends running these skills in this sequence. Each skill targets 
the lowest-scoring dimension and produces a concrete deliverable.

1. **[skill-name]** addresses [dimension]. Potential lift: [X points] on the 
   dimension score if executed and the issues are remediated.

2. **[skill-name]** addresses [dimension]. Potential lift: [X points].

[continue for as many skills as the audit recommends, typically 2 to 4]

## What the audit cannot tell you

This audit measures system health based on data already in Salesforce. It cannot 
diagnose:
- Activity capture failures (Einstein Activity Capture, Outreach sync gaps): a 
  dedicated `activity-capture-diagnostic` skill is planned for v0.2.
- Lead routing logic issues (LeanData, Salesforce Assignment Rules): planned 
  for v0.2.
- Compensation calculation errors: planned for v0.2.
- Whether reps actually know how to use the system: human conversation only.

## Re-run cadence

Re-run this audit:
- Quarterly as a recurring health check.
- After any major Salesforce migration or process change.
- When a new RevOps leader takes over the org.
- Before any sales planning cycle (annual or quarterly planning).
```

### Step 6: Guardrails on the output

- Do not soften the grade. A C is a C. An F is an F. The CRO needs the honest assessment.
- Do not list more than 4 skills in the remediation queue. If everything is broken, the user has bigger problems than this audit.
- Do not skip the "what the audit cannot tell you" section. Acknowledging limitations is the trust anchor.
- If overall grade is A, congratulate the user honestly. Recommend a quarterly re-run cadence and stop. Do not invent problems to justify the audit.
- If overall grade is F, flag this as a systemic issue requiring leadership intervention, not a remediation queue. Skills cannot fix culture or staffing problems.

## What good output looks like

See `examples/sample_output.md` for the audit run against the bundled Northwind Cloud dataset.

## What to avoid

- Treating the score as ground truth. It is one diagnostic view, calibrated to common B2B SaaS patterns. Adjust thresholds in the SKILL.md if your org is materially different (e.g., enterprise-only motion, transactional motion, regulated industries).
- Recommending skills that are not in this pack. The remediation queue must reference skills the user can actually run.
- Reporting the audit without action. The output is for the user to do something with, not to file.

## Appendix A: SOQL for Salesforce MCP

The audit needs three queries:

```sql
-- All opportunities (open and closed this quarter)
SELECT Id, Name, Account.Name, OwnerId, Owner.Name, Amount, StageName,
       Probability, ForecastCategoryName, CloseDate, LastActivityDate,
       NextStep, CreatedDate, IsClosed, IsWon
FROM Opportunity
WHERE IsClosed = false
   OR (IsClosed = true AND CloseDate = THIS_QUARTER)

-- Forecast history (12 weeks)
SELECT Rep__c, Week_Of__c, Commit_Amount__c, BestCase_Amount__c, Pipeline_Amount__c
FROM Forecast__c
WHERE Week_Of__c = LAST_N_WEEKS:12
ORDER BY Rep__c, Week_Of__c DESC

-- Rep roster
SELECT Id, Name, UserRole.Name, IsActive
FROM User
WHERE UserType = 'Standard'
  AND IsActive = true
  AND Profile.Name IN ('Sales User', 'Sales Manager')
```

If the user's org does not have a `Forecast__c` object, ask where forecast submissions are stored. The audit can run without forecast history but the Forecast Hygiene dimension cannot be scored; default to a flag of "Insufficient Data" for that dimension.

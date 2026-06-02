---
name: activity-capture-diagnostic
description: Diagnose sales activity capture quality across a team and grade it A through F. Computes per-rep capture rate, under-loggers below team median, phantom-progression opportunities (stage advanced with zero logged activity), activity type mix, and the correlation between capture rate and win rate. Use when the user asks about Einstein Activity Capture, Outreach sync gaps, missing call logs, low activity volume, or whether reps are actually working their pipeline.
---

# Activity Capture Diagnostic

## What this skill does

Diagnoses activity-capture quality for a sales team across four sub-dimensions, grades each A through F, and produces a prioritized list of reps and opportunities the manager should address. The output names specific under-loggers, specific phantom-progression deals, and gives the manager an advisory plan for the week.

This is the skill a Director of Revenue Operations runs when forecast credibility is suffering and they suspect the underlying problem is not pipeline composition but missing activity data. It is also the skill they run quarterly to keep activity-capture discipline from drifting.

## Upstream context

This skill is typically invoked by `salesforce-revops-audit` when activity or capture signals look weak (high stale-activity rate, low logged-activity volume, or phantom stage progression). It can also be run directly when a leader specifically wants to look at activity discipline. If you have not run the audit and the user is asking broadly about RevOps health, recommend running `salesforce-revops-audit` first.

## When to invoke

Invoke when the user says any of:
- "audit activity capture"
- "Einstein Activity Capture gaps"
- "Outreach sync issues"
- "are reps actually logging calls"
- "why is forecast credibility so low"
- "phantom progression"
- "activity volume by rep"
- "who is under-logging"

Do not invoke for:
- Pipeline composition issues (use `pipeline-hygiene-audit`)
- Forecast variance investigation (use `forecast-call-prep`)
- Single-deal pressure tests (use `deal-investigator`)
- Lead-side speed-to-lead questions (use `lead-routing-rule-analyzer`)

## Data sources, in order of preference

1. **Salesforce MCP**: query Task and Event records with the SOQL in Appendix A.
2. **CSV or JSON**: parse `activities.csv` or `activities.json` plus `opportunities.csv` (the diagnostic needs both).
3. **Sample data**: bundled synthetic dataset at `data/` in this repo.

The activities file may start with a comment line beginning with `#` carrying a canary GUID. Skip any leading `#`-prefixed lines before parsing the CSV header.

## Column discernment

Real Salesforce exports rarely match canonical names exactly. Custom suffixes (`__c`), renamed fields, and different cases are normal. Before parsing any data file, read `docs/column_mapping.md` and use it to map the export's actual headers to the canonical fields this skill needs (canonical activity fields are listed under "Canonical fields: activities" in that doc).

Procedure (full detail in `docs/column_mapping.md`):

1. Normalize each header in the export (lowercase, strip `__c`, replace `_` and `.` with space, drop noise tokens).
2. Score each header by token overlap against the canonical field's `header_tokens`, subtracting for `exclusion_tokens` hits.
3. Confirm the top candidate with a value fingerprint (pull two or three sample rows and check the values against the catalog's `value_fingerprint`).
4. If two headers tie above threshold, ask the user one question to disambiguate. If no header passes for a required field, ask the user to name the column. Do not guess.

Print a Mapping Report at the top of the diagnostic output. List each canonical field, the chosen export header, and the match outcome (matched, ambiguous, unmapped). Capture-related grades depend on the mapping, so the mapping must be auditable.

## Process

### Step 1: Acquire data

Pull all activity records for the last 60 to 90 days, and all open opportunities. For each activity, you need at minimum: activity_id, type, owner, related_opportunity_id, activity_date. For each opportunity, you need: id, owner_name, stage, amount, is_closed, is_won.

### Step 2: Score the four sub-dimensions

Each sub-dimension is scored 0 to 100, then converted to a letter grade: A (90 to 100), B (80 to 89), C (70 to 79), D (60 to 69), F (below 60).

**Sub-dimension 1: Capture Rate (weight: 30%)**

Compute total activities per rep over the lookback window. Compute team median.

Starts at 100. Subtract:
- 20 points for every rep more than 50% below team median (under-logger).
- 10 points for every rep between 30% and 50% below team median (light logger).
- 10 points if more than 25% of reps are under-loggers (systemic under-capture).

Cap deductions at 60 points (D floor; an F here requires Sub-dimension 4 to also fail).

**Sub-dimension 2: Phantom Progression (weight: 30%)**

A phantom-progression opportunity is one in Proposal stage or later that has zero logged activities in the lookback window. These deals advanced through stages without any recorded sales work.

Starts at 100. Subtract:
- 5 points per phantom opportunity over $100K, capped at 30 points.
- 20 points if more than 25% of advanced-stage open opportunities (Proposal, Negotiation, Verbal) are phantoms.
- 10 points if more than 15% of advanced-stage open opportunities are phantoms but less than 25%.

**Sub-dimension 3: Activity Type Mix (weight: 20%)**

Healthy mix for a B2B SaaS team: Calls roughly 30 to 45%, Emails 30 to 45%, Meetings 10 to 25%, Tasks 5 to 15%. The exact mix matters less than the absence of any single type. A rep with 100% email is not selling, they are nudging.

Starts at 100. Subtract:
- 15 points for every rep whose mix is more than 70% a single activity type.
- 10 points if team-wide Meetings are below 10% of total activity.
- 10 points if team-wide Calls are below 20% of total activity.

**Sub-dimension 4: Capture-Win Correlation (weight: 20%)**

Compute each rep's win rate over closed-this-quarter opportunities. Compute each rep's activity volume. Rank both. A healthy team shows positive correlation: top capturers are also top closers.

Starts at 100. Subtract:
- 30 points if the correlation is negative (top capturers are bottom closers, or vice versa). This is a strong signal of either activity-theatre (reps logging to look busy) or activity-deficit (closers winning despite poor records). Either case needs intervention.
- 15 points if the correlation is weakly positive (Spearman rho between 0 and 0.3) and the team has more than five reps.
- 0 points if rho is at or above 0.3.

If there are fewer than 12 closed-this-quarter opportunities total, flag as "insufficient data" and skip this sub-dimension; reweight the other three proportionally.

### Step 3: Compute the overall grade

Weighted average across the four sub-dimensions:

```
overall_score = (capture_rate * 0.30) +
                (phantom_progression * 0.30) +
                (activity_mix * 0.20) +
                (capture_win_correlation * 0.20)
```

Convert to letter grade using the same scale (A: 90+, B: 80 to 89, C: 70 to 79, D: 60 to 69, F: below 60).

### Step 4: Write the diagnostic report

Output using exactly this structure:

```
# Activity Capture Diagnostic, [date]

Run against [data source: Salesforce MCP / synthetic Northwind Cloud dataset / uploaded data].

Mapping Report: [matched N of M canonical fields. Unmapped: <list or none>.]

## Overall Capture Health: [Letter Grade] ([Score]/100)

[1 to 2 sentences. State the headline. Example: "D overall. Activity volume 
is concentrated in three healthy reps; the rest of the team is logging at 
less than half the team median. Three large opportunities have advanced to 
Proposal or later with zero logged activity, which is the single most 
material credibility risk in the report."]

## Dimensional Scores

| Sub-dimension | Grade | Score | Weight |
|---|---|---:|---:|
| Capture Rate | [Grade] | [Score]/100 | 30% |
| Phantom Progression | [Grade] | [Score]/100 | 30% |
| Activity Type Mix | [Grade] | [Score]/100 | 20% |
| Capture-Win Correlation | [Grade] | [Score]/100 | 20% |

## What's driving the grade

### Capture Rate: [Grade]
- Team median: N activities per rep over the lookback.
- Under-loggers (more than 50% below median): [Rep, Rep, Rep] with [N, N, N] activities.
- Light loggers (30 to 50% below median): [Rep, Rep] with [N, N] activities.

### Phantom Progression: [Grade]
- Advanced-stage open opps (Proposal, Negotiation, Verbal): N total.
- Phantoms (zero activities logged): N (X%).
- Phantom deals over $100K: N. Specific IDs flagged: `opp_xxxx`, `opp_xxxx`.

### Activity Type Mix: [Grade]
- Team mix: Calls X%, Emails X%, Meetings X%, Tasks X%.
- Reps with single-type concentration (>70% one type): [Rep with X% Email], etc.

### Capture-Win Correlation: [Grade]
- Spearman rho between activity volume and win rate: N.
- [One sentence interpretation: healthy positive, weak positive, or inverted.]

## Flagged items

### Under-loggers (warrant a capture-discipline conversation)
| Rep | Activities | Vs Team Median | Type Mix |
|---|---:|---:|---|
| [Rep] | N | -X% | C/E/M/T |
| ...

### Phantom-progression opportunities (advanced with zero activity)
| Opp ID | Stage | Amount | Owner |
|---|---|---:|---|
| `opp_xxxx` | [Stage] | $N | [Rep] |
| ...

## Recommended next steps

Run `deal-investigator` once per phantom-progression opportunity over $100K to determine whether the deal is real or pipeline padding. Specific IDs in priority order: `opp_xxxx`, `opp_xxxx`, `opp_xxxx`.

If the capture-win correlation is negative, also recommend re-running `forecast-call-prep` with low confidence flagged, because the top forecasters may not be the top deal-workers.

## What to do this week

Advisory, prioritized actions tied to the flagged items. These are suggestions for the manager, not orders for the reps.

1. **[Highest-priority rep]** warrants a capture-discipline conversation this week. Their logged activity is more than 50% below team median, and they own [N] open opportunities worth [$X]. The conversation worth having: are they working a different motion (referral-heavy, partner-led) that does not generate logged activity, or is the capture itself the gap?

2. **[Next rep]** has the same pattern at a smaller scale. Same conversation, lower urgency.

3. **Phantom-progression deals** belong on the next deal-review agenda. The manager should ask the owners what the stage-progression evidence was if no activity exists.

4. **Activity-mix outliers** (single-type concentration) are worth a coaching conversation, not an intervention. A rep at 90% email is probably running scared of phone calls; that is a skills gap, not a discipline gap.

## What the diagnostic cannot tell you

This diagnostic measures activity recorded in the CRM. It cannot diagnose:

- Whether the activity is high quality (a logged call could be a 90-second voicemail or a 45-minute discovery).
- Activity captured in third-party tools but never synced (Outreach, Salesloft, Gong gaps).
- Whether the rep is talking to the wrong people. Activity to a champion vs an economic buyer reads identically here.
- Whether the captured activity is real or fabricated. Activity-theatre exists.

## Re-run cadence

Re-run this diagnostic monthly during a capture-discipline initiative, or quarterly as a standing check. Re-run after any rollout of Outreach, Salesloft, or Einstein Activity Capture to verify sync is actually working.
```

### Step 5: Guardrails on the output

- Do not soften the grade. A D is a D.
- Do not list more than 10 phantom opportunities individually. If more, summarize the rest in aggregate.
- Do not name an under-logger without numbers. Always include the per-rep activity count and the team median for context.
- If the team has fewer than five reps, the Capture-Win Correlation sub-dimension is unreliable; flag insufficient data and skip rather than guess.
- The "What to do this week" section is advisory. Use phrasing like "warrants a conversation", "worth a coaching moment", "belongs on the next agenda". Do not write prescriptive orders for individual reps.
- If overall grade is A, congratulate the team honestly. Recommend a quarterly cadence and stop.
- If overall grade is F, flag this as a systemic capture-culture issue requiring leadership intervention, not per-rep coaching.

## What good output looks like

See `examples/sample_output.md` for the diagnostic run against the bundled Northwind Cloud dataset.

## What to avoid

- Confusing activity volume with sales effectiveness. A rep can over-log and under-sell, or under-log and over-sell. The diagnostic measures one signal, not the whole picture.
- Naming reps in the report without their per-rep number. "Marcus is under-logging" is gossip; "Marcus logged 4 activities vs team median 16" is data.
- Prescribing remediation steps for individual reps. The manager owns those conversations.
- Treating phantom-progression as proof of fraud. The most common explanation is sloppy capture, not fabricated deals.

## Appendix A: SOQL for Salesforce MCP

The diagnostic needs three queries:

```sql
-- All Task records in lookback window
SELECT Id, Subject, Type, OwnerId, Owner.Name, WhatId, ActivityDate,
       CreatedDate, Status
FROM Task
WHERE ActivityDate >= LAST_N_DAYS:90

-- All Event records in lookback window
SELECT Id, Subject, OwnerId, Owner.Name, WhatId, ActivityDate,
       CreatedDate
FROM Event
WHERE ActivityDate >= LAST_N_DAYS:90

-- Open opportunities (for phantom-progression and correlation checks)
SELECT Id, Name, OwnerId, Owner.Name, Amount, StageName, IsClosed, IsWon,
       CloseDate, LastActivityDate
FROM Opportunity
WHERE IsClosed = false OR (IsClosed = true AND CloseDate = THIS_QUARTER)
```

If the org has Einstein Activity Capture enabled, the EAC-synced records appear in Task and Event with `EACEventId` populated. The diagnostic does not need to distinguish manual from EAC records; both count as captured activity.

If the org uses Outreach or Salesloft with bi-directional sync, ask the user whether those activities are written back to Salesforce Task records. If not, the diagnostic will under-report and should warn the user.

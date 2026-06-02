---
name: lead-routing-rule-analyzer
description: Diagnose lead routing health and grade it A through F. Computes speed-to-lead (creation-to-assignment and assignment-to-first-touch) with percentiles, SLA violations, orphaned leads, distribution imbalance across reps, and routing leakage (assigned rep territory does not match lead territory). Use when the user asks about LeanData, Salesforce Assignment Rules, lead handoffs, speed-to-lead SLA, or whether leads are reaching the right reps.
---

# Lead Routing Rule Analyzer

## What this skill does

Diagnoses lead-routing health for a sales team across four sub-dimensions, grades each A through F, and produces a prioritized list of routing failures the manager and the RevOps function should address. The output names specific orphaned leads, specific SLA violations, and gives the manager an advisory plan for the week.

This is the skill a Director of Revenue Operations runs when conversion rates are inconsistent across reps and they suspect the underlying problem is upstream of the AE: leads are taking too long to reach reps, reaching the wrong reps, or never reaching anyone.

## Upstream context

This skill is typically invoked by `salesforce-revops-audit` when lead or routing signals appear (e.g., uneven conversion across reps in the same segment, conversion rates below industry norms, complaints about lead quality that turn out to be lead-timing issues). It can also be run directly when the user knows their routing rules are stale. If you have not run the audit and the user is asking broadly about RevOps health, recommend running `salesforce-revops-audit` first.

## When to invoke

Invoke when the user says any of:
- "audit lead routing"
- "LeanData health check"
- "Salesforce Assignment Rules audit"
- "speed-to-lead"
- "are leads reaching the right reps"
- "orphaned leads"
- "lead distribution by rep"
- "routing leakage"

Do not invoke for:
- Activity capture and logging (use `activity-capture-diagnostic`)
- Pipeline composition (use `pipeline-hygiene-audit`)
- Single-deal pressure tests (use `deal-investigator`)
- Forecast variance (use `forecast-call-prep`)

## Data sources, in order of preference

1. **Salesforce MCP**: query Lead records with the SOQL in Appendix A.
2. **CSV or JSON**: parse `leads.csv` or `leads.json` matching the schema in `docs/data_schema.md`.
3. **Sample data**: bundled synthetic dataset at `data/` in this repo.

The leads file may start with a comment line beginning with `#` carrying a canary GUID. Skip any leading `#`-prefixed lines before parsing the CSV header.

## Column discernment

Real Salesforce exports rarely match canonical names exactly. Custom suffixes (`__c`), renamed fields, and different cases are normal. Before parsing any data file, read `docs/column_mapping.md` and use it to map the export's actual headers to the canonical fields this skill needs (canonical lead fields are listed under "Canonical fields: leads" in that doc).

Procedure (full detail in `docs/column_mapping.md`):

1. Normalize each header in the export (lowercase, strip `__c`, replace `_` and `.` with space, drop noise tokens).
2. Score each header by token overlap against the canonical field's `header_tokens`, subtracting for `exclusion_tokens` hits.
3. Confirm the top candidate with a value fingerprint (pull two or three sample rows and check the values against the catalog's `value_fingerprint`).
4. If two headers tie above threshold, ask the user one question to disambiguate. If no header passes for a required field, ask the user to name the column. Do not guess.

Print a Mapping Report at the top of the diagnostic output. List each canonical field, the chosen export header, and the match outcome (matched, ambiguous, unmapped). Routing grades depend on the mapping, so the mapping must be auditable.

## Process

### Step 1: Acquire data

Pull all lead records from the last 60 to 90 days. For each lead, you need at minimum: lead_id, source, created_date, assigned_rep, assignment_date, first_touch_date, status, territory, segment, converted. You also need a rep-to-territory roster so the routing-leakage check has a ground truth to compare against. If the rep-to-territory mapping is not in the data, ask the user to provide it.

### Step 2: Score the four sub-dimensions

Each sub-dimension is scored 0 to 100, then converted to a letter grade: A (90 to 100), B (80 to 89), C (70 to 79), D (60 to 69), F (below 60).

**Sub-dimension 1: Speed-to-Lead (weight: 30%)**

Compute two latencies per assigned lead:
- Creation-to-assignment: hours between `created_date` and `assignment_date`.
- Assignment-to-first-touch: hours between `assignment_date` and `first_touch_date`.

Industry-standard SLA: creation-to-assignment under 24 hours, assignment-to-first-touch under 24 hours. Inbound SaaS teams often target much tighter (under 5 minutes for the first touch, e.g., Chili Piper / Drift research). The defaults below assume the standard SLA; adjust thresholds in this SKILL.md if your org targets tighter.

Starts at 100. Subtract:
- 20 points if median creation-to-assignment exceeds 24 hours.
- 15 points if more than 20% of leads breach the 48-hour creation-to-assignment SLA.
- 15 points if median assignment-to-first-touch exceeds 24 hours.
- 10 points if more than 25% of assigned leads have a null `first_touch_date` after 7+ days (never touched).

**Sub-dimension 2: Orphaned Leads (weight: 25%)**

Orphaned leads have a null `assigned_rep`. They are either stuck in a routing rule that did not fire, or they fell out the bottom of a round-robin queue.

Starts at 100. Subtract:
- 30 points if more than 15% of leads are orphaned.
- 15 points if more than 8% of leads are orphaned but less than 15%.
- 10 points if any orphaned lead is more than 30 days old (compounding aging).
- 10 points if orphaned leads are concentrated in a single source (e.g., 50%+ of orphans come from Web). Source-concentrated orphans usually point at one broken assignment rule.

**Sub-dimension 3: Distribution Balance (weight: 20%)**

Compute lead count per rep, restricted to reps with at least one lead. Compute the Gini coefficient or the simpler max/min ratio.

Starts at 100. Subtract:
- 20 points if the highest-receiving rep gets more than 3x the lowest-receiving rep's volume (after excluding territory-of-one reps).
- 15 points if the top three reps together hold more than 60% of all assigned leads.
- 10 points if any rep with a documented territory receives zero leads in the lookback window.

If a rep is the only owner of their territory, exclude them from this calculation; their concentration is by design, not by failure.

**Sub-dimension 4: Routing Leakage (weight: 25%)**

A routing leak is a lead whose assigned rep's territory does not match the lead's territory. This skill needs a rep-to-territory map; if absent, ask the user.

Starts at 100. Subtract:
- 25 points if more than 15% of assigned leads are leaked (rep territory does not match lead territory).
- 15 points if more than 8% of assigned leads are leaked.
- 10 points if leakage is concentrated on a single rep (one rep receives more than 30% of all leaked leads) or a single territory (one territory leaks more than 30% of its leads).

If the rep-to-territory map is not available, flag this sub-dimension as "insufficient data" and reweight the other three proportionally.

### Step 3: Compute the overall grade

Weighted average across the four sub-dimensions:

```
overall_score = (speed_to_lead * 0.30) +
                (orphaned_leads * 0.25) +
                (distribution_balance * 0.20) +
                (routing_leakage * 0.25)
```

Convert to letter grade using the same scale (A: 90+, B: 80 to 89, C: 70 to 79, D: 60 to 69, F: below 60).

### Step 4: Write the diagnostic report

Output using exactly this structure:

```
# Lead Routing Diagnostic, [date]

Run against [data source: Salesforce MCP / synthetic Northwind Cloud dataset / uploaded data].

Mapping Report: [matched N of M canonical fields. Unmapped: <list or none>.]

## Overall Routing Health: [Letter Grade] ([Score]/100)

[1 to 2 sentences. State the headline. Example: "C overall. Speed-to-lead 
is acceptable on the median but the long tail of stuck leads is significant. 
The bigger concern is routing leakage: 13% of assigned leads went to a rep 
whose territory does not match, which is a routing-rule problem the 
RevOps function owns, not the AEs."]

## Dimensional Scores

| Sub-dimension | Grade | Score | Weight |
|---|---|---:|---:|
| Speed-to-Lead | [Grade] | [Score]/100 | 30% |
| Orphaned Leads | [Grade] | [Score]/100 | 25% |
| Distribution Balance | [Grade] | [Score]/100 | 20% |
| Routing Leakage | [Grade] | [Score]/100 | 25% |

## What's driving the grade

### Speed-to-Lead: [Grade]
- Creation-to-assignment: median N hours, p90 N hours.
- Assignment-to-first-touch: median N hours, p90 N hours.
- 48-hour creation-to-assignment SLA breaches: N of M (X%).
- Never-touched after 7 days: N (X%).

### Orphaned Leads: [Grade]
- Orphaned (null assigned_rep): N of M (X%).
- Orphaned and over 30 days old: N.
- Source concentration of orphans: top source [X] with N% of orphans.

### Distribution Balance: [Grade]
- Top rep: [Name] with N leads. Bottom rep with at least one: [Name] with N.
- Top three reps share of assigned leads: X%.
- Reps with documented territory but zero leads: [Name, Name].

### Routing Leakage: [Grade]
- Leaked leads (rep territory does not match lead territory): N of M (X%).
- Top leaking rep: [Name] with N leaks.
- Top leaking territory: [Territory] with N leaks (X% of that territory's leads).

## Flagged items

### Orphaned leads to assign this week
| Lead ID | Source | Created | Days Old | Territory |
|---|---|---|---:|---|
| `lead_xxxx` | [Source] | [date] | N | [Territory] |
| ...

### SLA-breached leads
| Lead ID | Created | Assigned | Hours Late | Owner |
|---|---|---|---:|---|
| `lead_xxxx` | [date] | [date] | N | [Rep] |
| ...

### Routing-leakage examples
| Lead ID | Territory | Assigned Rep | Rep's Territory |
|---|---|---|---|
| `lead_xxxx` | [Lead Terr] | [Rep] | [Rep Terr] |
| ...

## Recommended next steps

If the routing-leakage score is low, the next step is a working session between RevOps and the team that owns assignment rules (often a Salesforce Admin or a LeanData specialist) to inspect the rules that are actually firing. The data here is the symptom; the rule logic is the cause.

If the speed-to-lead score is low, recommend re-running this diagnostic after one week of focused remediation to confirm the SLA breaches are dropping. A single intervention rarely fixes a queue backlog overnight.

If distribution balance is the only weak score, the conversation is with sales leadership about territory design, not with RevOps about rules.

## What to do this week

Advisory, prioritized actions tied to the flagged items. These are suggestions for the manager and for RevOps, not orders for the reps.

1. **Assign the orphaned leads** that are over [N] days old. These represent revenue waiting on the wrong side of the routing rules. The fastest fix is manual assignment while the underlying rule is investigated.

2. **Pull the assignment-rule audit trail** for the leaked-lead examples above. If a single rule is responsible for most leaks, that is the one-day fix worth doing first.

3. **The SLA-breach pattern** warrants a conversation with the reps named in the breach list. Worth asking whether the breach was a routing delay (their queue is empty when the lead arrives) or a workload issue (their queue is full when the lead arrives). The remediation is different in each case.

4. **Distribution imbalance** is a slower problem. If one rep is taking a disproportionate share of leads, the right venue for the conversation is the next territory planning cycle, not a midweek intervention.

## What the diagnostic cannot tell you

This diagnostic measures routing behavior recorded in the CRM. It cannot diagnose:

- Whether the leads themselves are high quality. A perfectly routed bad lead is still a bad lead.
- Whether the rep actually engaged after `first_touch_date`. The first touch could be a one-line email that went nowhere.
- Whether the assignment rules themselves are correctly designed. The diagnostic measures execution against the rule definitions, not whether the definitions are right.
- LeanData-specific routing logic that lives outside of standard Salesforce fields. If the org runs LeanData, the routing audit trail in LeanData itself is the authoritative source.

## Re-run cadence

Re-run this diagnostic monthly during a routing-rule overhaul, or quarterly as a standing check. Re-run after any change to assignment rules, any new territory carve-up, or any change to LeanData routing logic.
```

### Step 5: Guardrails on the output

- Do not soften the grade. A D is a D.
- Do not list more than 15 SLA-breached leads individually. Summarize the rest in aggregate.
- Do not list orphaned leads in groups larger than 20 individually; cap the table and report the remainder by source.
- The "What to do this week" section is advisory. Use phrasing like "worth a conversation", "warrants an inspection", "belongs in the next planning cycle". Do not write prescriptive orders for individual reps or for RevOps.
- If the team has fewer than four reps, the Distribution Balance sub-dimension is unreliable; flag insufficient data and reweight the other three.
- If overall grade is A, congratulate the team honestly. Recommend a quarterly cadence and stop.
- If overall grade is F, flag this as a systemic routing-architecture issue requiring a RevOps and Salesforce Admin working session, not per-lead triage.

## What good output looks like

See `examples/sample_output.md` for the diagnostic run against the bundled Northwind Cloud dataset.

## What to avoid

- Treating routing leakage as a rep failure. The rules belong to RevOps; the rep is downstream of the failure.
- Listing every orphaned lead. The pattern matters more than the exhaustive list.
- Recommending tooling changes (switch from Assignment Rules to LeanData, etc.). This skill diagnoses, it does not redesign.
- Ignoring the rep-to-territory map. Without it, the leakage check is impossible; ask the user rather than guess.

## Appendix A: SOQL for Salesforce MCP

The diagnostic needs one main query, plus a rep roster lookup:

```sql
-- Leads in lookback window
SELECT Id, LeadSource, CreatedDate, OwnerId, Owner.Name,
       AssignmentDate__c, First_Touch_Date__c, Status, Territory__c,
       Segment__c, IsConverted
FROM Lead
WHERE CreatedDate >= LAST_N_DAYS:90

-- Rep roster with territory (for routing-leakage check)
SELECT Id, Name, Territory__c, UserRole.Name
FROM User
WHERE UserType = 'Standard'
  AND IsActive = true
  AND Profile.Name IN ('Sales User', 'Sales Manager')
```

If the org does not have `AssignmentDate__c` and `First_Touch_Date__c` custom fields, ask where the assignment-time and first-touch-time are stored. Common alternatives include:
- LeanData routing logs (an external object).
- Salesforce Lead History (`LeadHistory`), filtered by `OwnerId` changes.
- Outreach or Salesloft sequence-enrolled timestamps.

If none of these are available, the diagnostic can still run on the Speed-to-Lead sub-dimension using `CreatedDate` and the first `LeadHistory` `OwnerId` change as a proxy. Flag the proxy in the report.

# Prompts

Copy-paste prompts that invoke each skill. Paste any of these into Claude Code, Claude Desktop, or any client that supports Agent Skills.

These prompts assume you've cloned the repo and are running Claude from inside `revops-skills/`. They reference paths relative to the repo root.

## Start here: the entry-point audit

If you don't know which skill to run, start here. This skill grades your org across five dimensions and tells you which other skill to run next.

```
Read skills/salesforce-revops-audit/SKILL.md and follow its instructions exactly. Run the audit against the data in the data/ directory. Today's date is [TODAY]. Produce the full audit report.
```

Replace `[TODAY]` with the current date (e.g., `May 22, 2026`). The skill uses the date to compute staleness, age, and quarter-to-date metrics.

A worked example of what this produces is in [`sample_audit_run.md`](sample_audit_run.md).

## Just took over a new RevOps role

Don't name a skill. Let the audit decide:

```
I just took over RevOps at [COMPANY]. I need to assess the state of our Salesforce org and figure out where to focus first. The data is in the data/ directory. Today is [TODAY].
```

Claude should recognize this as the textbook use case for `salesforce-revops-audit` and run it. If you want to test that the skill discovery is working, this is the prompt to use.

## Preparing for a forecast call

For the weekly Monday morning forecast meeting:

```
Read skills/forecast-call-prep/SKILL.md and follow its instructions exactly. Use the synthetic data in the data/ directory. Today's date is [TODAY]. Produce the weekly forecast call memo.
```

Output: a one-page memo with team commit reconciled against pipeline-weighted forecast and historical model, plus a recommended position for the leader to take into the call.

## Cleaning up pipeline before a forecast call or QBR

```
Read skills/pipeline-hygiene-audit/SKILL.md and follow its instructions exactly. Use the synthetic data in the data/ directory. Today's date is [TODAY]. Produce the per-rep hygiene punchlist.
```

Output: a per-rep punchlist with severity-ranked issues (stale activity, missing next step, past close dates) and specific opp IDs the manager should require fixed before the call.

## Pressure-testing a specific deal

```
Read skills/deal-investigator/SKILL.md and follow its instructions exactly. The target opportunity is [OPP_ID]. Use the synthetic data in the data/ directory. Today's date is [TODAY]. Produce the deal review memo.
```

Replace `[OPP_ID]` with the opportunity identifier (e.g., `opp_0029` in the bundled synthetic data, or `006xx000003ABCD` for a real Salesforce ID).

Output: a structured deal memo answering the five questions a CRO would ask in a deal review, with a recommendation of Commit / BestCase / Pipeline / Omit.

## Diagnosing activity capture (v0.2)

For surfacing under-loggers, phantom-progression deals, and the correlation between activity volume and win rate:

```
Read skills/activity-capture-diagnostic/SKILL.md and follow its instructions exactly. Use the synthetic data in the data/ directory. Today's date is [TODAY]. Produce the activity capture diagnostic report.
```

Output: a graded A through F report with per-rep activity counts, phantom-progression opportunities by ID, and an advisory "What to do this week" section.

## Diagnosing lead routing (v0.2)

For speed-to-lead, orphaned leads, distribution balance, and routing leakage:

```
Read skills/lead-routing-rule-analyzer/SKILL.md and follow its instructions exactly. Use the synthetic data in the data/ directory. Today's date is [TODAY]. Produce the lead routing diagnostic report.
```

Output: a graded A through F report with SLA-breached leads by ID, orphaned leads by source, and an advisory "What to do this week" section.

## Using your own Salesforce data

By default the prompts above point at `data/` (the bundled synthetic Northwind Cloud dataset). To use your own data, see [`bring_your_own_data.md`](bring_your_own_data.md) for the export instructions, then change the prompt to point at your data directory:

```
Read skills/salesforce-revops-audit/SKILL.md and follow its instructions exactly. Run the audit against the data in my_data/. Today's date is May 22, 2026. Produce the full audit report.
```

## Tips

- The skills are deterministic in approach but the model is not. Two runs against the same data may grade slightly differently on edge thresholds. That is by design; the skills capture practitioner judgment, not a fixed rubric.
- Always include "follow its instructions exactly" in the prompt. Without it, the model may approximate the SKILL.md instead of following them.
- Always state today's date. The skills compute staleness and aging relative to today, and getting the date wrong will skew every metric.
- If a skill produces output you disagree with, the SKILL.md is editable. Pull request welcome.

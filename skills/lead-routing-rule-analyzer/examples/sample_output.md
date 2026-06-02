# Lead Routing Diagnostic, May 20, 2026

Run against the synthetic Northwind Cloud dataset (Q2 FY2027, lookback 90 days).

Mapping Report: matched 10 of 10 canonical fields from `data/leads.json`. Unmapped: none.

## Overall Routing Health: B (82/100)

B overall. The medians on speed-to-lead are right at the 24-hour SLA boundary and the p90 tails are concerning, but no thresholds are tripped on the team aggregate. The bigger drag is orphaned leads: 17% of all leads have no assigned rep, and 24 of those have been sitting unassigned for more than 30 days. The routing-rule audit trail is the right next step.

## Dimensional Scores

| Sub-dimension | Grade | Score | Weight |
|---|---|---:|---:|
| Speed-to-Lead | A | 100/100 | 30% |
| Orphaned Leads | D | 60/100 | 25% |
| Distribution Balance | B | 80/100 | 20% |
| Routing Leakage | B | 85/100 | 25% |

## What's driving the grade

### Speed-to-Lead: A
- Creation-to-assignment: median 24h, p90 144h.
- Assignment-to-first-touch: median 24h, p90 168h.
- 48-hour creation-to-assignment SLA breaches: 26 of 149 (17.4%, below the 20% deduction threshold).
- 24-hour assignment-to-first-touch SLA breaches: 27 of 137 (19.7%).
- Never-touched after assignment: 12 of 149 (8.1%).
- Both medians are sitting exactly on the 24-hour SLA threshold. No deduction is triggered but the margin is thin; any drift puts the score at risk.

### Orphaned Leads: D
- Orphaned (null assigned_rep): 31 of 180 (17.2%, above the 15% deduction threshold).
- Orphaned and over 30 days old: 24.
- Source mix of orphans: Event 9, Outbound 7, Referral 7, Web 5, Partner 3. No single source above 50% concentration, so no source-specific routing rule failure is implied.

### Distribution Balance: B
- Top rep: Diego Marquez with 28 leads. Bottom rep with at least one: Yuki Tanaka with 4.
- Max-to-min ratio: 7.0x, above the 3x healthy threshold.
- Top three reps share of assigned leads: 53.7% (below the 60% concentration threshold).
- All ten reps received at least one lead in the lookback; no documented territory is sitting empty.

### Routing Leakage: B
- Leaked leads (assigned rep's territory does not match lead's territory): 20 of 149 (13.4%, above the 8% deduction threshold).
- Top leaking rep: Diego Marquez with 5 leaks.
- Top leaking territory: EMEA at 19% (6 of 31 EMEA leads went to non-EMEA reps).

## Flagged items

### Orphaned leads to assign this week
| Lead ID | Source | Created | Days Old | Territory |
|---|---|---|---:|---|
| `lead_0163` | Event | 2026-02-23 | 86 | NAMER-East |
| `lead_0048` | Outbound | 2026-02-25 | 84 | NAMER-West |
| `lead_0164` | Event | 2026-03-04 | 77 | EMEA |
| `lead_0098` | Event | 2026-03-08 | 73 | NAMER-East |
| `lead_0071` | Referral | 2026-03-09 | 72 | LATAM |
| `lead_0165` | Event | 2026-03-11 | 70 | NAMER-East |
| `lead_0141` | Web | 2026-03-13 | 68 | APAC |
| `lead_0109` | Event | 2026-03-19 | 62 | NAMER-East |
| `lead_0080` | Outbound | 2026-03-21 | 60 | LATAM |
| `lead_0073` | Referral | 2026-03-23 | 58 | NAMER-West |

14 additional orphaned leads over 30 days old not listed individually. See `data/leads.json` for the full set.

### SLA-breached leads (creation-to-assignment over 48 hours)
| Lead ID | Created | Assigned | Hours Late | Owner |
|---|---|---|---:|---|
| `lead_0123` | 2026-04-15 | 2026-04-27 | 288 | Hannes Vogel |
| `lead_0028` | 2026-03-05 | 2026-03-16 | 264 | Lakshmi Iyer |
| `lead_0083` | 2026-04-29 | 2026-05-10 | 264 | Tomas Lindgren |
| `lead_0084` | 2026-03-27 | 2026-04-07 | 264 | Wei Chen |
| `lead_0153` | 2026-03-26 | 2026-04-06 | 264 | Hannes Vogel |

21 additional SLA-breached leads not listed individually.

### Routing-leakage examples
| Lead ID | Territory | Assigned Rep | Rep's Territory |
|---|---|---|---|
| `lead_0008` | APAC | Diego Marquez | NAMER-West |
| `lead_0033` | EMEA | Diego Marquez | NAMER-West |
| `lead_0052` | LATAM | Marcus Bell | NAMER-Central |
| `lead_0054` | EMEA | Marcus Bell | NAMER-Central |
| `lead_0056` | NAMER-West | Wei Chen | LATAM |

15 additional routing-leakage examples not listed individually. Diego Marquez is responsible for 5 of 20 leaks (25%).

## Recommended next steps

The orphaned-leads score is the worst dimension and the easiest to act on. Manual assignment of the 24 orphans over 30 days old recovers immediate pipeline. In parallel, RevOps should pull the assignment-rule audit trail and inspect why those 24 leads never matched a rule.

The routing-leakage rate sits just above the deduction threshold. A working session between RevOps and whoever owns the assignment rules (often a Salesforce Admin or LeanData specialist) is the right venue to inspect why EMEA leads are leaking to NAMER reps. If a single rule is responsible, the fix is one day of work.

The speed-to-lead medians are at the SLA boundary, so any further degradation will trip the deduction. Worth setting up a weekly check of c2a and a2t medians for the next month.

## What to do this week

1. **Assign the 24 orphaned leads over 30 days old** by hand this week. Revenue is waiting on the wrong side of the routing rules. While that triage is happening, pull the assignment-rule audit trail in parallel.

2. **The leakage pattern on EMEA leads** (19% of EMEA going to non-EMEA reps) warrants a one-hour inspection of whichever assignment rule fires on `LeadSource = Event` or `LeadSource = Outbound` plus EMEA territory. Diego Marquez taking 5 of the 20 total leaks suggests a single misconfigured round-robin step rather than a systemic rule failure.

3. **The five reps named in the SLA-breach list** warrant a conversation about whether the breach was queue-empty (a routing delay before the lead reached them) or queue-full (a workload issue after the lead arrived). The remediation is different in each case.

4. **The distribution imbalance** (Diego at 28, Yuki at 4) is not actionable this week. The right venue is the next territory planning cycle, not a midweek intervention. Diego picks up disproportionately many EMEA and LATAM mis-routed leads, which inflates his count.

## What the diagnostic cannot tell you

This diagnostic measures routing behavior recorded in the CRM. It cannot diagnose:

- Whether the leads themselves are high quality. A perfectly routed bad lead is still a bad lead.
- Whether the rep actually engaged after `first_touch_date`. The first touch could be a one-line email that went nowhere.
- Whether the assignment rules themselves are correctly designed. The diagnostic measures execution against the rule definitions, not whether the definitions are right.
- LeanData-specific routing logic that lives outside of standard Salesforce fields. If the org runs LeanData, the routing audit trail in LeanData itself is the authoritative source.

## Re-run cadence

Re-run this diagnostic monthly during a routing-rule overhaul, or quarterly as a standing check. Re-run after any change to assignment rules, any new territory carve-up, or any change to LeanData routing logic.

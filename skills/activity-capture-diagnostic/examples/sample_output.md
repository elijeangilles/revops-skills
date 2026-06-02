# Activity Capture Diagnostic, May 20, 2026

Run against the synthetic Northwind Cloud dataset (Q2 FY2027, lookback 60 days).

Mapping Report: matched 7 of 7 canonical fields from `data/activities.json`. Unmapped: none.

## Overall Capture Health: F (58/100)

F overall. The team is logging 144 activities over the lookback, but the load is concentrated in seven of ten reps. Three reps are logging at less than half the team median, and 65% of advanced-stage open opportunities have zero logged activity. The phantom-progression rate is the single most material credibility risk in the report.

## Dimensional Scores

| Sub-dimension | Grade | Score | Weight |
|---|---|---:|---:|
| Capture Rate | F | 40/100 | 30% |
| Phantom Progression | F | 50/100 | 30% |
| Activity Type Mix | B | 85/100 | 20% |
| Capture-Win Correlation | C | 70/100 | 20% |

## What's driving the grade

### Capture Rate: F
- Team median: 16.5 activities per rep over the 60-day lookback.
- Under-loggers (more than 50% below median): Wei Chen (3 activities), Marcus Bell (4), Hannes Vogel (6). Three of ten reps, which is also above the 25% systemic-under-capture threshold.
- Light loggers (30 to 50% below median): none.
- The three under-loggers together account for 13 of 144 logged activities (9%) despite representing 30% of the team.

### Phantom Progression: F
- Advanced-stage open opps (Proposal, Negotiation, Verbal): 123 total.
- Phantoms (zero activities logged): 80 (65.0%), well above the 25% threshold.
- Phantom deals over $100K: 17. Top ten by amount are listed below in Flagged Items.

### Activity Type Mix: B
- Team mix: Calls 34.0%, Emails 43.1%, Meetings 17.4%, Tasks 5.6%. All inside the healthy bands.
- One rep with single-type concentration above 70%: Marcus Bell at 75% Email.

### Capture-Win Correlation: C
- Spearman rho between activity volume and closed-this-quarter win rate: -0.18.
- Weakly inverted. The lowest-volume reps (Wei Chen 3, Marcus Bell 4) both posted 100% win rates this quarter, but on tiny samples (4 of 4 and 1 of 1). The signal is suggestive of activity-theatre at the top or activity-deficit at the bottom, but the per-rep closed-deal counts are too small to be conclusive.

## Flagged items

### Under-loggers (warrant a capture-discipline conversation)
| Rep | Activities | Vs Team Median | Type Mix |
|---|---:|---:|---|
| Wei Chen | 3 | -82% | Email-heavy |
| Marcus Bell | 4 | -76% | 75% Email |
| Hannes Vogel | 6 | -64% | Mixed |

### Phantom-progression opportunities (advanced with zero activity)
| Opp ID | Stage | Amount | Owner |
|---|---|---:|---|
| `opp_0013` | Verbal | $444,000 | Aisha Okonkwo |
| `opp_0209` | Proposal | $438,500 | Lakshmi Iyer |
| `opp_0205` | Negotiation | $434,500 | Lakshmi Iyer |
| `opp_0001` | Proposal | $424,500 | Aisha Okonkwo |
| `opp_0026` | Proposal | $407,500 | Diego Marquez |
| `opp_0037` | Proposal | $341,500 | Diego Marquez |
| `opp_0002` | Proposal | $296,500 | Aisha Okonkwo |
| `opp_0005` | Negotiation | $287,500 | Aisha Okonkwo |
| `opp_0031` | Negotiation | $262,000 | Diego Marquez |
| `opp_0015` | Proposal | $245,500 | Aisha Okonkwo |

7 additional phantom opportunities over $100K not listed individually.

## Recommended next steps

Run `deal-investigator` once per phantom-progression opportunity over $100K, in priority order: `opp_0013`, `opp_0209`, `opp_0205`, `opp_0001`, `opp_0026`. These five represent over $2M of advanced-stage pipeline with no recorded sales work.

The correlation score is weakly negative, so also re-run `forecast-call-prep` with low confidence flagged. The top-volume reps and top-closing reps are not the same set this quarter, which means the forecast view from activity is not reliable for ranking.

## What to do this week

1. **Wei Chen, Marcus Bell, and Hannes Vogel** each warrant a capture-discipline conversation this week. The conversation worth having: are they working a different motion (referral-heavy, partner-led) that does not generate logged activity, or is the capture itself the gap? Wei Chen's 4 closed-won deals on only 3 logged activities is either remarkable territory or a sync problem with Outreach or Salesloft.

2. **Marcus Bell's 75% Email mix** is worth a coaching moment beyond the capture issue. A rep at three-quarters email is probably running scared of phone calls.

3. **The phantom-progression list belongs on the next deal-review agenda.** The manager should ask Aisha, Lakshmi, and Diego what the stage-progression evidence was if no activity exists for the listed deals.

4. **The aggregate phantom rate (65% of advanced-stage opps)** is too high to attribute to individual reps. The conversation here is about the team's habits at stage-transition: stages are being advanced in the CRM without the supporting activity being logged at the same time. This is a process problem, not a discipline problem on any one rep.

## What the diagnostic cannot tell you

This diagnostic measures activity recorded in the CRM. It cannot diagnose:

- Whether the activity is high quality (a logged call could be a 90-second voicemail or a 45-minute discovery).
- Activity captured in third-party tools but never synced (Outreach, Salesloft, Gong gaps).
- Whether the rep is talking to the wrong people. Activity to a champion vs an economic buyer reads identically here.
- Whether the captured activity is real or fabricated. Activity-theatre exists.

## Re-run cadence

Re-run this diagnostic monthly during a capture-discipline initiative, or quarterly as a standing check. Re-run after any rollout of Outreach, Salesloft, or Einstein Activity Capture to verify sync is actually working.

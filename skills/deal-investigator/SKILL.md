---
name: deal-investigator
description: Produce a structured deal review for a single opportunity. Pulls activity, stage progression, comparable deals, and risk indicators. Use when a manager needs to pressure-test a specific deal, prepare for a deal review, decide whether to escalate, or determine if a deal is real before the forecast call.
---

# Deal Investigator

## What this skill does

For a single opportunity, produces a structured review answering five questions a CRO would ask in a deal review:

1. Is this deal real?
2. Is the close date credible?
3. What is the actual risk?
4. What do comparable deals tell us?
5. What is the recommendation: commit, best case, omit, or escalate?

The output is a one-page deal memo, formatted for a deal review meeting.

## Upstream context

This skill is typically invoked by `salesforce-revops-audit` once per flagged opportunity when the audit's Deal Integrity dimension surfaces individual deals with 3+ risk flags. It can also be run directly when a manager wants to pressure-test a specific deal they have in mind. Unlike the other skills in this pack, this one operates on a single opportunity, so the audit will recommend running it multiple times in sequence when multiple deals are flagged.

## When to invoke

Invoke when the user says any of:
- "investigate this deal"
- "deal review for [opp name or id]"
- "is this deal real"
- "pressure-test [opp name]"
- "deep dive on this opportunity"
- "should we call this in commit"

Do not invoke for:
- Full-team forecast prep (use `forecast-call-prep`)
- Pipeline-wide hygiene scanning (use `pipeline-hygiene-audit`)
- Multi-deal portfolio analysis

## Data sources, in order of preference

1. **Salesforce MCP**: query the specific opportunity, its account, recent activity, and comparable closed opportunities (SOQL in Appendix A).
2. **CSV or JSON**: read `opportunities.csv` and identify the target opp by id or name.
3. **Sample data**: bundled synthetic dataset, with the user providing an opp_id from the dataset.

## Column discernment

Real Salesforce exports rarely match canonical names exactly. Custom suffixes (`__c`), renamed fields, and different cases are normal. Before parsing any data file, read `docs/column_mapping.md` and use it to map the export's actual headers to the canonical fields this skill needs.

Procedure (full detail in `docs/column_mapping.md`):

1. Normalize each header in the export (lowercase, strip `__c`, replace `_` and `.` with space, drop noise tokens).
2. Score each header by token overlap against the canonical field's `header_tokens`, subtracting for `exclusion_tokens` hits.
3. Confirm the top candidate with a value fingerprint (pull two or three sample rows and check the values against the catalog's `value_fingerprint`).
4. If two headers tie above threshold, ask the user one question to disambiguate. If no header passes for a required field, ask the user to name the column. Do not guess.

Note the mapping in a one-line footnote at the bottom of the deal memo: `Mapping: matched N of M required fields from <source>.`

## Process

### Step 1: Identify the target opportunity

User provides either an opp ID (`opp_0042`), an account name ("Acme Industries"), or partial description ("the Acme renewal Diego is working on"). If ambiguous, list the top three matches and ask which one. If exact match, proceed.

### Step 2: Gather the deal facts

Pull from the data source:

- Opportunity name, account, owner, segment, amount, stage, probability, forecast category
- Created date, close date, last activity date, days since last activity
- Next step
- Stage history if available
- Activity history if available

### Step 3: Identify comparable deals

Find 3-5 closed opportunities (won and lost) that share at least two of: same owner, same segment, similar amount band (within 50% of the target), same general industry signal from account name. These become the comparable set.

For each comparable, note: outcome (won/lost), days from creation to close, stage at which it was lost (if lost), final amount.

### Step 4: Compute the risk indicators

Score each indicator. Each is a binary risk flag.

| Indicator | Risk if |
|---|---|
| Stale activity | days_since_last_activity > 14 |
| Missing next step | next_step is empty |
| Close date plausibility | close_date < today (already passed) or stage = Discovery with close < 60 days |
| Stage age vs typical | days in current stage > 1.5x average for that stage for this owner |
| Amount vs comparable | target amount > 2x typical comparable amount |
| Forecast category alignment | category = Commit or BestCase but probability < 40 |
| Owner calibration history | owner is "optimist" profile AND deal is in Commit |

Count the flags. 0-1 flags = healthy. 2-3 = at risk. 4+ = at high risk.

### Step 5: Write the deal memo

Output using exactly this structure:

```
# Deal Review: [Account Name], [Opportunity Name]
[Opp ID] | [Owner] | [Segment] | $[Amount] | [Stage] | Close [date]

## The five questions

### 1. Is this deal real?
[2-3 sentences. State the position. "Yes, based on X." or "Maybe, but Y is missing." 
or "No, this looks like pipeline-padding."]

### 2. Is the close date credible?
[2-3 sentences. Compare the close date to typical deal cycle for this owner and 
this stage. Specify the realistic close date if the stated one is not credible.]

### 3. What is the actual risk?
[List the risk flags that fired, in order of materiality. Each one gets a single 
line. Do not pad.]

### 4. What do comparable deals tell us?
[2-3 sentences. State the owner's win rate on similar deals, the typical cycle 
length, and any common loss reason. Use the comparable set you identified.]

### 5. Recommendation
[Pick one: Commit, BestCase, Pipeline, or Omit. Then 1-2 sentences explaining 
the call. If Escalate, name who should drive it.]

## Deal facts
- Days in pipeline:                  N (created [date])
- Days in current stage:             N
- Days since last activity:          N
- Next step:                         [text or "MISSING"]
- Probability:                       N% (stage default: N%)
- Forecast category:                 [Pipeline / BestCase / Commit / Closed]

## Comparable deals
| Account | Amount | Outcome | Cycle | Notes |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |

## Risk indicators
- [✓ or ✗] Stale activity
- [✓ or ✗] Missing next step
- [✓ or ✗] Close date plausibility
- [✓ or ✗] Stage age vs typical
- [✓ or ✗] Amount vs comparable
- [✓ or ✗] Forecast category alignment
- [✓ or ✗] Owner calibration history

**Risk score: N flags. [Healthy / At risk / High risk]**
```

### Step 6: Guardrails on the output

- Do not soften the recommendation. If the call is Omit, say Omit. The manager needs the answer.
- Do not list comparable deals if there are fewer than 2 in the dataset. Instead say "Insufficient comparable history for this owner / segment combination" and proceed.
- Do not editorialize beyond what the data supports. Specifically, do not speculate on buyer intent, internal politics, or anything not in the data.
- If the deal is in Closed Won, refuse the request and tell the user this skill is for open opportunities only. Offer `close-won-analysis` instead (planned future skill).
- If the deal is in Closed Lost, refuse with the same message but offer a close-lost analysis instead.

## What good output looks like

See `examples/sample_output.md` for a worked example against an opp in the bundled Northwind Cloud dataset.

## What to avoid

- Reasoning about things not in the data. If you do not know the competitor, do not speculate about competition.
- Diplomatic recommendations. "It depends" is not a recommendation. Take a position.
- Long preambles. Get to the five questions fast.
- Restating the deal facts inside the prose. They are in the facts section at the bottom.

## Appendix A: SOQL for Salesforce MCP

```sql
-- The target opportunity
SELECT Id, Name, Account.Name, OwnerId, Owner.Name, Amount, StageName,
       Probability, ForecastCategoryName, CloseDate, LastActivityDate,
       NextStep, CreatedDate, IsClosed, IsWon
FROM Opportunity
WHERE Id = '[OPP_ID]'

-- Recent activity on the target
SELECT Id, Subject, ActivityDate, OwnerId, WhatId
FROM Task
WHERE WhatId = '[OPP_ID]'
  AND ActivityDate >= LAST_N_DAYS:60
ORDER BY ActivityDate DESC

-- Comparable closed deals from same owner and segment
SELECT Id, Name, Account.Name, Amount, StageName, CloseDate, CreatedDate, IsWon
FROM Opportunity
WHERE OwnerId = '[OWNER_ID]'
  AND IsClosed = true
  AND CloseDate = LAST_N_DAYS:365
ORDER BY CloseDate DESC
LIMIT 20
```

If stage history is needed and the org has the OpportunityHistory object, supplement with a query against that.

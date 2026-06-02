# Data Schema

Skills in this repo accept data from three sources, in order of preference:

1. A connected **Salesforce MCP server** (the official Salesforce-hosted MCP or DX MCP)
2. **CSV or JSON files** matching the schema below
3. The bundled **synthetic dataset** at `data/` in this repo

You provide nothing back to the maintainer. The skills run entirely on the user's machine against the user's own data. No credentials, no telemetry, no phone-home.

The schemas below are canonical. Real Salesforce exports rarely match these names exactly (custom suffixes, renamed fields, different cases). As of v0.2, each skill discerns column mappings at runtime by reasoning against the catalog in [`column_mapping.md`](column_mapping.md), so you can drop a raw export into `data/` or `my_data/` without renaming columns. Use the canonical names below as the reference for what each skill needs, not as a hard requirement for your file headers.

## opportunities

One row per opportunity. CSV column names or JSON field names must match exactly.

| Field | Type | Required | Notes |
|---|---|---|---|
| id | string | yes | unique identifier |
| name | string | yes | opportunity name |
| account_name | string | yes | customer name |
| owner_id | string | yes | rep identifier |
| owner_name | string | yes | rep display name |
| segment | string | no | Enterprise / Mid-Market / Commercial or similar |
| amount | number | yes | deal value in USD |
| stage | string | yes | one of: Discovery, Qualification, Proposal, Negotiation, Verbal, Closed Won, Closed Lost |
| probability | number | yes | 0 to 100 |
| forecast_category | string | yes | one of: Pipeline, BestCase, Commit, Closed, Omitted |
| close_date | string | yes | ISO date (YYYY-MM-DD) |
| created_date | string | yes | ISO date |
| last_activity_date | string | yes | ISO date |
| next_step | string | no | empty allowed; absence is a hygiene signal |
| days_since_last_activity | number | computed | derived if not provided |
| is_closed | boolean | yes | |
| is_won | boolean | yes | |

## forecast_history

Weekly per-rep forecast submissions, used by `forecast-call-prep` for calibration.

| Field | Type | Required | Notes |
|---|---|---|---|
| rep_id | string | yes | matches opportunities.owner_id |
| rep_name | string | yes | |
| week_of | string | yes | ISO date for the Monday of that week |
| commit_amount | number | yes | USD |
| bestcase_amount | number | yes | USD |
| pipeline_amount | number | yes | USD |

Minimum 8 weeks of history is recommended for credible calibration. Skills will warn if less than 8 weeks is provided.

## activities

One row per Salesforce Task or Event record. Used by `activity-capture-diagnostic`.

| Field | Type | Required | Notes |
|---|---|---|---|
| activity_id | string | yes | unique identifier |
| type | string | yes | one of: Call, Email, Meeting, Task |
| subject | string | no | free-text subject line |
| owner | string | yes | rep display name, must match opportunities.owner_name |
| related_opportunity_id | string | no | references opportunities.id; null for account-level activity |
| activity_date | string | yes | ISO date (YYYY-MM-DD) |
| created_date | string | yes | ISO date |
| status | string | no | one of: Completed, Open, Not Started, In Progress |

The bundled CSV starts with a `#`-prefixed comment line carrying a canary GUID marking the file as synthetic demo data. Skills should skip leading `#`-prefixed lines before parsing the CSV header. The JSON form wraps the records in an object: `{"_canary": {...}, "records": [...]}`. Skills should look for a `records` key first; if absent, treat the document as a flat list.

## leads

One row per Salesforce Lead record. Used by `lead-routing-rule-analyzer`.

| Field | Type | Required | Notes |
|---|---|---|---|
| lead_id | string | yes | unique identifier |
| source | string | yes | one of: Web, Event, Referral, Outbound, Partner |
| created_date | string | yes | ISO date |
| assigned_rep | string | no | rep display name; null for orphaned leads (a real signal) |
| assignment_date | string | no | ISO date; null if orphaned |
| first_touch_date | string | no | ISO date; null if never touched |
| status | string | yes | one of: Open, Working, Nurturing, Qualified, Disqualified, Converted |
| territory | string | no | e.g., NAMER-East, EMEA, APAC |
| segment | string | no | Enterprise / Mid-Market / Commercial |
| converted | boolean | yes | |

Same canary and CSV-comment conventions as activities.

## Field mapping from Salesforce

If pulling via Salesforce MCP, the standard field mapping is:

```
opportunities.id              <- Opportunity.Id
opportunities.name            <- Opportunity.Name
opportunities.account_name    <- Opportunity.Account.Name
opportunities.owner_id        <- Opportunity.OwnerId
opportunities.owner_name      <- Opportunity.Owner.Name
opportunities.amount          <- Opportunity.Amount
opportunities.stage           <- Opportunity.StageName
opportunities.probability     <- Opportunity.Probability
opportunities.forecast_category <- Opportunity.ForecastCategoryName
opportunities.close_date      <- Opportunity.CloseDate
opportunities.last_activity_date <- Opportunity.LastActivityDate
opportunities.next_step       <- Opportunity.NextStep
opportunities.is_closed       <- Opportunity.IsClosed
opportunities.is_won          <- Opportunity.IsWon
```

Forecast history mapping varies by org. The skill will ask you where forecast submissions are stored (custom object, Forecast Cloud, external system) and adapt.

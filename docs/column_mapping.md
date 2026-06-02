# Column Mapping Reference

Every skill in this pack reads this file before parsing any data. The skill
uses this catalog, the actual export's headers, and a few sample rows to map
real columns to canonical field names without the user editing anything.

The user drops a raw Salesforce export into `data/` or `my_data/`. The skill
infers which export column corresponds to each canonical field by reasoning
about header names and sample values against the patterns below. No manual
mapping step is required. The `tools/rename_columns.py` script still works
for users who prefer a deterministic pre-step, but it is no longer required.

## How the skill maps columns at runtime

For each canonical field below:

1. **Normalize every header in the export.** Lowercase the header, strip
   Salesforce custom suffixes (`__c`, `__pc`, `__r`), replace underscores
   and dots with spaces, and drop noise tokens (`field`, `custom`, `picklist`,
   `lookup`, `text`, `formula`, `id` when it appears alongside another noun).
2. **Score each header by token overlap** against the canonical field's
   `header_tokens`. Subtract for any `exclusion_tokens` hit.
3. **Confirm with a value fingerprint.** For each header above the minimum
   score, pull two or three sample row values and check them against
   `value_fingerprint`. A header that matches by name but fails the
   fingerprint loses.
4. **Pick the highest-scoring header that passes the fingerprint.** Record
   the mapping along with the score. **One header maps to at most one
   canonical field**; once a header is claimed, remove it from consideration
   for other canonical fields. Resolve high-specificity fields first
   (enums, IDs, dates) before name-like fields, since IDs and enums have
   tighter fingerprints than free-text names.
5. **If two headers tie** above threshold, ask the user one targeted question
   to pick. **If no header passes** for a required field, ask the user to
   name the column. Do not guess.
6. **Print a Mapping Report** at the top of the skill's output, listing each
   canonical field, the chosen export header, the match score, and any
   unmapped fields. The mapping is always auditable.

The same procedure works against any of the three data sources: Salesforce
MCP results (where headers usually match the standard REST API names), CSV
or JSON files (where headers may be anything), or the bundled synthetic
dataset (where headers already match canonical names).

## Canonical fields: opportunities

### id
- header_tokens: id, opp_id, opportunity_id, record_id, sfdc_id
- exclusion_tokens: account, owner, lead, parent, external
- value_fingerprint: short alphanumeric string, often prefixed (`opp_`, `006`), unique per row
- required: yes

### name
- header_tokens: name, opp_name, opportunity_name, deal_name, title
- exclusion_tokens: account, owner, rep, product, region
- value_fingerprint: free-text, typically 5 to 120 characters, usually contains the account name plus a deal type token
- required: yes

### account_name
- header_tokens: account, account_name, customer, company, organization, client
- exclusion_tokens: owner, rep, contact, partner, lead
- value_fingerprint: free-text company name, typically 2 to 80 characters
- required: yes

### owner_id
- header_tokens: owner_id, owner, owner_user_id, rep_id, ae_id, salesrep_id
- exclusion_tokens: account, contact, created_by, last_modified_by, manager
- value_fingerprint: short alphanumeric string, often prefixed (`rep_`, `005`), repeats across rows
- required: yes

### owner_name
- header_tokens: owner_name, owner_full_name, rep_name, ae_name, salesrep_name, opportunity_owner
- exclusion_tokens: account, contact, created_by, manager
- value_fingerprint: human name, two or more space-separated tokens
- required: yes

### segment
- header_tokens: segment, market_segment, tier, customer_tier, account_tier, vertical_tier
- exclusion_tokens: industry, region, geo, vertical_industry
- value_fingerprint: enum, one of (Enterprise, Mid-Market, Commercial, SMB, Strategic) or close synonyms
- required: no

### amount
- header_tokens: amount, value, tcv, acv, arr, mrr, total, contract, deal, price, revenue
- exclusion_tokens: count, qty, quantity, discount, tax, target, quota, list_price
- value_fingerprint: positive number, typically 500 to 5,000,000, no currency symbol after CSV parse
- required: yes

### stage
- header_tokens: stage, stagename, sales_stage, opportunity_stage, deal_stage, pipeline_stage
- exclusion_tokens: history, previous, prior, last
- value_fingerprint: enum, one of (Discovery, Qualification, Proposal, Negotiation, Verbal, Closed Won, Closed Lost) or close synonyms (Closed-Won, Won, Lost, Commit, Closed)
- required: yes

### probability
- header_tokens: probability, probability_pct, win_probability, confidence_pct, likelihood
- exclusion_tokens: forecast, calibration
- value_fingerprint: integer or float, 0 to 100
- required: yes

### forecast_category
- header_tokens: forecast_category, forecastcategoryname, forecast_bucket, commit_category, forecast_tier
- exclusion_tokens: history, override, manager
- value_fingerprint: enum, one of (Pipeline, BestCase, Best Case, Commit, Closed, Omitted) or close synonyms
- required: yes

### close_date
- header_tokens: close, closed, close_date, expected_close, target_close, projected_close
- exclusion_tokens: created, modified, activity, fiscal
- value_fingerprint: ISO date or US date format, within roughly plus or minus 730 days of today for open deals
- required: yes

### created_date
- header_tokens: created, created_date, created_on, opp_created, record_created, open_date
- exclusion_tokens: modified, last, activity, close
- value_fingerprint: ISO date or US date format, on or before today
- required: yes

### last_activity_date
- header_tokens: last_activity, last_activity_date, last_touch, last_touched, latest_activity, last_engagement
- exclusion_tokens: created, modified, close, due
- value_fingerprint: ISO date or US date format, on or before today; nulls allowed
- required: yes

### next_step
- header_tokens: next_step, nextstep, next_action, next_steps, action_plan
- exclusion_tokens: stage, history, completed
- value_fingerprint: free-text, often empty (an empty value is a meaningful hygiene signal, not a mapping failure)
- required: no

### is_closed
- header_tokens: is_closed, closed, isclosed, closed_flag, deal_closed
- exclusion_tokens: lost, won, reason
- value_fingerprint: boolean (True, False, 1, 0, Yes, No, true, false)
- required: yes

### is_won
- header_tokens: is_won, won, iswon, won_flag, deal_won, outcome_won
- exclusion_tokens: lost, closed, reason
- value_fingerprint: boolean (True, False, 1, 0, Yes, No, true, false)
- required: yes

## Canonical fields: forecast_history

### rep_id
- header_tokens: rep_id, owner_id, user_id, salesrep_id, ae_id
- exclusion_tokens: manager, region
- value_fingerprint: short alphanumeric string, repeats across rows
- required: yes

### rep_name
- header_tokens: rep_name, rep, owner_name, user_name, ae_name
- exclusion_tokens: manager
- value_fingerprint: human name
- required: yes

### week_of
- header_tokens: week_of, week, week_start, forecast_week, period
- exclusion_tokens: month, quarter, year
- value_fingerprint: ISO date or US date format, typically a Monday
- required: yes

### commit_amount
- header_tokens: commit, commit_amount, committed, commit_value
- exclusion_tokens: bestcase, pipeline, total
- value_fingerprint: positive number
- required: yes

### bestcase_amount
- header_tokens: bestcase, best_case, bestcase_amount, best_case_value, upside
- exclusion_tokens: commit, pipeline
- value_fingerprint: positive number, typically equal to or greater than commit
- required: yes

### pipeline_amount
- header_tokens: pipeline, pipeline_amount, total_pipeline, open_pipeline
- exclusion_tokens: commit, bestcase, closed
- value_fingerprint: positive number, typically equal to or greater than bestcase
- required: yes

## Canonical fields: activities

### activity_id
- header_tokens: id, activity_id, task_id, event_id, record_id
- exclusion_tokens: related, parent, owner
- value_fingerprint: short alphanumeric string, unique per row
- required: yes

### type
- header_tokens: type, activity_type, task_type, subtype, channel
- exclusion_tokens: status, outcome
- value_fingerprint: enum, one of (Call, Email, Meeting, Task) or close synonyms (Phone, EmailMessage, Event, ToDo)
- required: yes

### subject
- header_tokens: subject, title, description, summary, name
- exclusion_tokens: account, rep, related
- value_fingerprint: free-text, typically 5 to 200 characters
- required: no

### owner
- header_tokens: owner, owner_name, assigned_to, rep, user_name, created_by
- exclusion_tokens: account, related, contact
- value_fingerprint: human name, matches one of the reps in the opportunity owner roster
- required: yes

### related_opportunity_id
- header_tokens: related_opportunity_id, what_id, opportunity_id, related_opp, parent_opp
- exclusion_tokens: account, lead, contact
- value_fingerprint: short alphanumeric string matching the opportunities `id` field; nulls allowed for account-level activity
- required: no

### activity_date
- header_tokens: activity_date, due_date, completed_date, occurred_on, logged_date
- exclusion_tokens: created, modified
- value_fingerprint: ISO date or US date format
- required: yes

### created_date
- header_tokens: created, created_date, created_on, logged_at
- exclusion_tokens: modified, completed, due
- value_fingerprint: ISO date or US date format, on or before today
- required: yes

### status
- header_tokens: status, completion_status, state, outcome
- exclusion_tokens: forecast, stage
- value_fingerprint: enum, one of (Completed, Open, Not Started, In Progress, Logged)
- required: no

## Canonical fields: leads

### lead_id
- header_tokens: id, lead_id, record_id, sfdc_id
- exclusion_tokens: owner, account, opportunity, converted
- value_fingerprint: short alphanumeric string, unique per row
- required: yes

### source
- header_tokens: source, lead_source, channel, origin, acquisition_source
- exclusion_tokens: campaign, medium
- value_fingerprint: enum, one of (Web, Event, Referral, Outbound, Partner, Inbound, Other)
- required: yes

### created_date
- header_tokens: created, created_date, created_on, lead_created
- exclusion_tokens: assigned, first_touch, modified
- value_fingerprint: ISO date or US date format, on or before today
- required: yes

### assigned_rep
- header_tokens: assigned_rep, owner, owner_name, assigned_to, lead_owner, rep_name
- exclusion_tokens: created_by, modified_by, account
- value_fingerprint: human name matching the opportunity owner roster; nulls allowed (orphaned leads are a real signal)
- required: no

### assignment_date
- header_tokens: assignment_date, assigned_date, assigned_on, routed_date, owner_assigned_date
- exclusion_tokens: created, first_touch, modified
- value_fingerprint: ISO date or US date format, on or after the created_date
- required: no

### first_touch_date
- header_tokens: first_touch, first_touch_date, first_contact, first_response, time_to_first_touch
- exclusion_tokens: created, assigned, modified
- value_fingerprint: ISO date or US date format, on or after the assignment_date; nulls allowed
- required: no

### status
- header_tokens: status, lead_status, state, stage
- exclusion_tokens: forecast, history
- value_fingerprint: enum, one of (Open, Working, Nurturing, Qualified, Disqualified, Converted)
- required: yes

### territory
- header_tokens: territory, region, geo, sales_region, lead_territory
- exclusion_tokens: industry, segment
- value_fingerprint: enum (NAMER-East, NAMER-West, NAMER-Central, EMEA, APAC, LATAM) or close synonyms
- required: no

### segment
- header_tokens: segment, market_segment, tier, customer_tier
- exclusion_tokens: industry, region
- value_fingerprint: enum (Enterprise, Mid-Market, Commercial, SMB)
- required: no

### converted
- header_tokens: converted, is_converted, lead_converted, converted_flag
- exclusion_tokens: opportunity, account
- value_fingerprint: boolean (True, False, 1, 0, Yes, No)
- required: yes

## When inference is not enough

The skill stops and asks the user one targeted question in these cases:

1. **Tie above threshold.** Two headers score equally well for the same
   canonical field and both pass the value fingerprint. Example: an export
   with both `Renewal_TCV__c` and `Total_Contract_Value__c` for `amount`.
2. **Required field has no candidate.** No header passes the minimum
   score for a required canonical field. The skill lists the headers it
   could not place and asks which one is the missing field.
3. **Value fingerprint contradicts header.** A header named `Amount`
   contains values like `Negotiation, Proposal, Discovery`. The skill
   reports the contradiction and asks the user to confirm.

Non-required canonical fields (segment, next_step, subject, assigned_rep,
assignment_date, first_touch_date, territory, segment) are skipped silently
if no header maps, and the affected metric is flagged as "insufficient data"
in the skill's output.

## When to update this catalog

Add a new header pattern when a contributor brings a real-world Salesforce
export whose headers do not map cleanly with the existing tokens. The catalog
should grow by canonical field, not by org. One row per canonical field,
regardless of how many ways orgs spell it.

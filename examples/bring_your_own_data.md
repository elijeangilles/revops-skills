# Bring Your Own Data

The skills in this repo ship with a synthetic dataset (`data/`) so you can evaluate the pack without any setup. To use them against your real Salesforce org, you have two options.

## Option 1: Salesforce MCP (recommended for production use)

The skills are designed to read from any Salesforce MCP server. If you have one configured in your Claude client, the skills will use it automatically. No file export needed.

Salesforce supports two MCP servers as of 2026:

- **[Salesforce-hosted MCP](https://developer.salesforce.com/blogs/2026/04/salesforce-hosted-mcp-servers-are-now-generally-available)**: production-ready, runs in your Salesforce environment, GA since April 2026.
- **[Salesforce DX MCP](https://developer.salesforce.com/blogs/2025/06/introducing-mcp-support-across-salesforce)**: developer-focused, runs locally against your org.

Configure either one in your Claude Code or Claude Desktop client. The skills will detect the MCP and execute the SOQL queries listed in each `SKILL.md`'s Appendix A.

If you're using a Salesforce MCP, you can stop reading here. The skills handle the rest.

## Option 2: Export CSV files

If you don't have an MCP set up (or your security team hasn't approved one yet), export the data and put it in a `my_data/` directory at the repo root. The skills will read from there if you tell them to.

Three export methods, ordered fastest to most guided. Pick the one that matches your comfort level.

### Method A: SOQL via Workbench or Developer Console (fastest path)

If you know SOQL, this is the fastest export. Total time: about 60 seconds.

In [Salesforce Workbench](https://workbench.developerforce.com), Developer Console, or any SOQL tool, paste:

```sql
SELECT Id, Name, Account.Name, OwnerId, Owner.Name, Amount, StageName,
       Probability, ForecastCategoryName, CloseDate, LastActivityDate,
       NextStep, CreatedDate, IsClosed, IsWon
FROM Opportunity
WHERE IsClosed = false
   OR (IsClosed = true AND CloseDate = THIS_QUARTER)
```

Export the results to CSV. Save to `my_data/opportunities.csv`.

If your org has a Segment field on Account or Opportunity, add it to the SELECT clause. Most B2B SaaS orgs do.

After exporting, run the rename script in Step 3 below.

### Method B: Data Loader (for larger exports or recurring use)

[Data Loader](https://developer.salesforce.com/tools/data-loader) is Salesforce's official desktop export tool. Free, works on Mac and Windows.

1. Download and install Data Loader.
2. Sign in to your Salesforce org.
3. Choose **Export** (not Export All, which includes deleted records).
4. Select the **Opportunity** object.
5. Choose a destination file path (`my_data/opportunities.csv`).
6. Click **Next**, then **Create/Edit Query**.
7. Select the fields listed in Method A's SOQL query (Workbench's UI builder makes this point-and-click).
8. Set the WHERE clause: `IsClosed = false OR (IsClosed = true AND CloseDate = THIS_QUARTER)`.
9. Click **Finish** to run the export.

After downloading, run the rename script in Step 3 below.

### Method C: Salesforce Reports (no technical setup required)

This is the path for anyone who has built a Salesforce report before but doesn't want to install anything or write SOQL.

1. In Salesforce, go to **Reports** in the navigation bar.
2. Click **New Report**.
3. Choose the **Opportunities** report type.
4. Add these columns (use the field picker on the left):
   - Opportunity ID
   - Opportunity Name
   - Account Name
   - Opportunity Owner ID
   - Opportunity Owner
   - Amount
   - Stage
   - Probability (%)
   - Forecast Category
   - Close Date
   - Created Date
   - Last Activity
   - Next Step
   - Closed
   - Won
5. Set the filter to: **Closed = False** OR **(Close Date = THIS QUARTER AND Closed = True)**. This pulls open opportunities plus anything closed this quarter (won or lost).
6. Click **Run Report**, then **Export** in the top right.
7. Choose **Details Only** (not Formatted Report) and **CSV** format.
8. Save the file to `my_data/opportunities.csv` in your repo.

After downloading, run the rename script in Step 3 below. The script handles the column name differences between Salesforce Reports format and the format the skills expect.

### Step 3: rename columns (optional as of v0.2)

As of v0.2, the skills discern column mappings at runtime by reading
[`docs/column_mapping.md`](../docs/column_mapping.md) and reasoning about
your export's headers and sample values. You can drop a raw Salesforce export
into `my_data/` and the skills will figure out which column is which without
the rename script. The skill prints a Mapping Report at the top of its output
so the mapping is auditable.

The rename script below is still supported for users who prefer a
deterministic pre-step, or who want to confirm the mapping before running
any skill. It is no longer required.

```bash
python3 tools/rename_columns.py my_data/opportunities.csv
```

The script will:

- Auto-detect that your file is an opportunities export
- Rename Salesforce field names (`StageName`, `CloseDate`, etc.) to the skill-expected names (`stage`, `close_date`)
- Normalize date formats (US `MM/DD/YYYY` to ISO `YYYY-MM-DD`)
- Normalize boolean values (`FALSE` to `False`)
- Compute the `days_since_last_activity` column from `last_activity_date`
- Create a `.bak` backup of your original export
- Warn you if any required columns are missing

If you're more comfortable with manual editing, you can skip the script and rename columns yourself. The full schema reference is in [`docs/data_schema.md`](../docs/data_schema.md).

### Step 4: tell the skill to use your data

In any prompt from [`prompts.md`](prompts.md), change `data/` to `my_data/`:

```
Read skills/salesforce-revops-audit/SKILL.md and follow its instructions exactly. Run the audit against the data in my_data/. Today's date is May 22, 2026. Produce the full audit report.
```

The skill will read from `my_data/opportunities.csv` instead of the bundled synthetic data.

### File structure

After setup, your repo should look like:

```
revops-skills/
├── data/                       # synthetic, bundled with the repo (keep this for reference)
├── my_data/                    # your real data (gitignored)
│   ├── opportunities.csv
│   └── opportunities.csv.bak   # backup created by rename script
├── tools/
│   └── rename_columns.py
├── skills/
└── README.md
```

Add `my_data/` to your `.gitignore` so you don't accidentally commit Salesforce data to a public repo:

```bash
echo "my_data/" >> .gitignore
```

## Optional: activity export (for activity-capture-diagnostic, v0.2)

The `activity-capture-diagnostic` skill reads Salesforce Task and Event records. Export both objects and concatenate them, or run them separately and tell the skill which file is which.

### SOQL (fastest)

```sql
SELECT Id, Subject, Type, OwnerId, Owner.Name, WhatId, ActivityDate,
       CreatedDate, Status
FROM Task
WHERE ActivityDate >= LAST_N_DAYS:90

SELECT Id, Subject, OwnerId, Owner.Name, WhatId, ActivityDate, CreatedDate
FROM Event
WHERE ActivityDate >= LAST_N_DAYS:90
```

Save the union as `my_data/activities.csv`. The skill's column discernment (`docs/column_mapping.md`) handles header differences; no manual rename required.

### Data Loader

1. Choose **Export** in Data Loader.
2. Run the export twice, once for **Task** and once for **Event**, with the fields above and a date filter for the last 90 days.
3. Concatenate the two CSVs (Task plus Event) into `my_data/activities.csv`. The header rows should be identical; remove the second header before saving.

### Salesforce Reports

1. New Report on the **Tasks and Events** report type.
2. Add columns: Activity ID, Subject, Type, Assigned To Owner, Related To ID, Activity Date, Created Date, Status.
3. Filter: Activity Date in the last 90 days.
4. Run, export as CSV, Details Only.
5. Save as `my_data/activities.csv`.

## Optional: lead export (for lead-routing-rule-analyzer, v0.2)

The `lead-routing-rule-analyzer` skill reads Salesforce Lead records, plus a rep-to-territory roster (which the skill will ask for if absent).

### SOQL (fastest)

```sql
SELECT Id, LeadSource, CreatedDate, OwnerId, Owner.Name,
       AssignmentDate__c, First_Touch_Date__c, Status, Territory__c,
       Segment__c, IsConverted
FROM Lead
WHERE CreatedDate >= LAST_N_DAYS:90
```

If your org does not have `AssignmentDate__c` or `First_Touch_Date__c` custom fields, the skill can fall back to `LeadHistory` for the assignment timestamp (see the SKILL.md Appendix A for the proxy logic). Tell the skill which fallback to use.

Save the result as `my_data/leads.csv`.

### Data Loader

1. Choose **Export** in Data Loader.
2. Select the **Lead** object.
3. Add the fields listed in the SOQL above.
4. Filter on Created Date in the last 90 days.
5. Save as `my_data/leads.csv`.

### Salesforce Reports

1. New Report on the **Leads** report type.
2. Add columns: Lead ID, Lead Source, Created Date, Lead Owner, Assignment Date, First Touch Date, Lead Status, Territory, Segment, Converted.
3. Filter: Created Date in the last 90 days.
4. Run, export as CSV, Details Only.
5. Save as `my_data/leads.csv`.

## Optional: forecast history export

The `forecast-call-prep` skill calibrates rep variance against historical forecast submissions. If your org has a Forecast object (Salesforce Collaborative Forecasting writes to a private API, but most B2B SaaS orgs build a custom Forecast__c object), export it the same way.

Using SOQL:

```sql
SELECT Rep__c, Week_Of__c, Commit_Amount__c, BestCase_Amount__c, Pipeline_Amount__c
FROM Forecast__c
WHERE Week_Of__c = LAST_N_WEEKS:12
ORDER BY Rep__c, Week_Of__c DESC
```

Save as `my_data/forecast_history.csv`, then run the rename script:

```bash
python3 tools/rename_columns.py my_data/forecast_history.csv
```

The script handles the `__c` suffixes automatically.

If your org doesn't have a Forecast object, `forecast-call-prep` will still run but will skip the calibration section and warn you.

## Data minimization

The skills only need the fields listed above. Resist the temptation to export everything; the smaller your export, the less likely you are to violate your org's data handling policies.

You should not include:

- Personal information (email addresses, phone numbers)
- Customer financial details beyond Amount
- Custom fields containing sensitive information
- Activity records (the skills only need `LastActivityDate`, not the activity bodies)

## A note on data residency

When you run the skills against your data, the data is sent to Anthropic's servers as part of the conversation with Claude. If your org has data residency requirements (GDPR, SOC 2, HIPAA, etc.), check with your security team before exporting production data.

Anonymizing the data first (replacing real rep names with placeholders, scaling dollar amounts) is a common compromise that preserves the analytical signal without exposing real customer information.

## Roadmap

v0.2 ships automatic column mapping at runtime via Claude reasoning, so the rename script in `tools/rename_columns.py` is no longer required. It is kept for users who prefer a deterministic pre-step. The catalog of header tokens and value fingerprints is in [`docs/column_mapping.md`](../docs/column_mapping.md).

## Troubleshooting

**Skill says it can't find the file**: check the path. The skill expects relative paths from the repo root (`my_data/opportunities.csv`, not `/Users/you/Downloads/my_data/opportunities.csv`).

**Rename script says required columns are missing**: your export didn't include all the fields the skills need. Re-run the export with all the columns listed in Method A's SOQL query.

**Numbers look wrong**: most often the date field. The skills compute staleness relative to "today's date" in the prompt, not the system date. If you forget to include the date in the prompt, the skills will use whatever Claude assumes, which may be wrong.

**Different output each run**: this is by design. The skills capture practitioner judgment, not a fixed rubric. Two runs against the same data may grade slightly differently on edge thresholds.

**Rename script fails on encoding**: if your CSV was exported with non-UTF-8 encoding (Windows exports sometimes use UTF-16 or Latin-1), open the file in Excel or a text editor, save as UTF-8, then re-run the script.

# Changelog

All notable changes to this project are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] 2026-06-02

v0.2 delivers on the activity-capture and lead-routing skills the v0.1 audit promised it could not yet diagnose. It also removes the manual rename step that was the most common friction point for users bringing real Salesforce exports.

### Added

- **activity-capture-diagnostic** (diagnostic tier). Grades activity capture A through F across four sub-dimensions (Capture Rate, Phantom Progression, Activity Type Mix, Capture-Win Correlation). Surfaces under-loggers below team median and phantom-progression opportunities (advanced stages with zero logged activity). Output includes a "Recommended next steps" section that routes phantom-progression opp IDs to `deal-investigator`, and a "What to do this week" section with advisory, prioritized actions.
- **lead-routing-rule-analyzer** (diagnostic tier). Grades lead routing A through F across four sub-dimensions (Speed-to-Lead, Orphaned Leads, Distribution Balance, Routing Leakage). Surfaces orphaned leads, SLA violations, distribution imbalance, and routing leakage (assigned-rep territory does not match lead territory).
- **Runtime column mapping** via [`docs/column_mapping.md`](docs/column_mapping.md). Skills now reason about a real Salesforce export's headers and sample values against a shared catalog, mapping arbitrary column names to canonical fields without the user editing anything. Drop a raw export with custom field names (`Total_Contract_Value__c`, `Stage_Name_Picklist`) into `data/` or `my_data/` and the skills figure out the mapping. The `tools/rename_columns.py` script is still supported as an optional deterministic pre-step.
- **Synthetic activity dataset** at `data/activities.csv` and `data/activities.json`. 144 Salesforce-style Task and Event records spanning all ten existing reps, with planted signals: three under-loggers (Wei Chen, Marcus Bell, Hannes Vogel) and broad phantom-progression coverage across advanced-stage opportunities.
- **Synthetic lead dataset** at `data/leads.csv` and `data/leads.json`. 180 Salesforce-style Lead records, with planted signals: 17% orphaned leads, 13% routing leakage, distribution imbalance (Diego Marquez at 28 leads vs Yuki Tanaka at 4), and 48-hour creation-to-assignment SLA breaches.
- **Canary GUID** in each new data file (`activities.*` and `leads.*`) marking the data as synthetic demo content not intended for training corpora. CSV files prepend a `#`-prefixed comment line; JSON files wrap records in `{"_canary": {...}, "records": [...]}`.
- **Schema entries** for `activities` and `leads` in [`docs/data_schema.md`](docs/data_schema.md).
- **SOQL-first export instructions** in [`examples/bring_your_own_data.md`](examples/bring_your_own_data.md) for Task, Event, and Lead objects (followed by Data Loader and Salesforce Reports paths).
- **Copy-paste prompts** for both new skills in [`examples/prompts.md`](examples/prompts.md).
- **Sample outputs** for both new skills at `skills/activity-capture-diagnostic/examples/sample_output.md` and `skills/lead-routing-rule-analyzer/examples/sample_output.md`, generated against the bundled synthetic dataset.

### Changed

- **salesforce-revops-audit routing.** Two new recommendation lines added in-place:
  - Dimension 1 (Pipeline Health) now recommends `lead-routing-rule-analyzer` if pipeline coverage is below 2.5x AND lead data is available.
  - Dimension 2 (Data Quality) now recommends `activity-capture-diagnostic` if more than 20% of open opportunities have stale activity.
  - Dimension 5 (Process Integrity) now recommends `activity-capture-diagnostic` when phantom-progression flags fire, and points the residual remediation reference at v0.3 (was v0.2).
- **Audit's "What the audit cannot tell you" section.** Activity capture and lead routing moved from "planned for v0.2" to now-available with pointers to the new skills. Compensation calculation errors moved to "planned for v0.3."
- **README.md** updated to six total skills, with the new diagnostic tier, the v0.2 release noted in the roadmap, and the column-mapping capability called out.
- **All four v0.1 SKILL.md files** received a new "Column discernment" section between "Data sources" and "Process," telling Claude how to read the column-mapping catalog and produce a Mapping Report at the top of its output.
- **examples/bring_your_own_data.md** notes the rename script is now optional.

### Notes

- The v0.1 opportunities, forecast_history, quarterly_actuals, and company datasets are bit-for-bit identical to v0.1. The synthetic data generator was extended with new generator functions for activities and leads; the existing RNG sequence is preserved, so all v0.1 sample outputs remain reproducible.
- No breaking changes to existing skill outputs. v0.1 prompts work identically.

## [0.1.0] 2026-05-20

Initial release.

### Added

- **salesforce-revops-audit**. The entry-point skill. Grades a Salesforce org A through F across five dimensions (Pipeline Health, Data Quality, Forecast Hygiene, Deal Integrity, Process Integrity) and produces a prioritized remediation queue mapped to the remediation skills.
- **forecast-call-prep**. Produces a one-page executive summary for a weekly forecast call, reconciling rep commit, pipeline-weighted forecast, and historical model forecast. Names risks, upsides, and coverage concerns by rep, with a recommended position for the leader to take into the call.
- **pipeline-hygiene-audit**. Scans open pipeline for hygiene issues (stale activity, missing next steps, unrealistic close dates, stage-amount mismatches, probability-stage mismatches, orphaned opps) and produces a per-rep punchlist ranked by severity.
- **deal-investigator**. Produces a structured deal review for a single opportunity, answering the five questions a CRO would ask in a deal review, with risk indicators and a Commit / BestCase / Pipeline / Omit recommendation.
- **Synthetic Northwind Cloud dataset**. 10 sales reps with varied calibration profiles, 251 opportunities, 12 weeks of forecast history, quarterly actuals. Deterministic via fixed seed.
- **tools/rename_columns.py**. Deterministic Salesforce-export rename helper covering the standard Salesforce field names and common Reports and Data Loader variations.
- **docs/data_schema.md**. Canonical schema reference for opportunities and forecast history.
- **examples/bring_your_own_data.md**. Export-and-rename instructions for users bringing real Salesforce data via SOQL, Data Loader, or Salesforce Reports.
- **examples/prompts.md**. Copy-paste prompts for invoking each skill.
- **examples/sample_audit_run.md**. Worked example of `salesforce-revops-audit` against the bundled dataset.
- **TESTING.md**. Test sequence for validating skills against the synthetic data and against a real Salesforce org.

[0.2.0]: https://github.com/elijeangilles/revops-skills/releases/tag/v0.2.0
[0.1.0]: https://github.com/elijeangilles/revops-skills/releases/tag/v0.1.0

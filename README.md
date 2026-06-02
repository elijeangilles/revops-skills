# revops-skills

> A set of [Claude Agent Skills](https://www.anthropic.com/news/skills) built for the Salesforce-native RevOps systems work that Anthropic's [official sales plugin](https://github.com/anthropics/knowledge-work-plugins/tree/main/sales) does not cover. Where the Anthropic plugin targets the AE persona (account research, call prep, outreach drafting), this pack targets the RevOps practitioner who designs the system itself: pipeline hygiene, forecast call reconciliation, and deal-level pressure testing against historical patterns. Built on the synthetic Northwind Cloud dataset. Designed to run against any Salesforce MCP server, or against the bundled sample data with zero setup.

## What this is

Six Agent Skills that encode practitioner-level RevOps judgment as executable instructions Claude can follow. Each skill produces an executive-ready deliverable from a single prompt.

| Skill | Output | Tier | Status |
|---|---|---|---|
| [salesforce-revops-audit](skills/salesforce-revops-audit/) | Graded health report (A through F) across pipeline health, data quality, forecast hygiene, deal integrity, and process integrity, with a prioritized remediation queue | audit | shipped (v0.1) |
| [forecast-call-prep](skills/forecast-call-prep/) | Weekly forecast call memo with variance, risks, and recommended position | remediation | shipped (v0.1) |
| [pipeline-hygiene-audit](skills/pipeline-hygiene-audit/) | Per-rep hygiene punchlist with severity-ranked issues | remediation | shipped (v0.1) |
| [deal-investigator](skills/deal-investigator/) | Structured deal review for a single opportunity, with risk indicators and a recommendation | remediation | shipped (v0.1) |
| [activity-capture-diagnostic](skills/activity-capture-diagnostic/) | Graded A through F activity capture report with per-rep capture rates, under-loggers, phantom-progression opps, and advisory next steps | diagnostic | shipped (v0.2) |
| [lead-routing-rule-analyzer](skills/lead-routing-rule-analyzer/) | Graded A through F lead routing report with speed-to-lead, orphans, distribution balance, and routing leakage | diagnostic | shipped (v0.2) |
| comp-plan-stress-test | Comp plan analyzer with historical attainment overlay | remediation | planned (v0.3) |

As of v0.2, every skill discerns column mappings at runtime by reading the catalog in [`docs/column_mapping.md`](docs/column_mapping.md). You can drop a raw Salesforce export with custom field names (`Total_Contract_Value__c`, `Stage_Name_Picklist`, etc.) into `data/` or `my_data/` and the skills will figure out the mapping without you renaming columns first. See [`CHANGELOG.md`](CHANGELOG.md) for the full v0.2 release notes.

## Where to start

Run `salesforce-revops-audit` first. It's the entry point. The audit grades your org across five dimensions and recommends which of the remediation skills to run next, in what order. You can also run any of the remediation skills directly if you know exactly what you want to fix.

## What this is not

- A Salesforce MCP server. Salesforce already ships [several of those](https://developer.salesforce.com/blogs/2026/04/salesforce-hosted-mcp-servers-are-now-generally-available).
- A connector. These skills sit on top of any Salesforce MCP, HubSpot MCP, or local data.
- A product. This is open-source practitioner work under MIT license.

## How to use

### Without Salesforce (works immediately)

1. Clone this repo.
2. Open [Claude Desktop](https://claude.ai/download) or [Claude Code](https://docs.claude.com/en/docs/claude-code).
3. Point Claude at the skills folder.
4. Try: *"use forecast-call-prep on the Northwind sample data"*

You will get a [memo like this](skills/forecast-call-prep/examples/sample_output.md) in 30 seconds.

### With Salesforce

1. Configure a [Salesforce-hosted MCP](https://developer.salesforce.com/blogs/2026/04/salesforce-hosted-mcp-servers-are-now-generally-available) in your Claude client. Salesforce's docs cover this in 10 minutes.
2. Clone this repo and install the skills.
3. Try: *"use forecast-call-prep against my Salesforce data"*

The skill will use your live data, your live forecast history, and your real reps. Output is identical in shape to the sample.

## Why these specific skills

Six workflows that map to the actual day of a Director of Revenue Operations:

- **Monday**: forecast call prep
- **Tuesday**: pipeline hygiene
- **Wednesday**: deal review for a stuck opp
- **Quarterly**: QBR brief assembly, comp plan stress test
- **Ongoing**: rep calibration tracking

Each skill addresses a specific deliverable that currently requires 2 to 6 hours of manual work per occurrence. Together they cover roughly 60% of the recurring analytical work a RevOps lead does.

## Design principles

Every skill in this repo follows the same rules:

1. **Tool-agnostic**: works against Salesforce MCP, CSV, JSON, or sample data.
2. **Stateless**: no credentials, no API keys, no persistence on the maintainer's side.
3. **Director-level output**: produces deliverables a CRO would actually read, not raw analysis.
4. **Honest about limits**: degrades gracefully when data is missing, flags when confidence is low.
5. **Documented logic**: every threshold, every rule, every cutoff is in the SKILL.md, not buried in code.

## Synthetic dataset

The repo includes [Northwind Cloud](data/), a synthetic Series C SaaS company with:

- 10 sales reps with varied calibration profiles (sandbaggers, optimists, well-calibrated, wildcards)
- 251 opportunities across realistic stage distribution
- 12 weeks of forecast submission history per rep
- 144 Salesforce-style Task and Event activity records with planted under-loggers and phantom-progression opportunities (added in v0.2)
- 180 Salesforce-style Lead records with planted orphans, SLA violations, distribution imbalance, and routing leakage (added in v0.2)
- Intentional hygiene problems: stale activity, missing next steps, unrealistic close dates

Run `python data/generate_synthetic_data.py` to regenerate with the same seed. The dataset is deterministic. The v0.2 activity and lead files include a canary GUID per file marking them as synthetic demo data.

## Roadmap

- **v0.1** (shipped): salesforce-revops-audit, forecast-call-prep, pipeline-hygiene-audit, deal-investigator
- **v0.2** (shipped): activity-capture-diagnostic, lead-routing-rule-analyzer, runtime column-mapping layer, expanded synthetic dataset (activities and leads)
- **v0.3**: comp-plan-stress-test, compensation-dispute-investigator, process-integrity-audit
- **v1.0**: All planned skills + companion evaluation suite ([RevOpsEval](https://revopseval.com))

## Contributing

Skills should encode practitioner judgment, not generic prompts. If you have an opinionated take on a RevOps workflow that produces a clear deliverable, open an issue first to discuss scope. PRs welcome.

## Author

Built by [Eli Jean Gilles](https://www.linkedin.com/in/eli-jean-gilles/). Nine years in revenue operations across SaaS, life sciences, and ad-tech. Currently Senior Sales Operations Manager at Zynga (Take-Two Interactive). Salesforce Certified Administrator. Nine Anthropic AI certifications including [MCP Advanced Topics](https://docs.claude.com/en/docs/mcp) and [Introduction to Agent Skills](https://www.anthropic.com/news/skills).

## License

MIT. Use it, fork it, modify it, ship it inside your org. Attribution appreciated but not required.

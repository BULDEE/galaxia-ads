---
name: ads-warehouse-setup
description: Set up the all-accounts ads warehouse that powers ads-analysis. Configures BigQuery, the native Data Transfer Service connectors (Google Ads + Meta, free), a TikTok pipe, a service account, and the BigQuery MCP wiring. Triggers on "set up ads warehouse", "connect my ad accounts to BigQuery".
---

# Ads warehouse setup

One-time runbook to land Google Ads + Meta + TikTok into one BigQuery dataset and wire the BigQuery MCP so the `ads-analysis` skill can query it.

## 1. BigQuery dataset

In a Google Cloud project, create a dataset (e.g. `ads`) in a region close to you (`EU` or `US`). Enable the BigQuery API and the BigQuery Data Transfer API.

## 2. Native connectors (free, daily)

**Google Ads** (BigQuery Data Transfer Service, native, free):
- BigQuery console > Data transfers > Create transfer > Source "Google Ads".
- Note (2026): new transfers require MFA on the authorizing user (since May 7 2026), and backfills are capped at 37 months (since June 1 2026). Plan history needs accordingly.

**Meta / Facebook Ads** (native DTS connector, free during preview):
- Data transfers > Create transfer > Source "Facebook Ads".
- Restrict to specific ad accounts at setup (multi-account friendly). 12 objects available. Minimum interval 24h.
- Reference: docs.cloud.google.com/bigquery/docs/facebook-ads-transfer

## 3. TikTok pipe (the gap)

No native DTS connector for TikTok. Options:
- **Windsor.ai** (from ~$19/mo): cheapest, near-zero effort, TikTok to BigQuery.
- Supermetrics / Fivetran / Dataslayer: enterprise-grade, more expensive.
- Self-host an open-source TikTok Ads extractor writing to BigQuery (more effort).

Pick Windsor.ai for a small portfolio; revisit if volume grows.

## 4. Service account (for the agent, non-interactive)

For an autonomous agent (Hermes) or headless use, create a service account with the role `roles/bigquery.dataViewer` on the dataset and `roles/bigquery.jobUser` on the project. Download its JSON key. Scope: `https://www.googleapis.com/auth/bigquery`.

## 5. Wire the BigQuery MCP

**Claude Code** (this plugin's `.mcp.json` already points at the managed endpoint):
```json
{ "mcpServers": { "bigquery": { "type": "http", "url": "https://bigquery.googleapis.com/mcp" } } }
```
Claude Code handles the Google OAuth on first use. The managed server exposes `list_dataset_ids`, `list_table_ids`, `get_table_info`, `execute_sql`, `execute_sql_readonly` (queries cap at 3 min / 3000 rows).

**Hermes / self-host** (more control, service-account auth): run the open-source `googleapis/mcp-toolbox` (GA) with the service-account JSON, expose it over HTTP, and add it to Hermes `~/.hermes/config.yaml` under `mcp_servers`.

## 6. Verify

- `list_dataset_ids` returns your `ads` dataset.
- `list_table_ids` shows the connector tables (e.g. `google_ads_*`, `meta_ads_*`, `tiktok_*`).
- A small `execute_sql_readonly` (e.g. `SELECT COUNT(*) FROM ...`) returns rows.

Then the `ads-analysis` skill can run deep SQL over all accounts.

## Note on scope

This warehouse holds raw, platform self-reported numbers. It is the exploration substrate, not the attribution layer. Attributed ROAS/MER stays in the attribution product (Metrikia MCP). See the `ads-analysis` escalation rule.

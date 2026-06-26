---
name: ads-analysis
description: Deep ad-hoc analysis of ad performance across all accounts via BigQuery SQL (Google Ads + Meta + TikTok in one warehouse). Use for granular cohorts, cross-platform joins, creative-level breakdowns, anomaly detection. Triggers on "analyze my ads", "cross-platform ad query", "ad-hoc ads SQL".
---

# Ads analysis (BigQuery)

Deep, ad-hoc analysis over the ads warehouse. This skill is for the questions that a fixed attribution product cannot answer out of the box: arbitrary SQL across all accounts and platforms, joined, on full history.

## Escalation rule (read first, do this every time)

There are two read layers. Pick the right one, do not waste a BigQuery query on a question the attribution layer answers in one call.

1. **Attribution / business-truth questions** (ROAS, MER, CPA, "which campaign actually drove revenue", attributed conversions): use the **Metrikia MCP** (`get_metrics`, `get_campaign_performance`, `get_attribution_journey`, `ask_diana`). It returns MTA/CRM-attributed numbers. This is the source of truth. Do NOT recompute ROAS from raw BigQuery rows: BigQuery has platform self-reported numbers, not attributed business truth, and re-deriving it would contradict the attribution layer.

2. **Granular / novel / cross-platform questions outside that fixed schema** (custom cohorts, hour-of-day spend curves, creative-level joins across Meta + Google + TikTok, raw daily spike detection, anything not pre-modeled): use **BigQuery** (`execute_sql_readonly` on the warehouse).

If unsure, ask the attribution layer first; only fall back to BigQuery when its schema cannot express the answer.

## Prerequisite

The BigQuery MCP server is wired (see this plugin's `.mcp.json` and the `ads-warehouse-setup` skill). The ads warehouse must be fed by BigQuery Data Transfer Service (Google Ads + Meta, native) and a TikTok pipe (Windsor.ai or similar). Tools: `list_dataset_ids`, `list_table_ids`, `get_table_info`, `execute_sql_readonly`.

## Procedure

1. **Confirm escalation**: is this an attribution question (use Metrikia) or a granular/novel one (use BigQuery)? State which and why.
2. **Discover schema**: `list_table_ids` on the ads dataset, `get_table_info` on the relevant tables before writing SQL (table and column names vary by connector).
3. **Write read-only SQL**: prefer `execute_sql_readonly`. Mind the limits: queries cap at 3 minutes and return up to 3,000 rows. Aggregate in SQL, do not pull raw rows to count in the agent.
4. **Normalize across platforms**: spend, impressions, clicks, conversions live under different column names per source. Map them explicitly in the query (a CTE per platform, then UNION).
5. **Report**: lead with the answer, then the SQL used, then caveats (these are platform self-reported numbers, daily-lagged; for attributed ROAS see Metrikia).

## Example shapes (adapt to the real schema)

Cross-platform spend last 30 days:
```sql
WITH g AS (SELECT 'google' AS platform, date, SUM(cost_micros)/1e6 AS spend FROM `proj.ads.google_ads_stats` WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) GROUP BY date),
     m AS (SELECT 'meta' AS platform, date, SUM(spend) AS spend FROM `proj.ads.meta_ads_insights` WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) GROUP BY date)
SELECT platform, SUM(spend) AS spend_30d FROM (SELECT * FROM g UNION ALL SELECT * FROM m) GROUP BY platform ORDER BY spend_30d DESC;
```

## Pitfalls

- Never present a BigQuery-derived ROAS as business truth. Raw warehouse numbers are platform self-reported (last-click). Attributed ROAS is Metrikia's job.
- Data is daily-lagged (DTS floor is 24h) and platforms restate conversions for 24-72h. Do not alarm on the last 1-2 days of raw data.
- Always inspect the schema first; connector table layouts change.

## Verification

The answer states which layer was used and why. Any ROAS/attribution claim came from Metrikia, not BigQuery. SQL is aggregated server-side and respects the 3,000-row cap.

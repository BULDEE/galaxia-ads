# galaxia-ads

Claude Code plugin for ad analysis and operations, layered around an attribution source of truth.

## The idea: layer, do not duplicate

As of 2026, Google, Meta and TikTok all ship official MCP servers, and BigQuery has an official MCP. That does not mean an agent should hit raw platform APIs to compute performance. Performance attribution is the hardest, highest-value problem, and it belongs in a dedicated attribution layer. This plugin sits around that layer, covering the two things attribution products do not: deep ad-hoc SQL, and write/ops.

```
Attribution (source of truth)   ->  Metrikia MCP
  "true ROAS / MER / which campaign actually drove revenue"
  MTA + CRM attributed, pre-modeled, the hot path. Not in this plugin.

Deep / ad-hoc analysis          ->  BigQuery MCP (this plugin)
  "any granular SQL across all accounts joined, on history"
  Raw warehouse, full SQL. Native DTS for Google + Meta (free), Windsor.ai for TikTok.

Write / ops                     ->  Provider MCPs (this plugin)
  "pause / scale / create (lands PAUSED)"
  Meta / Google Ads / TikTok official MCPs, per-account OAuth.
```

The plugin **never recomputes ROAS or attribution**. That stays in the attribution layer (Metrikia MCP). Re-deriving it from raw BigQuery rows would use un-attributed platform numbers and contradict the source of truth.

## Skills

| Skill | Role |
|-------|------|
| `ads-warehouse-setup` | One-time setup: BigQuery dataset, native DTS connectors (Google + Meta, free), TikTok pipe, service account, BigQuery MCP wiring. |
| `ads-analysis` | Deep ad-hoc SQL over the ads warehouse, with an explicit escalation rule (attribution layer first, BigQuery only for what it cannot answer). |
| `ads-ops` | Write/ops over the official provider MCPs (Meta, Google Ads, TikTok), with the PAUSED-by-default guardrail. |

## MCP servers

`.mcp.json` wires the official Google **BigQuery MCP** (managed endpoint `https://bigquery.googleapis.com/mcp`). Claude Code handles the Google OAuth on first use. Provider MCPs (Meta `mcp.facebook.com/ads`, Google Ads, TikTok, or unified Pipeboard) are wired per the `ads-ops` skill as needed.

## Requirements

- A Google Cloud project with BigQuery and the Data Transfer API enabled.
- Ad accounts connected via DTS (Google + Meta) and a TikTok pipe (Windsor.ai or similar).
- For attributed ROAS: an attribution layer such as the Metrikia MCP (separate, not bundled here).

## Notes (2026)

- BigQuery MCP queries cap at 3 minutes / 3,000 rows. Aggregate in SQL.
- Native DTS is daily (24h floor). Platforms restate conversions for 24-72h.
- Google Ads DTS requires MFA (since May 2026) and caps backfills at 37 months (since June 2026).
- Provider MCPs are beta: pin versions, expect churn. Created entities land PAUSED.

## License

Apache-2.0. (c) Alexandre Mallet / BULDEE.

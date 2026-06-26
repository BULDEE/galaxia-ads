# galaxia-ads

Claude Code plugin for ad analysis and operations, orchestrated around an attribution source of truth.

## Why this exists (the honest version)

As of 2026 every platform ships an official MCP (Meta, Google Ads, TikTok), and BigQuery has an official MCP. So why a plugin? Because the provider MCPs are walled gardens with self-reported, last-click numbers that cannot join across platforms. They leave two gaps:

1. **Attributed business truth** (real ROAS / MER via MTA + CRM): filled by the **Metrikia MCP**.
2. **Arbitrary cross-platform analysis**: filled by a unified provider MCP (reads) or BigQuery (full historical SQL).

This plugin is the **router and orchestration** around those layers. It does not re-implement attribution, and it does not make BigQuery the centerpiece.

## The stack

```
Metrikia MCP        ->  the brain, wired by default (.mcp.json)
  "true ROAS / MER / which campaign actually drove revenue"
  MTA + CRM attributed. The default answer for performance questions.

Provider MCPs       ->  reads + write/ops
  Meta / Google Ads / TikTok official, or unified Pipeboard (one auth, cross-platform).
  Live platform state and actions. Numbers are self-reported (sanity check, not truth).

BigQuery MCP        ->  OPTIONAL deep-SQL escape hatch
  Only when a question needs arbitrary cross-platform/historical SQL the Metrikia schema
  cannot express, and only if an ads warehouse is set up. Not required for most use.
```

The plugin **never recomputes ROAS or attribution**. That stays in Metrikia. Re-deriving it from raw BigQuery rows would use un-attributed platform numbers and contradict the source of truth.

## Skills

| Skill | Role |
|-------|------|
| `ads-router` | The brain. Routes any ad question to the right layer (Metrikia first, provider for ops/reads, BigQuery only when needed). Read this first. |
| `ads-ops` | Write/ops over provider MCPs (Meta, Google Ads, TikTok, or unified Pipeboard), with the PAUSED-by-default guardrail. |
| `ads-analysis` | OPTIONAL: deep ad-hoc SQL over a BigQuery ads warehouse, for what Metrikia and provider reads cannot answer. |
| `ads-warehouse-setup` | OPTIONAL: one-time setup of the BigQuery warehouse (native DTS for Google + Meta, Windsor.ai for TikTok) and the BigQuery MCP wiring. Only if you need the deep-SQL layer. |

## MCP servers (`.mcp.json`)

- `metrikia` (`https://mcp.metrikia.io/api/v1/mcp`): the attribution brain, wired by default. Claude Code handles OAuth on first use. (If you also run the standalone Metrikia plugin, you only need one.)
- `bigquery` (`https://bigquery.googleapis.com/mcp`): the optional deep-SQL layer. Harmless if no warehouse exists; the router only reaches for it when needed.

Provider MCPs (Meta `mcp.facebook.com/ads`, Google Ads, TikTok, or unified Pipeboard) are wired per the `ads-ops` skill, since they need per-account OAuth.

## Requirements

- A Metrikia account + MCP access (the attribution brain). Required.
- A provider MCP for reads/ops (Pipeboard recommended). For actions.
- BigQuery + a fed ads warehouse. Optional, only for deep SQL.

## Notes (2026)

- Provider and BigQuery MCPs are beta/preview: pin versions, expect churn. Created entities land PAUSED.
- BigQuery MCP queries cap at 3 minutes / 3,000 rows. Native DTS is daily (24h floor).

## License

Apache-2.0. (c) Alexandre Mallet / BULDEE.

---
name: ads-router
description: The brain of the ads stack. Given any ad question, routes it to the right MCP layer: Metrikia for attributed ROAS (default), provider MCPs for cross-platform reads and write-ops, BigQuery only for arbitrary historical SQL a warehouse exists. Triggers on any ads question ("how are my ads doing", "which campaign performs", "pause X", "spend by platform").
---

# Ads router

Read this first on ANY ad question. It decides which layer answers, so the agent never re-derives attribution or wastes a heavy query. The layers are complementary, not interchangeable.

## Decision tree

```
Is the question about TRUE performance / business value?
  ("ROAS, MER, CPA, which campaign actually drove revenue, attributed conversions")
  -> Metrikia MCP. This is the source of truth (MTA + CRM attribution).
     get_metrics, get_campaign_performance, get_attribution_journey, get_budget_advice, ask_diana.
     Do NOT compute this anywhere else. Platform/warehouse numbers are self-reported, not attributed.

Is it a WRITE / OPS action?
  ("pause, scale, shift budget, create draft")
  -> Provider MCP (ads-ops skill). Meta / Google Ads / TikTok official, or unified Pipeboard.
     Decide from Metrikia truth, act here. Created entities land PAUSED.

Is it a platform-native READ across one or more platforms?
  ("current delivery, today's spend per platform, account list, a platform's own number")
  -> Provider MCP (unified Pipeboard recommended for cross-platform reads in one auth).
     Treat numbers as platform self-reported (last-click), a sanity check, not truth.

Is it an ARBITRARY granular SQL question Metrikia's fixed schema cannot express,
needing joins across all platforms and full history?
  ("hour-of-day spend curves joined across Meta+Google+TikTok over 18 months, custom cohorts")
  -> BigQuery MCP (ads-analysis skill). OPTIONAL: only if the ads warehouse is set up
     (see ads-warehouse-setup). If no warehouse exists, say so and offer the simpler path
     (Metrikia + provider reads) instead of pushing a warehouse build.
```

## Hard rules

1. **Attribution is Metrikia, always.** Never present a platform or BigQuery ROAS as business truth. If asked for ROAS, call Metrikia. If Metrikia is unavailable, say the number is unattributed, do not fake it.
2. **BigQuery is the exception, not the default.** Most questions resolve with Metrikia (truth) plus a provider read (live platform state). Reach for BigQuery only when the question genuinely needs arbitrary cross-platform/historical SQL. Do not propose building a warehouse for a question Metrikia answers in one call.
3. **Decide from truth, act from ops.** Performance decisions use Metrikia. Writes go through provider MCPs with the PAUSED guardrail.

## Why this stack (for the record)

Provider MCPs are walled gardens with self-reported, last-click numbers and cannot join across platforms. They leave two gaps: attributed business truth (filled by Metrikia) and arbitrary cross-platform analysis (filled by a unified provider MCP like Pipeboard for reads, or BigQuery for full SQL). So the coherent stack is Metrikia as the brain, provider MCPs for reads/ops, and BigQuery as an optional deep-analysis escape hatch, not the centerpiece.

## Prerequisites

- Metrikia MCP wired (this plugin's `.mcp.json`, OAuth on first use). Required.
- Provider MCP for ops/reads (ads-ops skill). Recommended: Pipeboard unified.
- BigQuery MCP + warehouse (ads-warehouse-setup). Optional.

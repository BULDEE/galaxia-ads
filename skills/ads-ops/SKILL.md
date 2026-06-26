---
name: ads-ops
description: Write and operations actions on ad platforms via the official provider MCPs (Meta Ads, Google Ads, TikTok). Pause/scale campaigns, shift budgets, create drafts. Read-only platform numbers as a sanity cross-check. Triggers on "pause campaign", "shift budget", "create ad draft".
---

# Ads operations (provider MCPs)

Thin layer for ACTING on ad platforms, plus platform-native reads as a sanity cross-check. Reads for decision-making stay in the attribution layer (Metrikia) and the warehouse (BigQuery); this skill is for write/ops.

## Provider MCPs (official, 2026, beta)

| Platform | Endpoint / repo | R/W | Notes |
|----------|-----------------|-----|-------|
| Meta Ads | `mcp.facebook.com/ads` (AI Ads Connectors) | read + write | 29 tools, Business OAuth, per-account, no long-lived tokens stored |
| Google Ads | `github.com/google-marketing-solutions/google_ads_mcp` (self-host) | read-only | GAQL `search`, needs a Developer Token |
| TikTok Ads | platform-hosted | read + write | full lifecycle |
| Unified (community) | `github.com/pipeboard-co/meta-ads-mcp` (Pipeboard) | read + write | one auth for Meta+Google+TikTok+Snap+Reddit, mature |

Wire the ones you use in Claude Code `.mcp.json` or Hermes `~/.hermes/config.yaml`. These are beta: pin versions, expect churn.

## Safety guardrails (non-negotiable)

1. **Created entities land PAUSED.** The official MCPs create campaigns/ads in a paused state on purpose. Never auto-activate. Surface the created draft to the user and require an explicit human go before activation.
2. **Confirm before any write.** Pause, budget change, status change: state the exact action and the target (account, campaign id, current value, new value), then ask before executing.
3. **No bulk destructive actions** without an itemized list the user approved.

## Procedure

1. **Decide from truth, act from ops.** Base the decision on Metrikia (attributed ROAS) and/or BigQuery (granular analysis). Do NOT decide from platform self-reported numbers.
2. **Resolve the target**: list accounts/campaigns via the provider MCP, confirm the exact id with the user.
3. **State the action**: account, campaign, current state/value, proposed change.
4. **Confirm**, then execute the write via the provider MCP.
5. **Verify**: read back the new state from the provider MCP and report it.

## Sanity cross-check (read)

Platform self-reported numbers (last-click, platform-attributed) are useful only to sanity-check the warehouse pipeline or spot a platform-side delivery issue. They are NOT the source of truth for performance. For any ROAS/performance decision, use Metrikia. State clearly when a number is platform self-reported.

## Pitfalls

- Never present platform self-reported ROAS as business truth.
- Never activate a created entity automatically.
- Beta MCPs: a tool may change or fail. Degrade gracefully, tell the user, do not retry a write blindly.

## Verification

Every write was confirmed by the user beforehand and read back afterward. Created entities are left PAUSED. No performance decision was made on platform self-reported numbers alone.

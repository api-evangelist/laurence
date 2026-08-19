---
name: Diagnose hourly Amazon Marketing Stream performance and bid moves
description: >-
  Use Laurence's pre-stored Amazon Marketing Stream data to find the hours where performance moved,
  and line those hours up against the bid updates that were live at the time. This is the surface
  Amazon's own Ads MCP server does not cover.
api: mcp/laurence-mcp.yml
endpoint: https://mcp.laurence.com/mcp
operations:
  - list_allowed_ads_profiles
  - get_campaigns
  - get_ad_groups
  - get_ams_events
  - get_bids_and_observations
  - get_campaign_keywords
generated: '2026-08-13'
method: generated
source: >-
  https://www.laurence.com/blog/laurence-mcp-launch (tool names, parameters, interval semantics and
  the AMS positioning)
---

# Diagnose hourly stream performance and bid moves

Amazon Marketing Stream (AMS) is the hourly event firehose — impressions, clicks, cost, conversions
and placements at keyword granularity. Amazon's own Ads MCP server does not expose it, and standing
it up yourself means an SNS → SQS → Lambda pipeline into a warehouse. Laurence has already landed it
in ClickHouse, so `get_ams_events` answers in seconds rather than the minutes an on-demand Amazon
report takes. **This is the reason to reach for this server rather than Amazon's.**

**Prerequisite:** complete `laurence-connect-and-scope` first.

## Steps

1. **Resolve scope.** `list_allowed_ads_profiles` → a valid `profile_id`.

2. **Narrow to the campaigns that matter.** `get_campaigns`, and `get_ad_groups` where the question
   is ad-group shaped. Collect `campaign_ids` — passing them as a filter is what keeps a stream pull
   from hitting the row cap.

3. **Choose granularity deliberately.** `get_ams_events` takes an `interval`:
   - **daily** — rollups per Pacific day. Use this to find *which day* moved.
   - **hourly** — raw rows, **capped by `limit`**. Use this only after you know which day, and only
     with `campaign_ids` (and `keyword_id` where you have one) set.

   Going straight to hourly across a whole account is the standard way to get a silently truncated
   answer. Find the day daily, then zoom.

   Optional filters worth using: `match_type` and `placement` — placement in particular explains a
   lot of apparent keyword volatility, because top-of-search behaves nothing like rest-of-search.

4. **Line the movement up against the bids.** `get_bids_and_observations` returns bid updates paired
   with the **same-hour performance snapshot** (metrics present when available). It accepts
   `keyword_id`, `campaign_ids`, Pacific date bounds and a row `limit`. This is the tool that answers
   *"did the bid change cause this, or did the market?"* — a spend spike at 14:00 with no bid update
   in the same hour is a market move, not a Laurence action.

5. **State the current position.** `get_campaign_keywords` returns keywords and their current bids
   for the campaigns in question, so a recommendation is framed against where the bid actually sits
   now rather than where it sat during the window.

## Rules

- **Everything is Pacific.** `get_ams_events` rolls up per Pacific day and
  `get_bids_and_observations` takes Pacific date bounds. Say "Pacific" in your output — an hourly
  finding stated in the wrong timezone is worse than no finding.
- **`limit` truncates without telling you.** No cursor, no offset, no next-page token. If a result
  count equals your limit exactly, assume you were cut off and narrow the filters.
- **"Metrics when present."** `get_bids_and_observations` does not guarantee a metrics snapshot on
  every row. Do not average across rows as though every row carried metrics, and do not treat a
  missing snapshot as a zero.
- **Read-only.** You can explain a bid move; you cannot make one. Laurence's optimizer reprices
  hourly on its own.
- **Never invent an identifier.** Every `profile_id`, `campaign_id`, `ad_group_id` and `keyword_id`
  must have come from a prior tool response in this session.

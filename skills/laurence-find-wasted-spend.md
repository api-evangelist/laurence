---
name: Find wasted Amazon Ads spend from search terms and negatives
description: >-
  Locate search terms that take clicks without producing orders for a given Amazon Ads profile, then
  check whether those terms are already negated, so the user gets a defensible waste list rather than
  a raw report dump.
api: mcp/laurence-mcp.yml
endpoint: https://mcp.laurence.com/mcp
operations:
  - list_allowed_ads_profiles
  - get_campaigns
  - get_search_term_data
  - get_campaign_keywords
  - get_campaign_negative_keywords
  - get_daily_sales
generated: '2026-08-13'
method: generated
source: >-
  https://www.laurence.com/blog/laurence-mcp-launch (tool names, parameters and stated example
  prompts)
---

# Find wasted Amazon Ads spend

Laurence's own launch documentation names this as a first-class use case: *"Show me search terms
with clicks but no orders in the last 14 days."* Do it properly.

**Prerequisite:** complete `laurence-connect-and-scope` first. You need a valid `profile_id`.

## Steps

1. **Resolve scope.** `list_allowed_ads_profiles` → pick the `profile_id` for the store the user
   means. If more than one is allowed and the user was ambiguous, ask — do not pick for them.

2. **Frame the account.** `get_campaigns` with that `profile_id` returns the campaigns (Sponsored
   Products, Sponsored Brands, Sponsored Display). Note which `campaign_ids` are in scope for the
   question; you will pass them as filters rather than pulling the whole account.

3. **Pull search terms.** `get_search_term_data` with `profile_id`, the relevant `campaign_ids`, and
   a `limit`. This returns queries with attributed performance. Terms carrying clicks and spend with
   no attributed orders are the waste candidates.

   **Watch the truncation.** `limit` is a hard row cap with no pagination. If you hit it, narrow by
   `campaign_ids` or by `keyword_id` and pull again — do not report a total computed from a
   truncated set.

4. **Check what is already negated before recommending anything.**
   `get_campaign_negative_keywords` takes ad group IDs; get those from `get_ad_groups` for the
   campaigns in question. A term already on a negative list is not an actionable finding, and
   recommending it wastes the user's attention. This step is what separates a useful answer from a
   report dump.

5. **Cross-check the matched keywords.** `get_campaign_keywords` for the same `campaign_ids` returns
   keywords and their current bids. A wasteful *search term* is often the result of a broad or
   phrase keyword, not an exact one — name the keyword driving the waste, not just the query.

6. **Put it in context.** `get_daily_sales` for the profile over the same window (optionally filtered
   by ASIN) gives units and revenue, so the waste number can be expressed against actual sales rather
   than in isolation.

## Rules

- **Pacific time.** Any date window you state must be `America/Los_Angeles`. "Last 14 days" resolves
  against Pacific days, not the user's local clock or UTC.
- **You cannot fix it here.** The tool surface is read-only — there is no tool to add a negative
  keyword or lower a bid. Report the finding and say explicitly that applying it happens in Laurence
  or Seller Central, not through this server.
- **Attribution lag is real.** Recent days under-report orders on Amazon. Do not call a term
  "wasted" on one or two days of data; say what window you used and note the lag.
- **Never invent a `profile_id`, `campaign_id`, `keyword_id` or ASIN.** Every identifier you pass
  must have come out of a prior tool response in this same session.

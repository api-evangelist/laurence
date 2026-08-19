---
name: Connect to Laurence MCP and resolve your Amazon Ads scope
description: >-
  Authenticate against the Laurence MCP server and establish which Amazon Ads profile_id values the
  signed-in user is permitted to query, before attempting any data tool. Every other Laurence skill
  depends on this one.
api: mcp/laurence-mcp.yml
endpoint: https://mcp.laurence.com/mcp
operations:
  - list_allowed_ads_profiles
generated: '2026-08-13'
method: generated
source: >-
  https://www.laurence.com/blog/laurence-mcp-launch (tool names and semantics),
  live probes 2026-08-13 (auth behavior)
---

# Connect to Laurence MCP and resolve your Amazon Ads scope

Laurence MCP is a **remote, OAuth-protected, read-only** MCP server over Amazon Advertising and
Amazon Marketing Stream (AMS) data. It is available to Laurence customers only. There is no API key
and no stdio package.

## 1. Register the server

Use the remote-HTTP transport. The provider's documented endpoint is the Modal deployment URL, but
the server's own RFC 9728 metadata names the branded host as canonical — either resolves to the
same server.

```
claude mcp add --transport http laurence https://mcp.laurence.com/mcp
```

Codex: `codex mcp add laurence --url https://mcp.laurence.com/mcp`, then
`codex mcp login laurence --scopes laurence:mcp`.

## 2. Expect a 401 on the first call, and follow it

An unauthenticated `initialize` or `tools/list` returns:

```
HTTP/2 401
www-authenticate: Bearer error="invalid_token", error_description="Authentication required",
  resource_metadata="https://mcp.laurence.com/.well-known/oauth-protected-resource/mcp"

{"error": "invalid_token", "error_description": "Authentication required"}
```

**Do not treat this as an outage.** It is the RFC 9728 discovery handshake. Follow
`resource_metadata` to learn the authorization server (`https://www.laurence.com`), then use the
OAuth 2.0 authorization-code flow with **PKCE S256** and scope **`laurence:mcp`**. The
authorization server supports dynamic client registration with
`token_endpoint_auth_methods_supported: ["none"]`, so a public client registers itself — no
pre-issued credential is needed. Sign-in happens in a browser; the session persists in the IDE
afterward.

Full auth profile: `authentication/laurence-authentication.yml`. Scopes: `scopes/laurence-scopes.yml`.

## 3. Always call `list_allowed_ads_profiles` first

Authorization is enforced **per Amazon Ads profile at call time**, not by OAuth scope. The single
`laurence:mcp` scope gates the whole tool surface; it tells you nothing about which stores you may
read.

Call `list_allowed_ads_profiles` before anything else. It returns every `profile_id` the signed-in
user may query. **Every other tool requires a `profile_id` drawn from that list.** Never guess or
carry over a `profile_id` from another account, another user's session, or a prior conversation.

## 4. Know the boundaries before you plan a task

- **Read-only.** All nine tools are reads. There is no write, mutation, bid-change or
  campaign-management tool. If a user asks you to change a bid or pause a campaign, say plainly that
  the MCP surface cannot do it — Laurence's own optimizer performs those actions.
- **Pacific time.** Date bounds and rollups are `America/Los_Angeles`, not UTC. Convert before you
  compare against anything else.
- **Row limits, not cursors.** `get_bids_and_observations`, `get_ams_events` and
  `get_search_term_data` take a `limit`. There is no cursor, offset or next-page token — a large
  result set is **truncated, silently, at the limit**. If a count looks suspiciously round, narrow
  the filters rather than assuming you saw everything.
- **No idempotency keys and no documented rate limits.** Neither is needed for reads, but there is
  also no `Retry-After` or `RateLimit-*` header to pace against; back off conservatively on your own.
  See `rate-limits/laurence-rate-limits.yml`.
- **Errors are OAuth-shaped, not problem+json.** `{"error": ..., "error_description": ...}`. Tool-level
  error semantics are undocumented — see `errors/laurence-problem-types.yml`.

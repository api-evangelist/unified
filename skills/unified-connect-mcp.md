---
name: unified-connect-mcp
description: >-
  Connect to Unified's remote Model Context Protocol server at https://mcp.unified.com/mcp,
  register a client dynamically, complete the OAuth authorization-code + PKCE flow, and verify the
  operations host is healthy before and after. Use this when an agent needs to reach a Unified
  customer's social advertising data and has no prior credentials.
api: unified:unified-mcp-service
operations:
  - root__get
  - health_check_health_get
  - readiness_check_health_ready_get
  - liveness_check_health_live_get
generated: '2026-09-02'
method: generated
source: >-
  openapi/unified-mcp-service-openapi.json (operationIds verified against the fetched spec),
  well-known/unified-oauth-authorization-server.json,
  well-known/unified-oauth-protected-resource.json, mcp/unified-mcp.yml
---

# Connect to the Unified MCP server

Unified's agent surface is a single remote MCP endpoint. Everything below the authorization step
is gated: the tool names, their input schemas and their consequences are **not published
anywhere**, so this skill takes you as far as an authorized session and then stops rather than
guessing at tools that may not exist.

## 1. Confirm the service is up (anonymous)

Two GETs from the published OpenAPI, both unauthenticated:

- `root__get` — `GET https://api.unified.com/` returns the service identity:
  `{"message": "...", "data": {"service": "unified-mcp-service", "version": "...", "build": "...", "environment": "..."}}`.
  Record `version` and `build`. Unified publishes no changelog, so this payload is the only
  version signal that exists — capture it on every run so a change is detectable.
- `health_check_health_get` — `GET https://api.unified.com/health` returns
  `{"status": "healthy", "message": "Service is running"}`.

`readiness_check_health_ready_get` (`/health/ready`) and `liveness_check_health_live_get`
(`/health/live`) return the same `HealthCheckResponse` shape and are the right probes if you are
distinguishing "started" from "ready to serve".

The same document and the same endpoints are served on `https://mcp.unified.com` — it is one
build behind two hostnames.

## 2. Discover the authorization requirements

Do not hardcode endpoints. Read them:

1. `GET https://mcp.unified.com/.well-known/oauth-protected-resource/mcp` (RFC 9728) →
   `{"resource": "https://mcp.unified.com/mcp", "authorization_servers": ["https://mcp.unified.com/"], "scopes_supported": ["openid"], "bearer_methods_supported": ["header"]}`.
   The bare `/.well-known/oauth-protected-resource` path **404s** — the resource suffix is
   required.
2. `GET https://mcp.unified.com/.well-known/oauth-authorization-server` (RFC 8414) → the
   `authorization_endpoint`, `token_endpoint`, `registration_endpoint`, `revocation_endpoint`,
   `code_challenge_methods_supported` and `scopes_supported`.

If you skipped step 2, an unauthenticated `POST https://mcp.unified.com/mcp` will hand you the
same pointer: the 401 carries `WWW-Authenticate: Bearer error="invalid_token", ...,
resource_metadata="https://mcp.unified.com/.well-known/oauth-protected-resource/mcp"`. Following
the challenge is the correct behaviour; retrying the same call with the same token is not.

## 3. Register and authorize

- Register: `POST https://mcp.unified.com/register` (RFC 7591 dynamic client registration). No
  pre-provisioned client id is needed.
- Authorize: `https://mcp.unified.com/authorize`, `response_type=code`, PKCE with
  `code_challenge_method=S256` (the server advertises **only** S256), `scope=openid`.
- Token: `POST https://mcp.unified.com/token`, grant `authorization_code`; refresh with
  `refresh_token`. Client authentication is `client_secret_post` or `client_secret_basic`.
- Revoke on teardown: `POST https://mcp.unified.com/revoke`.

Send the token as `Authorization: Bearer <token>` — `bearer_methods_supported` is `["header"]`
only, so a query-parameter token will not work.

## 4. Open the session and discover tools at runtime

`POST https://mcp.unified.com/mcp` with `Content-Type: application/json`,
`Accept: application/json, text/event-stream`, and the bearer token. Run the MCP handshake, then
`tools/list`.

**Discover, do not assume.** This repository deliberately records no tool list for Unified,
because none is published and an anonymous `tools/list` returns 401. Read the tools and their
`inputSchema` from the live response.

## 5. Operate carefully — what the contract does not tell you

These are real gaps, and an agent should plan around them rather than around an assumption:

- **No error catalogue beyond the auth error.** The OpenAPI declares no 4xx or 5xx responses.
  Observed behaviour: 401 returns `{"error": "...", "error_description": "..."}`; an unrouted path
  returns a plain-text `Not Found`. Do not expect `application/problem+json`.
- **No rate-limit signal.** No `RateLimit-*`, `X-RateLimit-*` or `Retry-After` header is emitted
  on any response. You cannot budget from headers; back off on your own schedule and treat any
  429 or 5xx as a signal to slow down.
- **No documented reversal.** Unified publishes no write surface anonymously, so no cancel,
  refund, void or undo operation and no reversal window is on record. If `tools/list` returns a
  tool that writes, **confirm the reversal path with the customer before calling it** — this
  profile cannot tell you whether the action can be taken back.
- **No idempotency key.** None is documented or observed. Assume a retried write may duplicate.
- **Pre-1.0 versioning.** The production service reports a `0.0.x` version with no published
  versioning or deprecation policy. Re-read `root__get` and compare `version`/`build` between
  runs; a bump may change behaviour without notice.
- **Correlation.** Responses carry an `apigw-requestid` header. Log it — it is the only handle you
  can quote to Unified support (`hello@unified.com`) when something fails.

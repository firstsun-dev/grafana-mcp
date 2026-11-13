# Design: Grafana MCP fixed-destination Worker proxy

## Request flow

```text
ChatGPT / Claude / Claude Code / approved headless MCP client
  | HTTPS https://mcp.firstsun.org/grafana/mcp
  v
Cloudflare Access: grafana-mcp (Managed OAuth + explicit Service Token policy)
  | Cf-Access-Jwt-Assertion
  v
Exact Workers Route: mcp.firstsun.org/grafana/mcp
  v
grafana-mcp-proxy Worker
  | exact Host/path checks; signed JWT verify (issuer/JWKS/Grafana AUD/exp/nbf)
  | sanitize and allowlist request headers; inject private origin bearer
  | fixed-path Streamable HTTP, preserve streaming and session semantics
  v
Scoped HTTP Workers VPC Service binding (single target host:port)
  v
Existing firstsun-nas Tunnel (private route, NOT public ingress)
  v
NAS loopback-only official grafana/mcp-grafana container
  | read-only Grafana Service Account token
  v
Existing private Grafana API -> Prometheus datasource, dashboards and alerts

Unchanged: https://mcp.firstsun.org/anas/mcp -> anas-mcp Custom Domain
```

The NAS port `127.0.0.1:18080` is **illustrative**, not a deployed choice. Preflight must confirm a free port and that the real `cloudflared` process can reach the selected listener. The upstream must be configured to serve `/grafana/mcp` (prefer official base-path support) rather than relying on arbitrary path rewrites.

## Responsibility split

- This repo owns application logic, tests, Wrangler *binding declaration only*, and thin Worker CI caller.
- `infra-config` owns Route, Access application, AUD, Service Token allowlist, VPC Service registration, NAS container, origin secrets and Terraform cutover.
- `.github` owns the cross-service protocol/naming policy and shared CI workflow.
- `anas-mcp` is out of scope. The existing Custom Domain, DNS and Tunnel are not moved.

One service has one independent Access application/AUD; there is no identity, token or proxy fallback between ANAS and Grafana.

## External interface and authentication

Only canonical `https://mcp.firstsun.org/grafana/mcp` is routable for Grafana. Requests to bare `/grafana`, trailing slash, encoded/doubled path, prefix collision, unknown or alternate hosts MUST NOT reach MCP tools. Do not configure `workers.dev` or preview URLs, and do not declare public Worker routes in this repository.

Access owns OAuth RFC 9728/8414 discovery, authorization code + PKCE, allowed redirects and headless Service Token evaluation. The Worker validates the signed `Cf-Access-Jwt-Assertion` using the approved team JWKS and checks algorithm, issuer, Grafana AUD, `exp` and `nbf` before initiating VPC traffic. Use a proven verification pattern from `anas-mcp`, without a generic authentication framework. Missing/invalid assertions fail with generic 401; unavailable verification/config fails closed with generic 503. Do not interpret caller-provided JWT contents before verification. ANAS-only assertions are denied.

Cloudflare-provided OAuth challenges and protected-resource metadata must identify the Grafana resource, not ANAS. Verify live behavior against both services before accepting the implementation.

## Private upstream and secret boundaries

The proxy uses exactly one fixed HTTP VPC Service binding registered against the existing `firstsun-nas` Tunnel; it cannot select other hostnames, ports or services based on request headers or body. No public fallback `fetch(request)`, Tunnel hostname, new DNS record or VPC Network binding.

The proxy reconstructs upstream requests from a minimal allowlist:
- Preserve required MCP request headers (`Accept`, `Content-Type`, `MCP-Protocol-Version`, `Mcp-Session-Id`, `Last-Event-ID` where relevant).
- Never forward inbound `Authorization`, `Cookie`, `Cf-Access-*`, `CF-Access-*`, `X-Grafana-*`, proxy/forwarding headers or unapproved origin-control headers.
- Insert only the Worker-held origin caller credential as `Authorization: Bearer <origin-token>`.
- Keep downstream status, MCP/session headers, content type, cache policy and event-stream framing intact through a safe response-header allowlist.
- Never use the external OAuth token to authenticate directly to the NAS container.

The matching upstream caller secret belongs only to the Worker and NAS MCP server. The Grafana Service Account Token stays on NAS, never in the Worker, Terraform output, clients or logs. Use the approved secrets workflow and pin the official upstream container release/digest.

## MCP transport

Support the methods implemented by the pinned official upstream:
- `POST` for `initialize`, `tools/list` and `tools/call`.
- `GET` for response streams only if supported by that release.
- `DELETE` for session termination only if supported by that release.

Preserve JSON-RPC envelopes, `text/event-stream` responses, session IDs, protocol-version headers, resumability identifiers, cancellation and meaningful status codes. Do not collect SSE bodies, rewrite JSON-RPC messages, cache responses or invent session persistence. Verify end-to-end against real VPC/Tunnel traffic; unit mocks alone are insufficient. Reject unsupported methods and invalid transport headers.

## Read-only and resource protection

NAS runs an independently managed official `grafana/mcp-grafana` container with `--disable-write`, a reviewed minimal tool-category allowlist and a least-privilege Grafana Service Account. Use the existing private Grafana URL and existing Prometheus datasource. Confirm whether datasource query permissions can be restricted in the installed Grafana edition.

Even read-only PromQL may be expensive: adopt conservative request body size, query time-window, concurrency and timeout limits that preserve valid MCP streaming. Validate limits from the pinned upstream and Workers platform before assigning numeric values. The worker must fail closed on VPC/backend outages and avoid unintended load on the monitoring stack.

## Quality, CI and deployment

- Tests: JWT validation, wrong AUD, spoofed headers, canonical path, disallowed methods, fixed upstream, streaming/session headers, cancellation, downstream errors and write-tool unavailability.
- Observability: safe request outcome/latency and backend connectivity only; never log assertions, credentials, PromQL, private metrics or MCP tool content.
- Workflow: thin caller of `firstsun-dev/.github/.github/workflows/_cf-worker-v2.yml@v2`, production-only, checks on branch push, main deploy only after rollout readiness, no additional preview/Dev endpoints.
- Wrangler: `workers_dev=false` and `preview_urls=false`, only service binding/vars needed for Worker; Terraform exclusively manages public Routes and Access.
- Production smoke must not expect an unauthenticated 200 from the protected route. Use safe Access challenge checks and authorized MCP tests in the staged cutover.

## Staged rollout and rollback

1. Read-only preflight: ANAS baseline, route precedence, Access discovery, existing NAS Tunnel `config_src`/`remote_config` drift, VPC permissions/QUIC/SSE compatibility and private origin access.
2. Prepare official pinned NAS container independently of `nas_monitor`, enforce read-only account and origin token, verify local MCP tools.
3. Implement/test the Proxy Worker with no public alternate route or OAuth bypass.
4. In `infra-config`, create approved private VPC/Access resources and inspect a **refreshed full Terraform plan**; stop on unrelated changes, replacements, destroys or any ANAS/Tunnel/DNS mutation.
5. With explicit operator approval, stage infrastructure and enable the exact Route **last**; perform real OAuth PKCE, Service Token, streaming, auth-negative and read-only query acceptance tests.
6. Re-run ANAS OAuth/Service Token and routing regression before/after cutover.
7. If failure occurs, disable only Grafana's Route/Access registration and restore Grafana-only Worker/container versions. Do not roll back ANAS or replace the shared Tunnel/Custom Domain.

This design does **not** authorize production deployment or Terraform apply.


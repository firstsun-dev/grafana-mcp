# grafana-mcp project

## Purpose

Provide a read-only remote Grafana MCP endpoint for Firstsun's ChatGPT, Claude, Claude Code and authorized headless clients without exposing the NAS monitoring stack or duplicating the upstream Grafana MCP tools.

**Status:** planning only. No Worker, NAS container, Cloudflare configuration or production deployment has been implemented by this repository.

## Canonical contract

- Public MCP resource: `https://mcp.firstsun.org/grafana/mcp` (exact path; no aliases).
- Service slug: `grafana`. Cloudflare Worker: `grafana-mcp-proxy`.
- Interactive authentication: Cloudflare Access Managed OAuth with PKCE.
- Machine authentication: a separately approved, explicitly scoped Cloudflare Access Service Token.
- Origin authentication: signed, Grafana-specific Access JWT validated by the Worker; a separate private bearer token between Worker and NAS MCP.
- Private data access: Worker HTTP VPC Service -> existing `firstsun-nas` Tunnel -> loopback-only official `grafana/mcp-grafana` -> internal Grafana API and existing Prometheus datasource.
- All Grafana tools are read-only; official upstream implementation is used unchanged.

## Ownership boundaries

| Repository | Owner of |
| --- | --- |
| `firstsun-dev/.github` | Organization-wide `mcp-service-hosting` contract and cross-repository onboarding specification |
| `firstsun-dev/grafana-mcp` | Proxy Worker, its application OpenSpec, JWT verification, fixed-target Streamable HTTP forwarding, tests, thin CI caller and operation guide |
| `firstsun-dev/infra-config` | NAS Ansible/Compose, Grafana read-only service account provisioning, Cloudflare Access app/policies/AUD, exact Worker Route, scoped HTTP VPC Service and cutover runbook |
| `firstsun-dev/anas-mcp` | Existing analytics MCP, unchanged |

The shared hostname remains an `anas-mcp` Worker Custom Domain for unmatched paths. This project MUST NOT become a multi-service proxy or provision any Cloudflare Route/Custom Domain in its Wrangler configuration.

## Engineering and safety rules

- Follow `firstsun-dev/.github/openspec/changes/add-grafana-mcp-on-nas/` before making implementation decisions.
- Favor the smallest proxy that verifies access and forwards an allowlisted MCP request to one fixed private destination.
- Preserve MCP stream/session semantics; never buffer SSE into JSON.
- Use a dedicated origin caller token held by the Worker; Grafana Service Account credentials stay exclusively on NAS.
- Never log/accessibly expose tokens, signed assertions, PromQL, private metrics or tool payloads.
- Keep `workers_dev=false` and `preview_urls=false`, use centrally managed `_cf-worker-v2.yml@v2` with a thin production-only caller.
- A documentation or PR merge does not authorize `terraform apply`, public route activation, secrets changes or production cutover.
- Unverified platform behavior (Workers VPC HTTP streaming, route precedence, existing Tunnel drift, OAuth metadata discovery) is a rollout blocker, not an excuse to add a public bypass.

## Active application change

- [`bootstrap-grafana-mcp`](changes/bootstrap-grafana-mcp/proposal.md) — pending implementation, documents the application contract and staged verification.


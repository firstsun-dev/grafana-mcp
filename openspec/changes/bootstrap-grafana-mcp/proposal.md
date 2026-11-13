# Proposal: Bootstrap the Grafana MCP Proxy Worker

## Status

**Proposed — documentation only.** This change defines the implementation contract in the newly initialized `firstsun-dev/grafana-mcp` repository. No runtime is implemented and no production resource is applied by this change.

The accepted organization-wide design remains:
- [Grafana MCP onboarding](https://github.com/firstsun-dev/.github/tree/main/openspec/changes/add-grafana-mcp-on-nas)
- [Shared MCP hostname convention](https://github.com/firstsun-dev/.github/tree/main/openspec/changes/standardize-mcp-path-hosting)
- [Terraform-side service registration](https://github.com/firstsun-dev/infra-config/blob/main/docs/mcp-service-registration.md)

## Why

Firstsun already has `https://mcp.firstsun.org/anas/mcp` and a NAS-hosted Grafana/Prometheus stack. The requested outcome is a single, read-only Grafana MCP resource consumable by ChatGPT and Claude-family clients, reusing the shared hostname and current NAS Tunnel rather than publishing a second origin, building a generic gateway or reimplementing Grafana MCP.

## Proposed application changes

1. Create `grafana-mcp-proxy` as a small TypeScript Worker in this repository.
2. Validate the exact external `/grafana/mcp` path and a signed Grafana-specific Cloudflare Access assertion before any private forwarding.
3. Keep Access Managed OAuth and authorized headless Service Tokens in the *separate* Grafana Access application owned by `infra-config`.
4. Forward only allowed MCP Streamable HTTP operations to one private HTTP VPC Service. Replace client authentication with a distinct origin-only token; never forward caller assertions or Grafana credentials.
5. Preserve MCP request/response and SSE streaming semantics with bounded resource usage, safe errors, minimal telemetry and comprehensive negative tests.
6. Deploy with a thin caller of the existing organization Worker v2 workflow; keep Worker-owned public routes and preview hosts disabled.

## Scope and ownership

**In this repository:** application OpenSpec, Worker source/tests, binding declarations, README/operations documentation and thin CI/CD caller.

**Coordinated in `infra-config`:** existing NAS Tunnel preflight, dedicated official pinned `grafana/mcp-grafana` container, secret injection, Grafana read-only account, VPC Service, exact Worker Route, independent Access application/policies and full Terraform plan. Changes to these resources require separate reviews.

**Not changed:** `anas-mcp` Worker, Custom Domain, ANAS Access AUD/policies, existing NAS Tunnel, DNS, existing Grafana/Prometheus stack lifecycle and the organization CI engine.

## Dependencies and uncertainties

Before implementing or enabling the public route, verify:
- Exact Worker Route precedence over the existing ANAS Custom Domain and correct protected-resource OAuth metadata for both paths.
- Cloudflare Workers VPC HTTP `fetch()`, QUIC connection and live SSE/session behavior through the existing `firstsun-nas` Tunnel.
- Tunnel `config_src` versus live `remote_config` discrepancy, without making unrelated Tunnel changes.
- Reachability of a dedicated NAS loopback port from cloudflared and private Docker networking to Grafana.
- Pinned upstream `grafana/mcp-grafana` image/options, tool allowlist and minimum Grafana permissions.

Any failed compatibility gate requires a design review. Do not silently introduce public NAS ingress, wildcard routes, broader network access or alternate hosts.

## Non-goals

- Implementing or forking Grafana MCP tools, a standalone Prometheus MCP or a shared MCP Gateway.
- Grafana writes, arbitrary HTTP/SQL access, dynamic upstream selection or automatic remediation.
- New domain/DNS, a new Cloudflare Tunnel, broad VPC Network, publicly exposed container or extra datastore.
- Production `terraform apply`, credentials, deployment, route activation or updating other MCP services as part of this documentation change.

## Acceptance

A later implementation is complete only when real ChatGPT/Claude-compatible OAuth/PKCE and an approved headless Service Token can initialize MCP, enumerate read-only tools, run a Prometheus query, search dashboards and list alerts through the fixed private route; requests with wrong/missing auth, wrong AUD, malicious headers, unsupported paths or write operations fail closed; streaming and sessions work; ANAS still passes regression; and the approved refreshed full Terraform plan contains no unrelated change or resource replacement.


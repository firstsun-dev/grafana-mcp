# Tasks: Bootstrap grafana-mcp

Documentation commits do not imply implementation. A task is checked only after evidence exists in the owning repository or live environment. Do not create multiple commits per checklist item; prefer focused implementation PRs.

## 0. Application specification

- [x] Initialize the `firstsun-dev/grafana-mcp` repository with a documentation-only identity.
- [x] Reference the organization `add-grafana-mcp-on-nas` contract without copying its infrastructure ownership.
- [x] Specify proxy runtime behavior, security requirements and acceptance cases.
- [ ] Obtain review/approval of this application OpenSpec before implementation.

## 1. Preflight (no writes)

- [ ] Verify current production ANAS `/anas/mcp`, Access Managed OAuth and Service Token operation.
- [ ] Verify exact Worker Route precedence with existing Custom Domain and separate per-path OAuth protected-resource metadata.
- [ ] Reconcile existing NAS Tunnel `config_src` vs API `remote_config` observations, **without modifying the Tunnel**.
- [ ] Confirm approved HTTP Workers VPC Service, existing Tunnel QUIC/HTTP streaming and NAS local host/port reachability.
- [ ] Confirm official `grafana/mcp-grafana` pinned version, valid CLI flags, Streamable HTTP behavior, Host/Origin restrictions and credential setup.
- [ ] Confirm NAS-to-private-Grafana API Docker network path, read-only Grafana service account and required datasource permissions.
- [ ] Stop for design review if SSE/session routing, Access challenge or VPC restrictions cannot work without a public bypass.

## 2. Private NAS origin (owner: infra-config)

- [ ] Add standalone `nas_grafana_mcp` Ansible role and Compose project; keep `nas_monitor` lifecycle unaffected.
- [ ] Pin official upstream image; serve internal `/grafana/mcp`, disable all writes, allowlist read-only tools and bind loopback only.
- [ ] Provision distinct NAS upstream caller and Grafana Service Account secrets via approved secret management.
- [ ] Set conservative resource/health limits and restrictive Host/Origin handling.
- [ ] Test private `initialize`, `tools/list`, Prometheus query, dashboard search and alert read; verify no write tools.

## 3. Proxy Worker (owner: grafana-mcp)

- [ ] Add minimal TypeScript Worker and Wrangler configuration (no public Route, workers.dev or preview URLs).
- [ ] Reject all noncanonical hosts/paths and unsupported methods.
- [ ] Verify signed Access JWT issuer/JWKS/algorithm/Grafana AUD/exp/nbf; fail closed on bad/unavailable auth.
- [ ] Forward only fixed-path requests through scoped VPC Service; scrub caller identity/auth and Grafana override headers.
- [ ] Add Worker-held, separate origin bearer; ensure no Grafana Service Account credential enters the Worker.
- [ ] Preserve MCP POST/optional GET/DELETE, SSE, status codes and session/resumption headers.
- [ ] Apply safe body/concurrency/query timeout limits and redact sensitive telemetry.
- [ ] Add automated security and protocol regression tests.
- [ ] Add production-only thin caller to shared `_cf-worker-v2.yml@v2` with safe test/smoke behavior.

## 4. Edge resources (owner: infra-config)

- [ ] Add exact Grafana Worker Route, distinct path-scoped Access app, OAuth DCR/PKCE, explicit human allowlist and independent Service Token policy.
- [ ] Add Grafana-specific AUD and fixed-host/port HTTP VPC Service using the **existing** `firstsun_nas` Tunnel.
- [ ] Configure private Worker binding and approved non-secret outputs without exposing secrets.
- [ ] Run Terraform format/validate and inspect **full refreshed plan**; reject unrelated drift, destroys, replacements and any ANAS/Custom-Domain/DNS/Tunnel change.
- [ ] Await explicit operator approval before applying edge resources; enable the public Route last.

## 5. End-to-end acceptance

- [ ] Verify unauthenticated Cloudflare Access 401 and correct Grafana-specific `WWW-Authenticate`/resource metadata.
- [ ] Test real ChatGPT and Claude-compatible Managed OAuth discovery/PKCE, `initialize` and `tools/list`.
- [ ] Test explicitly authorized headless Service Token independently.
- [ ] Prove Grafana/ANAS JWT AUDs are mutually rejected, and no cross-service routes or policies leak.
- [ ] Verify Prometheus query, dashboard read/search, alert listing, and unavailable/denied writes.
- [ ] Verify upstream caller token boundary, sanitized headers, invalid paths, outage behavior and no public origin access.
- [ ] Verify streaming POST/GET/DELETE where upstream supports it, cancellation and session/resumption headers across VPC/Tunnel.
- [ ] Re-run existing ANAS OAuth and Service Token regression before and after Grafana cutover.
- [ ] Capture redacted CI results, Terraform plan, live endpoint evidence and rollback readiness.

## 6. Operations

- [ ] Document scoped Grafana-only rollback, secret rotation, image update and availability monitoring.
- [ ] Mark application implemented only after live client, security, private routing and ANAS regression gates pass.

## Mandatory no-go

No production `terraform apply`, unrelated Tunnel recreation, public NAS ingress, wildcard route, shared ANAS token/AUD, write tools, unsafe header forwarding, committed secrets or unverified VPC streaming workaround.


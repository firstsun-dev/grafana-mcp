# Grafana MCP Proxy — application capability specification

This app-level spec refines, and does not replace, the organization-wide `mcp-service-hosting` contract in `firstsun-dev/.github`. All requirements below describe the target behavior, **not an implemented service**.

## ADDED Requirements

### Requirement: Canonical shared-host endpoint with strict isolation

The service SHALL expose Grafana MCP only at `https://mcp.firstsun.org/grafana/mcp` via the independently managed exact Worker Route. It SHALL NOT define its own public Custom Domain, fallback to ANAS, or accept variant paths.

#### Scenario: Canonical request

- **WHEN** an approved client requests the exact public Grafana MCP URL
- **THEN** the Grafana-specific Route and Access application handle the request
- **AND** only the Grafana proxy and private upstream MAY execute tools

#### Scenario: Invalid route or host

- **WHEN** a client requests `/grafana`, `/grafana/mcp/`, `/grafana/mcp2`, an encoded/doubled path, unknown slug or alternate hostname
- **THEN** no Grafana or ANAS MCP tool is invoked from that request

### Requirement: Signed and independent Cloudflare Access authentication

Cloudflare Access SHALL support Managed OAuth with PKCE and an explicitly authorized headless Service Token on Grafana's own Access application and AUD. The Worker SHALL verify the signed `Cf-Access-Jwt-Assertion` (issuer, JWKS signature, allowed algorithm, Grafana AUD, expiration and not-before) before contacting VPC.

#### Scenario: Valid approved human or machine client

- **WHEN** an approved client completes the corresponding Access authentication
- **THEN** the Worker accepts only a valid Grafana-scoped Access assertion and permits MCP protocol handling
- **AND** the resource metadata/challenge identifies Grafana rather than ANAS

#### Scenario: Wrong or missing assertion

- **WHEN** the assertion is missing, spoofed, expired, malformed or issued for ANAS
- **THEN** the Worker fails closed before any VPC fetch or tool invocation
- **AND** no sensitive claim, credential or JWKS detail is returned

#### Scenario: Verification dependencies fail

- **WHEN** the Worker lacks its expected Access configuration or the identity verification dependency is unavailable
- **THEN** access is denied with a generic failure and no private upstream request

### Requirement: Private fixed-destination forwarding and credential separation

The Worker SHALL send allowed MCP traffic only through a single host/port scoped Workers VPC Service over the existing NAS Tunnel, using a separate origin-only bearer token. The NAS SHALL keep the Grafana Service Account credential private.

#### Scenario: A client supplies override headers

- **WHEN** an authenticated caller supplies `Authorization`, `Cookie`, `Cf-Access-*`, `X-Grafana-*` or forwarding/target-override headers
- **THEN** the Worker rejects or removes them before the private call
- **AND** the only upstream authorization is the Proxy Worker's dedicated origin credential

#### Scenario: Origin or VPC is unavailable

- **WHEN** the private backend cannot be reached or returns an authorization failure
- **THEN** the Worker returns a bounded fail-closed response without exposing secrets or falling back to a public URL

### Requirement: MCP Streamable HTTP correctness

The Worker SHALL preserve upstream-supported MCP HTTP semantics, including `POST` and any supported `GET`/`DELETE` session operations, JSON-RPC bodies, streaming and required MCP request/response headers.

#### Scenario: Initialize and call tools

- **WHEN** an authorized client sends `initialize`, `tools/list` or `tools/call`
- **THEN** the upstream receives an equivalent MCP request and its meaningful status/content-type/session headers are relayed

#### Scenario: Long-running event stream

- **WHEN** upstream responds with `text/event-stream` or a valid session/resumption identifier
- **THEN** the Worker streams data without buffering/reformatting or losing required protocol and session headers
- **AND** cancellation and timeouts do not create an unauthorized fallback

#### Scenario: Unsupported operation

- **WHEN** a client sends an unsupported method, invalid MCP transport headers or an oversized request
- **THEN** the proxy rejects it without invoking a Grafana tool

### Requirement: Read-only Grafana capabilities and bounded query cost

The NAS SHALL run the pinned official `grafana/mcp-grafana` implementation with write tools disabled, a reviewed read-only tool allowlist, a loopback-only listener and a least-privilege Grafana Service Account. The proxy SHALL enforce reasonable resource bounds.

#### Scenario: Monitoring analysis

- **WHEN** an approved MCP client requests a supported Prometheus query, dashboard search/read or alert list
- **THEN** Grafana's existing private API/datasources return only data allowed by the read-only service account

#### Scenario: Mutation attempt

- **WHEN** a client attempts to create, modify or delete Grafana resources
- **THEN** no corresponding write tool is exposed or authorized
- **AND** the Grafana configuration remains unchanged

### Requirement: Independent rollout and safe observability

Changes SHALL use the organization-managed Worker v2 workflow and a staged, operator-approved `infra-config` cutover. Logging SHALL be limited to non-sensitive operational metadata.

#### Scenario: Test and deployment

- **WHEN** this service is implemented
- **THEN** tests cover auth, route isolation, malicious headers, streaming/session lifecycle, errors and writes
- **AND** production deployment SHALL NOT enable the exact Route before the NAS/VPC/Access path is proven safe

#### Scenario: ANAS regression and rollback

- **WHEN** Grafana's public route is enabled or rolled back
- **THEN** `https://mcp.firstsun.org/anas/mcp` continues to pass its own OAuth, Service Token and tool-regression tests
- **AND** no ANAS Custom Domain, DNS, Access, AUD or existing Tunnel is replaced or repointed

#### Scenario: Telemetry and plan inspection

- **WHEN** a request is observed or Terraform changes are reviewed
- **THEN** logs omit secrets, JWTs, tool payloads and private PromQL/metrics
- **AND** any refreshed full plan containing unrelated change, destroy or replacement blocks production cutover


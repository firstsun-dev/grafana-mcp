# grafana-mcp

Planned read-only Grafana MCP proxy for Firstsun clients (ChatGPT, Claude, Claude Code and other OAuth-capable MCP clients).

**Status: specification only.** This repository is not yet an implemented or deployed MCP service.

- Canonical endpoint: `https://mcp.firstsun.org/grafana/mcp`
- Intended Worker: `grafana-mcp-proxy` (Cloudflare Access JWT verification + protocol-preserving private HTTP forwarding)
- Grafana tools: official `grafana/mcp-grafana` container on the existing NAS; no tool reimplementation
- Infrastructure owner: `firstsun-dev/infra-config` (NAS Ansible, Workers VPC Service, Terraform Route and Cloudflare Access)
- Existing `/anas/mcp`, Custom Domain and `firstsun-nas` Tunnel must remain unchanged

The organization-level architecture is defined in [add-grafana-mcp-on-nas](https://github.com/firstsun-dev/.github/tree/main/openspec/changes/add-grafana-mcp-on-nas). Application-level requirements, implementation plan and acceptance criteria live under [openspec/changes/bootstrap-grafana-mcp](openspec/changes/bootstrap-grafana-mcp).

No production deploy, Terraform apply or secret provisioning is authorized by these documents.

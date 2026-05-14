# avx-health

A SKILL.md skill for running live health sweeps against an [Aviatrix](https://aviatrix.ai) network fabric via the Aviatrix MCP server.

Works with any agentic MCP client that supports the SKILL.md skill format (Claude Code, and others).

## What it does

Invoked as `/avx-health`, the skill runs a tiered parallel sweep using `aviatrix_*` MCP tools and emits a RAG scorecard with numbered findings and investigation playbooks.

**Domains checked:**
- Gateway up/down/degraded status
- Site-to-Cloud tunnel health
- DCF (Distributed Cloud Firewall) enforcement coverage and log freshness
- IPS/Suricata alert volume and severity
- FlowIQ traffic pipeline health and top-talker anomaly detection
- BGP session state and uptime
- Audit trail freshness
- Container health (opt-in, `--deep`)
- Gateway CPU, memory, and throughput (`perf`)

**Invocation patterns:**
```
/avx-health             full sweep
/avx-health --deep      full sweep + container health per gateway
/avx-health bgp         BGP sessions only
/avx-health dcf         DCF enforcement + log freshness only
/avx-health traffic     FlowIQ top talkers only
/avx-health s2c         Site-to-Cloud tunnel status only
/avx-health audit       Audit trail freshness only
/avx-health perf        Gateway CPU%, memory%, and throughput
```

## Prerequisites

- An agentic MCP client that supports the SKILL.md skill format
- Aviatrix MCP server configured in your client (`https://platform.mcp.aviatrix.com/mcp`)
- An Aviatrix MCP API key with at minimum `controller:read` scope — generate one at [platform-login.mcp.aviatrix.com](https://platform-login.mcp.aviatrix.com)

## Installation

### Plugin install (Claude Code)

```bash
git clone https://github.com/nickda/avx-health.git ~/.claude/plugins/avx-health
# Restart your client — /avx-health appears automatically
```

### Manual

Copy `skills/avx-health/SKILL.md` into your client's skills directory.

## Notes

- Auto-detects MCP server prefix (`mcp__*__aviatrix_list_gateways`) — works against any environment without hardcoding.
- FireNet-aware: enforcement gaps on spokes routing through a vendor firewall are not scored as findings.
- All tool calls use HTTPS port 443 only, matching Aviatrix MCP server egress policy.

# Aviatrix MCP Skills for Claude Code

Claude Code skills for operating Aviatrix network fabric via the [Aviatrix MCP server](https://mcp.aviatrix.com).

## Skills

### `/avx-health` — Fabric Health Sweep

Live health check across an Aviatrix fabric. Runs a tiered parallel sweep using `aviatrix_*` MCP tools and emits a RAG scorecard with numbered findings and investigation playbooks.

**Checks:**
- Gateway up/down/degraded status
- Site-to-Cloud tunnel health
- DCF (Distributed Cloud Firewall) enforcement coverage and log freshness
- IPS/Suricata alert volume and severity
- FlowIQ traffic pipeline health and anomaly detection
- BGP session state and uptime
- Audit trail freshness
- Container health (opt-in)
- Gateway CPU, memory, and throughput

**Invocation:**
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

- [Claude Code](https://claude.ai/code) CLI installed
- [Aviatrix MCP server](https://mcp.aviatrix.com) configured in your Claude Code environment
- An Aviatrix MCP API key with at minimum `controller:read` scope

## Installation

### As a Claude Code plugin (recommended)

```bash
# Clone this repo
git clone https://github.com/nicholasdavitashvili/public-skills.git ~/.claude/plugins/aviatrix-mcp-skills

# Restart Claude Code — skills appear as /avx-health automatically
```

### Manual

Copy `skills/avx-health/SKILL.md` into your Claude Code skills directory and invoke via `/avx-health`.

## Notes

- The skill auto-detects which MCP server prefix to use (`mcp__*__aviatrix_list_gateways`) — works against dev, staging, production, and customer environments without hardcoding.
- FireNet-enabled transits are accounted for: enforcement gaps on spokes routing through a vendor firewall are not scored as findings.
- All tool calls use HTTPS port 443 only, matching Aviatrix MCP server egress policy.

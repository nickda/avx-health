# avx-health

A SKILL.md skill for running live health sweeps against an [Aviatrix](https://aviatrix.ai) network fabric via the [Aviatrix MCP server](https://platform-login.mcp.aviatrix.com).

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

## Try the demo

No sign-up required. Run this to configure your MCP client and start immediately:

```bash
# Claude Code: writes .mcp.json in the current directory
curl -s -X POST https://platform.mcp.aviatrix.com/auth/demo-config \
  | jq '.configs["claude-code"]' > .mcp.json
```

Restart your client, then run `/avx-health` to sweep the demo fabric.

For other clients (Claude Desktop, Cursor, Windsurf, Cline), or to skip `jq`, visit [platform-login.mcp.aviatrix.com/login/agent-signup](https://platform-login.mcp.aviatrix.com/login/agent-signup) for ready-to-paste configs.

Demo configs are rate-limited to 10 requests/hour per IP.

## Prerequisites

- An agentic MCP client that supports the SKILL.md skill format
- Aviatrix MCP server configured in your client (`https://platform.mcp.aviatrix.com/mcp`)
- An Aviatrix MCP API key with at minimum `controller:read` scope; generate one at [platform-login.mcp.aviatrix.com](https://platform-login.mcp.aviatrix.com)

## Installation

### Clone and symlink (stays up to date)

```bash
git clone https://github.com/nickda/avx-health.git
mkdir -p ~/.claude/skills/avx-health
ln -s "$(pwd)/avx-health/SKILL.md" ~/.claude/skills/avx-health/SKILL.md
```

### One-liner

```bash
mkdir -p ~/.claude/skills/avx-health
curl -fsSL https://raw.githubusercontent.com/nickda/avx-health/main/SKILL.md \
  -o ~/.claude/skills/avx-health/SKILL.md
```

Restart your client after installing; `/avx-health` appears automatically.

## Notes

- Auto-detects MCP server prefix (`mcp__*__aviatrix_list_gateways`), works against any environment without hardcoding.
- FireNet-aware: enforcement gaps on spokes routing through a vendor firewall are not scored as findings.
- All tool calls use HTTPS port 443 only, matching Aviatrix MCP server egress policy.

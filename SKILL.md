---
name: avx-health
description: "Aviatrix fabric health sweep via aviatrix_* MCP. Checks gateways, S2C, BGP, DCF, IPS, traffic, audit, FireNet. Use when network seems off, pre-demo check needed, or asking if everything's ok. RAG scorecard with numbered findings and investigation playbooks. Not for tracelogs or hardening audits."
user-invocable: true
argument-hint: '[--deep] [--pdf] [bgp|dcf|traffic|s2c|audit|perf|logs|firenet]'
version: 1.5.3
status: active
depends-on: []
feeds-into: [avx-tshoot]
---

# Aviatrix Fabric Health Sweep

You are running a live health check against an Aviatrix fabric. The goal is a fast, complete
picture: what's degraded, what's missing coverage, what needs investigation. Everything runs
against the `aviatrix_*` MCP tools — no tracelogs, no CLI, no manual steps.

## Invocation patterns

```
/avx-health             full sweep (Tier 1 + Tier 2)
/avx-health --deep      full sweep + Tier 3 container health per gateway
/avx-health --pdf       full sweep + PDF export (type "done" or "q" when finished investigating)
/avx-health bgp         BGP sessions only
/avx-health dcf         DCF enforcement + log freshness only (also calls aviatrix_list_firenet_inspection for exclusion)
/avx-health traffic     FlowIQ top talkers only
/avx-health s2c         S2C tunnel status only
/avx-health audit       Audit trail freshness only
/avx-health perf        Gateway CPU%, memory%, and throughput — all gateway types (spoke/transit/edge)
/avx-health firenet     FireNet inspection coverage — which transits, which spokes inspected
/avx-health logs        Pull recent controller command log entries; optionally search for a pattern
```

**MCP server detection:** Look at available tools — find the one that matches
`mcp__*__aviatrix_list_gateways`. Use that server prefix for all calls. Never hardcode
a server name; this skill runs against dev, stg, prod, and customer environments.

**Time windows (fixed):**
- IPS alerts: last 24h
- DCF log freshness: last 24h (gap = potential crash-loop)
- Traffic anomalies: last 7d (FlowIQ)
- Audit trail: last 7d for sweep; expands during investigation

---

## Aviatrix architecture facts

These facts constrain how you interpret diagnostic data. Violating them produces outputs that expert users will immediately distrust.

**Fabric control plane:** Aviatrix Controller programs VPC/VNet route tables via cloud provider APIs. No routing protocol (BGP, OSPF) runs between transit and spoke gateways.

**Transit-to-spoke data plane:** Two IPsec tunnels per spoke (primary + backup transit GW), ECMP load-balanced. This is ActiveMesh. Never describe transit-spoke connectivity problems as "BGP issues."

**BGP = external peering only.** BGP runs between transit/edge gateways and on-prem routers, VGW, or S2C peers. `aviatrix_run_bgp_diag` checks those external sessions. HTTP 500 on a spoke = FRR not active, expected. A down BGP session on a transit GW means the on-prem/VGW peer is affected — not the transit-spoke fabric.

**IPS/IDS (Suricata) = egress/ingress + east-west only with symmetric routing.** Suricata can inspect east-west traffic, but only when symmetric routing is in place (single gateway per path, no HA gateway groups). In HA deployments — the production norm — east-west IPS is not reliable because return traffic may flow through a different gateway. Do not treat IPS alert absence as proof that east-west traffic is passing uninspected; in HA fabrics, east-west inspection requires FireNet with a vendor appliance.

**FlowIQ = NetFlow Agent dependent.** Pipeline requires: (a) NetFlow Agent enabled in CoPilot Settings → Logging Services, (b) UDP 2055 open from gateway EIPs to CoPilot, (c) `oteld_enabled` matching gateway version (8.2+ = OTEL/TCP; older = UDP 2055 — mismatch = silent data loss). `boundary_at_start: true` is the symptom of any pipeline failure, not just staleness.

---

## Execution: Tiered parallel sweep

Run Tier 1 first (all calls in parallel — they have no dependencies on each other). Once
Tier 1 completes, run Tier 2 in parallel using the data you collected. Emit the scorecard
after Tier 2. Only run Tier 3 if `--deep` was passed.

### Tier 1 — all parallel

| Call | Purpose |
|---|---|
| `aviatrix_list_gateways` | Gateway up/down/degraded status |
| `aviatrix_list_s2c_connections` | S2C tunnel status |
| `aviatrix_get_dcf_enforced_gateways` | Which spokes have DCF enforcement active |
| `aviatrix_list_firenet_inspection` | Which transit GWs have FireNet active; which spokes are inspected vs not. Use this data in DCF enforcement scoring — spokes in `inspected` lists are covered by a third-party FW and should not be flagged as DCF enforcement gaps. |
| `aviatrix_get_detected_intrusions` | IPS/Suricata alerts (last 24h) |
| `aviatrix_get_flowiq_top_talkers` | Top external destinations + internal sources (7d). Check `boundary_at_start` in response — `true` means FlowIQ aggregation index has no data for this window (pipeline broken/stale). Root cause is always one of: NetFlow Agent disabled in CoPilot, UDP 2055 blocked, or `oteld_enabled` mismatch — see Architecture Facts above. If `true`, call `aviatrix_get_top_egress(category=ips, ...)` and `aviatrix_get_top_egress(category=urls, ...)` for the same 7d window as a DCF-log-based fallback. Note in scorecard: source is firewall logs (top-10 truncated; counts not volumetric). |
| `aviatrix_get_dcf_audit_summary` | Last DCF config change event (audit trail freshness) |

### Tier 2 — parallel, uses Tier 1 data

| Call | Purpose |
|---|---|
| `aviatrix_run_bgp_diag` on **every** gateway from Tier 1 | External BGP session state (on-prem/VGW peers only — transit-spoke fabric uses IPsec, not BGP). HTTP 500 = FRR not running — suppress silently, expected on most gateways. Report only gateways that return data. Cap at 10 concurrent calls. |
| `aviatrix_count_dcf_logs` per **enforced** gateway (from Tier 1) | Count of DCF logs in last 24h per gateway. Zero = potential inspection gap or crash-loop. Batch in groups of 10 if >20 enforced gateways. |

### `perf` invocation — standalone only, not part of default sweep

Call `aviatrix_get_gateway_performance` with default params (`minutes_back=15`, all gateway types).
Sort results by CPU % descending. No scoring thresholds — gateway instance size determines what's
"high", which isn't available in the response. **Do not render an ASCII table.** Use the formats below.

Users can pass explicit `start_time`/`end_time` for historical analysis or `gateway_names` to
target specific gateways. Forward any such args directly to the tool.

**Visualization block:** Render a code-fenced block with CPU and memory bars, followed by an
indented metadata line per gateway.
Bar width = 20 chars of `=`, padded with spaces to 20. Gateway names left-padded to a fixed width
for alignment. N/A values render as `[--------------------]`.

```
Gateway Resource Utilization (15 min avg)
gw-spoke-east-prod    CPU [================    ] 80%  Mem [===================] 95%
                      type=spoke  account=eng-aws  region=us-east-2  RX=23.6 GB  TX=21.8 GB
gw-spoke-west-prod    CPU [=========           ] 45%  Mem [============        ] 60%
                      type=spoke  account=ops-aws  region=us-west-2  RX=10.1 GB  TX=8.4 GB
```

Formula: `bar_len = round(value / 100 * 20)`. Clamp to [0, 20].
Metadata line: type, account, region, RX, TX — all on a single indented line below the bar line.

### `firenet` invocation — standalone only

Call `aviatrix_list_firenet_inspection` and emit a structured summary:

```
FireNet Inspection Coverage
transit-gw-east    [egress inspection]
  Inspected (3): spoke-a, spoke-b, spoke-c
  Not inspected (1): spoke-dev

transit-gw-west
  Inspected (2): spoke-d, spoke-e
  Not inspected (0): —
```

List every transit returned. Mark egress inspection transits explicitly. If the tool returns
no entries, emit: `No FireNet-enabled transit gateways found.` Not scored — informational only.

### `logs` invocation — standalone only

1. Call `aviatrix_get_controller_logs` with `max_lines=200` (default).
2. Optionally, if the user passed a search term after `logs` (e.g. `/avx-health logs bgp`), also call `aviatrix_search_controller_logs(search_pattern=<term>, max_lines=200)` and display matches separately.
3. Emit the raw log block as returned by the tool. No scoring, no thresholds.

Use `aviatrix_search_controller_logs` inside investigation playbooks to correlate controller-side events with a specific finding (e.g. search for a gateway name, action keyword like "bgp", or timestamp range).

---

### Tier 3 — only with `--deep`, parallel

| Call | Purpose |
|---|---|
| `aviatrix_search_gateway_syslogs` per gateway, search term `avx-gw-trafficserver` | Container restart detection. Warn user before starting if gateway count >10. |

---

## Scoring thresholds

| Domain | ✅ Green | 🟡 Amber | 🔴 Red |
|---|---|---|---|
| Gateways | All up | 1-2 down/degraded | 3+ down |
| S2C Tunnels | All up | 1-2 down | 3+ down or >20% of total |
| DCF Enforcement | All non-FireNet spokes enforced | 1-2 unenforced (named) | 3+ unenforced |
| FireNet Coverage | Informational — no pass/fail threshold | — | — |
| IPS Alerts | 0 alerts (24h) | 1-5, or high-volume with single benign signature | 6+ diverse signatures, or any blocked/high-severity alert |
| FlowIQ Pipeline | `boundary_at_start: false`, data current | Data gap 1-7d | `boundary_at_start: true` (>7d gap or pipeline broken) |
| Traffic Anomalies | All top IPs = known CDN/cloud | Unknown IPs present | Unknown IP >1 GB |
| Audit Trail | Events in last 7d | No events 7-30d | No events >30d |
| BGP Sessions | All up, all >7d uptime | Any session <7d uptime | Any session down |
| DCF Log Freshness | Logs in last 24h per enforced GW | Last log 24h-7d ago | Enforced GW with 0 logs or >7d gap |
| Container Health (`--deep`) | No restarts | Restarts <10 | Restarts >100 (crash-loop) |

**FireNet and DCF enforcement:** Spokes listed in the `inspected` field of `aviatrix_list_firenet_inspection` results are covered by a third-party firewall (Palo Alto, Fortinet, CheckPoint). They must not be counted as DCF enforcement gaps — vendor FW handles inspection. Exclude them before computing the DCF Enforcement score. Include a FireNet Coverage row in the scorecard showing how many transit GWs have FireNet active and how many spokes are under vendor inspection. In `dcf` standalone mode (not full sweep), always call `aviatrix_list_firenet_inspection` first so the exclusion list is available before scoring.

**IPS alert scoring uses homogeneity:** 2000 identical alerts all permitted (e.g. same ET POLICY signature, zero blocks) = amber. Ten alerts across different signatures with any block action = red. Count alone doesn't determine severity — signature diversity and action (permit vs block) do.

**FlowIQ pipeline check:** `boundary_at_start: true` in the `[flow-metadata]` block of the
`get_flowiq_top_talkers` response means the aggregation index has no data for the requested
window — probe returns empty results **silently**, which would otherwise produce a false green
Traffic Anomalies score. Always check this field first. Common causes: gateway upgrade changed
netflow transport (UDP→TCP/OTEL mismatch), netflow agent disabled post-upgrade, CoPilot NSG
blocking UDP 2055 from gateway EIPs. Data cutoff date aligns with the triggering event.

**Traffic anomaly classification:** Flag IPs whose owner field (from `get_flowiq_top_talkers`)
does not match known CDN/cloud providers: Fastly, Akamai, CloudFront, AWS Global Accelerator,
Azure CDN, GCP CDN, Cloudflare. Unknown owner + high volume = flag it.

**Top talkers visualization:** After the scorecard table, render a code-fenced block with
relative volume bars for the top external destinations. Top IP = full 20-char `=` bar. Others
scaled linearly. Append `⚠ unknown` for flagged IPs. Only emit when Traffic Anomalies is
🟡 or 🔴 (skip when green — saves space).

```
Top External Destinations (7d egress)
1. 1.2.3.4    (Cloudflare CDN)      [====================] 4.2 GB
2. 5.6.7.8    (Alibaba AS37963)     [=========           ] 1.8 GB  ⚠ unknown
3. 9.10.11.12 (AWS CloudFront)      [====                ] 0.8 GB
```

Formula: `bar_len = round(ip_bytes / max_bytes * 20)`. Clamp to [1, 20].

**DCF log freshness red** is the most actionable signal: an enforced gateway going silent
almost always means the `avx-gw-trafficserver` container is crash-looping. The investigation
playbook will confirm it.

---

## Scorecard output format

After Tier 2 (or after each domain for single-domain runs), emit:

```
## Aviatrix Fabric Health — YYYY-MM-DD HH:MM UTC

| Domain             | Status | Finding                                            |
|--------------------|--------|----------------------------------------------------|
| Gateways           | ✅     | 12/12 up                                           |
| S2C Tunnels        | ✅     | 8/8 up                                             |
| DCF Enforcement    | 🟡     | marketing-azure-spoke-all: no enforcement          |
| FireNet Coverage   | ℹ️     | 1 transit (transit-fw-east): 3 spokes inspected    |
| IPS Alerts         | ✅     | 0 alerts (24h)                                     |
| FlowIQ Pipeline    | 🔴     | boundary_at_start: true — no data since 2026-05-01  |
| Traffic Anomalies  | 🟡     | 47.91.64.21 (Alibaba, 14 GB) — unknown dependency  |
| Audit Trail        | 🔴     | Last event: 2025-12-12 (5 months stale)            |
| BGP Sessions       | ✅     | 4 sessions up, all >7d uptime                      |
| DCF Log Freshness  | 🔴     | marketing-azure-spoke-all: 0 logs since 2026-01-29 |

3 critical findings. 2 warnings. Investigate now?
- [1] FlowIQ pipeline broken — no data since 2026-05-01 (🔴)
- [2] DCF log gap on marketing-azure-spoke-all (🔴)
- [3] Audit trail stale 5 months (🔴)
- [4] Alibaba traffic unidentified (🟡)
- [5] DCF enforcement gap on marketing-azure-spoke-all (🟡)
```

If everything is green: `All checks passed. No findings.`

The user responds with numbers ("1 3"), "all", or a finding name. After each investigation,
show which findings remain open and ask what to investigate next.

When `--pdf` is active, also accept `done` or `q` as a signal to close the session and generate
the PDF. Display this hint below the finding list when `--pdf` is active:
`Type a finding number to investigate, or "done"/"q" to generate PDF.`

---

## Investigation playbooks

Each playbook ends with a **Resolution signal** — the minimal tool call that confirms the issue is resolved. Do not close a finding until the signal is green.

### DCF log gap (enforced GW, no recent logs)

Two root causes account for most DCF log gaps. Determine which applies before
investigating further.

**Step 0 — establish the cutoff date:**
Call `aviatrix_count_dcf_logs` with progressively older windows (24h, 7d, 30d, 90d) across
all enforced gateways. Identify the point where count drops to 0. Note the pattern:
- **Same cutoff date across all gateways** — fabric-level event (feature flag, upgrade, CoPilot restart). Go to Branch B first.
- **Different cutoff dates per gateway** — per-gateway failure (crash-loop most likely). Go to Branch A.

**Branch A — Crash-loop:**

1. Call `aviatrix_get_gateway_performance` with `gateway_names=[suspect_gw]` — high CPU (>85%) or memory pressure alongside a log gap strengthens the crash-loop diagnosis
2. Call `aviatrix_search_gateway_syslogs` on the named gateway, search term `avx-gw-trafficserver`
3. Count restart events. Estimate cycle time: (uptime at first restart) → (uptime at last restart) / N
4. Crash-loop confirmed if: >100 restarts OR cycle time <10 min
5. Extract: container image version, exit code (SIGABRT = code 134/n/a, clean shutdown = 0)
6. Estimate incident start: restart_count × avg_cycle_time, working backward from now
7. Report findings + recommend escalation to Aviatrix support with gateway name, container version, and CPU/memory at time of investigation

**Branch B — Observability sink diversion (8.2+ build, single cutoff, all gateways):**

Controller 8.2 introduced a `dcf_logs_obs_sink` feature flag that diverts DCF logs from
the traditional rsyslog profile 9 pathway to a newer delivery channel. If enabled but the
obs_sink delivery channel fails, all logs stop silently while rsyslog appears correctly
configured.

1. Call `aviatrix_search_controller_logs(search_pattern="dcf_logs_obs_sink", max_lines=200)` — if a feature_info dump appears in the log with `enabled: true`, this flag is active. Absence of the dump does not rule it out; it only surfaces when the controller logs a periodic feature dump.
2. Call `aviatrix_search_controller_logs(search_pattern="conduit", max_lines=200)` — look for obs_sink delivery errors or handoff failures
3. Cross-reference the cutoff date against known events: controller upgrade to 8.2, CoPilot migration, direct API calls outside Terraform
4. If `dcf_logs_obs_sink: enabled=true` is confirmed: surface the finding and this resolution path to the customer's Aviatrix admin — the flag can be disabled via `POST /v2/api action=enable_feature feature_name=dcf_logs_obs_sink enable=false`. This is a controller API call outside MCP scope; recommend they test it and open an Aviatrix support ticket referencing the flag and controller build.
5. If obs_sink is active and the delivery channel is the intended path (not a regression): check that CoPilot OTEL receiver (TCP 31284) is reachable from gateway EIPs — NSG blocking this port silently drops all obs_sink delivery.

**Resolution signal:** `aviatrix_count_dcf_logs` on the affected gateway with a 24h window returns > 0.

### Audit trail stale

1. Call `aviatrix_search_dcf_audit` with 30d window; if 0 results, try 90d; if still 0, try all-time
2. Check user attribution across returned events — is the `user` field populated or always empty?
3. Report: last event date, total events all-time, user attribution quality, estimated stale duration
4. If all-time returns 0 events: CoPilot audit logging may never have been enabled or was disabled

**Resolution signal:** `aviatrix_get_dcf_audit_summary` returns at least one event within the last 7 days.

### FlowIQ pipeline broken (`boundary_at_start: true`)

FlowIQ aggregation index has no data for the queried window. Data cutoff date aligns with
the triggering event (upgrade, config change, network change).

1. Establish the cutoff: retry `get_flowiq_top_talkers` with progressively older windows
   (try 7d ago, 14d ago, 30d ago) until `boundary_at_start: false` — the last window with
   data indicates when the pipeline stopped.
2. Cross-check the cutoff date against known changes: controller upgrade, CoPilot restart,
   NSG/firewall rule change.
3. Check CoPilot → Settings → FlowIQ: verify netflow is enabled, collector IP/port are
   correct, and `oteld_enabled` matches the gateway version (8.2+ gateways use OTEL/TCP;
   older gateways use ipt-netflow/UDP — mismatch = silent data loss).
4. Check network: UDP port 2055 must be open from gateway spoke EIPs to CoPilot private IP.
   If NSG added since cutoff date = root cause.
5. Report: cutoff date, likely trigger, fix required (CoPilot setting change or NSG rule).
6. **DCF-log fallback while FlowIQ is down:** Call `aviatrix_get_top_egress(category=ips, start_time=<window_start>, end_time=<now>)` and `category=urls` in parallel. This aggregates over CoPilot DCF firewall logs (not netflow) — top-10 rank-truncated, volume counts not representative of full traffic. Only useful if DCF enforcement is active on at least some spokes. Report as supplemental visibility, not a replacement for FlowIQ.

**Resolution signal:** `aviatrix_get_flowiq_top_talkers` returns `boundary_at_start: false` and a non-empty result set for the original failing window.

### Traffic anomaly (unknown external IP)

1. Call `aviatrix_get_flowiq_top_talkers` with `by_gateway=true` to find which spoke(s) originate the traffic
2. Call `aviatrix_get_flowiq_top_talkers` with `direction=east-west` — identifies which internal RFC1918 source(s) are generating traffic toward the suspect external IP (correlate source spoke with the egress gateway found above)
3. For each RFC1918 source IP from step 2: call `aviatrix_get_cloud_workloads(ip=<src_ip>)` to identify the owning VM (instance type, account, VPC, cloud tags). This converts "traffic from 10.x.x.x" into a named workload. Run in parallel if multiple source IPs.
4. Call `aviatrix_count_dcf_logs` for that IP (24h, then 7d) — if 0, DCF has no visibility into this flow
5. Use the `owner` field from the top_talkers response for ASN/service identification (no external DNS needed)
6. Report: volume, originating spokes, internal source IPs (east-west), owning VMs (from step 3), ASN owner, whether DCF covers those spokes
7. If DCF enforced on spoke but logs = 0 for this IP: flag as potential inspection gap — DCF may be up but the inspection container may be silently failing

**Resolution signal:** Re-run `aviatrix_get_flowiq_top_talkers` for the same window; the IP now resolves to a known CDN/cloud owner, or volume has dropped to noise level. If the concern was unmanaged internal source traffic: the owning VM identified in step 3 has been reviewed by the customer.

### DCF enforcement gap

0. **FireNet check from Tier 1 data:** Cross-reference each unenforced gateway against the `inspected` lists returned by `aviatrix_list_firenet_inspection` (already collected in Tier 1). If the gateway appears in an `inspected` list, vendor FW handles inspection — close this finding, note which FireNet transit covers it. Only proceed to steps 1-3 for gateways not in any FireNet `inspected` list.
1. Call `aviatrix_get_dcf_gateway_rules` for each genuinely unenforced gateway (not FireNet-covered)
2. Distinguish: (a) enforcement disabled + rules exist = enforcement was turned off, (b) enforcement disabled + no rules = was never configured
3. Call `aviatrix_search_controller_logs(search_pattern=<gw_name>)` to check whether a recent API call (e.g. enforcement disable, policy change) correlates with when the gap appeared
4. Report gap type, rule count, any correlating controller log events, and whether this appears intentional (test/dev spoke with no policies is different from a prod spoke that had enforcement then lost it)

**Resolution signal:** `aviatrix_get_dcf_enforced_gateways` lists the gateway as enforced, or `aviatrix_get_dcf_gateway_rules` shows at least one active rule with enforcement enabled on the gateway.

### BGP session short uptime or down

1. Call `aviatrix_run_bgp_diag` on the flagged gateway for full neighbor detail. On transit gateways, heavy commands (`show ip bgp`, `show ip route bgp`, `show running`) return a `job_id` instead of inline output — poll `aviatrix_get_bgp_diag_result(job_id=...)` every 15s until status is `complete` or `error`. Fast commands (`show ip bgp summary`, `show ip bgp neighbors`) always return synchronously.
2. Call `aviatrix_get_gateway_performance` with `gateway_names=[suspect_gw]` — high CPU can cause BGP hold-timer expiry even when the session appears "up"
3. Call `aviatrix_search_controller_logs(search_pattern=<gw_name>)` — look for recent gateway operations (upgrade, config push, HA event) that may have triggered the session reset. Short uptime + recent controller action = likely cause.
4. Check: neighbor state, prefix counts received/sent, uptime, BFD presence
5. Flag independently: no BFD (failover takes up to hold-timer seconds, typically 9s); fragmented /28 prefixes where a supernet would do; asymmetric prefix counts between sessions
6. Report: peer AS, current uptime, BFD state, CPU/memory at investigation time, any controller log events near the session start time, any prefix anomalies

**Resolution signal:** `aviatrix_run_bgp_diag` on the gateway returns `Established` state with uptime > 7 days (green threshold) and prefix counts consistent with peer expectations.

### S2C tunnel down

1. Call `aviatrix_run_s2c_diagnostic` on each down connection
2. If HTTP 403: write-level credentials are required for this tool — fall back to raw status from `aviatrix_list_s2c_connections` and note the access limitation
3. Report: IKE phase, last error string, rekey timing if visible

**Resolution signal:** `aviatrix_list_s2c_connections` shows `UP` status for the affected tunnel.

### Container health (`--deep`)

Already collected in Tier 3. Report per-gateway: container name, restart count, estimated cycle
time, exit code. No additional tool calls needed unless the user wants to inspect a specific
gateway's full syslog.

---

## PDF export (`--pdf`)

When `--pdf` is passed, accumulate all output emitted during the run in a markdown buffer
(scorecard + all investigations). When the user types `done` or `q`, fire the PDF pipeline.

### Step 1: build exec summary

Compose this block as the first section of the document:

```
## Executive Summary

**Controller:** <controller_url>
**Swept:** <YYYY-MM-DD HH:MM UTC>
**Overall status:** 🔴 CRITICAL

| Severity    | Count |
|-------------|-------|
| 🔴 Critical | N     |
| 🟡 Warning  | N     |
| ✅ Green    | N     |

**Key findings:**
- <one-liner per 🔴 and 🟡 finding only; omit green domains>
```

### Step 2: write markdown to /tmp

Write `/tmp/avx-health-<YYYY-MM-DD-HHMMSS>.md` containing:
1. Exec summary (Step 1)
2. Full scorecard (verbatim)
3. All investigation outputs (verbatim, in order triggered)

### Step 3: invoke /make-pdf

```
/make-pdf generate --cover --toc \
  --title "Aviatrix Fabric Health Report" \
  --author "avx-health" \
  --date "<YYYY-MM-DD>" \
  /tmp/avx-health-<timestamp>.md \
  /tmp/avx-health-<timestamp>.pdf
```

### Step 4: print path

```
PDF saved: /tmp/avx-health-2026-05-14-143022.pdf
```

---

## Constraints

- **Port 443 only.** No raw TCP sockets, no external DNS, no WHOIS. IP classification uses
  the `owner` field from `get_flowiq_top_talkers` or `aviatrix_classify_ip` if available.
- **BGP 500s are normal.** Most gateways don't run BGP (FRR not active). HTTP 500 from
  `aviatrix_run_bgp_diag` = BGP not enabled — suppress silently.
- **Standalone spokes (no transit) are valid.** A spoke with empty `transit_gw_name` is not a
  health concern. It may be a NAT egress gateway for a single VPC, a local S2C peer, or
  intentionally isolated. Do not flag as an issue or finding. If the user explicitly asks for
  topology detail, note it as: `ℹ️ No transit attachment (standalone spoke)` — informational only,
  not counted in any score.
- **S2C diagnostic needs write credentials.** 403 from `aviatrix_run_s2c_diagnostic` means
  the API key has read-only scope. Don't retry; report the limitation and use list data instead.
- **DCF audit summary time range.** `aviatrix_get_dcf_audit_summary` may not support a time
  filter. If it returns recent events, use the most recent event timestamp. If the response
  has no timestamp, use `aviatrix_search_dcf_audit` with a 7d window as the freshness check.
- **`--deep` gateway count warning.** If the fabric has >10 gateways and `--deep` is passed,
  tell the user how many syslog calls will be made and confirm before proceeding.
- **`--pdf` buffer.** When `--pdf` is active, accumulate all output in a markdown buffer during
  the run. Write the file only when the user types `done` or `q`. Do not write intermediate
  files mid-investigation.
- **Docs lookup (last resort).** If a feature name, flag, or setting encountered in live tool
  output is not covered by these playbooks, run `WebSearch("site:docs.aviatrix.com <term>")` then
  `WebFetch` the top result. Cap: 1 search + 1 fetch per ambiguity. Do not fetch for anything
  already covered by inline playbook text. Synthesize the result into the finding; do not dump
  raw HTML. Check the controller version (major.minor) against any version tag in the docs URL
  or page — if they differ, note: "docs found for vX.Y; controller is vA.B — verify
  applicability." If no relevant docs page is found, note the gap and proceed with available data.

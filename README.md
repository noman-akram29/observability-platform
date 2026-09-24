# Observability Lab

## Architecture

Two physical Windows hosts, each running WSL2 Ubuntu, treated as four distinct logical hosts:

- **Noman-Win** — Windows 11 Pro, physical host (172.16.9.158)
- **Noman-Linux** — WSL2 Ubuntu 24.04.4 on Noman-Win (172.27.24.46, NAT'd, private to Noman-Win)
- **Faran-Win** — Windows 11, physical host (172.16.9.20)
- **Faran-Linux** — WSL2 Ubuntu on Faran-Win (172.18.236.106, NAT'd, private to Faran-Win)

Both WSL2 guests use default NAT networking (not mirrored). Each is reachable only from
its own physical host unless a port-forwarding rule is added on that host. LAN connectivity
between Noman-Win and Faran-Win is confirmed working (ICMP + TCP). WSL-to-WSL connectivity
across hosts (Noman-Linux <-> Faran-Linux) does not exist yet and will require a
`netsh interface portproxy` rule on the WSL guest's own Windows host when needed.

Observability backend (Prometheus, Grafana, and later Loki/Tempo/Alertmanager) will run on
Noman-Linux. Faran-Linux will act as a remote monitored target.

## Environment

| Host | Role | OS | IP | Firewall |
|---|---|---|---|---|
| Noman-Win | Physical host | Windows 11 Pro | 172.16.9.158 | Enabled (all profiles) |
| Noman-Linux | Observability backend | Ubuntu 24.04.4 LTS (WSL2) | 172.27.24.46 | N/A (WSL, NAT'd) |
| Faran-Win | Physical host | Windows 11 | 172.16.9.20 | Disabled (all profiles) — LAB ONLY |
| Faran-Linux | Monitored target | Ubuntu (WSL2) | 172.18.236.106 | N/A (WSL, NAT'd) |

## Repository Structure

observability-lab/
├── README.md
├── docs/architecture/
├── prometheus/
│ ├── prometheus.yml
│ ├── rules/
│ └── targets/
└── grafana/provisioning/


## Phase 1 — Prometheus

### Purpose
Pull-based metrics collection. Prometheus scrapes `/metrics` HTTP endpoints on a
schedule rather than waiting for targets to push data, so target unavailability
itself becomes observable (`up == 0`) instead of producing silence.

### Architecture
Single binary: scrape manager, TSDB (local disk), PromQL query engine, rule
evaluator, all in one process. No external database.

### Installation
- Version: v3.12.0 (verified against official prometheus.io/download.json —
  Prometheus 3.x line; classic console-template UI removed, new web UI is default)
- Binary installed to `/usr/local/bin/` (not version-controlled)
- Config lives in `prometheus/prometheus.yml` (version-controlled)

### Configuration
Minimal config, self-scrape only:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```
`alerting:` and `rule_files:` intentionally omitted (not commented out) since
neither is in use yet — added explicitly in Phase 2.

### Commands
```bash
promtool check config prometheus/prometheus.yml
prometheus --config.file=prometheus/prometheus.yml --storage.tsdb.path=prometheus/data
```

### Verification
- `curl http://localhost:9090/-/healthy` and `/-/ready`
- `curl http://localhost:9090/api/v1/targets` — target health = "up"
- `curl -G .../api/v1/query --data-urlencode 'query=up'` — value = "1"

### PromQL
- `up` — synthetic liveness gauge per target
- `prometheus_tsdb_head_series` — gauge, active series count (1001 from
  self-monitoring alone — histograms fan out into many series per metric name)
- `rate(prometheus_http_requests_total[1m])` — counter-to-per-second-rate;
  sliding window, not a running total — can legitimately read 0

### Troubleshooting
| Symptom | Cause | Diagnostic | Fix |
|---|---|---|---|
| Empty response, 0 bytes, no error | Unescaped PromQL special chars (`()[]`) in raw curl URL | `curl -s -o /dev/null -w "%{http_code}"` | Use `curl -G --data-urlencode` |
| `rate()` returns empty result | Not enough samples yet in window, or truly no recent traffic to that handler | Re-run after generating traffic | Wait, or generate traffic, and re-check |
| Startup log shows stale lockfile warning | Previous process did not shut down gracefully (crash, kill -9, OOM) | Check for this WARN on every startup | Not fatal — WAL replay handles it; investigate why prior process died |

### Production Notes
- Bound to `0.0.0.0:9090` by default — reachable from LAN if firewall allowed it.
  Currently blocked by Noman-Win's firewall (deliberate, not yet revisited).
- Default retention: 15 days. Not yet tuned for this lab's actual needs.
- WAL provides durability against hard kills (verified: kill -9 test, 3 WAL
  segments replayed cleanly, zero data loss, only lockfile cleanup was skipped).
- LAB ONLY: running in foreground manually. Production needs a supervised
  service (systemd) with restart policy — not yet implemented.

### What I should understand
- Pull vs push model and why `up` matters
- job vs instance label distinction
- Counter vs gauge behavior
- rate() is windowed, not cumulative
- One metric name can be many time series (cardinality)
- Graceful vs hard shutdown, and what the WAL actually protects

## Phase 2 — Node Exporter

### Purpose
Translates kernel/OS state (/proc, /sys) into Prometheus exposition format.
Prometheus has no native knowledge of CPU/memory/disk - Node Exporter is the
reference implementation of the "exporter" pattern.

### Architecture
Single stateless binary, no config file, controlled via CLI flags only.
Architecturally identical to any other scrape target - "just another job"
in Prometheus's eyes, same as self-monitoring.

### Installation
- Version: v1.12.1 (verified against Docker Hub / Helm chart release refs)
- Binary installed to `/usr/local/bin/`
- No config file - default collectors only, no flags used yet

### Configuration
Added as a second scrape job (prometheus.yml):
```yaml
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
```
Job named "node" (category of thing monitored), not "node_exporter" (the
tool) - enables clean `by (instance)` grouping once multiple hosts exist.

### Commands
```bash
node_exporter                 # foreground, default collectors
curl http://localhost:9100/metrics
```

### Verification
- Raw `/metrics` inspected directly before Prometheus ingestion (1054 node_ metric lines)
- `prometheus_tsdb_head_series`: 1001 -> 2210 after onboarding this one target
- Both targets (`prometheus`, `node`) show `health: "up"` via /api/v1/targets

### PromQL
See `prometheus/rules/example-queries.md` for verified queries:
CPU %, memory %, disk %, network rate, load average, host availability.

### Troubleshooting
| Symptom | Cause | Diagnostic | Fix |
|---|---|---|---|
| `up{job="node"}` = 0 | Node Exporter process not running | `ps aux \| grep node_exporter` | Restart node_exporter |
| Detection lag on failure | Scrape interval bounds detection speed | Check `lastScrape` timing in /api/v1/targets | Expected - fastest detection = scrape_interval |
| Some collectors produce zero metrics | WSL2 lacks hardware interfaces (hwmon, rapl, nvme, etc.) | `curl .../metrics \| grep <collector>` | Expected on WSL2 - not a bug |
| Identical avail_bytes across mountpoints | WSL2 mounts share one underlying virtual disk | Compare `device=` label across mountpoints | Expected - not independent filesystems |

### Production Notes
- LAB ONLY: manual foreground execution. Production needs systemd service.
- Exporter port 9100 currently only reachable locally on Noman-Linux -
  cross-host exposure (Faran-Linux) requires the portproxy work noted in
  the Architecture section.
- rate() valid only on counters (node_cpu_seconds_total,
  node_network_*_bytes_total) - never on gauges (node_memory_*, node_load1).

### What I should understand
- Exporter pattern: translates OS state to Prometheus format, nothing more
- Node Exporter is architecturally "just another target" - no special casing
- Multi-label counters (cpu x mode) compound cardinality
- Detection latency is bounded by scrape_interval, not instantaneous
- WSL2-specific metric quirks (shared disk, missing hardware collectors)

## Phase 3 — Grafana

### Purpose
Visualization and query federation layer. Grafana stores no metrics/logs/traces
itself - every panel queries Prometheus (or Loki/Tempo later) live, at render
time. Its only persistent state is its own metadata (dashboards, datasources,
users) in an internal SQLite DB.

### Architecture
Full application tree (not a single binary) - binary + web assets + default
config + bundled plugins. Sits above Prometheus as a query/view layer, never
holds time-series data itself.

### Installation
- Version: v13.1.1 (verified via FreeBSD ports update trail - v13.0 is a major
  release: /api path deprecated in favor of /apis, React 18->19 internally)
- Application tree installed to `/opt/grafana` (not version-controlled)
- Custom config + provisioning in `grafana/` (version-controlled)

### Configuration
`grafana.ini` overrides only `http_port` and `[paths]` (provisioning, data) -
everything else falls back to Grafana's own defaults.ini.

Datasource provisioned as code (`provisioning/datasources/prometheus.yml`):
- `access: proxy` - Grafana's backend queries Prometheus, browser never talks
  to Prometheus directly (security boundary)
- `editable: false` - prevents UI drift from what's in git

Dashboard provisioned as code (`provisioning/dashboards/provider.yml` +
`noman-linux-host-overview.json`) - built manually first via UI, then
exported and committed, per "no blind dashboard imports" rule.

### Commands
```bash
/opt/grafana/bin/grafana server \
  --config=grafana/grafana.ini \
  --homepath=/opt/grafana
```

### Verification
- `curl .../api/health` -> database: "ok"
- `curl .../api/datasources` -> Prometheus datasource present, access=proxy
- `curl .../api/search?query=Noman-Linux` -> dashboard present, same uid
  across manual creation and provisioning-driven restart
- Startup log: `provisioning.dashboard ... "finished to provision dashboards"`
  confirms file-based provisioning actually applied, not just present in DB

### PromQL / Dashboard
6 panels: Host Availability (stat), CPU/Memory/Disk %, Network Receive Rate,
System Load. All queries reused from Phase 2's verified query set.

### Troubleshooting
| Symptom | Cause | Diagnostic | Fix |
|---|---|---|---|
| Datasource works but panel shows wrong magnitude/unit | Wrong unit selected in panel (e.g. "milli" prefix instead of correct SI scale) | Compare panel value against direct `curl` query to Prometheus | Re-select correct unit under Standard Options |
| All panels show "No data" with red warning triangles | Prometheus (the datasource) is down - NOT the same as a target being down | Click the warning triangle for the actual connection error | Restart Prometheus; Grafana auto-recovers on next refresh, no manual nudge needed |
| Graph line looks continuous across a known outage window | Grafana connects across null/missing points by default | Compare against known outage timestamps | Deliberate dashboard design choice - not a bug, revisit null-fill settings if gaps should be visible |
| SQLite "database is locked" on startup | Concurrent internal writes during migrations/provisioning | Check `/api/health` after startup completes | Benign at this scale; production moves to Postgres/MySQL |

### Production Notes
- LAB ONLY: manual foreground execution, default admin credentials changed
  but no SSO/RBAC configured.
- SQLite is the internal DB - fine for single-instance lab use, not
  recommended once multiple Grafana instances or real concurrent load exist.
- "No data" vs "DOWN" distinction matters operationally - a dashboard alone
  cannot distinguish "target unhealthy" from "monitoring backend unhealthy."
  This gap is exactly why "monitor the monitoring stack" is a later phase.

### What I should understand
- Grafana holds no time-series data - it is a live query/view layer only
- access: proxy vs direct - security boundary, browser never talks to backend directly
- Provisioning-as-code prevents UI/git drift (editable: false enforces this)
- "No data" (datasource unreachable) != "DOWN" (target unreachable but datasource fine) - different failure domains, different detection needs
- Grafana auto-retries a failed datasource with no manual intervention required

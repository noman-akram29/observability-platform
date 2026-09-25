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

## Cross-Host Networking — Faran-Linux Onboarded

### Problem
Noman-Linux (WSL2 NAT, 172.27.x) and Faran-Linux (WSL2 NAT, 172.18.x) each sit
behind their own private Hyper-V virtual switch, invisible to the other
physical host. No route exists between the two WSL subnets directly.

### Solution
`netsh interface portproxy` on Faran-Win forwards its real LAN IP:port into
Faran-Linux's private WSL IP:port:
```powershell
netsh interface portproxy add v4tov4 listenaddress=172.16.9.20 listenport=9100 connectaddress=172.18.236.106 connectport=9100
```
Full path proven end-to-end: Noman-Linux (WSL NAT egress) -> LAN ->
Faran-Win:9100 (portproxy) -> Faran-Linux:9100 (Node Exporter).

LAB ONLY caveat: portproxy rules are NOT persistent across Windows reboots,
and the connectaddress (WSL2 internal IP) can change across WSL restarts
(DHCP-leased from the internal switch). Production would need this
re-applied via a startup script bound to the current WSL IP - not
implemented here.

### Configuration
Faran-Linux added to the existing `job: "node"` (not a new job - same
category of thing being monitored), with a `host` label added to
distinguish true host identity from the network path used to reach it
(instance shows the portproxy address, not Faran-Linux's real WSL IP):
```yaml
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          host: "Noman-Linux"
      - targets: ["172.16.9.20:9100"]
        labels:
          host: "Faran-Linux"
```

### Verification
- Reachability proven in isolated stages: Noman-Win -> Faran-Win (portproxy)
  first, then Noman-Linux -> same path, separating "does the portproxy work"
  from "does Noman-Linux's own WSL NAT egress work"
- Both `node` targets show `health: "up"` with distinct `host` labels
- `node_load1` returns two genuinely distinct values from two real hosts

### Troubleshooting
| Symptom | Cause | Diagnostic | Fix |
|---|---|---|---|
| Orphaned time series with missing label after a relabel/config change | Prometheus does not migrate history when a label set changes - it starts a new series and abandons the old one | `query{label=""}` to match the label-absent series explicitly | Expected behavior, not a bug. Old series goes stale after ~5 min (default staleness timeout, NOT the scrape interval) and stops appearing in instant queries |
| `up=0`, error = "connection refused" | AMBIGUOUS: could mean the target process died, OR a network path/proxy between Prometheus and the target broke | Cannot be determined from Prometheus's error text alone - must check target process, then network path, then any proxy/forwarding layer, in order | This is a real, unavoidable limitation - `up` tells you THAT something failed, never WHY |
| Cross-host target unreachable despite exporter and firewall both fine | WSL2-to-WSL2 across two hosts has no route by default (double NAT) | Test in isolated hops: Windows-to-Windows first, then WSL-to-Windows, before assuming exporter is broken | netsh portproxy on the target's own Windows host |

### What I should understand
- Two independent WSL2 NAT boundaries do not compose into a route - each
  needs its own forwarding solution, tested in isolation before combining
- Relabeling/adding labels creates new series, orphans old ones - a
  real, quiet cardinality cost of routine config changes
- up=0 is a binary signal only - diagnosing WHY requires checking every
  layer in the path, not trusting the error text to be specific
- Per-target failure isolation confirmed directly (Faran-Linux down did
  not affect Noman-Linux's health at all)

## Phase 4 — Alertmanager

### Purpose
Takes Prometheus's alert-rule evaluations (is a condition true?) and handles
what happens next: routing, grouping, deduplication, inhibition, delivery.
Deliberately a separate process from Prometheus - detection logic and
notification/routing logic are different concerns.

### Architecture
Standalone binary (not a library inside Prometheus). Always initializes a
gossip/cluster subsystem (port 9094) even running solo - this is the
mechanism used for HA deduplication across multiple Alertmanager replicas
in production; irrelevant but harmless for our single-instance lab.

### Installation
- Version: v0.34.0 (verified via Helm chart update reference)
- CAUTION: `apt install prometheus-alertmanager` installs v0.26.0 (8 minor
  versions behind) AND auto-starts a systemd service bound to :9093 -
  had to stop/disable/purge it before using the correct binary. Real
  lesson: never blindly follow an apt "not found" suggestion in this
  project - always check whether an official binary is the intended path.
- Binary + amtool installed to /usr/local/bin/ (not version-controlled)
- Config in `alertmanager/alertmanager.yml` (version-controlled)

### Configuration
```yaml
route:
  group_by: [\'alertname\', \'host\']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: \'local-webhook\'

receivers:
  - name: \'local-webhook\'
    webhook_configs:
      - url: \'http://127.0.0.1:5001/\'
```
Receiver is a local Python HTTP listener (`webhook-listener.py`) - chosen
deliberately over email/Slack for the first pass, to prove the pipeline
mechanism using only components already in the lab, before adding real
external notification dependencies.

group_by includes `host` (not just alertname) - prevents Noman-Linux and
Faran-Linux failures from silently merging into one ambiguous notification.

### Alert Rules (prometheus/rules/host-alerts.yml)
| Alert | Threshold | for: | severity | Reasoning |
|---|---|---|---|---|
| HostDown | up{job="node"}==0 | 1m | critical | Fast - full unavailability is unambiguous and urgent |
| PrometheusTargetDown | up{job="prometheus"}==0 | 1m | critical | LIMITATION: can only catch Prometheus degrading while still partially alive - a full crash means nothing is left to evaluate this rule or notify. True crash detection needs an EXTERNAL watchdog (later "monitor the monitoring stack" phase) |
| HighCPU | >85% | 5m | warning | Headroom before real saturation; 5m filters transient spikes (e.g. our own curl/jq bursts earlier) |
| HighMemory | >90% | 5m | warning | Higher than CPU - much of "used" memory is reclaimable cache, not genuinely committed |
| DiskAlmostFull | >85% | 10m | warning | Disk fill is always a slow trend, never a legitimate transient - longest window, no benefit to a short one |

### Verification
- Full alert lifecycle proven for HostDown: inactive -> pending (activeAt
  matches up flip) -> firing (exactly for: duration later) -> resolved
  (real endsAt timestamp) -> inactive, via real webhook payloads
- Measured real latency: firing to notification ~30s (group_wait);
  condition-cleared to resolution-notification ~1m45s (evaluation_interval
  + Alertmanager resolve_timeout stacking)

### Troubleshooting
| Symptom | Cause | Diagnostic | Fix |
|---|---|---|---|
| Rule reaches "pending" then resets to "inactive" without firing | rate() over a [5m] window needs the FULL window filled with consistently-over-threshold samples - a rolling window still containing pre-load idle samples reports a lower value than true current load, so a short/interrupted stress test can dip back under threshold before for: completes | Run one continuous load for significantly longer than for: alone suggests (window fill time + for: duration, not just for: duration) | Sustained, uninterrupted load; watch the raw query value ramp up over time, not just the alert state |
| bash uses a stale binary path after replacing an apt-installed tool | bash caches (hashes) resolved command locations per shell session | `which <tool>` shows an old path that no longer exists | `hash -r` to force PATH re-resolution |

### What I should understand
- Alert lifecycle: inactive -> pending -> firing -> resolved, each
  transition driven by a real, measurable timing mechanism (for:,
  group_wait, resolve_timeout) - not instantaneous
- rate() window fill time and for: duration COMPOUND - a synthetic test
  needs to account for both, not just the stated for: value
- Labels drive routing; annotations are display-only, and templating a
  label that does not exist on that alert\'s series silently renders empty
  rather than erroring
- Self-monitoring has a hard limit: a rule cannot detect the total failure
  of the system evaluating it - only external monitoring can
- Never trust an apt "package not found" suggestion without checking
  whether it conflicts with an intentionally-versioned binary install

## Production Hardening — systemd Services

### Purpose
Every component up to this point ran manually in a foreground terminal -
explicitly marked LAB ONLY throughout every phase. This converts all five
to systemd services with automatic restart, closing that gap.

### What changed
All five components (Prometheus, Node Exporter, Grafana, Alertmanager, and
our own webhook-listener test tool) now run as systemd units with
`Restart=on-failure` and `RestartSec=5`. Unit files committed to
`systemd/` in this repo (copies of what is installed at
`/etc/systemd/system/`).

Key differences from manual execution:
- `ExecStart` uses full absolute paths (no reliance on PATH or CWD except
  where `WorkingDirectory` is explicitly set for relative config paths)
- Output goes to the systemd journal, not a terminal -
  `journalctl -u <service> -f` replaces watching a foreground terminal
- Grafana's service is deliberately named `grafana-lab`, not `grafana` -
  avoids future collision with Grafana's own official package unit name
  (`grafana-server.service`), which does not exist here since we installed
  manually via tarball, not apt

### Verification
- Each service individually hard-killed (`kill -9`) and confirmed to
  auto-recover with a new PID within the RestartSec window, no manual
  intervention
- ALL FIVE simultaneously hard-killed (`pkill -9` across all processes at
  once) and confirmed full-stack auto-recovery: all targets returned to
  `up`, dashboard intact, all 5 alert rules reloaded correctly from disk
  and back to `inactive`

### Troubleshooting
| Symptom | Cause | Diagnostic | Fix |
|---|---|---|---|
| Service shows `activating (auto-restart)` repeatedly, never reaches `active (running)` | A genuine startup failure being masked as a restart loop - NOT actual recovery | `journalctl -u <service> -n 30` to see the real exit error, not just `systemctl status`'s truncated summary | Diagnose the real error (in our case: leftover foreground process still holding port 9094, blocking the new systemd-managed instance from binding it) |
| Alertmanager fails to bind port 9094 | An old foreground/manual instance was never actually stopped before starting the systemd service - two processes competing for the same port | `sudo ss -tlnp \| grep 9094` to find the real PID holding the port | Kill the orphaned process explicitly, then start the service fresh |

### What I should understand
- `Restart=on-failure` is the actual production behavior every prior
  phase's "LAB ONLY: manual execution" caveat was pointing toward
- `systemctl status` showing `activating (auto-restart)` is a FAILURE
  signal, not a recovery signal - a service endlessly restarting without
  reaching `active (running)` needs `journalctl` investigation immediately,
  not assumption that "it'll figure itself out"
- Moving to systemd changes where output goes (journal, not terminal) -
  a real operational shift, not just a cosmetic one
- Full-stack simultaneous failure recovery is a meaningfully different,
  stronger guarantee than individually-tested single-component recovery

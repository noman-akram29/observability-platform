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

_Not started._

## Phase 3 — Grafana

_Not started._

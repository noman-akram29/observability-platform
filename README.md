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

_In progress._

## Phase 2 — Node Exporter

_Not started._

## Phase 3 — Grafana

_Not started._

# Node Exporter — Verified PromQL Queries (Noman-Linux, local)

CPU utilization %:
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

Memory utilization %:
100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)

Disk utilization % (root):
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

Network receive rate (bytes/sec), per interface:
rate(node_network_receive_bytes_total[5m])

System load (1 min):
node_load1

Host availability:
up{job="node"}

Notes:
- WSL2 caveat: /, /mnt/wslg/distro, /var/lib/docker often share one
  underlying virtual disk (/dev/sde) - identical avail_bytes across
  mountpoints is expected, not a bug.
- eth0 network counters may show 0 if no real cross-network traffic has
  occurred recently - traffic to localhost goes over lo, not eth0.
- rate() is only valid on counters (node_cpu_seconds_total,
  node_network_receive_bytes_total). Never apply rate() to a gauge
  (node_memory_*, node_load1) - use direct arithmetic instead.

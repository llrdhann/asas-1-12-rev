# Server B cross-monitoring (Prometheus A)

Prometheus/Grafana/Alertmanager on Server A monitor Server B without managing
or changing Server B. The friend who owns Server B remains responsible for its
applications, exporters, firewall, and monitoring stack.

## Required endpoints

| Purpose | Endpoint | Owner |
|---|---|---|
| Host metrics | `172.20.3.242:9100/metrics` | Existing Node Exporter on B |
| MariaDB metrics | `172.20.3.242:9104/metrics` | Existing mysqld_exporter on B |
| Moodle HTTP/HTTPS | Exact URL in `peer_moodle_url` | Existing Moodle on B; probed by Blackbox Exporter on A |

Restrict TCP 9100 and 9104 to the current Server A source address configured in `inventory/host_vars/serverA.yml` (`server_ip`). Update this allowlist if Server A IP changes.
Do not expose MariaDB TCP 3306 to Prometheus or to the general network. The
mysqld_exporter must use a dedicated least-privilege database account.

Set `peer_moodle_url` in `inventory/group_vars/managed/main.yml` to the real
Moodle URL reachable from Server A. The Moodle scrape job and Moodle alerts are
intentionally omitted while this value is empty.

## PromQL panels

### Server health (Node Exporter, job `peer_node_exporter`)

Server/exporter scrape status:

```promql
up{job="peer_node_exporter"}
```

CPU usage percent:

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{job="peer_node_exporter",mode="idle"}[5m])) * 100)
```

RAM usage percent:

```promql
100 * (1 - node_memory_MemAvailable_bytes{job="peer_node_exporter"} / node_memory_MemTotal_bytes{job="peer_node_exporter"})
```

Root filesystem usage percent:

```promql
100 * (1 - node_filesystem_avail_bytes{job="peer_node_exporter",mountpoint="/",fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{job="peer_node_exporter",mountpoint="/",fstype!~"tmpfs|overlay"})
```

Network receive/transmit bytes per second:

```promql
rate(node_network_receive_bytes_total{job="peer_node_exporter",device!~"lo|docker.*|veth.*"}[5m])
rate(node_network_transmit_bytes_total{job="peer_node_exporter",device!~"lo|docker.*|veth.*"}[5m])
```

Uptime seconds:

```promql
time() - node_boot_time_seconds{job="peer_node_exporter"}
```

### Moodle (Blackbox Exporter, job `peer_moodle_http`)

HTTP availability (`1` success, `0` failure):

```promql
probe_success{job="peer_moodle_http"}
```

Response time seconds:

```promql
probe_duration_seconds{job="peer_moodle_http"}
```

HTTP status code:

```promql
probe_http_status_code{job="peer_moodle_http"}
```

TLS certificate days remaining:

```promql
(probe_ssl_earliest_cert_expiry{job="peer_moodle_http"} - time()) / 86400
```

### MariaDB (mysqld_exporter, job `peer_mariadb`)

Exporter scrape status:

```promql
up{job="peer_mariadb"}
```

Database health (`1` healthy, `0` exporter cannot connect to MariaDB):

```promql
mysql_up{job="peer_mariadb"}
```

Connected threads:

```promql
mysql_global_status_threads_connected{job="peer_mariadb"}
```

Running threads:

```promql
mysql_global_status_threads_running{job="peer_mariadb"}
```

Queries per second:

```promql
rate(mysql_global_status_queries{job="peer_mariadb"}[5m])
```

Slow queries added in the last 10 minutes:

```promql
increase(mysql_global_status_slow_queries{job="peer_mariadb"}[10m])
```

Connection limit usage percent:

```promql
100 * (mysql_global_status_threads_connected{job="peer_mariadb"} / mysql_global_variables_max_connections{job="peer_mariadb"})
```

## Alerts configured on Server A

- `PeerNodeExporterDown`: Server B Node Exporter scrape unavailable for 2 minutes.
- `PeerHighCPU`: CPU over configured threshold (default 85%) for 5 minutes.
- `PeerHighMemory`: RAM over configured threshold (default 90%) for 5 minutes.
- `PeerHighDisk`: root filesystem over configured threshold (default 90%) for 5 minutes.
- `PeerMoodleDown`: Moodle HTTP probe fails for 2 minutes (only when URL is configured).
- `PeerMoodleSlow`: Moodle probe duration over 5 seconds for 5 minutes (only when URL is configured).
- `PeerMoodleSSLCertificateExpiring`: Moodle TLS certificate has fewer than 14 days remaining (only when URL is configured).
- `PeerMariaDBExporterDown`: MariaDB exporter scrape unavailable for 2 minutes.
- `PeerMariaDBDown`: exporter responds but reports that it cannot connect to MariaDB.
- `PeerMariaDBHighConnectionUsage`: connected threads exceed 80% of `max_connections` for 5 minutes.
- `PeerMariaDBSlowQueriesIncreasing`: new slow queries are observed in the 10-minute window.

`up == 0` reports that Prometheus cannot scrape the endpoint; it does not
prove the entire server is powered off. A failed Node Exporter scrape can also
mean exporter failure, firewall filtering, or a network path problem.

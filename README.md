# Moodle migration and monitoring automation

## Current topology and ownership

- **Server A (`172.20.3.100`)**: the only Ansible deployment target in this repository. It will receive the Moodle migration from Server `.211`.
- **Source Server (`172.20.3.211`)**: migration source and rollback system. Do not modify or retire it until Server A passes all validation.
- **Peer Server (`172.20.3.242`)**: belongs to a friend and runs its own applications and monitoring stack. This repository does not manage or change anything on it. Prometheus on Server A only scrapes the peer's explicitly exposed monitoring endpoints.

Only `serverA` is an Ansible deployment target. `serverB` is inventory data used to render Prometheus scrape targets; no play targets `serverB`.

## Deployment safety and scope

- All deployment playbooks target `serverA` only. `serverB` is a peer-only inventory entry for Prometheus scrape targets.
- Bind9 is installed and configured natively on the Ubuntu host by the `bind9` role.
- Moodle PHP-FPM, MariaDB 10.11, Nginx, and the local MariaDB exporter run in Docker Compose.
- The old fresh-install Moodle 5.1 role remains disabled and is not part of any playbook.
- Restore automation verifies the audited source archives and stages them; it intentionally does not overwrite live code or import SQL until the archive layout and SQL directives have been checked.

## Repository layout

```text
.
├── ansible.cfg
├── AUDIT-STATUS.md
├── inventory/
│   ├── hosts.ini
│   ├── group_vars/managed/main.yml
│   └── host_vars/{serverA,serverB}.yml
├── playbooks/
│   ├── prepare-serverA.yml
│   ├── prepare-moodle-db.yml
│   ├── restore-moodle.yml
│   ├── site.yml
│   ├── validate.yml
│   ├── ssl-renew.yml
│   └── reset-monitoring.yml
└── roles/{common,bind9,docker_engine,ssl,moodle,website,monitoring}/
```

All active inventory variables live under `inventory/`. Do not recreate duplicate `group_vars/` or `host_vars/` directories at repository root.

## Secrets

Create the vault file from the placeholder example:

```bash
cp inventory/group_vars/managed/vault.yml.example inventory/group_vars/managed/vault.yml
nano inventory/group_vars/managed/vault.yml
ansible-vault encrypt inventory/group_vars/managed/vault.yml
```

Use newly generated secrets. A credential was present in the old example file; revoke/rotate it if it is still active. Never commit `vault.yml`, Moodle `config.php`, `.my.cnf`, database dumps, private keys, certificates, or Moodle runtime data.

Install the required collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

Check the inventory and syntax before running anything:

```bash
ansible-inventory --graph
ansible-playbook playbooks/prepare-serverA.yml --syntax-check
ansible-playbook playbooks/prepare-serverA.yml --list-hosts
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml --list-hosts
ansible-playbook playbooks/prepare-moodle-db.yml --syntax-check
ansible-playbook playbooks/restore-moodle.yml --syntax-check
```

The `--list-hosts` output must list **only `serverA`**. Do not run `ansible all` or target `serverB`.

## Cross-monitoring Server B (read-only from Server A)

Prometheus on Server A collects only these peer signals; it does not install or configure services on Server B:

- Host health/resources from the friend's existing Node Exporter at `172.20.3.242:9100`.
- Moodle HTTP/HTTPS availability, response time, HTTP status and TLS expiry through Blackbox Exporter running locally on Server A. Set `peer_moodle_url` in `inventory/group_vars/managed/main.yml` to the exact Moodle URL reachable from Server A. Until set, the Moodle scrape job and Moodle-specific alerts are omitted; the dashboard panels show no data for Moodle.
- MariaDB health and database metrics from the friend's existing MySQL/MariaDB Exporter at `172.20.3.242:9104` (`mysql_up`, connections, running threads, queries/sec and slow-query counter).

The friend must explicitly authorize and expose the exporter endpoints to Server A, preferably restricted by firewall to the current Server A IP configured as `server_ip` in `inventory/host_vars/serverA.yml`. Do not expose MariaDB TCP 3306 publicly; Prometheus should scrape the exporter endpoint, not the database port. The exporter must use a dedicated least-privilege database account on Server B. No Ansible play targets Server B. Docker Engine metrics are disabled by default because the requested scope is host, Moodle and database health.

This repository configures monitoring on Server A only. Local Prometheus jobs cover Server A Node Exporter, Moodle HTTPS/latency via Blackbox Exporter, and MariaDB via a least-privilege local mysqld-exporter account. The Server A Grafana dashboard includes host CPU/RAM, Moodle HTTPS availability, and MariaDB health/connections/queries. Any reverse monitoring from Server B to Server A remains the friend's responsibility on Server B.

## Reset and redeploy monitoring on Server A

A separate reset playbook is provided at `playbooks/reset-monitoring.yml`. It
is restricted to the `serverA` inventory host and verifies the stable
`server_role: primary` identity. Its connection address follows `server_ip`
from `inventory/host_vars/serverA.yml`. It does not SSH to, stop, or modify
Server B or the old Moodle source server.

- Configuration reset (default): stops the monitoring services and removes
  the Ansible-managed Prometheus, Alertmanager, Blackbox Exporter, and Grafana
  provisioning files. Prometheus/Alertmanager history and Grafana data remain.
  Run `ansible-playbook playbooks/prepare-serverA.yml --limit serverA` afterward to recreate the monitoring files and start the services.

  ```bash
  ansible-playbook playbooks/reset-monitoring.yml --syntax-check
  ansible-playbook playbooks/reset-monitoring.yml
  ansible-playbook playbooks/prepare-serverA.yml --limit serverA
  ```

- Full reset: additionally removes monitoring data/config directories,
  custom exporter binaries and systemd units, Grafana package/repository, and
  monitoring service accounts. This deletes Prometheus and Alertmanager
  history and Grafana's database/dashboard state. It requires an explicit
  confirmation variable:

  ```bash
  ansible-playbook playbooks/reset-monitoring.yml --syntax-check
  ansible-playbook playbooks/reset-monitoring.yml \\
    -e reset_mode=full -e confirm_full_reset=true
  ansible-playbook playbooks/prepare-serverA.yml --limit serverA
  ```

Use full reset only when a clean reinstall is intended. The reset playbook
never includes `serverB` in its host pattern.


## Standalone Server A deployment (architecture matched to source `.211`)

The deployment intentionally mirrors the audited source topology: **Bind9 runs natively on Ubuntu and is fully managed by Ansible**; Moodle PHP-FPM, MariaDB 10.11, and Nginx run in Docker Compose. Moodle is pinned to the source code archive (5.2.3+ Build 20260916), with PHP 8.3-FPM and the source Nginx routing behavior. Static website content is not included in this repository; copy the exact source `web/` files to `/opt/moodle/web` on A before the final deploy. A minimal placeholder is only created if the directory is empty.

### Before deployment

1. The three already-audited backup archives are expected on Server A under `/opt/moodle/backup/` (the location confirmed during the source audit) using these exact names: `moodle.sql.gz`, `moodle-source.tar.gz`, and `moodledata.tar.gz`. The restore playbook checks their recorded SHA-256 values before extracting anything.
2. Create `inventory/group_vars/managed/vault.yml` from the example and encrypt it with `ansible-vault encrypt`. Use new long random DB root, DB user, exporter, and Grafana passwords (hex output from `openssl rand -hex 32` is suitable for Compose `.env` and PHP config quoting). Never use credentials from the old source Compose file.
3. Review `bind_network_cidr` and any forwarders for the actual LAN. Recursion is restricted to localhost and that CIDR; do not expose open recursion.
4. Confirm Server A is a clean target and has a separate rollback backup before any data import. The restore playbook refuses to overwrite non-empty Moodle directories or import into a non-empty database. Keep the separate rollback SQL backup intact.

### Deployment sequence

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory --graph
ansible-playbook playbooks/prepare-serverA.yml --syntax-check
ansible-playbook playbooks/prepare-serverA.yml --list-hosts
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml --list-hosts
ansible-playbook playbooks/prepare-moodle-db.yml --syntax-check
ansible-playbook playbooks/restore-moodle.yml --syntax-check
```

`--list-hosts` must show only `serverA`. Run the infrastructure pass after Vault is ready with `ansible-playbook playbooks/prepare-serverA.yml --limit serverA`. This prepares the host, automated native Bind9, Docker, SSL, website placeholder and monitoring without touching Moodle data. The full `site.yml` adds Moodle only after `/opt/moodle/moodle/public/index.php` has been restored; the Moodle role intentionally fails closed if that entry point is absent. Stage and inspect backups with:

```bash
ansible-playbook playbooks/restore-moodle.yml --syntax-check
ansible-playbook playbooks/restore-moodle.yml --limit serverA \
  -e confirm_moodle_restore=true \
  -e restore_confirmation_text=RESTORE_MOODLE_TO_172.20.3.100
```

This playbook verifies SHA-256 and extracts archives only to `/srv/moodle/restore-staging`; it does **not** replace live code, restore moodledata into the live path, or import SQL. Inspect the extracted tree and SQL dump header first, then perform the controlled cutover/restore with a fresh backup of A. This is intentionally conservative because archive layout and SQL directives must be confirmed before destructive import. After a verified manual cutover places the audited source in `/opt/moodle/moodle`, moodledata in `/opt/moodle/moodledata`, and the source SQL has been imported into the new MariaDB database, run `ansible-playbook playbooks/site.yml --limit serverA`, then `playbooks/validate.yml`. SQL import is deliberately not automated in this package until the exact dump directives and live target rollback procedure have been rechecked. To prepare the destination database container without starting Moodle, run `ansible-playbook playbooks/prepare-moodle-db.yml --limit serverA`. Inspect the SQL dump header/directives and take a separate backup of A before importing; do not drop or recreate the named MariaDB volume as a shortcut.

### SSL and IP changes

A single new self-signed certificate is generated on A with DNS SANs for the apex, `www`, and `e-learning` domains plus the current `server_ip`. Its private key stays on A with mode 0600 and is never committed. Change `server_ip` in `inventory/host_vars/serverA.yml`; `ansible_host` follows it automatically. Then rerun `site.yml`: the SSL role checks all SANs and regenerates the certificate if the IP/domain set no longer matches. Force regeneration explicitly with `ansible-playbook playbooks/ssl-renew.yml --limit serverA`. Update external/public DNS and router/firewall rules separately; Ansible only manages the host's authoritative Bind9 zones. Self-signed HTTPS remains untrusted by browsers until the certificate is installed as trusted on client devices.

### Important limitations / cutover notes

- The original source certificate contained the old IP `.45`; A receives a new certificate with the currently configured `server_ip`.
- Bind9 forward and reverse zones are rendered from variables, and all configuration/zone files are checked with `named-checkconf` and `named-checkzone` before service reload. Recursion is LAN-restricted.
- The Moodle restore playbook is staging-only by design. Do not claim the Moodle migration is complete until the SQL dump structure, archive layout, config.php, ownership, DB import, login, course files, cron, and plugin/theme behavior are validated.
- The Docker Compose uses the same named MariaDB volume identity as the source (`moodle_mariadb_data`) only if explicitly set in Compose; on A this creates a new local volume. Do not point A at `.211` over the network.
- The existing monitoring role continues to target Server A only and scrapes Server B as an external peer. The local MariaDB exporter account is provisioned by Ansible with a least-privilege SQL grant; its password is kept in Vault and the generated client file is mode 0400, owned by the exporter container UID/GID 65534.


### Changing Server A IP

The single source of truth is `inventory/host_vars/serverA.yml` (`server_ip`). `ansible_host` is templated from it, and the Bind9 forward records, reverse PTR host octet, Node Exporter bind address, exporter publish address, Grafana domain, and SSL IP SAN consume the configured value. Restore/reset safety checks use the stable `server_role: primary` identity rather than pinning the old IP.

If Server A moves to a different IPv4 /24 subnet, also update `bind_network_cidr`, `bind_reverse_zone`, and the reverse-zone network/file settings in `inventory/group_vars/managed/main.yml`; the reverse zone is intentionally explicit because its name depends on the subnet, not only the host address. Update the network interface address, gateway, and DNS resolver in Ubuntu/Netplan separately before reconnecting Ansible. This repository does not guess the NIC name or gateway and does not change the host's network interface automatically. Ensure SSH is reachable at the new IP before running Ansible.

After the OS network is reachable at the new IP, run `ansible-playbook -i inventory/hosts.ini playbooks/site.yml --limit serverA`. The self-signed certificate is regenerated automatically when its IP SAN no longer matches. Use `playbooks/ssl-renew.yml` only when a forced renewal is wanted. Public DNS, router/NAT, DHCP reservations, and client trust stores remain separate changes.

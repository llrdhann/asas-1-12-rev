# Audit status (2026-11-17)

## Static fixes applied in this revision

- Prometheus now scrapes Server A Node Exporter at `server_ip:9100`, matching the Node Exporter systemd bind address.
- The local MySQL Exporter runs explicitly as UID/GID `65534:65534`; its mounted `.my.cnf` is owned by that UID/GID and mode `0400`, avoiding the root-only file permission failure while keeping credentials private.
- Blackbox Exporter now has separate modules: the Server A local probe sets Moodle Host/SNI explicitly, while the Server B probe derives Host/SNI from `peer_moodle_url` instead of incorrectly reusing Server A’s hostname.
- SSL validation now requires both certificate and private key, validates the private key, and checks that the public keys match. After certificate regeneration, the PHP image is rebuilt and Nginx is explicitly restarted to load the new mounted certificate.
- Moodle restore staging now requires exactly one `index.php` candidate, rather than accepting ambiguous archive layouts.
- HTTPS validation now uses `curl --resolve` so the request reaches local Nginx with the correct Moodle hostname and TLS SNI without depending on public DNS or the host resolver.

## Still intentionally not automated / must be completed

1. `restore-moodle.yml` only verifies SHA-256 and extracts archives to `/srv/moodle/restore-staging`. It does not copy the source into `/opt/moodle/moodle`, restore `moodledata` into `/opt/moodle/moodledata`, import `moodle.sql.gz`, or implement a tested rollback. Therefore a full `site.yml` run will stop at the Moodle role until a human-reviewed cutover has populated the live paths.
2. The actual source static website files from `.211:/opt/moodle/web` are not bundled. The website role creates only a placeholder if no `index.html` exists. Copy the original website content before treating the site as migrated.
3. Ubuntu Netplan/NIC address, gateway, and resolver are not managed. Configure the host network and confirm SSH access at the new `server_ip` before running Ansible.
4. Server B remains peer-only. Its firewall allowlist must permit the current Server A IP to reach only the agreed exporter endpoints (9100 and 9104); this repository does not alter Server B.
5. If changing subnets, update `bind_network_cidr`, `bind_reverse_zone`, and `bind_reverse_file` along with `server_ip`. Public DNS, router/NAT, DHCP reservations, and client trust of the self-signed certificate are external tasks.
6. `peer_moodle_url` is blank by default, so the Server B Moodle HTTP probe and related alerts remain disabled until the real reachable URL is configured.
7. A previously initialized MariaDB Docker volume retains its original credentials; changing Vault passwords does not rewrite existing database accounts. Verify whether `moodle_mariadb_data` is fresh or already initialized before expecting the new Vault password to authenticate. Do not delete the volume as a troubleshooting shortcut.
8. The repository does not contain a real encrypted `vault.yml`; create it from the example, set unique credentials, encrypt it, and keep it out of Git.

## Validation performed in this audit environment

- ZIP archive integrity checked with `unzip -t`.
- All 23 YAML files parsed successfully with PyYAML.
- Ansible CLI and the `community.docker` collection were not available here, so `ansible-playbook --syntax-check`, inventory evaluation, Jinja rendering against the real facts, and a live deployment were not run.
- No SSH connection to Server A or Server B was available; runtime health is not certified by this audit.

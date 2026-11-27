# Moodle Migration and Monitoring Automation — Server A

Repository ini mengotomatisasi deployment infrastruktur Server A, migrasi Moodle, website statis, DNS, dan monitoring menggunakan Ansible.

Server A merupakan satu-satunya target deployment dalam repository ini. Server B hanya digunakan sebagai peer monitoring dan tidak dikelola oleh Ansible pada repository ini.

> **Penting:** Restore Moodle mengubah direktori aplikasi dan mengimpor SQL ke database tujuan. Jalankan hanya pada Server A yang memang disiapkan untuk migrasi. Pastikan backup rollback tersedia dan sudah diverifikasi. Restore belum menyediakan rollback otomatis jika proses gagal di tengah jalan.

---

## 1. Topologi dan Batas Pengelolaan

| Host | IP saat konfigurasi terakhir | Peran |
|---|---|---|
| Server A | `10.28.95.38` | Target deployment Ansible |
| Server B | `10.28.95.39` | Peer monitoring saja |
| Server sumber lama | `172.20.3.211` | Sumber backup dan rollback |

IP di atas merupakan konfigurasi saat dokumentasi ini diperbarui. Jika alamat berubah, sesuaikan inventory dan konfigurasi jaringan terkait.

### Pembagian layanan

**Server A:**
- Ubuntu Server.
- BIND9 sebagai DNS server native.
- Docker Engine dan Docker Compose.
- Moodle menggunakan PHP-FPM.
- MariaDB 10.11.
- Nginx sebagai reverse proxy.
- Prometheus.
- Grafana.
- Alertmanager.
- Node Exporter.
- Blackbox Exporter.
- MariaDB Exporter.

**Server B:**
- Menyediakan endpoint monitoring yang dapat diakses Server A.
- Tidak menjadi target deployment Ansible.
- Memiliki Moodle dan database independen dari Server A jika layanan tersebut dijalankan di sana.
- Pengelolaan service, exporter, firewall, dan database Server B dilakukan secara terpisah oleh pengelolanya.

### Domain

| Layanan | Domain |
|---|---|
| Website statis | `lingga-ramadhan.net` |
| Website statis WWW | `www.lingga-ramadhan.net` |
| Moodle Server A | `e-learning.lingga-ramadhan.net` |
| Moodle Server B | `e-learning.lingskuy-ramadhan.net` |

Pastikan domain Moodle Server B benar-benar sesuai dengan konfigurasi peer yang digunakan. Jika domain berubah, perbarui `peer_moodle_url` pada group variables.

### Batas pengelolaan

- Semua playbook deployment menggunakan target `serverA`.
- `serverB` hanya digunakan sebagai sumber target monitoring.
- Jangan menjalankan deployment menggunakan `--limit all`.
- Jangan mengubah target deployment menjadi `all`.
- Ansible tidak mengatur Netplan, interface jaringan, IP NIC, gateway, atau resolver OS Server A.
- Konfigurasi jaringan harus sudah benar sebelum playbook deployment dijalankan.
- Jangan menganggap Server B sebagai bagian dari deployment Moodle Server A.

---

## 2. Prasyarat

### 2.1. Control node

Control node membutuhkan:

- Linux atau WSL.
- Python 3.
- Ansible.
- OpenSSH client.
- Git.
- rsync.

Instalasi pada Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y python3 python3-pip ansible openssh-client git rsync
```

Instal collection Ansible yang dibutuhkan:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2.2. Server A

Server A membutuhkan:

- Ubuntu Server yang kompatibel dengan role repository.
- IP statis atau DHCP reservation yang sudah dikonfigurasi.
- Koneksi SSH dari control node.
- User Ansible dengan hak sudo.
- Koneksi internet untuk instalasi package dan Docker image.
- Port yang diperlukan sudah diizinkan melalui firewall.

Port yang perlu diperhatikan:

| Port | Fungsi |
|---|---|
| 22 | SSH |
| 53 TCP/UDP | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3000 | Grafana |
| 9090 | Prometheus |
| 9093 | Alertmanager |
| 9100 | Node Exporter |
| 9104 | MariaDB Exporter |
| 9115 | Blackbox Exporter |

Sesuaikan akses firewall dengan kebutuhan jaringan. Endpoint monitoring tidak perlu dibuka ke internet publik.

### 2.3. Tes koneksi SSH

Contoh:

```bash
ssh ling@10.28.95.38
```

Ganti username dan IP sesuai konfigurasi Server A.

Pastikan SSH berhasil sebelum menjalankan Ansible.

---

## 3. File dan Variabel yang Harus Disesuaikan

Semua path berikut relatif dari root repository.

### 3.1. Inventory — `inventory/hosts.ini`

Contoh:

```ini
[managed]
serverA ansible_user=ling

[monitoring_peers]
serverB

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Penjelasan:

- `managed`: host yang dikelola oleh Ansible.
- `monitoring_peers`: host yang hanya menjadi sumber data monitoring.
- `serverA`: target deployment.
- `serverB`: peer monitoring saja.

Jangan memasukkan Server B ke grup `managed`.

Jangan mengisi `ansible_host` Server B jika Server B tidak akan diakses melalui Ansible. Variabel `server_ip` pada `host_vars/serverB.yml` sudah cukup untuk kebutuhan monitoring.

### 3.2. Server A — `inventory/host_vars/serverA.yml`

Contoh konfigurasi saat ini:

```yaml
---
server_ip: "10.28.95.38"
server_network_cidr: "10.28.95.0/24"

ansible_host: "{{ server_ip }}"
server_role: primary

base_domain: "lingga-ramadhan.net"
moodle_subdomain: "e-learning"
```

Keterangan:

| Variabel | Kegunaan |
|---|---|
| `server_ip` | IP aktual Server A |
| `server_network_cidr` | Network dan prefix subnet |
| `ansible_host` | Alamat yang digunakan Ansible untuk SSH |
| `server_role` | Penanda peran Server A |
| `base_domain` | Domain utama |
| `moodle_subdomain` | Subdomain Moodle Server A |

`server_ip` harus sama dengan IP yang benar-benar terpasang pada interface jaringan Server A.

Mengubah nilai ini tidak otomatis mengubah IP pada Netplan.

Domain Moodle dibentuk dari `moodle_subdomain` dan `base_domain`.

### 3.3. Server B — `inventory/host_vars/serverB.yml`

Contoh konfigurasi saat ini:

```yaml
---
server_ip: "10.28.95.39"
server_role: monitoring_peer_only
```

Jika IP Server B berubah, sesuaikan nilai `server_ip`.

Pastikan alamat tersebut benar-benar milik Server B dan endpoint monitoring dapat diakses dari Server A.

Repository ini tidak menginstal atau mengubah layanan pada Server B.

### 3.4. Group variables — `inventory/group_vars/managed/main.yml`

Periksa variabel berikut:

| Variabel | Kegunaan |
|---|---|
| `bind_network_cidr` | Subnet yang diizinkan mengakses DNS |
| `bind_reverse_zone` | Zona reverse DNS |
| `bind_reverse_file` | File reverse DNS |
| `bind_forwarders` | DNS forwarder |
| `peer_host` | Nama host peer monitoring |
| `peer_ip` | IP peer yang diambil dari host variables |
| `peer_moodle_url` | URL Moodle peer |
| `peer_moodle_monitor_enabled` | Mengaktifkan atau menonaktifkan probe Moodle peer |
| `peer_docker_monitor_enabled` | Mengaktifkan atau menonaktifkan monitoring Docker peer |
| `peer_mariadb_monitor_enabled` | Mengaktifkan atau menonaktifkan monitoring MariaDB peer |
| `peer_node_exporter_target` | Target Node Exporter peer |
| `peer_mariadb_exporter_target` | Target MariaDB Exporter peer |
| `moodle_wwwroot` | URL Moodle Server A |
| `ssl_domains` | Domain untuk sertifikat |
| `website_server_names` | Domain website statis |
| `moodle_version_expected` | Versi Moodle yang diharapkan |
| `moodle_build_expected` | Build Moodle yang diharapkan |
| `moodle_source_sha256` | Hash source Moodle |
| `moodledata_sha256` | Hash moodledata |
| `moodle_sql_sha256` | Hash SQL backup |

Contoh relasi variabel peer:

```yaml
peer_host: serverB
peer_ip: "{{ hostvars[peer_host].server_ip }}"

peer_moodle_url: "https://e-learning.lingskuy-ramadhan.net"

peer_moodle_monitor_enabled: false
peer_docker_monitor_enabled: false
peer_mariadb_monitor_enabled: false

peer_node_exporter_target: "{{ peer_ip }}:{{ node_exporter_port }}"
peer_mariadb_exporter_target: "{{ peer_ip }}:{{ mysql_exporter_port }}"
```

Nilai di atas menunjukkan konfigurasi feature flag peer yang saat ini dinonaktifkan. Sesuaikan dengan kondisi layanan Server B.

**Perhatian:**
- Pastikan hostname pada `peer_moodle_url` benar-benar sesuai dengan domain Moodle Server B.
- Gunakan `https://` apabila peer menggunakan HTTPS.
- Jangan menggunakan domain Server A sebagai domain peer jika keduanya berbeda.
- Mengaktifkan monitoring Moodle, Docker, atau MariaDB tidak otomatis menginstal exporter pada Server B.
- Pastikan endpoint yang diaktifkan memang tersedia dan dapat diakses dari Server A.

Jika Server B sedang tidak aktif atau layanan peer belum tersedia, biarkan feature flag terkait dalam keadaan `false`.

Jika Server A pindah subnet, sesuaikan juga variabel jaringan BIND9 dan konfigurasi reverse DNS.

### 3.5. Ansible Vault

Buat file Vault dari contoh:

```bash
cp inventory/group_vars/managed/vault.yml.example \
  inventory/group_vars/managed/vault.yml
```

Buat password acak yang berbeda untuk setiap layanan:

```bash
openssl rand -hex 32
```

Edit file:

```bash
nano inventory/group_vars/managed/vault.yml
```

Variabel yang harus diisi:

```yaml
vault_grafana_admin_password: "PASSWORD_RANDOM"
vault_moodle_db_root_password: "PASSWORD_RANDOM"
vault_moodle_db_password: "PASSWORD_RANDOM"
vault_mysql_exporter_password: "PASSWORD_RANDOM"
```

Enkripsi:

```bash
ansible-vault encrypt inventory/group_vars/managed/vault.yml
```

Untuk mengedit kembali:

```bash
ansible-vault edit inventory/group_vars/managed/vault.yml
```

Jangan commit file Vault asli, password, atau secret ke Git.

Semua playbook yang membutuhkan Vault dijalankan menggunakan:

```bash
--ask-vault-pass
```

Jangan menempelkan password, app password SMTP, atau secret lain ke terminal output yang akan dibagikan maupun ke repository publik.

---

## 4. Backup Moodle

Restore membutuhkan tiga file backup di Server A:

```text
/opt/moodle/backup/
├── moodle.sql.gz
├── moodle-source.tar.gz
└── moodledata.tar.gz
```

### 4.1. SHA-256 backup

Hash backup yang digunakan pada proses migrasi project:

```yaml
moodle_source_sha256: "5fc407337058c89721f3ed96a8badd59ab3a8e234e3eb2db4690e5389716073a"

moodledata_sha256: "4292cc42c7d86bd6ce732c23cad0f4f267e915ead45649afb29b3c15c325fa17"

moodle_sql_sha256: "878de0beec19a42dac9e6b343d6561ccebef3e888eef4f403feae2a88432188c"
```

Verifikasi backup di Server A:

```bash
sudo sha256sum \
  /opt/moodle/backup/moodle-source.tar.gz \
  /opt/moodle/backup/moodledata.tar.gz \
  /opt/moodle/backup/moodle.sql.gz
```

Jika menggunakan backup yang berbeda:

1. Hitung ulang SHA-256.
2. Perbarui variabel hash di group variables.
3. Periksa versi dan build Moodle.
4. Pastikan struktur arsip sesuai.
5. Jangan menonaktifkan validasi hash hanya agar restore dapat dilanjutkan.

Struktur source yang diharapkan:

```text
moodle/
└── public/
    └── version.php
```

Struktur moodledata yang diharapkan:

```text
moodledata/
└── filedir/
```

Simpan backup rollback secara terpisah. Jangan memasukkan backup ke Git.

### 4.2. Website statis

Repository ini tidak menyertakan konten asli website dari `/opt/moodle/web` pada Server sumber lama.

Sebelum menjalankan deployment website, salin konten asli ke:

```text
/opt/moodle/web
```

Jika folder kosong, role website dapat membuat halaman placeholder.

Jangan menganggap migrasi website statis berhasil hanya karena placeholder dapat dibuka.

Website statis `lingga-ramadhan.net` merupakan layanan terpisah dari Moodle `e-learning.lingga-ramadhan.net`. Jangan mengganti document root website statis dengan direktori Moodle.

---

## 5. Instalasi dan Validasi Ansible

Jalankan dari root repository.

### 5.1. Instal collection

```bash
ansible-galaxy collection install -r requirements.yml
```

### 5.2. Periksa inventory

```bash
ansible-inventory -i inventory/hosts.ini --graph
```

```bash
ansible-inventory -i inventory/hosts.ini --list
```

Pastikan:

- `serverA` berada dalam grup `managed`.
- `serverB` berada dalam grup `monitoring_peers`.
- Server B tidak menjadi target deployment.

### 5.3. Tes koneksi

```bash
ansible -i inventory/hosts.ini serverA -m ping
```

### 5.4. Syntax check seluruh playbook

```bash
for p in playbooks/*.yml; do
  echo "===== $p ====="
  ansible-playbook -i inventory/hosts.ini "$p" --syntax-check
done
```

Jika playbook membutuhkan Vault, tambahkan `--ask-vault-pass`.

### 5.5. Periksa target playbook

```bash
for p in playbooks/*.yml; do
  echo "===== $p ====="
  ansible-playbook -i inventory/hosts.ini "$p" --list-hosts
done
```

Untuk playbook deployment dan reset Server A, target harus hanya Server A.

Jangan melanjutkan jika ada playbook yang menargetkan Server B secara tidak sengaja.

---

## 6. Urutan Deployment dan Restore Moodle

Semua perintah dijalankan dari root repository di control node.

### Tahap 1 — Deploy infrastruktur Server A

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --limit serverA --tags infrastructure --ask-vault-pass
```

Tahap ini menjalankan role infrastruktur yang memiliki tag `infrastructure`, yaitu:

- Common.
- Bind9.
- Docker Engine.
- SSL.
- Website.
- Monitoring.

Role Moodle tidak dijalankan pada tahap ini.

### Tahap 2 — Restore source, moodledata, dan SQL

**Perhatian: tahap ini mengubah data dan database.**

Sebelum melanjutkan, pastikan:

- Tiga file backup tersedia.
- Hash backup cocok.
- Password Vault sudah benar.
- Database tujuan belum berisi tabel.
- Direktori tujuan sesuai.
- Backup rollback tersedia.

Jalankan:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/restore-moodle.yml \
  --limit serverA --ask-vault-pass \
  -e confirm_moodle_restore=true \
  -e restore_confirmation_text=RESTORE_MOODLE_TO_10.28.95.38
```

Sesuaikan alamat IP pada teks konfirmasi dengan `server_ip` aktual.

Restore akan mempersiapkan direktori dan konfigurasi Compose, menyalakan MariaDB, menunggu database siap, kemudian memulihkan source Moodle, moodledata, dan SQL.

Proses restore melakukan pemeriksaan kesiapan proses MariaDB dan autentikasi root sebelum memeriksa tabel database tujuan.

Restore menolak direktori tujuan yang tidak kosong dan database yang sudah berisi tabel.

**Jika restore gagal di tengah proses:**

Jangan langsung menjalankan ulang playbook.

Periksa:

- Isi direktori source Moodle.
- Isi direktori moodledata.
- Status dan log MariaDB.
- Kondisi database tujuan.
- Log Ansible pada tahap terakhir.

Lakukan pemulihan manual dari backup rollback atau bersihkan target hanya setelah dipastikan aman.

Restore belum menyediakan rollback otomatis.

### Tahap 3 — Deploy Moodle, PHP-FPM, Nginx, dan config.php

Hanya setelah restore berhasil:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --limit serverA --tags moodle --ask-vault-pass
```

Role Moodle akan menjalankan konfigurasi aplikasi, membangun image PHP `moodle-php:8.3`, membuat konfigurasi Moodle, dan menjalankan layanan terkait.

Arsitektur layanan Moodle menggunakan Docker Compose:

- `moodle-db`: MariaDB.
- `moodle`: PHP-FPM.
- `moodle-nginx`: Nginx reverse proxy.

Nginx melayani website statis dan Moodle melalui konfigurasi virtual host yang terpisah.

### Tahap 4 — Validasi otomatis

```bash
ansible-playbook -i inventory/hosts.ini playbooks/validate.yml \
  --limit serverA --ask-vault-pass
```

Validasi mencakup pemeriksaan konfigurasi BIND9, DNS lokal, konfigurasi Compose, status container, dan endpoint HTTPS Moodle lokal dengan Host dan TLS SNI yang sesuai.

Validasi otomatis bukan pengganti pengujian manual.

### Menjalankan role tertentu

Monitoring saja:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags monitoring --limit serverA --ask-vault-pass
```

DNS saja:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags dns --limit serverA --ask-vault-pass
```

Seluruh infrastruktur tanpa Moodle:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags infrastructure --limit serverA --ask-vault-pass
```

Moodle saja:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags moodle --limit serverA --ask-vault-pass
```

### Pengujian manual sebelum cutover

Sebelum mematikan Server sumber lama:

- Buka `https://e-learning.lingga-ramadhan.net`.
- Pastikan halaman Moodle tampil.
- Uji login menggunakan akun yang sudah ada.
- Periksa beberapa kursus.
- Periksa user dan enrolment.
- Periksa aktivitas dan file yang diunggah.
- Periksa log PHP, Nginx, dan MariaDB.
- Periksa cron Moodle.
- Pastikan website statis menggunakan konten asli.
- Uji akses dari client LAN.
- Uji akses dari jaringan luar jika memang diperlukan.
- Pastikan layanan tidak bergantung lagi pada Server sumber lama.

Jangan menghapus atau mematikan Server sumber lama sebelum pengujian selesai dan pemilik project menyetujui cutover.

---

## 7. Mengubah IP Server A atau Subnet

Konfigurasi jaringan dan konfigurasi Ansible merupakan dua hal yang berbeda.

- Netplan mengatur IP interface, gateway, dan DNS pada sistem operasi.
- Ansible menggunakan variabel `server_ip` dan `server_network_cidr` untuk konfigurasi layanan.

Mengubah variabel Ansible tidak otomatis mengubah IP NIC.

### 7.1. Identifikasi interface jaringan

Jalankan pada Server A:

```bash
ip -br addr
```

Periksa routing:

```bash
ip route
```

Periksa file Netplan:

```bash
ls -l /etc/netplan/
```

Periksa konfigurasi aktif:

```bash
sudo netplan get
```

Contoh output interface:

```text
lo       UNKNOWN  127.0.0.1/8
enp0s3   UP       10.10.10.254/24
enp0s8   UP       10.0.2.15/24
enp0s9   UP       10.28.95.38/24
```

Tentukan interface yang benar-benar digunakan untuk jaringan Server A.

Jangan mengasumsikan nama interface selalu `enp0s9`, karena nama NIC dapat berbeda antar VM atau perangkat.

### 7.2. Backup konfigurasi Netplan

Sebelum mengubah konfigurasi:

```bash
sudo cp -a /etc/netplan /etc/netplan.backup
```

Periksa file YAML yang tersedia:

```bash
ls -l /etc/netplan/
```

Edit file yang memang digunakan:

```bash
sudo nano /etc/netplan/NAMA_FILE.yaml
```

Jangan membuat konfigurasi duplikat tanpa memeriksa konfigurasi yang sudah ada.

### 7.3. Contoh konfigurasi Netplan

Contoh berikut berdasarkan konfigurasi jaringan Server A pada lingkungan pengujian.

Edit file Netplan yang digunakan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Contoh isi konfigurasi:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 10.10.10.254/24

    enp0s8:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false

    enp0s9:
      addresses:
        - 10.28.95.38/24
      nameservers:
        addresses:
          - 10.28.95.38
          - 8.8.8.8
          - 1.1.1.1
        search:
          - "~."
```

Keterangan:

| Interface | Fungsi |
|---|---|
| `enp0s3` | Interface jaringan internal dengan IP statis `10.10.10.254/24`. |
| `enp0s8` | Mendapatkan konfigurasi IP melalui DHCP, tetapi tidak menggunakan DNS dari DHCP. |
| `enp0s9` | Interface jaringan utama Server A dengan IP statis `10.28.95.38/24`. |

**Hal yang perlu diperhatikan:**

- Sesuaikan nama interface berdasarkan hasil `ip -br addr`.
- Ganti `10.28.95.38/24` sesuai IP dan subnet Server A yang digunakan.
- Sesuaikan DNS pertama dengan IP Server A apabila BIND9 berjalan pada Server A.
- DNS publik `8.8.8.8` dan `1.1.1.1` merupakan contoh resolver tambahan.
- Konfigurasi ini tidak menetapkan default gateway secara eksplisit. Pastikan default route berasal dari konfigurasi jaringan yang memang digunakan, misalnya DHCP pada interface lain.

Jangan menyalin konfigurasi secara langsung jika susunan NIC, DHCP, gateway, atau subnet pada lingkungan berbeda.

### 7.4. Validasi dan terapkan Netplan

Periksa sintaks konfigurasi:

```bash
sudo netplan generate
```

Jika tidak ada error, gunakan:

```bash
sudo netplan try
```

`netplan try` lebih aman untuk perubahan jaringan melalui SSH karena konfigurasi dapat dibatalkan jika tidak dikonfirmasi.

Pastikan koneksi tetap berjalan sebelum mengonfirmasi perubahan.

Setelah berhasil, periksa:

```bash
ip -br addr
ip route
```

Uji DNS dan konektivitas:

```bash
ping -c 3 IP_GATEWAY
```

```bash
getent hosts google.com
```

Jika SSH terputus, gunakan console VM atau akses lokal untuk memperbaiki konfigurasi.

### 7.5. Sinkronisasi variabel Ansible

Setelah Netplan benar dan Server A bisa dijangkau melalui IP baru, ubah:

`inventory/host_vars/serverA.yml`

Contoh:

```yaml
---
server_ip: "10.28.95.38"
server_network_cidr: "10.28.95.0/24"

ansible_host: "{{ server_ip }}"
server_role: primary

base_domain: "lingga-ramadhan.net"
moodle_subdomain: "e-learning"
```

Pastikan IP berada dalam rentang host subnet yang benar.

### 7.6. Sesuaikan Bind9

Pada:

`inventory/group_vars/managed/main.yml`

Pastikan konfigurasi jaringan DNS menggunakan variabel subnet Server A, bukan subnet lama yang sudah tidak digunakan.

Contoh:

```yaml
bind_network_cidr: "{{ server_network_cidr }}"
```

Periksa pula konfigurasi reverse DNS, termasuk:

```yaml
bind_reverse_zone: "..."
bind_reverse_file: "..."
```

Nama zona reverse DNS harus sesuai dengan subnet dan delegasi reverse DNS yang benar-benar dimiliki.

**Reverse DNS tidak otomatis berubah hanya karena prefix subnet berubah.**

Jika hanya memperoleh delegasi reverse DNS untuk subnet `/28`, konfigurasi harus disesuaikan dengan delegasi classless reverse DNS menggunakan mekanisme RFC 2317. Jangan menggunakan nama zona contoh tanpa memastikan delegasi dari pengelola zona reverse DNS induk.

### 7.7. Deploy ulang infrastruktur

Setelah Netplan, inventory, dan variabel DNS sudah benar:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --limit serverA --tags infrastructure --ask-vault-pass
```

Kemudian validasi:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/validate.yml \
  --limit serverA --ask-vault-pass
```

**Urutan penting:** Netplan terlebih dahulu, variabel Ansible berikutnya, lalu deployment infrastruktur.

### 7.8. Memperbarui sertifikat SSL

Jika domain atau identitas sertifikat berubah, jalankan:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ssl-renew.yml \
  --limit serverA --ask-vault-pass
```

Sertifikat yang digunakan adalah self-signed. Browser/client tidak otomatis mempercayainya.

Periksa kembali SAN sertifikat dan pastikan layanan HTTPS berjalan setelah pembaruan.

---

## 8. Monitoring Server B

Prometheus pada Server A mengambil data monitoring dari endpoint yang disediakan Server B.

Target peer diambil dari:

```yaml
peer_host: serverB
peer_ip: "{{ hostvars[peer_host].server_ip }}"
```

Endpoint yang dapat dipantau bergantung pada feature flag dan layanan yang tersedia di Server B.

| Service | Port default | Keterangan |
|---|---:|---|
| Node Exporter | 9100 | CPU, memory, disk, dan network |
| MariaDB Exporter | 9104 | Metrik MariaDB jika diaktifkan |
| Docker metrics | Sesuai konfigurasi Docker daemon | Jika endpoint Docker metrics tersedia |
| Moodle HTTP probe | URL Moodle peer | Melalui Blackbox Exporter di Server A |

### Ketentuan monitoring peer

- Server A hanya membaca endpoint monitoring Server B.
- Server B tidak menjadi target deployment Ansible.
- Pengelola Server B harus mengaktifkan exporter yang diperlukan.
- Firewall Server B harus mengizinkan koneksi dari IP Server A.
- Jangan membuka port database `3306` ke internet.
- Jika Server B mati, target monitoring peer akan berstatus down. Kondisi tersebut tidak berarti Moodle Server A ikut mati.
- Feature flag yang bernilai `false` tidak akan mengaktifkan monitoring endpoint tersebut.

### Dashboard

Dashboard Server A dan Server B menampilkan metrik yang disesuaikan dengan target masing-masing.

Dashboard Server B mencakup antara lain:

- Status Node Exporter peer.
- Status HTTP Moodle peer jika probe diaktifkan.
- Status MariaDB peer jika exporter diaktifkan.
- Active Alerts peer.
- CPU.
- Memory.
- Root disk.
- Network RX/TX.
- Response time Moodle.
- Active connections MariaDB jika metrik tersedia.

Label alert yang digunakan:

- Alert Server A menggunakan label `host: serverA`.
- Alert peer menggunakan label `peer: "{{ peer_host }}"`.

Query Active Alerts pada dashboard harus mengikuti label tersebut agar jumlah alert yang ditampilkan sesuai dengan host yang dipantau.

---

## 9. Reset Monitoring

Reset monitoring hanya menyentuh komponen monitoring Server A.

Periksa isi:

```text
playbooks/reset-monitoring.yml
```

### 9.1. Reset konfigurasi monitoring

```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset-monitoring.yml \
  --limit serverA --ask-vault-pass
```

Reset mode default mempertahankan data historis Prometheus/Alertmanager dan database Grafana sesuai implementasi playbook.

Setelah reset, deploy kembali monitoring:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags monitoring --limit serverA --ask-vault-pass
```

### 9.2. Reset penuh monitoring

Reset penuh dapat menghapus data historis Prometheus/Alertmanager dan state Grafana.

Jalankan hanya jika memang ingin menghapus data tersebut:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset-monitoring.yml \
  --limit serverA --ask-vault-pass \
  -e reset_mode=full \
  -e confirm_full_reset=true
```

Jangan menjalankan reset penuh jika data historis masih diperlukan.

---

## 10. Reset Deployment Moodle Server A

Playbook:

```text
playbooks/reset-serverA.yml
```

Digunakan untuk menghapus deployment Moodle Server A agar proses restore dan deployment dapat diulang dari kondisi awal.

Reset ini **bukan** reset seluruh infrastruktur Server A.

### 10.1. Komponen yang dihapus

Reset deployment Moodle mencakup komponen berikut sesuai implementasi playbook:

- Container Moodle melalui Docker Compose.
- Docker volumes yang digunakan deployment Moodle, termasuk volume database.
- Direktori source Moodle aktif.
- Direktori `moodledata` aktif.
- File runtime `.env`.
- File runtime `docker-compose.yml`.
- Image PHP legacy `moodle-servera-php:8.3` jika masih tersedia.

**Perhatian:** Database Moodle aktif beserta data pada volume terkait akan dihapus.

Image PHP legacy dihapus untuk membersihkan sisa deployment versi sebelumnya. Image tersebut bukan image yang digunakan oleh konfigurasi terbaru.

Pastikan backup tersedia dan sudah diverifikasi.

### 10.2. Komponen yang dipertahankan

Reset dirancang untuk mempertahankan komponen di luar deployment Moodle, antara lain:

- Backup Moodle:
  - `moodle.sql.gz`
  - `moodle-source.tar.gz`
  - `moodledata.tar.gz`
- Repository Ansible.
- Docker Engine.
- BIND9 dan konfigurasi DNS.
- Grafana.
- Prometheus.
- Alertmanager.
- Konfigurasi monitoring.
- Website statis `lingga-ramadhan.net`.
- Konfigurasi Nginx dan SSL di luar file runtime Compose yang dihapus.

Periksa implementasi playbook sebelum mengasumsikan file atau direktori tertentu dipertahankan. Reset ini tidak dimaksudkan untuk menghapus seluruh Docker image, Docker Engine, atau data monitoring.

### 10.3. Pemeriksaan sebelum reset

Periksa sintaks:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset-serverA.yml \
  --syntax-check --ask-vault-pass
```

Pastikan:

- Target hanya `serverA`.
- `server_role` adalah `primary`.
- Backup tersedia.
- SHA-256 backup cocok.
- Tidak ada data Moodle aktif yang masih diperlukan.
- IP Server A sesuai dengan teks konfirmasi.

### 10.4. Menjalankan reset

Contoh:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset-serverA.yml \
  --limit serverA --ask-vault-pass \
  -e confirm_serverA_reset=true \
  -e reset_confirmation_text=RESET_SERVERA_MOODLE_TO_10.28.95.38
```

Ganti IP pada teks konfirmasi dengan nilai `server_ip` aktual.

Teks konfirmasi harus sama persis dengan format yang diperiksa oleh playbook:

```text
RESET_SERVERA_MOODLE_TO_<IP_SERVER_A>
```

Playbook memiliki pemeriksaan keamanan untuk mencegah reset dijalankan pada target yang salah.

### 10.5. Setelah reset

Setelah reset berhasil, ulangi proses:

1. Restore source, moodledata, dan SQL.
2. Deploy role Moodle.
3. Jalankan validasi otomatis.
4. Lakukan pengujian manual.

**Jangan menjalankan reset ini hanya untuk mengganti IP, subnet, SSL, atau konfigurasi monitoring.** Reset menghapus data deployment Moodle aktif.

---

## 11. Git dan File Rahasia

### Boleh masuk Git

- Playbook.
- Role.
- Template `.j2`.
- Inventory non-rahasia.
- `vault.yml.example`.
- README.
- Dokumentasi.

### Jangan commit

- `inventory/group_vars/managed/vault.yml` asli.
- Password dan secret.
- `.env`.
- `config.php`.
- `.my.cnf`.
- Private key dan certificate yang bersifat rahasia.
- SQL dump.
- Source archive.
- `moodledata`.
- Backup rollback.

Sebelum commit:

```bash
git status --short
git diff --check
git diff --cached --stat
```

Periksa isi staged changes:

```bash
git diff --cached
```

Pastikan tidak ada secret atau file backup yang ikut masuk Git.

### Contoh commit dan push

Periksa branch:

```bash
git branch --show-current
```

Stage hanya file yang memang akan di-commit:

```bash
git add README.md
```

Untuk perubahan lainnya, tambahkan file yang sudah diperiksa secara eksplisit.

Periksa staged changes:

```bash
git diff --cached --check
git diff --cached --stat
git diff --cached
```

Jika sudah benar:

```bash
git commit -m "docs: update deployment and reset documentation"
git push origin main
```

Sesuaikan nama branch jika tidak menggunakan `main`.

---

## 12. Troubleshooting dan Batasan

### SSH gagal

Periksa:

- IP Server A.
- Gateway dan routing.
- Firewall.
- Username.
- SSH key.
- Hak sudo.

Pastikan SSH berfungsi sebelum menjalankan Ansible.

### Backup ditolak

Periksa:

- Nama file.
- SHA-256.
- Versi dan build Moodle.
- Struktur arsip source.
- Struktur arsip moodledata.

Jangan menonaktifkan pemeriksaan hash untuk melewati validasi.

### Restore gagal karena direktori tidak kosong

Jangan langsung menghapus direktori.

Periksa isi direktori dan pastikan data boleh diganti sebelum melanjutkan.

### Database tidak bisa login setelah Vault berubah

Password pada MariaDB volume yang sudah pernah diinisialisasi tidak otomatis berubah hanya karena nilai Vault diperbarui.

Jangan menghapus volume sebagai jalan pintas. Periksa akun database, volume, dan prosedur rotasi password.

### Moodle tidak tampil

Periksa:

- Status container.
- Log PHP-FPM.
- Log Nginx.
- Log MariaDB.
- `config.php`.
- DNS.
- Host dan SNI Nginx.
- Permission source dan moodledata.
- Koneksi database.

### Website masih placeholder

Pastikan konten website asli sudah disalin ke:

```text
/opt/moodle/web
```

pada Server A.

### Server B tidak menampilkan data monitoring

Periksa:

- Server B aktif.
- Exporter berjalan.
- Feature flag monitoring sesuai kebutuhan.
- Target IP benar.
- Port exporter benar.
- Firewall mengizinkan IP Server A.
- Konfigurasi Prometheus sudah diperbarui.

### Grafana tidak dapat mengambil data Prometheus

Periksa:

- Prometheus berjalan.
- Prometheus listen pada alamat yang sesuai.
- Port Prometheus dapat diakses.
- URL datasource Grafana benar.
- Target scrape Prometheus berstatus sesuai kondisi layanan.

### Batasan pengujian

Syntax check dan validasi Ansible tidak menjamin seluruh layanan berhasil secara end-to-end.

Lakukan pengujian bertahap:

1. Syntax check.
2. Inventory dan daftar host.
3. Koneksi SSH.
4. Deployment infrastruktur.
5. Restore Moodle.
6. Deployment Moodle.
7. Validasi otomatis.
8. Pengujian manual.

Simpan output setiap tahap untuk dokumentasi dan audit project.

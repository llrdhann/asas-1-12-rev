# Moodle migration and monitoring automation — Server A

Repository ini menyiapkan **Server A** sebagai server mandiri untuk website, Moodle, DNS, dan monitoring. Moodle dipindahkan dari backup Server `.211`. Server B hanya dipantau sebagai peer; tidak dikelola oleh Ansible di repository ini.

> **Status:** dokumentasi ini menjelaskan alur yang ada di playbook. Restore mengubah direktori aplikasi Moodle dan mengimpor SQL ke database tujuan yang harus kosong. Lakukan hanya pada Server A baru/yang sudah disiapkan untuk migrasi, setelah backup rollback tersedia. Restore belum memiliki rollback otomatis. Jangan menganggap migrasi selesai sebelum semua validasi manual berhasil.

## 1. Topologi dan batas pengelolaan

| Host | IP contoh | Peran |
|---|---:|---|
| Server A | `10.194.156.38` | Satu-satunya target deployment Ansible |
| Server B | `172.20.3.242` | Peer monitoring saja; milik pihak lain |
| Server sumber lama | `172.20.3.211` | Sumber backup dan rollback; tidak diubah playbook |

- Semua playbook deployment menggunakan `hosts: serverA`.
- `serverB` hanya menjadi sumber target scrape Prometheus. Jangan menjalankan deployment dengan `--limit all` atau mengubah pola host menjadi `all`.
- Bind9 berjalan native pada Ubuntu Server A.
- Moodle PHP-FPM, MariaDB 10.11, dan Nginx berjalan menggunakan Docker Compose.
- Website statis: `lingga-ramadhan.net` dan `www.lingga-ramadhan.net`.
- Moodle: `e-learning.lingga-ramadhan.net`.
- Ansible **tidak** mengatur Netplan/NIC, IP interface, gateway, atau resolver OS Server A. IP dan SSH harus sudah berfungsi sebelum playbook dijalankan.

## 2. Prasyarat

### Di komputer control node

- Linux/WSL dengan Python 3, Ansible, OpenSSH client, `git`, dan `rsync`.
- Akses SSH ke Server A menggunakan user yang dapat menjalankan `sudo`.
- Repository sudah di-clone dan perintah dijalankan dari direktori root repository.

Contoh instalasi pada Ubuntu/Debian control node:

```bash
sudo apt update
sudo apt install -y python3 python3-pip ansible openssh-client git rsync
```

Jika distro menyediakan versi Ansible terlalu lama, gunakan metode instalasi Ansible yang sesuai distro dan pastikan versi tersebut mendukung modul `community.docker` yang dipakai repository.

### Di Server A

- Ubuntu Server yang kompatibel dengan role/package repository ini.
- IP statis atau DHCP reservation sudah diatur manual.
- SSH dari control node sudah bisa masuk.
- User Ansible memiliki hak sudo.
- Port yang diperlukan sudah diizinkan sesuai jaringan: SSH dari control node; DNS 53 TCP/UDP; HTTP 80 dan HTTPS 443 dari client; monitoring sesuai kebutuhan.
- Belum ada instalasi Moodle/DB yang harus dipertahankan pada path dan volume yang dikelola playbook.

Tes koneksi sebelum deployment:

```bash
ssh ling@10.194.156.38
```

Ganti user dan IP sesuai Server A. Setelah SSH berhasil, lanjutkan dari control node.

## 3. File dan variabel yang harus disesuaikan

Ubah file berikut **sebelum menjalankan playbook**. Path di bawah relatif dari root repository.

### A. `inventory/hosts.ini`

Ubah user SSH Ansible jika bukan `ling`:

```ini
[managed]
serverA ansible_user=ling

[monitoring_peers]
serverB ansible_host=172.20.3.242 server_ip=172.20.3.242
```

`serverB` dipakai untuk data monitoring saja. Jangan masukkan Server B ke grup `managed`.

### B. `inventory/host_vars/serverA.yml`

Ubah alamat Server A. `ansible_host` mengikuti `server_ip` secara otomatis:

```yaml
server_ip: "10.194.156.38"
ansible_host: "{{ server_ip }}"
server_role: primary
```

`server_ip` harus sama dengan IP yang benar-benar sudah terpasang dan bisa diakses melalui SSH. Jangan mengandalkan playbook untuk mengganti IP NIC.

### C. `inventory/group_vars/managed/main.yml`

Periksa nilai berikut dan ubah hanya jika kondisi jaringan/layanan berbeda:

| Variabel | Kapan perlu diubah |
|---|---|
| `bind_network_cidr` | Subnet LAN tempat DNS boleh melayani query/recursive |
| `bind_reverse_zone` | Jika subnet/reverse DNS berubah |
| `bind_reverse_file` | Jika nama/path file reverse zone disesuaikan |
| `bind_forwarders` | Jika DNS host perlu meneruskan query ke resolver tertentu; default `[]` |
| `peer_moodle_url` | Isi URL Moodle Server B yang benar-benar dapat dijangkau dari Server A; kosong berarti probe HTTP Moodle peer tidak dibuat |
| `peer_moodle_monitor_enabled` | Set `false` jika tidak ingin probe Moodle Server B |
| `peer_node_exporter_target` / `peer_mariadb_exporter_target` | Ubah port/target jika exporter Server B memang memakai endpoint lain |
| `moodle_wwwroot` | Ubah hanya jika hostname Moodle memang berubah; harus konsisten dengan DNS dan Nginx |
| `ssl_domains` dan `website_server_names` | Ubah jika domain aplikasi berubah |
| `moodle_version_expected` / `moodle_build_expected` | Harus cocok dengan source Moodle yang akan dipulihkan |
| `moodle_*_sha256` | Harus cocok dengan tiga arsip backup yang akan dipakai; hitung ulang jika arsip berubah |

Jika Server A pindah ke subnet lain, jangan hanya mengubah `server_ip`. Sesuaikan `bind_network_cidr`, `bind_reverse_zone`, dan `bind_reverse_file` juga. Periksa pula konfigurasi router/firewall dan DNS eksternal.

### D. `inventory/host_vars/serverB.yml`

Jika IP Server B berubah, ubah `server_ip` di file ini. Pastikan alamat tersebut benar-benar milik peer dan endpoint exporter tersedia. Repository ini tidak menginstal atau mengubah exporter di Server B.

Pihak pengelola Server B perlu mengizinkan koneksi dari IP Server A ke endpoint exporter yang disepakati (default Node Exporter `9100` dan MySQL/MariaDB Exporter `9104`). Jangan membuka port database `3306` ke publik.

### E. `inventory/group_vars/managed/vault.yml`

Buat dari contoh, isi password baru, lalu enkripsi. Jangan commit file hasilnya:

```bash
cp inventory/group_vars/managed/vault.yml.example inventory/group_vars/managed/vault.yml
openssl rand -hex 32
nano inventory/group_vars/managed/vault.yml
ansible-vault encrypt inventory/group_vars/managed/vault.yml
```

Isi empat variabel ini dengan nilai random yang berbeda:

```yaml
vault_grafana_admin_password: "GANTI_DENGAN_RANDOM_HEX"
vault_moodle_db_root_password: "GANTI_DENGAN_RANDOM_HEX"
vault_moodle_db_password: "GANTI_DENGAN_RANDOM_HEX"
vault_mysql_exporter_password: "GANTI_DENGAN_RANDOM_HEX"
```

Buat nilai random terpisah untuk setiap password, jangan gunakan contoh literal di atas. Untuk mengedit setelah dienkripsi:

```bash
ansible-vault edit inventory/group_vars/managed/vault.yml
```

Jalankan playbook dengan `--ask-vault-pass` agar Ansible meminta password Vault. Jangan menaruh password Vault atau secret lain langsung di command line, README, inventory non-encrypted, atau Git.

### F. Backup Moodle di Server A

Restore mengharapkan tiga file ini sudah ada di Server A:

```text
/opt/moodle/backup/moodle.sql.gz
/opt/moodle/backup/moodle-source.tar.gz
/opt/moodle/backup/moodledata.tar.gz
```

Nama file dan SHA-256 harus cocok dengan nilai di `inventory/group_vars/managed/main.yml`. Pada backup yang diaudit untuk project ini, nilainya adalah:

```yaml
moodle_source_sha256: "5fc407337058c89721f3ed96a8badd59ab3a8e234e3eb2db4690e5389716073a"
moodledata_sha256: "4292cc42c7d86bd6ce732c23cad0f4f267e915ead45649afb29b3c15c325fa17"
moodle_sql_sha256: "878de0beec19a42dac9e6b343d6561ccebef3e888eef4f403feae2a88432188c"
```

Verifikasi di Server A:

```bash
sudo sha256sum \
  /opt/moodle/backup/moodle-source.tar.gz \
  /opt/moodle/backup/moodledata.tar.gz \
  /opt/moodle/backup/moodle.sql.gz
```

Jika menggunakan backup berbeda, jangan menyalin hash di atas begitu saja. Hitung SHA-256 arsip baru, perbarui variabel yang sesuai, lalu pastikan `moodle_version_expected` dan `moodle_build_expected` cocok dengan `moodle/public/version.php` di arsip tersebut. Arsip sumber harus memiliki struktur `moodle/public/version.php`; arsip data harus memiliki `moodledata/filedir`.

Simpan backup rollback Server A secara terpisah. Jangan masukkan backup SQL, source archive, moodledata, atau file runtime ke Git.

### G. Website statis

Repository ini **tidak** menyertakan konten website asli dari `/opt/moodle/web` di Server `.211`. Sebelum menjalankan `site.yml`, salin isi website asli ke `/opt/moodle/web` pada Server A. Jika folder kosong, role website akan membuat halaman placeholder `index.html`.

Jangan menganggap website statis sudah dimigrasikan hanya karena placeholder berhasil tampil.

## 4. Instalasi awal di control node

Dari root repository:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory -i inventory/hosts.ini --graph
ansible-inventory -i inventory/hosts.ini --list
```

Periksa hasil inventory: `serverA` harus berada di grup `managed`; `serverB` hanya di `monitoring_peers`.

Tes koneksi:

```bash
ansible -i inventory/hosts.ini serverA -m ping --ask-vault-pass
```

Jika koneksi SSH sudah memakai key dan tidak membutuhkan Vault untuk ping, perintah ping juga dapat dijalankan tanpa `--ask-vault-pass`.

Periksa syntax dan daftar target seluruh playbook:

```bash
for p in playbooks/*.yml; do
  echo "===== $p ====="
  ansible-playbook -i inventory/hosts.ini "$p" --syntax-check
  ansible-playbook -i inventory/hosts.ini "$p" --list-hosts
done
```

Untuk playbook deployment, daftar host harus hanya `serverA`. `reset-monitoring.yml` juga hanya menargetkan Server A. Jangan lanjut jika ada playbook yang menargetkan Server B.

## 5. Urutan deployment dan restore Moodle

Jalankan per tahap. Pastikan tahap sebelumnya sukses sebelum lanjut. Semua perintah berikut dijalankan dari root repository di control node.

### Tahap 1 — Deploy infrastruktur Server A

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --limit serverA --ask-vault-pass --skip-tags moodle
```

Tahap ini menjalankan role common, Bind9, Docker Engine, SSL, website, dan monitoring tanpa menjalankan role Moodle.

### Tahap 2 — Restore source, moodledata, dan SQL

**Perhatian: ini tahap yang mengubah data.**

Playbook restore akan mempersiapkan direktori dan konfigurasi Compose, menyalakan MariaDB saja, menunggu database siap, kemudian memulihkan source Moodle, moodledata, dan SQL.

Pastikan tiga backup, hash, Vault, dan backup rollback sudah benar. Kemudian jalankan:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/restore-moodle.yml \
  --limit serverA --ask-vault-pass \
  -e confirm_moodle_restore=true \
  -e restore_confirmation_text=RESTORE_MOODLE_TO_10.194.156.38
```

Ganti IP pada teks konfirmasi sesuai `server_ip` aktual. Konfirmasi harus sama persis.

Restore tetap menolak direktori tujuan yang tidak kosong dan database yang sudah berisi tabel. Restore belum menyediakan rollback otomatis untuk kegagalan di tengah proses. Jika playbook berhenti setelah mulai menyalin source/data atau saat impor SQL, jangan langsung menjalankan ulang. Periksa direktori live, database, dan log MariaDB; lakukan pemulihan manual dari backup rollback atau bersihkan target hanya setelah dipastikan aman.

### Tahap 3 — Deploy Moodle, PHP-FPM, Nginx, dan config.php

Hanya setelah restore berhasil:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --limit serverA --ask-vault-pass --tags moodle
```

Role Moodle membuat `config.php` dari template dan Vault, membangun image PHP, lalu menjalankan layanan Moodle.

### Tahap 4 — Validasi otomatis

```bash
ansible-playbook -i inventory/hosts.ini playbooks/validate.yml \
  --limit serverA --ask-vault-pass
```

Playbook validasi memeriksa konfigurasi/zone Bind9, DNS lokal, konfigurasi Compose, status container, dan endpoint HTTPS Moodle lokal. Lolos validasi ini belum menggantikan tes login dan pemeriksaan konten Moodle.

### Menjalankan role tertentu dari `site.yml`

`site.yml` menggunakan tags agar role tertentu bisa dijalankan tanpa harus menjalankan semua role.

Contoh menjalankan monitoring saja:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags monitoring --limit serverA --ask-vault-pass
```

Menjalankan DNS saja:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags dns --limit serverA --ask-vault-pass
```

Menjalankan semua role infrastruktur tanpa Moodle:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --skip-tags moodle --limit serverA --ask-vault-pass
```

Menjalankan Moodle saja:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags moodle --limit serverA --ask-vault-pass
```

### Validasi manual sebelum mematikan Server `.211`

- Buka `https://e-learning.lingga-ramadhan.net` dan pastikan sertifikat self-signed dipahami/ditangani pada client.
- Login dengan akun Moodle yang sudah ada.
- Periksa beberapa kursus, user, enrolment, aktivitas, dan file yang diunggah.
- Periksa log container PHP, Nginx, dan MariaDB serta cron Moodle.
- Pastikan website statis menggunakan konten asli, bukan placeholder.
- Uji akses dari client LAN dan, jika diperlukan, dari jaringan luar melalui DNS/NAT yang benar.
- Uji bahwa Moodle tetap berfungsi ketika Server `.211` dimatikan atau tidak dapat dijangkau. Lakukan hanya setelah semua validasi dan backup rollback tersedia.

Jangan menghapus atau mematikan Server `.211` sebelum pengujian ini selesai dan pemilik project menyetujui cutover.

## 6. Mengubah IP Server A atau subnet

Konfigurasi jaringan Server A dipisahkan menjadi dua variabel:

- `server_ip`: alamat IP yang digunakan Server A.
- `server_network_cidr`: alamat network beserta prefix subnet yang digunakan Server A.

Pemisahan ini diperlukan karena alamat IP server dan subnet tidak selalu menggunakan prefix `/24`.

### 6.1. Mengatur IP dan subnet

Atur kedua variabel di `inventory/host_vars/serverA.yml`.

**Contoh konfigurasi subnet `/24`:**

```yaml
server_ip: "10.28.95.38"
server_network_cidr: "10.28.95.0/24"
ansible_host: "{{ server_ip }}"
server_role: primary
```

**Contoh konfigurasi subnet `/28`:**

```yaml
server_ip: "10.28.95.38"
server_network_cidr: "10.28.95.32/28"
ansible_host: "{{ server_ip }}"
server_role: primary
```

Pastikan `server_ip` termasuk dalam rentang alamat host pada `server_network_cidr`.

Jika menggunakan subnet `/28`, contoh `10.28.95.32/28` memiliki rentang:

- Network: `10.28.95.32`
- Host: `10.28.95.33`–`10.28.95.46`
- Broadcast: `10.28.95.47`

### 6.2. Konfigurasi BIND9

Pada `inventory/group_vars/managed/main.yml`, gunakan variabel subnet dari `serverA.yml`:

```yaml
bind_network_cidr: "{{ server_network_cidr }}"
```

Variabel `bind_network_cidr` digunakan untuk menentukan jaringan yang diizinkan mengakses layanan DNS sesuai konfigurasi BIND9.

**Reverse DNS merupakan konfigurasi yang berbeda.** Variabel berikut menentukan zona reverse DNS dan file zona yang digunakan:

```yaml
bind_reverse_zone: "95.28.10.in-addr.arpa"
bind_reverse_file: "/etc/bind/db.10.28.95"
```

Nilai di atas merupakan contoh zona reverse DNS untuk seluruh jaringan `10.28.95.0/24`. Jangan otomatis menggantinya hanya karena alamat server menggunakan subnet `/28`.

Jika Server A hanya mendapat delegasi reverse DNS untuk `10.28.95.32/28`, diperlukan konfigurasi *classless reverse DNS delegation* menggunakan mekanisme RFC 2317. Salah satu contoh nama zona untuk rentang tersebut adalah:

```text
32-47.95.28.10.in-addr.arpa
```

Nama zona tersebut bukan konfigurasi yang bisa langsung digunakan tanpa delegasi dari pengelola zona reverse DNS induk.

Karena itu:

- Gunakan zona `/24` apabila memang mendapat kewenangan mengelola seluruh `/24`.
- Jika hanya mendapat delegasi `/28`, sesuaikan konfigurasi dengan delegasi RFC 2317 yang diberikan oleh pengelola DNS induk.
- Jangan mengubah reverse zone hanya berdasarkan perubahan prefix subnet tanpa memeriksa delegasi DNS yang tersedia.

### 6.3. Langkah ketika jaringan berubah

1. Sesuaikan konfigurasi Netplan dengan IP, prefix subnet, gateway, dan DNS jaringan yang baru.
2. Perbarui `server_ip` pada `inventory/host_vars/serverA.yml`.
3. Perbarui `server_network_cidr` jika prefix atau network berubah.
4. Periksa `bind_reverse_zone` dan `bind_reverse_file`. Pastikan keduanya sesuai dengan delegasi reverse DNS yang tersedia.
5. Pastikan konfigurasi DNS, gateway, dan koneksi SSH sudah sesuai dengan jaringan baru.
6. Jalankan kembali konfigurasi infrastruktur:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --limit serverA --tags infrastructure --ask-vault-pass
```

7. Jalankan validasi:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/validate.yml \
  --limit serverA --ask-vault-pass
```

**Catatan:** Mengubah variabel Ansible tidak otomatis mengubah konfigurasi IP pada Netplan. Keduanya harus disesuaikan. Pastikan juga koneksi SSH tetap dapat dijangkau setelah perubahan jaringan.

### 6.4. Memperbarui sertifikat SSL

Jika IP atau domain Server A berubah dan sertifikat SSL perlu diperbarui, gunakan playbook khusus berikut:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ssl-renew.yml \
  --limit serverA --ask-vault-pass
```

Playbook ini digunakan untuk menjalankan proses pembaruan sertifikat SSL tanpa harus menjalankan seluruh role infrastruktur.

Sertifikat yang dibuat adalah *self-signed*. Browser/client tidak otomatis mempercayainya sehingga sertifikat perlu dipercaya secara manual pada client yang digunakan.

**Catatan:** Pembaruan sertifikat diperlukan apabila perubahan IP atau domain membuat identitas pada sertifikat (SAN) tidak lagi sesuai. Periksa hasil sertifikat dan pastikan layanan HTTPS tetap berjalan setelah pembaruan.


## 7. Monitoring Server B

Prometheus pada Server A membaca endpoint yang sudah disediakan oleh pengelola Server B:

- Node Exporter: default `172.20.3.242:9100`.
- MySQL/MariaDB Exporter: default `172.20.3.242:9104`.
- Blackbox HTTP probe Moodle peer: aktif hanya jika `peer_moodle_url` diisi dengan URL yang dapat dijangkau Server A.

Minta persetujuan pemilik Server B dan batasi firewall agar hanya IP Server A yang dapat mengakses endpoint exporter. Jangan membuka port database `3306` ke internet. Tidak ada playbook di repository ini yang menginstal, mengubah, atau merestart service di Server B.

## 8. Reset monitoring

Reset hanya menyentuh monitoring Server A. Periksa isi `playbooks/reset-monitoring.yml` sebelum menjalankannya.

Reset konfigurasi default (mempertahankan data historis Prometheus/Alertmanager dan database Grafana):

```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset-monitoring.yml \
  --limit serverA --ask-vault-pass
```

Setelah reset selesai, deploy kembali role monitoring:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml \
  --tags monitoring --limit serverA --ask-vault-pass
```

Reset penuh menghapus data historis Prometheus/Alertmanager dan state Grafana. Jangan jalankan kecuali memang menghendaki penghapusan tersebut:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/reset-monitoring.yml \
  --limit serverA --ask-vault-pass \
  -e reset_mode=full -e confirm_full_reset=true
```

## 9. Git dan file rahasia

Boleh masuk Git:

- Playbook, role, template `.j2`, inventory non-rahasia, `vault.yml.example`, README, dan dokumentasi.

Jangan commit:

- `inventory/group_vars/managed/vault.yml` asli, password, `.env`, `config.php`, `.my.cnf`, private key/certificate, SQL dump, source archive, `moodledata`, atau backup rollback.

Sebelum commit:

```bash
git status --short
git diff --check
git diff --cached --stat
```

Pastikan tidak ada secret atau file backup dalam staged changes.

## 10. Troubleshooting dan batasan penting

- **SSH gagal:** cek IP, route, firewall, username, key, dan sudo di luar Ansible terlebih dahulu.
- **Backup ditolak:** cocokkan nama file, SHA-256, versi/build Moodle, dan struktur arsip. Jangan mematikan pemeriksaan hash hanya agar playbook lanjut.
- **Restore berhenti karena direktori tidak kosong:** jangan hapus direktori secara otomatis. Periksa isinya dan pastikan boleh diganti.
- **Database tidak bisa login setelah perubahan Vault:** password pada MariaDB volume yang sudah pernah diinisialisasi tidak otomatis berubah hanya karena Vault diganti. Jangan menghapus volume sebagai jalan pintas; verifikasi akun/volume dan lakukan prosedur rotasi yang benar.
- **Moodle tidak tampil:** periksa status dan log container, `config.php`, DNS, Nginx Host/SNI, permission source/data, dan koneksi database.
- **Website masih placeholder:** salin konten website asli ke `/opt/moodle/web` pada Server A.
- **Server B tidak ada data:** pastikan exporter aktif, URL/port benar, dan firewall mengizinkan IP Server A.

Tidak ada pengujian live deployment yang dijamin oleh pemeriksaan statis repository. Jalankan `--syntax-check`, `--list-hosts`, tes koneksi, lalu deployment bertahap. Simpan output setiap tahap untuk audit.

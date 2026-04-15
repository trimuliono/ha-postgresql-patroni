# Dokumentasi HA PostgreSQL Cluster dengan Patroni + etcd + HAProxy + Keepalived
## Rocky Linux 9 — Setup Lengkap

**Versi:** PostgreSQL 16, Patroni, etcd  
**OS:** Rocky Linux 9  

---

## Topologi Cluster

```
                        ┌─────────────────────────┐
                        │   Floating IP (VIP)      │
                        │   10.100.13.248          │
                        └────────────┬────────────┘
                                     │ Keepalived VRRP
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
   ┌──────────┴──────────┐ ┌────────┴────────┐ ┌──────────┴──────────┐
   │    confluent-1       │ │   confluent-2   │ │    confluent-3       │
   │   10.100.13.241      │ │  10.100.13.242  │ │   10.100.13.243      │
   │                      │ │                 │ │                      │
   │  PostgreSQL 16       │ │  PostgreSQL 16  │ │  PostgreSQL 16       │
   │  Patroni             │ │  Patroni        │ │  Patroni             │
   │  etcd                │ │  etcd           │ │  etcd                │
   │  HAProxy             │ │  HAProxy        │ │  HAProxy             │
   │  Keepalived (MASTER) │ │  Keepalived     │ │  Keepalived          │
   └──────────────────────┘ └─────────────────┘ └──────────────────────┘
```

| Node | Hostname | IP | Role |
|------|----------|----|------|
| Node 1 | confluent-1 | 10.100.13.241 | Primary (awal) |
| Node 2 | confluent-2 | 10.100.13.242 | Replica |
| Node 3 | confluent-3 | 10.100.13.243 | Replica |
| VIP | - | 10.100.13.248 | Floating IP (Keepalived) |

**Port yang digunakan:**

| Port | Service | Keterangan |
|------|---------|-----------|
| 5432 | PostgreSQL | Database |
| 8008 | Patroni REST API | Health check |
| 2379 | etcd client | DCS client |
| 2380 | etcd peer | DCS peer communication |
| 5000 | HAProxy | Primary (read/write) |
| 5001 | HAProxy | Replica (read only) |
| 7000 | HAProxy Stats | Monitoring page |

---

## Urutan Instalasi

```
1. Pre-req (semua node)
2. PostgreSQL 16 (semua node)
3. Patroni (semua node)
4. etcd (semua node)
5. HAProxy (semua node)
6. Keepalived (semua node)
7. Start service dengan urutan: etcd → Patroni → HAProxy → Keepalived
8. Testing
```

---

## Step 1 — Persiapan Semua Node

Lakukan di **semua node** (confluent-1, confluent-2, confluent-3).

### 1.1 Update sistem dan install pre-req

```bash
sudo dnf update -y
sudo dnf install -y epel-release
sudo dnf install -y python3 python3-pip curl net-tools vim wget tar gnupg2
```

### 1.2 Set timezone

```bash
sudo timedatectl set-timezone Asia/Jakarta
timedatectl status
```

### 1.3 Set hostname (sesuaikan per node)

```bash
# Di confluent-1
sudo hostnamectl set-hostname confluent-1

# Di confluent-2
sudo hostnamectl set-hostname confluent-2

# Di confluent-3
sudo hostnamectl set-hostname confluent-3
```

### 1.4 Set /etc/hosts di semua node

```bash
sudo nano /etc/hosts
```

Tambahkan baris berikut:

```
10.100.13.241   confluent-1
10.100.13.242   confluent-2
10.100.13.243   confluent-3
```

Verifikasi:

```bash
cat /etc/hosts
ping confluent-1 -c 2
ping confluent-2 -c 2
ping confluent-3 -c 2
```

### 1.5 Konfigurasi Firewall

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp   # PostgreSQL
sudo firewall-cmd --permanent --add-port=8008/tcp   # Patroni REST API
sudo firewall-cmd --permanent --add-port=2379/tcp   # etcd client
sudo firewall-cmd --permanent --add-port=2380/tcp   # etcd peer
sudo firewall-cmd --permanent --add-port=5000/tcp   # HAProxy primary
sudo firewall-cmd --permanent --add-port=5001/tcp   # HAProxy replica
sudo firewall-cmd --permanent --add-port=7000/tcp   # HAProxy stats
sudo firewall-cmd --permanent --add-protocol=vrrp   # Keepalived VRRP
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### 1.6 Konfigurasi SELinux

Ada dua pendekatan yang bisa dipilih:

**Opsi A — Set SELinux ke Permissive (lebih mudah, kurang aman):**

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Verifikasi
getenforce
# Output: Permissive
```

**Opsi B — Tetap Enforcing dengan policy yang tepat (rekomendasi production):**

> Di Rocky Linux 9 dengan SELinux enforcing, ada **dua** hal yang perlu dikonfigurasi untuk HAProxy:
> 1. Izinkan HAProxy **bind** ke port 5000/5001/7000 → `semanage port`
> 2. Izinkan HAProxy **connect** ke port PostgreSQL 5432 → `setsebool haproxy_connect_any`
>
> Tanpa keduanya, HAProxy tidak bisa meneruskan koneksi ke PostgreSQL meskipun service berjalan normal.

```bash
# Install policy tools jika belum ada
sudo dnf install -y policycoreutils-python-utils

# 1. Labeli port HAProxy agar diizinkan SELinux
sudo semanage port -a -t haproxy_port_t -p tcp 5000
sudo semanage port -a -t haproxy_port_t -p tcp 5001
sudo semanage port -a -t haproxy_port_t -p tcp 7000

# Verifikasi label sudah terdaftar
sudo semanage port -l | grep -E "5000|5001|7000"

# 2. Izinkan HAProxy connect ke port PostgreSQL (5432)
sudo setsebool -P haproxy_connect_any 1

# Verifikasi
getsebool haproxy_connect_any
# Output: haproxy_connect_any --> on
```

> **Catatan:** Di environment ini (Rocky Linux 9 dengan SELinux enforcing), Opsi B yang digunakan. Step ini dilakukan setelah HAProxy diinstall di Step 5.

---

## Step 2 — Instalasi PostgreSQL 16

Lakukan di **semua node**.

### 2.1 Install repo PGDG

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### 2.2 Disable modul PostgreSQL bawaan Rocky (wajib!)

```bash
sudo dnf -qy module disable postgresql
```

### 2.3 Install PostgreSQL 16

> **Catatan:** `postgresql16-devel` tidak diinstall karena membutuhkan `perl(IPC::Run)` yang tidak tersedia di repo Rocky 9. Package ini tidak diperlukan untuk kebutuhan Patroni HA cluster.

```bash
sudo dnf install -y postgresql16-server postgresql16-contrib
```

### 2.4 Stop dan disable service PostgreSQL default

> Patroni yang akan mengelola PostgreSQL, bukan systemd langsung.

```bash
sudo systemctl stop postgresql-16
sudo systemctl disable postgresql-16

# Verifikasi
sudo systemctl status postgresql-16
# Output: disabled
```

### 2.5 Install dependency Python untuk Patroni

```bash
sudo dnf install -y python3-pip python3-devel gcc
pip3 install --upgrade pip
```

### 2.6 Install Patroni dan dependency

> **Catatan Rocky Linux 9:** Gunakan flag `--break-system-packages` karena Rocky 9 menggunakan "externally managed" Python environment. Tanpa flag ini, pip3 akan menolak instalasi.

```bash
sudo pip3 install patroni python-etcd psycopg2-binary --break-system-packages

# Verifikasi binary tersedia
which patroni
# Output: /usr/local/bin/patroni

patroni --version
# Output: patroni 4.x.x
```

> **Jika `patroni` tidak ditemukan di PATH setelah install:** Binary mungkin terinstall di `~/.local/bin/`. Solusi: copy ke `/usr/local/bin/` agar bisa diakses oleh systemd service yang berjalan sebagai user `postgres`.
>
> ```bash
> sudo cp ~/.local/bin/patroni /usr/local/bin/patroni
> sudo chmod 755 /usr/local/bin/patroni
> ls -la /usr/local/bin/patroni
> ```

---

## Step 3 — Instalasi dan Konfigurasi etcd

Lakukan di **semua node**.

> **Catatan:** Package `etcd` tidak tersedia di repo default Rocky Linux 9 (`dnf install etcd` akan gagal). Instalasi dilakukan secara manual menggunakan binary resmi dari GitHub.

### 3.1 Download dan install etcd binary

```bash
# Set versi etcd
ETCD_VER=v3.5.17

# Download binary
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd.tar.gz

# Extract
tar xzvf /tmp/etcd.tar.gz -C /tmp/

# Copy binary ke /usr/local/bin
sudo cp /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo cp /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo cp /tmp/etcd-${ETCD_VER}-linux-amd64/etcdutl /usr/local/bin/

# Set permission
sudo chmod +x /usr/local/bin/etcd
sudo chmod +x /usr/local/bin/etcdctl
sudo chmod +x /usr/local/bin/etcdutl

# Verifikasi
etcd --version
etcdctl version
```

### 3.2 Buat direktori etcd

> Tidak perlu membuat user baru — menggunakan user `postgres` yang sudah ada, supaya konsisten dengan Patroni.

```bash
# Buat direktori data dan config
sudo mkdir -p /data/etcd
sudo mkdir -p /etc/etcd
sudo chown postgres:postgres /data/etcd
```

### 3.3 Buat systemd service untuk etcd

```bash
sudo nano /etc/systemd/system/etcd.service
```

```ini
[Unit]
Description=etcd distributed key-value store
Documentation=https://etcd.io/docs
After=network.target

[Service]
Type=notify
User=postgres
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=5s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

### 3.4 Konfigurasi etcd

> Format konfigurasi menggunakan `KEY=VALUE` (EnvironmentFile systemd), berbeda dengan Ubuntu yang menggunakan format YAML. Parameternya sama, hanya formatnya yang berbeda.

**Buat file konfigurasi di masing-masing node:**

```bash
sudo nano /etc/etcd/etcd.conf
```

**Isi untuk confluent-1 (10.100.13.241):**

```bash
ETCD_NAME="confluent-1"
ETCD_INITIAL_CLUSTER_TOKEN="cluster123"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER="confluent-1=http://10.100.13.241:2380,confluent-2=http://10.100.13.242:2380,confluent-3=http://10.100.13.243:2380"
ETCD_DATA_DIR="/data/etcd"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.100.13.241:2380"
ETCD_LISTEN_PEER_URLS="http://10.100.13.241:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.100.13.241:2379,http://127.0.0.1:2379"
ETCD_LISTEN_CLIENT_URLS="http://10.100.13.241:2379,http://127.0.0.1:2379"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LOG_LEVEL="info"
ETCD_AUTO_COMPACTION_RETENTION="24"
```

**Isi untuk confluent-2 (10.100.13.242):**

```bash
ETCD_NAME="confluent-2"
ETCD_INITIAL_CLUSTER_TOKEN="cluster123"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER="confluent-1=http://10.100.13.241:2380,confluent-2=http://10.100.13.242:2380,confluent-3=http://10.100.13.243:2380"
ETCD_DATA_DIR="/data/etcd"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.100.13.242:2380"
ETCD_LISTEN_PEER_URLS="http://10.100.13.242:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.100.13.242:2379,http://127.0.0.1:2379"
ETCD_LISTEN_CLIENT_URLS="http://10.100.13.242:2379,http://127.0.0.1:2379"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LOG_LEVEL="info"
ETCD_AUTO_COMPACTION_RETENTION="24"
```

**Isi untuk confluent-3 (10.100.13.243):**

```bash
ETCD_NAME="confluent-3"
ETCD_INITIAL_CLUSTER_TOKEN="cluster123"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER="confluent-1=http://10.100.13.241:2380,confluent-2=http://10.100.13.242:2380,confluent-3=http://10.100.13.243:2380"
ETCD_DATA_DIR="/data/etcd"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.100.13.243:2380"
ETCD_LISTEN_PEER_URLS="http://10.100.13.243:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.100.13.243:2379,http://127.0.0.1:2379"
ETCD_LISTEN_CLIENT_URLS="http://10.100.13.243:2379,http://127.0.0.1:2379"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LOG_LEVEL="info"
ETCD_AUTO_COMPACTION_RETENTION="24"
```

### 3.6 Enable dan start etcd di semua node

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd
```

### 3.7 Verifikasi etcd cluster

```bash
etcdctl member list
# Output contoh:
# 1234567890: name=confluent-1 ...
# 2345678901: name=confluent-2 ...
# 3456789012: name=confluent-3 ...

etcdctl endpoint health
# Output: http://127.0.0.1:2379 is healthy
```

> **Penting:** Lakukan Step 3.1 s/d 3.6 di **semua node** sebelum verifikasi cluster. etcd membutuhkan quorum minimal 2 dari 3 node untuk healthy. Jika hanya 1 node running, cluster belum terbentuk.

---

## Step 4 — Konfigurasi Patroni

Lakukan di **semua node**.

### 4.1 Buat direktori Patroni

```bash
sudo mkdir -p /etc/patroni
sudo mkdir -p /data/postgresql/16/main
sudo mkdir -p /data/log/postgresql
sudo chown -R postgres:postgres /data/postgresql
sudo chown -R postgres:postgres /data/log/postgresql

# Buat file pgpass
sudo touch /etc/patroni/16-main.pgpass
sudo chown postgres:postgres /etc/patroni/16-main.pgpass
sudo chmod 600 /etc/patroni/16-main.pgpass
```

### 4.2 Buat file konfigurasi Patroni

> **Catatan perbedaan dari Ubuntu:** Konfigurasi `pg_createcluster` dan `pg_clonecluster` dihapus karena merupakan tool khusus Ubuntu/Debian yang tidak tersedia di Rocky Linux. Patroni di Rocky menggunakan `initdb` dan `pg_basebackup` bawaan PostgreSQL secara langsung.

```bash
sudo nano /etc/patroni/16-main.yml
```

**Isi untuk confluent-1 (10.100.13.241):**

```yaml
scope: "16-main"
namespace: "/postgresql-common/"
name: confluent-1

etcd3:
  hosts:
    - 10.100.13.241:2379
    - 10.100.13.242:2379
    - 10.100.13.243:2379

restapi:
  listen: 10.100.13.241:8008
  connect_address: 10.100.13.241:8008

bootstrap:
  dcs:
    ttl: 10
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 64MB
    check_timeline: true
    postgresql:
      use_pg_rewind: true
      remove_data_directory_on_rewind_failure: false
      remove_data_directory_on_diverged_timelines: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 512MB
        max_wal_senders: 20
        max_replication_slots: 20
        wal_log_hints: "on"
        track_commit_timestamp: "off"
      pg_hba:
        - local   all             all                                     peer
        - host    all             all             127.0.0.1/32            md5
        - host    all             all             ::1/128                 md5
        - host    all             all             10.100.13.241/24        md5
        - host    all             all             10.100.13.242/24        md5
        - host    all             all             10.100.13.243/24        md5
        - local   replication     all                                     peer
        - host    replication     all             127.0.0.1/32            md5
        - host    replication     all             ::1/128                 md5
        - host    replication     all             10.100.13.242/24        md5
        - host    replication     all             10.100.13.243/24        md5
        - host    all             all             0.0.0.0/0               md5
        - host    replication     all             0.0.0.0/0               md5

postgresql:
  listen: "*:5432"
  connect_address: 10.100.13.241:5432
  use_unix_socket: true
  data_dir: /data/postgresql/16/main
  bin_dir: /usr/pgsql-16/bin
  pgpass: /etc/patroni/16-main.pgpass
  authentication:
    replication:
      username: "replicator"
      password: "rep-pass"
    superuser:
      username: "postgres"
      password: "YourSuperuserPassword"
  parameters:
    unix_socket_directories: '/var/run/postgresql/'
    logging_collector: 'on'
    log_directory: '/data/log/postgresql'
    log_filename: 'postgresql-16-main.log'
    log_rotation_age: '1d'
    log_rotation_size: '100MB'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

**Untuk confluent-2 (10.100.13.242)**, salin config di atas lalu ubah hanya 3 bagian ini:

```yaml
name: confluent-2

restapi:
  listen: 10.100.13.242:8008
  connect_address: 10.100.13.242:8008

postgresql:
  listen: "*:5432"
  connect_address: 10.100.13.242:5432
```

**Untuk confluent-3 (10.100.13.243)**, salin config di atas lalu ubah hanya 3 bagian ini:

```yaml
name: confluent-3

restapi:
  listen: 10.100.13.243:8008
  connect_address: 10.100.13.243:8008

postgresql:
  listen: "*:5432"
  connect_address: 10.100.13.243:5432
```

**Perbedaan konfigurasi Ubuntu vs Rocky 9:**

| Parameter | Ubuntu (kantor) | Rocky 9 |
|---|---|---|
| `bin_dir` | `/usr/lib/postgresql/16/bin` | `/usr/pgsql-16/bin` |
| `data_dir` | `/database/postgresql/16/main` | `/data/postgresql/16/main` |
| `config_dir` | `/etc/postgresql/16/main` | *(tidak perlu)* |
| `pgpass` | `/etc/postgresql/16/main/16-main.pgpass` | `/etc/patroni/16-main.pgpass` |
| `log_directory` | `/database/log/postgresql` | `/data/log/postgresql` |
| `listen port` | `*:15432` | `*:5432` |
| `bootstrap method` | `pg_createcluster` | *(dihapus, tidak ada di Rocky)* |
| `create_replica_method` | `pg_clonecluster` | *(dihapus, tidak ada di Rocky)* |

### 4.3 Buat systemd service untuk Patroni

```bash
sudo nano /etc/systemd/system/patroni@.service
```

```ini
[Unit]
Description=Patroni PostgreSQL HA - %i
After=syslog.target network.target etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/%i.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
```

### 4.4 Set permission direktori Patroni

> **Penting:** Direktori `/etc/patroni` harus dimiliki oleh user `postgres` agar Patroni bisa membaca file konfigurasi saat berjalan sebagai service.

```bash
sudo chown -R postgres:postgres /etc/patroni
sudo chmod 755 /etc/patroni
sudo chmod 644 /etc/patroni/16-main.yml

# Verifikasi
ls -la /etc/ | grep patroni
# Output: drwxr-xr-x  2 postgres postgres ... patroni
```

### 4.5 Buat symlink patronictl agar bisa diakses tanpa sudo

```bash
# patronictl terinstall di /usr/local/bin/ — buat symlink ke /usr/bin/ agar mudah dipanggil
sudo ln -s /usr/local/bin/patronictl /usr/bin/patronictl

# Verifikasi
patronictl -c /etc/patroni/16-main.yml list
```

### 4.6 Enable dan start Patroni

```bash
sudo systemctl daemon-reload
sudo systemctl enable patroni@16-main

# Start di confluent-1 dulu (akan jadi primary)
sudo systemctl start patroni@16-main
sudo systemctl status patroni@16-main

# Tunggu 30 detik, lalu start di confluent-2 dan confluent-3
sudo systemctl start patroni@16-main
```

### 4.7 Verifikasi Patroni cluster

```bash
patronictl -c /etc/patroni/16-main.yml list

# Output contoh:
# + Cluster: 16-main (7625919654464255739) -----------+----+-------------+-----+------------+-----+
# | Member      | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
# +-------------+---------------+---------+-----------+----+-------------+-----+------------+-----+
# | confluent-1 | 10.100.13.241 | Leader  | running   |  1 |             |     |            |     |
# | confluent-2 | 10.100.13.242 | Replica | streaming |  1 |   0/504E298 |   0 |  0/504E298 |   0 |
# | confluent-3 | 10.100.13.243 | Replica | streaming |  1 |   0/504E298 |   0 |  0/504E298 |   0 |
# +-------------+---------------+---------+-----------+----+-------------+-----+------------+-----+
```

---

## Step 5 — Instalasi dan Konfigurasi HAProxy

Lakukan di **semua node**.

### 5.1 Install HAProxy

```bash
sudo dnf install -y haproxy
```

### 5.2 Konfigurasi HAProxy

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

```
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    daemon

defaults
    log global
    mode tcp
    retries 3
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout check 5s

# Stats page (akses via http://IP:7000)
listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /
    stats refresh 10s
    stats show-node

# Primary — read/write (port 5000)
# HAProxy cek endpoint /primary di Patroni REST API port 8008
listen primary
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server confluent-1 10.100.13.241:5432 check port 8008
    server confluent-2 10.100.13.242:5432 check port 8008
    server confluent-3 10.100.13.243:5432 check port 8008

# Replica — read only (port 5001)
# HAProxy cek endpoint /replica di Patroni REST API port 8008
listen standbys
    bind *:5001
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server confluent-1 10.100.13.241:5432 check port 8008
    server confluent-2 10.100.13.242:5432 check port 8008
    server confluent-3 10.100.13.243:5432 check port 8008
```

### 5.3 Konfigurasi SELinux untuk HAProxy

> **Penting di Rocky Linux 9:** SELinux memblokir dua hal berbeda yang keduanya harus difix:
> 1. HAProxy **bind** ke port non-standar (5000, 5001, 7000) → fix dengan `semanage port`
> 2. HAProxy **connect** ke port PostgreSQL (5432) → fix dengan `setsebool haproxy_connect_any`
>
> **Tanpa keduanya, HAProxy tidak bisa meneruskan koneksi ke PostgreSQL meskipun stats page menunjukkan backend UP.**

```bash
# Install policy tools jika belum ada
sudo dnf install -y policycoreutils-python-utils

# 1. Izinkan HAProxy bind ke port 5000, 5001, 7000
sudo semanage port -a -t haproxy_port_t -p tcp 5000
sudo semanage port -a -t haproxy_port_t -p tcp 5001
sudo semanage port -a -t haproxy_port_t -p tcp 7000

# Verifikasi
sudo semanage port -l | grep -E "5000|5001|7000"

# 2. Izinkan HAProxy connect ke port PostgreSQL (5432) — INI YANG PALING KRITIS
sudo setsebool -P haproxy_connect_any 1

# Verifikasi
getsebool haproxy_connect_any
# Output: haproxy_connect_any --> on
```

> **Catatan:** Jika `semanage port -a` muncul error `already defined`, gunakan `-m` (modify):
> ```bash
> sudo semanage port -m -t http_port_t -p tcp 5000
> sudo semanage port -m -t http_port_t -p tcp 5001
> sudo semanage port -m -t http_port_t -p tcp 7000
> ```

### 5.4 Konfigurasi HAProxy

```bash
sudo tee /etc/haproxy/haproxy.cfg << 'EOF'
global
    nbthread 4
    maxconn 20000

defaults
    mode tcp
    option tcplog
    option tcpka
    retries 3
    timeout client 1h
    timeout connect 10s
    timeout server 1h
    timeout check 10s

# Stats page (akses via http://IP:7000)
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats refresh 10s
    stats show-node

# Primary — read/write (port 5000)
# HAProxy cek endpoint /primary di Patroni REST API port 8008
listen primary
    bind *:5000
    option httpchk
    http-check connect port 8008
    http-check send meth GET uri /primary
    http-check expect status 200
    default-server inter 2s fall 2 rise 3 on-marked-down shutdown-sessions
    server confluent-1 10.100.13.241:5432 maxconn 5000 check port 8008
    server confluent-2 10.100.13.242:5432 maxconn 5000 check port 8008
    server confluent-3 10.100.13.243:5432 maxconn 5000 check port 8008

# Standbys — read only (port 5001)
# HAProxy cek endpoint /replica di Patroni REST API port 8008
listen standbys
    bind *:5001
    balance leastconn
    option httpchk
    http-check connect port 8008
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 2s fall 2 rise 3 on-marked-down shutdown-sessions
    server confluent-1 10.100.13.241:5432 maxconn 5000 check port 8008
    server confluent-2 10.100.13.242:5432 maxconn 5000 check port 8008
    server confluent-3 10.100.13.243:5432 maxconn 5000 check port 8008
EOF
```

> **Catatan perbedaan syntax HAProxy 2.x vs 1.x:** Gunakan `http-check connect` + `http-check send` + `http-check expect` (bukan `option httpchk GET /primary HTTP/1.0`). Syntax lama tidak valid di HAProxy 2.8+ untuk mode TCP dengan httpchk.

### 5.5 Enable dan start HAProxy

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

### 5.6 Verifikasi HAProxy

```bash
# Cek port listening
ss -tlnp | grep haproxy

# Akses stats page
curl http://10.100.13.241:7000/
```



---

## Step 6 — Instalasi dan Konfigurasi Keepalived

Lakukan di **semua node**.

### 6.1 Install Keepalived

```bash
sudo dnf install -y keepalived
```

### 6.2 Buat script health check HAProxy

```bash
sudo nano /etc/keepalived/chk_haproxy.sh
```

```bash
#!/bin/bash
if systemctl is-active --quiet haproxy; then
    exit 0
else
    exit 1
fi
```

```bash
sudo chmod +x /etc/keepalived/chk_haproxy.sh
```

### 6.3 Cek nama interface jaringan

```bash
ip a
# Catat nama interface, contoh: eth0, enp0s3, ens32, ens192, dll
# Di environment ini interface yang digunakan adalah: ens32
```

### 6.4 Konfigurasi Keepalived

```bash
sudo nano /etc/keepalived/keepalived.conf
```

**Untuk confluent-1 (MASTER):**

```
global_defs {
    router_id postgresql_ha
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/chk_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens32          # sesuaikan dengan nama interface
    virtual_router_id 51
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass P@ssw0rd123
    }

    virtual_ipaddress {
        10.100.13.248/24
    }

    track_script {
        chk_haproxy
    }
}
```

**Untuk confluent-2 (BACKUP):**

```
global_defs {
    router_id postgresql_ha
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/chk_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32          # sesuaikan dengan nama interface
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass P@ssw0rd123
    }

    virtual_ipaddress {
        10.100.13.248/24
    }

    track_script {
        chk_haproxy
    }
}
```

**Untuk confluent-3 (BACKUP):**

```
# Sama seperti confluent-2, ubah priority menjadi:
priority 99
```

### 6.5 Enable dan start Keepalived

```bash
sudo systemctl enable keepalived
sudo systemctl restart keepalived
sudo systemctl status keepalived
```

### 6.6 Verifikasi Floating IP

```bash
# Di confluent-1 (MASTER), floating IP harus muncul
ip a show ens32 | grep 10.100.13.248
# Harus ada: inet 10.100.13.248/24 scope global secondary ens32

# Ping floating IP dari node lain
ping 10.100.13.248 -c 3
```

---

## Step 7 — Urutan Start Service (Setelah Reboot)

Jika semua node direstart, jalankan service dengan urutan berikut:

```bash
# 1. Start etcd di semua node
sudo systemctl start etcd

# 2. Verifikasi etcd cluster sehat
etcdctl member list
etcdctl endpoint health

# 3. Start Patroni di semua node (confluent-1 dulu)
sudo systemctl start patroni@16-main

# 4. Verifikasi cluster Patroni
patronictl -c /etc/patroni/16-main.yml list

# 5. Start HAProxy di semua node
sudo systemctl start haproxy

# 6. Start Keepalived di semua node
sudo systemctl start keepalived
```

---

## Step 8 — Testing

### 8.1 Cek status cluster Patroni

```bash
patronictl -c /etc/patroni/16-main.yml list
```

Output yang diharapkan:

```
+ Cluster: 16-main (7625919654464255739) -----------+----+-------------+-----+------------+-----+
| Member      | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+-------------+---------------+---------+-----------+----+-------------+-----+------------+-----+
| confluent-1 | 10.100.13.241 | Leader  | running   |  1 |             |     |            |     |
| confluent-2 | 10.100.13.242 | Replica | streaming |  1 |   0/504E298 |   0 |  0/504E298 |   0 |
| confluent-3 | 10.100.13.243 | Replica | streaming |  1 |   0/504E298 |   0 |  0/504E298 |   0 |
+-------------+---------------+---------+-----------+----+-------------+-----+------------+-----+
```

> **Catatan:** Kolom `Lag in MB` di Patroni versi lama diganti dengan `Receive LSN` / `Replay LSN` di versi terbaru. Replica status `streaming` (bukan `running`) adalah normal dan menandakan replikasi streaming aktif berjalan.

### 8.2 Cek health endpoint Patroni

```bash
# Cek primary
curl http://10.100.13.241:8008/primary
# Output: HTTP 200 = ini primary

# Cek replica
curl http://10.100.13.242:8008/replica
# Output: HTTP 200 = ini replica

# Cek info lengkap node
curl http://10.100.13.241:8008/patroni | python3 -m json.tool
```

### 8.3 Koneksi ke PostgreSQL via Floating IP

> **Sebelum test ini:** Pastikan VIP sudah aktif di confluent-1 dengan `ip a show ens32 | grep 10.100.13.248`. Jika VIP belum muncul, cek status keepalived.

```bash
# Koneksi ke primary (port 5000) via floating IP
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5000 user=postgres sslmode=disable" \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"

# Output yang diharapkan:
#  inet_server_addr | pg_is_in_recovery
# ------------------+-------------------
#  10.100.13.241    | f               <- f = false = ini primary
# (1 row)

# Koneksi ke replica (port 5001) via floating IP
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5001 user=postgres sslmode=disable" \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"

# Output yang diharapkan:
#  inet_server_addr | pg_is_in_recovery
# ------------------+-------------------
#  10.100.13.242    | t               <- t = true = ini replica
# (1 row)
```

> **Jika muncul error "server closed the connection unexpectedly":** Kemungkinan besar SELinux memblokir HAProxy connect ke port 5432. Jalankan `sudo setsebool -P haproxy_connect_any 1` di semua node lalu restart HAProxy. Lihat bagian Troubleshooting di bawah.

### 8.4 Test write ke primary

```bash
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5000 user=postgres sslmode=disable"

# Di dalam psql:
CREATE DATABASE testdb;
\c testdb
CREATE TABLE test (id serial, nama text, waktu timestamp default now());
INSERT INTO test (nama) VALUES ('test-1'), ('test-2'), ('test-3');
SELECT * FROM test;
```

### 8.5 Test read dari replica

```bash
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5001 user=postgres sslmode=disable" -d testdb

# Di dalam psql:
SELECT * FROM test;
# Data harus sudah tereplikasi dari primary
```

### 8.6 Test failover — simulasi primary mati

```bash
# 1. Catat siapa primary saat ini
patronictl -c /etc/patroni/16-main.yml list

# 2. Stop Patroni di node primary (misal confluent-1)
# Lakukan di confluent-1:
sudo systemctl stop patroni@16-main

# 3. Cek dari node lain — siapa yang jadi leader baru
# Tunggu 30-60 detik
patronictl -c /etc/patroni/16-main.yml list

# Output yang diharapkan:
# confluent-2 atau confluent-3 sekarang jadi Leader

# 4. Test koneksi masih bisa via floating IP
psql -h 10.100.13.248 -p 5000 -U postgres -c "SELECT inet_server_addr();"

# 5. Hidupkan kembali confluent-1
# Di confluent-1:
sudo systemctl start patroni@16-main

# Patroni otomatis join kembali sebagai replica
patronictl -c /etc/patroni/16-main.yml list
```
<img width="1213" height="456" alt="image" src="https://github.com/user-attachments/assets/79d2d60c-b978-4fda-b300-491945172e9f" />
<img width="908" height="237" alt="image" src="https://github.com/user-attachments/assets/3d4a5df7-0ac2-447d-a882-26cedfb3fba5" />
<img width="1199" height="268" alt="image" src="https://github.com/user-attachments/assets/4c8c6976-0063-4c26-a959-8d44c7b4c8d8" />


### 8.7 Test failover manual (switchover)

```bash
# Switchover ke node tertentu
patronictl -c /etc/patroni/16-main.yml switchover \
  --leader confluent-3 \
  --candidate confluent-2 \
  --force

# Verifikasi
patronictl -c /etc/patroni/16-main.yml list
```
<img width="1672" height="638" alt="image" src="https://github.com/user-attachments/assets/b26faf95-1bbc-4eff-9562-74c3c211f0e2" />

### 8.8 Test Keepalived failover

```bash
# 1. Cek VIP ada di confluent-1
ip a show ens32 | grep 10.100.13.248

# 2. Stop Keepalived di confluent-1
sudo systemctl stop keepalived

# 3. Cek VIP pindah ke confluent-2
# Di confluent-2:
ip a show ens32 | grep 10.100.13.248
# VIP harus muncul di sini

# 4. Hidupkan kembali Keepalived di confluent-1
sudo systemctl start keepalived
# VIP kembali ke confluent-1 (karena priority lebih tinggi)
```
<img width="754" height="146" alt="image" src="https://github.com/user-attachments/assets/908067ed-e2d2-4a5e-8d03-832fae801bae" />
<img width="788" height="114" alt="image" src="https://github.com/user-attachments/assets/ac139164-8a2a-4f1c-92fe-1d7a04a5dd67" />
<img width="778" height="203" alt="image" src="https://github.com/user-attachments/assets/6af922db-48cb-4383-b6fa-ed931024dd01" />

### 8.9 Cek HAProxy stats

Buka browser dan akses:

```
http://10.100.13.248:7000/
```
<img width="1901" height="949" alt="image" src="https://github.com/user-attachments/assets/ae52bf06-cbda-41e5-84ab-4a9098be9d5c" />

Atau via curl:

```bash
curl http://10.100.13.248:7000/ | grep -E "UP|DOWN"
```

---

## Troubleshooting

### Patroni gagal start — tidak bisa konek ke etcd

```bash
# Cek etcd running
sudo systemctl status etcd

# Cek log Patroni
sudo journalctl -u patroni@16-main -n 50 --no-pager

# Test koneksi ke etcd manual
curl http://127.0.0.1:2379/health
```

### PostgreSQL tidak bisa start via Patroni

```bash
# Cek log PostgreSQL
sudo -u postgres cat /data/postgresql/16/main/log/postgresql-*.log

# Cek permission direktori
ls -la /data/postgresql/
# Harus milik user postgres:postgres
sudo chown -R postgres:postgres /data/postgresql
```

### psql via VIP gagal — "server closed the connection unexpectedly"

Ini terjadi ketika HAProxy sudah sehat (backend UP) tapi PostgreSQL menolak koneksi yang masuk.

**Langkah diagnosis:**

```bash
# 1. Pastikan VIP aktif di node MASTER
ip a show ens32 | grep 10.100.13.248
# Harus muncul: inet 10.100.13.248/24

# 2. Pastikan HAProxy backend UP (bukan error routing)
curl -s http://10.100.13.248:7000/ | grep -E "primary|standbys"
# primary/confluent-1 harus UP, standbys/confluent-2 dan /confluent-3 harus UP

# 3. Coba koneksi langsung ke PostgreSQL (bypass HAProxy)
psql -h 10.100.13.241 -p 5432 -U postgres -c "SELECT 1;"
# Jika ini juga gagal → masalah ada di pg_hba.conf atau PostgreSQL

# 4. Cek pg_hba.conf — pastikan koneksi dari luar diizinkan
sudo -u postgres cat /data/postgresql/16/main/pg_hba.conf | grep -v "^#" | grep -v "^$"
```

**Perbaikan pg_hba.conf jika koneksi dari network ditolak:**

Pastikan di dalam konfigurasi Patroni (`/etc/patroni/16-main.yml`) bagian `pg_hba` sudah ada baris berikut:

```yaml
pg_hba:
  - host    all             all             10.100.13.0/24          md5
  - host    all             all             0.0.0.0/0               md5
  - host    replication     all             0.0.0.0/0               md5
```

Setelah edit `16-main.yml`, reload konfigurasi Patroni:

```bash
# Reload pg_hba tanpa restart PostgreSQL
patronictl -c /etc/patroni/16-main.yml reload 16-main

# Atau jika perlu restart:
patronictl -c /etc/patroni/16-main.yml restart 16-main confluent-1
```

**Verifikasi setelah perbaikan:**

```bash
psql -h 10.100.13.248 -p 5000 -U postgres -c "SELECT inet_server_addr(), pg_is_in_recovery();"
# Harus berhasil dan mengembalikan IP primary dengan pg_is_in_recovery = f
```

### HAProxy menandai semua server DOWN / psql via VIP gagal "server closed the connection unexpectedly"

Ada dua kemungkinan penyebab di Rocky Linux 9 — keduanya terkait SELinux:

**Kemungkinan 1: HAProxy tidak bisa bind ke port 5000/5001/7000**

```bash
# Cek SELinux denial
sudo ausearch -m avc -ts recent | grep haproxy | grep "dest=500\|dest=700"

# Fix: label port
sudo semanage port -a -t haproxy_port_t -p tcp 5000
sudo semanage port -a -t haproxy_port_t -p tcp 5001
sudo semanage port -a -t haproxy_port_t -p tcp 7000
sudo systemctl restart haproxy
```

**Kemungkinan 2: HAProxy tidak bisa connect ke PostgreSQL port 5432 (paling umum)**

Gejala: HAProxy stats menunjukkan backend **UP** (health check Patroni REST API berhasil), tapi koneksi psql tetap gagal dengan "server closed the connection unexpectedly". Di audit log terlihat:

```
avc: denied { name_connect } for comm="haproxy" dest=5432
scontext=haproxy_t tcontext=postgresql_port_t permissive=0
```

```bash
# Cek denial
sudo ausearch -m avc -ts recent | grep haproxy | grep "dest=5432"

# Fix: izinkan HAProxy connect ke semua port
sudo setsebool -P haproxy_connect_any 1

# Verifikasi
getsebool haproxy_connect_any
# Output: haproxy_connect_any --> on

sudo systemctl restart haproxy
```

**Verifikasi Patroni REST API:**

```bash
# Cek primary — harus HTTP 200
curl -o /dev/null -w "%{http_code}" http://10.100.13.241:8008/primary

# Cek replica — harus HTTP 200
curl -o /dev/null -w "%{http_code}" http://10.100.13.242:8008/replica
# 503 = ini primary (bukan replica) — normal

# Cek log HAProxy
sudo journalctl -u haproxy -n 50 --no-pager
```



### Floating IP tidak berpindah saat failover

```bash
# Cek log Keepalived
sudo journalctl -u keepalived -n 50 --no-pager

# Pastikan VRRP tidak diblokir firewall
sudo firewall-cmd --list-protocols | grep vrrp

# Cek priority antar node
sudo grep priority /etc/keepalived/keepalived.conf
```

### Cek semua service berjalan normal

```bash
# Jalankan di semua node
for svc in etcd patroni@16-main haproxy keepalived; do
  echo "=== $svc ==="
  sudo systemctl is-active $svc
done
```

---

## Referensi

| Dokumentasi | URL |
|-------------|-----|
| Patroni | https://patroni.readthedocs.io |
| etcd | https://etcd.io/docs |
| HAProxy | https://www.haproxy.org/download/2.6/doc/configuration.txt |
| PostgreSQL 16 | https://www.postgresql.org/docs/16/ |
| Rocky Linux 9 | https://docs.rockylinux.org |

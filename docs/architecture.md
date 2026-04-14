# Arsitektur HA PostgreSQL Cluster

## Gambaran Besar

Setup ini membangun PostgreSQL cluster dengan High Availability (HA) — artinya database tetap bisa diakses meskipun salah satu node mati. Ini dicapai dengan menggabungkan 4 tools yang masing-masing punya peran berbeda.

```
Aplikasi / Client
       │
       ▼
[Floating IP 10.100.13.248]  ← dikelola Keepalived
       │
       ▼
[HAProxy :5000 / :5001]      ← load balancer, tau mana primary/replica
       │              │
       ▼              ▼
[PostgreSQL]    [PostgreSQL]  ← dikelola Patroni
[Primary]       [Replica]
       │
       ▼
[etcd cluster]               ← penyimpan "siapa yang jadi leader"
```

---

## Peran Tiap Komponen

### PostgreSQL 16
Database engine yang sebenarnya. Menyimpan data. Di setup ini PostgreSQL **tidak dijalankan langsung via systemd** — melainkan dikelola sepenuhnya oleh Patroni.

### Patroni
"Otak" dari cluster. Patroni yang memutuskan siapa Primary, siapa Replica, dan apa yang terjadi kalau Primary mati. Patroni expose REST API di port 8008 yang digunakan HAProxy untuk health check.

### etcd
Tempat Patroni menyimpan state cluster — siapa leader, timeline berapa, konfigurasi DCS. Semua node Patroni membaca dan menulis ke etcd. etcd sendiri juga cluster (3 node) dan butuh minimal 2 dari 3 node hidup (quorum).

### HAProxy
Load balancer yang menerima koneksi dari client dan meneruskannya ke node yang tepat. HAProxy tahu mana Primary dan mana Replica dengan cara **nanya ke Patroni REST API** (`/primary` atau `/replica`) setiap beberapa detik.

### Keepalived
Mengelola **Floating IP** (Virtual IP / VIP) `10.100.13.248`. VIP ini yang dipakai client untuk konek — jadi client tidak perlu tahu IP node mana yang sedang aktif. Keepalived memindahkan VIP ke node lain jika node MASTER mati.

---

## Alur Failover Otomatis

Ketika confluent-1 (Primary) tiba-tiba mati:

```
1. Patroni di confluent-2 & confluent-3 mendeteksi leader hilang
   (via etcd — leader key expired)

2. Patroni melakukan election:
   → Node dengan data paling up-to-date dipilih jadi Primary baru
   → Misal: confluent-2 terpilih

3. confluent-2 promote dirinya jadi Primary
   → pg_is_in_recovery() berubah dari true → false

4. HAProxy health check mendeteksi perubahan:
   → /primary di confluent-1: tidak merespons (DOWN)
   → /primary di confluent-2: HTTP 200 (UP)
   → Traffic port 5000 dialihkan ke confluent-2

5. Keepalived:
   → Jika confluent-1 juga mati, VIP berpindah ke confluent-2
   → Client tetap konek via 10.100.13.248 tanpa ganti config

6. Saat confluent-1 hidup kembali:
   → Patroni otomatis join sebagai Replica
   → pg_basebackup sync data dari Primary baru
```

Downtime yang terjadi: **30–60 detik** (waktu election + HAProxy detect)

---

## Keputusan Desain

### Kenapa semua service di tiap node?
etcd, HAProxy, dan Keepalived diinstall di semua 3 node — bukan di node terpisah. Ini membuat setup lebih sederhana dan setiap node bisa berdiri sendiri. Trade-off: resource lebih besar per node.

### Kenapa HAProxy pakai Patroni REST API untuk health check?
HAProxy tidak bisa "nanya" ke PostgreSQL apakah dia Primary atau Replica secara langsung tanpa query. Patroni menyediakan endpoint HTTP khusus (`/primary`, `/replica`) yang mengembalikan HTTP 200 jika node punya role tersebut — ini jauh lebih mudah diintegrasikan dengan HAProxy.

### Kenapa PostgreSQL tidak distart via systemd?
Patroni perlu kontrol penuh atas lifecycle PostgreSQL — kapan start, kapan stop, kapan promote. Kalau PostgreSQL distart via systemd secara independen, Patroni tidak bisa mengelolanya. Ini adalah pola standar di setup Patroni.

### Kenapa etcd pakai user postgres?
Untuk menyederhanakan permission management. Daripada membuat user baru khusus etcd, kita reuse user `postgres` yang sudah ada — karena etcd dan Patroni perlu akses ke direktori yang sama.

### SELinux: kenapa Enforcing bukan Permissive?
Di production, SELinux Enforcing lebih aman. Meski butuh konfigurasi tambahan (`semanage port` dan `setsebool haproxy_connect_any`), ini adalah cara yang benar. Permissive hanya untuk development/testing.

---

## Perbedaan Setup Ubuntu vs Rocky Linux 9

| Aspek | Ubuntu | Rocky Linux 9 |
|---|---|---|
| PostgreSQL bin_dir | `/usr/lib/postgresql/16/bin` | `/usr/pgsql-16/bin` |
| etcd install | `apt install etcd` | Manual dari GitHub binary |
| Patroni install | pip (normal) | pip `--break-system-packages` |
| SELinux | Tidak ada (AppArmor) | Ada, perlu konfigurasi khusus |
| Bootstrap method | `pg_createcluster` | Tidak ada, Patroni pakai `initdb` langsung |

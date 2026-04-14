# PostgreSQL

## Apa itu?

PostgreSQL adalah relational database management system (RDBMS) open source yang sudah ada sejak 1996. Di sini menggunakan versi 16, yang diinstall dari repo resmi PGDG (PostgreSQL Global Development Group) — bukan dari repo bawaan Rocky Linux yang versinya lebih lama.

## Kenapa dipakai di project ini?

PostgreSQL adalah database yang akan diproteksi oleh setup HA ini. Semua data aplikasi Hasura disimpan di PostgreSQL. Tujuan project ini adalah memastikan PostgreSQL tetap available (tidak downtime) meski ada node yang mati.

## Cara kerjanya dalam konteks project ini

Di setup ini, **PostgreSQL tidak dijalankan langsung via systemd**. Patroni yang mengontrol lifecycle-nya — kapan start, kapan stop, kapan promote jadi Primary. Karena itu service `postgresql-16` di systemd sengaja di-disable:

```bash
sudo systemctl disable postgresql-16
```

Patroni menjalankan PostgreSQL dengan parameter yang sesuai dengan role node saat itu (Primary atau Replica).

### Streaming Replication

Primary terus-menerus mengirim WAL (Write-Ahead Log) ke Replica secara real-time. Replica menerapkan WAL tersebut sehingga datanya selalu sync dengan Primary.

```
Primary                    Replica
   │   WAL stream            │
   │ ──────────────────────► │
   │                         │
   │  (setiap ada perubahan) │
```

Status sync bisa dicek dengan:
```bash
patronictl -c /etc/patroni/16-main.yml list
# Kolom "Receive LSN" dan "Replay LSN" menunjukkan posisi sync Replica
```

### Port dan Koneksi

| Port | Keterangan |
|---|---|
| 5432 | Port PostgreSQL default. Koneksi langsung ke node. |
| 5000 | Port HAProxy untuk Primary (via floating IP) |
| 5001 | Port HAProxy untuk Replica (via floating IP) |

Dalam production, aplikasi selalu konek ke port HAProxy (5000/5001) — bukan langsung ke 5432.

## Komponen penting yang perlu dipahami

**pg_hba.conf** — File yang menentukan siapa boleh konek ke PostgreSQL. Di setup Patroni, konfigurasi ini dikelola di `patroni.yml` bagian `pg_hba`, bukan diedit langsung.

**pg_rewind** — Tool untuk "rewind" bekas Primary agar bisa sync dengan Primary baru setelah failover. Diaktifkan via `use_pg_rewind: true` di konfigurasi Patroni.

**pg_basebackup** — Tool untuk clone data directory dari Primary ke Replica baru. Dipakai Patroni secara otomatis saat ada node baru bergabung.

**WAL (Write-Ahead Log)** — Semua perubahan data ditulis ke WAL dulu sebelum ke disk. Ini yang dikirim ke Replica untuk replikasi.

**pg_is_in_recovery()** — Fungsi SQL untuk cek apakah node sedang dalam mode recovery (Replica) atau tidak (Primary):
```sql
SELECT pg_is_in_recovery();
-- f = false = ini Primary
-- t = true  = ini Replica
```

## Hal yang saya pelajari

- Di Rocky Linux 9, **wajib** disable modul PostgreSQL bawaan sebelum install dari PGDG repo: `dnf module disable postgresql`. Kalau tidak, akan conflict versi.
- `postgresql16-devel` tidak bisa diinstall di Rocky 9 karena butuh `perl(IPC::Run)` yang tidak tersedia. Tapi ini tidak masalah karena tidak dibutuhkan untuk HA cluster.
- PostgreSQL punya dua "direktur": kalau dijalankan normal pakai systemd, kalau pakai Patroni maka Patroni yang jadi direkturnya. Keduanya tidak bisa jalan bersamaan.

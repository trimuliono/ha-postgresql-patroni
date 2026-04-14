# Patroni

## Apa itu?

Patroni adalah template/framework open source (ditulis Python) untuk membuat PostgreSQL cluster yang High Available. Patroni yang bertugas memutuskan siapa Primary, siapa Replica, dan apa yang terjadi kalau ada node mati.

Sederhananya: Patroni adalah "manajer" yang mengatur semua node PostgreSQL dalam cluster.

## Kenapa dipakai di project ini?

Tanpa Patroni, jika Primary mati, kita harus manual promote Replica jadi Primary baru — proses yang butuh waktu dan rawan human error. Patroni mengotomatiskan seluruh proses ini (election, promotion, reconfig) dalam hitungan detik.

## Cara kerjanya dalam konteks project ini

Patroni berjalan di ketiga node sebagai service systemd (`patroni@16-main`). Setiap instance Patroni:

1. **Berkomunikasi dengan etcd** — untuk membaca/menulis state cluster (siapa leader, timeline berapa)
2. **Mengelola PostgreSQL** — start, stop, promote, rewind sesuai role yang didapat
3. **Expose REST API di port 8008** — untuk health check oleh HAProxy

### Proses Election (saat Primary mati)

```
Primary (confluent-1) mati
         │
         ▼
etcd: leader key expired (TTL habis, default 10 detik)
         │
         ▼
Patroni di confluent-2 & confluent-3 mendeteksi tidak ada leader
         │
         ▼
Election: siapa yang punya LSN tertinggi? → confluent-2 menang
         │
         ▼
confluent-2: promote PostgreSQL → jadi Primary baru
confluent-3: reconfigure streaming replication → ikut Primary baru
         │
         ▼
HAProxy health check mendeteksi perubahan dalam ~6 detik
Traffic diarahkan ke confluent-2
```

### REST API Endpoint Penting

| Endpoint | HTTP Code | Artinya |
|---|---|---|
| `GET :8008/primary` | 200 | Node ini adalah Primary |
| `GET :8008/primary` | 503 | Node ini bukan Primary |
| `GET :8008/replica` | 200 | Node ini adalah Replica |
| `GET :8008/replica` | 503 | Node ini bukan Replica |
| `GET :8008/patroni` | 200 | Info lengkap state node |

HAProxy menggunakan endpoint ini untuk health check dan routing.

### Konfigurasi Kunci di `16-main.yml`

```yaml
# DCS settings — parameter failover
bootstrap:
  dcs:
    ttl: 10                        # Detik sebelum leader dianggap mati
    loop_wait: 10                  # Interval Patroni cek state
    retry_timeout: 10              # Timeout untuk operasi DCS
    maximum_lag_on_failover: 64MB  # Replica dengan lag > 64MB tidak eligible jadi Primary
```

```yaml
# PostgreSQL settings yang dikelola Patroni
postgresql:
  parameters:
    wal_level: replica             # WAL harus level replica untuk streaming replication
    hot_standby: "on"             # Replica bisa menerima read query
    max_wal_senders: 20           # Maks koneksi replication dari Primary
    max_replication_slots: 20     # Maks replication slots
```

### Perintah patronictl yang Sering Dipakai

```bash
# Lihat status cluster
patronictl -c /etc/patroni/16-main.yml list

# Failover manual (switchover) ke node tertentu
patronictl -c /etc/patroni/16-main.yml switchover \
  --master confluent-1 \
  --candidate confluent-2 \
  --force

# Restart PostgreSQL di satu node (via Patroni, bukan systemd)
patronictl -c /etc/patroni/16-main.yml restart 16-main confluent-1

# Reload konfigurasi (misal setelah edit pg_hba)
patronictl -c /etc/patroni/16-main.yml reload 16-main
```

## Komponen penting yang perlu dipahami

**scope** — Nama cluster di etcd. Semua node yang punya scope sama dianggap satu cluster. Di sini: `16-main`.

**use_pg_rewind** — Opsi yang memungkinkan bekas Primary di-"rewind" dan langsung bergabung sebagai Replica tanpa perlu full basebackup. Lebih cepat.

**tags** — Metadata per node. `nofailover: true` artinya node ini tidak akan dipilih jadi Primary meski Primary mati. Berguna untuk node yang hanya untuk backup/reporting.

**`@` di service name** — `patroni@16-main` berarti Patroni menggunakan systemd template service, di mana `16-main` adalah nama instance. Bisa menjalankan beberapa cluster Patroni di satu server dengan nama berbeda.

## Hal yang saya pelajari

- Di Rocky Linux 9, Patroni diinstall via pip dengan flag `--break-system-packages`. Tanpa flag ini pip3 menolak karena Rocky 9 menggunakan "externally managed" Python environment.
- Setelah install, binary `patroni` kadang tidak langsung ada di PATH. Perlu dicopy manual dari `~/.local/bin/` ke `/usr/local/bin/` agar bisa diakses service systemd yang berjalan sebagai user `postgres`.
- Patroni tidak pakai `pg_createcluster` (tool Ubuntu) — di Rocky langsung pakai `initdb` bawaan PostgreSQL.
- Jangan restart PostgreSQL langsung via systemd saat dalam cluster — selalu lewat `patronictl restart` agar Patroni tetap aware.

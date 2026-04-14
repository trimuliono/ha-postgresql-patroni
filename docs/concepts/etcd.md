# etcd

## Apa itu?

etcd adalah distributed key-value store — database sederhana yang menyimpan data dalam format `key = value`, tapi didesain untuk berjalan di banyak node sekaligus dan tetap konsisten meski ada node yang mati.

Analogi mudah: etcd itu seperti "papan pengumuman bersama" yang bisa dibaca dan ditulis oleh semua node, dan selalu menampilkan informasi yang sama di semua node.

## Kenapa dipakai di project ini?

Patroni butuh tempat menyimpan state cluster yang bisa diakses semua node secara konsisten — siapa yang jadi leader sekarang, di timeline berapa, konfigurasi apa yang berlaku. Tempat ini disebut DCS (Distributed Configuration Store), dan di setup ini menggunakan etcd.

Tanpa etcd, Patroni tidak bisa bekerja karena tidak ada "sumber kebenaran tunggal" tentang siapa Primary.

## Cara kerjanya dalam konteks project ini

etcd berjalan di ketiga node (confluent-1, 2, 3) dan membentuk cluster sendiri. Cluster etcd ini menggunakan **Raft consensus algorithm** — sebuah cara untuk memastikan semua node sepakat tentang satu nilai meskipun beberapa node tidak merespons.

### Raft & Quorum

Dengan 3 node etcd, cluster bisa tetap berfungsi selama minimal **2 dari 3 node** hidup. Kalau hanya 1 yang hidup, cluster berhenti (tidak mau terima write) untuk mencegah split-brain.

```
3 node hidup → normal, quorum terpenuhi
2 node hidup → normal, quorum terpenuhi (1 node boleh mati)
1 node hidup → STOP, tidak ada quorum
```

### Apa yang Disimpan Patroni di etcd?

```
/postgresql-common/16-main/leader    → "confluent-1"
/postgresql-common/16-main/members/confluent-1  → {role: primary, state: running, ...}
/postgresql-common/16-main/members/confluent-2  → {role: replica, state: streaming, ...}
/postgresql-common/16-main/config    → {ttl: 10, loop_wait: 10, ...}
```

Key `/leader` ini punya TTL (Time To Live). Patroni di node Primary harus terus memperbarui key ini. Kalau Primary mati dan tidak memperbarui key dalam TTL, key expired → election dimulai.

### Port etcd

| Port | Fungsi |
|---|---|
| 2379 | Client port — Patroni konek ke sini untuk baca/tulis |
| 2380 | Peer port — etcd antar node saling komunikasi di sini |

### Konfigurasi Per Node

Setiap node punya konfigurasi berbeda hanya di bagian IP:

```bash
ETCD_NAME="confluent-1"                  # nama unik per node
ETCD_LISTEN_PEER_URLS="http://10.100.13.241:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.100.13.241:2379,http://127.0.0.1:2379"
ETCD_INITIAL_CLUSTER="confluent-1=http://10.100.13.241:2380,
                       confluent-2=http://10.100.13.242:2380,
                       confluent-3=http://10.100.13.243:2380"
```

### Verifikasi Health

```bash
# Cek semua member
etcdctl member list

# Cek health semua endpoint
etcdctl endpoint health

# Lihat semua key yang tersimpan (untuk debug)
etcdctl get / --prefix --keys-only
```

## Komponen penting yang perlu dipahami

**ETCD_INITIAL_CLUSTER_STATE="new"** — Hanya dipakai saat pertama kali cluster dibentuk. Jika node mati dan hidup lagi, state ini sudah tidak relevan (etcd ingat state dari data directory-nya).

**ETCD_HEARTBEAT_INTERVAL** — Seberapa sering leader etcd mengirim heartbeat ke follower. Default: 100ms.

**ETCD_ELECTION_TIMEOUT** — Berapa lama follower menunggu heartbeat sebelum memulai election baru. Default: 1000ms (10x heartbeat interval).

**ETCD_AUTO_COMPACTION_RETENTION** — etcd menyimpan history semua perubahan. Compaction membersihkan history lama agar tidak habis disk. Di sini: 24 (jam).

## Hal yang saya pelajari

- Di Rocky Linux 9, `dnf install etcd` **tidak akan berhasil** karena package etcd tidak ada di repo default. Harus download binary langsung dari GitHub release page.
- etcd di setup ini berjalan sebagai user `postgres` (bukan user terpisah) untuk menyederhanakan permission management.
- etcd butuh semua node running sebelum cluster terbentuk. Kalau hanya 1 node yang distart, etcd akan "nunggu" quorum dan Patroni tidak bisa connect.
- Format konfigurasi etcd di Rocky menggunakan `KEY=VALUE` (EnvironmentFile systemd), bukan YAML seperti di beberapa tutorial Ubuntu.

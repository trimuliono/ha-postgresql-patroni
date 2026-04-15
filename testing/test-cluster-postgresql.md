# Testing Cluster PostgreSQL


**Environment:** confluent-1, confluent-2, confluent-3  
**Status:** ✅ PASSED

Tujuan: Verifikasi bahwa cluster PostgreSQL berdiri dengan benar dalam kondisi normal (semua node hidup).

---

## Test 8.1 — Status Cluster Patroni

**Perintah:**
```bash
patronictl -c /etc/patroni/16-main.yml list
```

**Output:**
```
+ Cluster: 16-main (7625919654464255739) -----------+----+-------------+-----+------------+-----+
| Member      | Host          | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+-------------+---------------+---------+-----------+----+-------------+-----+------------+-----+
| confluent-1 | 10.100.13.241 | Leader  | running   |  1 |             |     |            |     |
| confluent-2 | 10.100.13.242 | Replica | streaming |  1 |   0/504E298 |   0 |  0/504E298 |   0 |
| confluent-3 | 10.100.13.243 | Replica | streaming |  1 |   0/504E298 |   0 |  0/504E298 |   0 |
+-------------+---------------+---------+-----------+----+-------------+-----+------------+-----+
```

**Hasil:** ✅ PASS
- confluent-1 sebagai Leader (Primary)
- confluent-2 dan confluent-3 sebagai Replica dengan status `streaming`
- Lag = 0 di kedua Replica (fully sync)

**Catatan:** Status `streaming` (bukan `running`) di Replica adalah normal — menandakan streaming replication aktif berjalan.

---

## Test 8.3 — Koneksi via Floating IP

**Perintah:**
```bash
# Test primary (port 5000)
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5000 user=postgres sslmode=disable" \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"

# Test replica (port 5001)
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5001 user=postgres sslmode=disable" \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"
```

**Output port 5001 (Replica):**
```
 inet_server_addr | pg_is_in_recovery
------------------+-------------------
 10.100.13.243    | t
(1 row)
```

**Hasil:** ✅ PASS
- Koneksi via VIP berhasil
- Port 5001 dirouting ke confluent-3 (10.100.13.243)
- `pg_is_in_recovery = t` — benar, ini Replica

**Catatan masalah:** Saat test port 5000, terjadi kesalahan input — tanda kutip tidak tertutup sehingga psql masuk ke prompt interaktif. Test tetap berhasil setelah command diperbaiki.

---

## Test 8.4 — Write ke Primary

**Perintah:**
```bash
PGPASSWORD=YourSuperuserPassword psql \
  "host=10.100.13.248 port=5000 user=postgres sslmode=disable"

postgres=# CREATE DATABASE testdb;
testdb=# CREATE TABLE test (id serial, nama text, waktu timestamp default now());
testdb=# INSERT INTO test (nama) VALUES ('test-1'), ('test-2'), ('test-3');
testdb=# INSERT INTO test (nama) VALUES ('test-1'), ('test-2'), ('test-3');
testdb=# SELECT * FROM test;
```

**Output:**
```
 id |  nama  |           waktu
----+--------+----------------------------
  1 | test-1 | 2026-04-13 23:17:30.320142
  2 | test-2 | 2026-04-13 23:17:30.320142
  3 | test-3 | 2026-04-13 23:17:30.320142
  4 | test-1 | 2026-04-13 23:18:00.631852
  5 | test-2 | 2026-04-13 23:18:00.631852
  6 | test-3 | 2026-04-13 23:18:00.631852
(6 rows)
```

**Hasil:** ✅ PASS — Write ke Primary via VIP berhasil.

---

## Test 8.5 — Read dari Replica (Verifikasi Replikasi)

**Perintah:**
```bash
PGPASSWORD=YourSuperuserPassword psql \
  -h 10.100.13.248 -p 5001 -U postgres -d testdb

testdb=# SELECT * FROM test;
```

**Output:**
```
 id |  nama  |           waktu
----+--------+----------------------------
  1 | test-1 | 2026-04-13 23:17:30.320142
  2 | test-2 | 2026-04-13 23:17:30.320142
  3 | test-3 | 2026-04-13 23:17:30.320142
  4 | test-1 | 2026-04-13 23:18:00.631852
  5 | test-2 | 2026-04-13 23:18:00.631852
  6 | test-3 | 2026-04-13 23:18:00.631852
(6 rows)
```

**Hasil:** ✅ PASS — Data yang ditulis ke Primary berhasil tereplikasi ke Replica (confluent-3).

**Catatan masalah:** Saat pertama mencoba dengan `-d testdb` di akhir connection string format DSN, muncul error:
```
FATAL: Peer authentication failed for user "host=10.100.13.248 port=5001 user=postgres sslmode=disable"
```
Psql salah parsing — seluruh string DSN dianggap sebagai nama user. Solusi: gunakan flag `-h`, `-p`, `-U`, `-d` secara eksplisit.

---

## Ringkasan

| Test | Status | Catatan |
|---|---|---|
| 8.1 Cek cluster Patroni | ✅ PASS | 3 node running, Lag = 0 |
| 8.3 Koneksi via Floating IP | ✅ PASS | Primary & Replica routing benar |
| 8.4 Write ke Primary | ✅ PASS | CREATE DB, CREATE TABLE, INSERT berhasil |
| 8.5 Read dari Replica | ✅ PASS | Replikasi berjalan, data sync |

**Kesimpulan:** Cluster PostgreSQL dalam kondisi sehat. Siap untuk Testing HA.

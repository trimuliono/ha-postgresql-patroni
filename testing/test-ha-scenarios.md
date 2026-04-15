# Testing HA — Skenario Failover PostgreSQL


**Environment:** confluent-1, confluent-2, confluent-3  
**Status:** 🔄 PLANNED

Tujuan: Verifikasi bahwa cluster PostgreSQL tetap available saat ada node yang mati atau terjadi failover.

---

## Pre-condition Sebelum Testing

Pastikan semua kondisi berikut terpenuhi sebelum mulai:

```bash
# 1. Semua service running
for svc in etcd patroni@16-main haproxy keepalived; do
  echo "=== $svc ===" && sudo systemctl is-active $svc
done

# 2. Cluster healthy
patronictl -c /etc/patroni/16-main.yml list
# Semua node harus running/streaming, Lag = 0

# 3. VIP ada di confluent-1
ip a show ens32 | grep 10.100.13.248

# 4. Koneksi via VIP berfungsi
psql -h 10.100.13.248 -p 5000 -U postgres -c "SELECT inet_server_addr();"
```

---

## Skenario 8.6 — Simulasi Primary Mati (Failover Otomatis)

**Tujuan:** Membuktikan Patroni otomatis memilih Primary baru saat Primary mati mendadak.

**Langkah:**

```bash
# 1. Catat kondisi awal
patronictl -c /etc/patroni/16-main.yml list

# 2. Stop Patroni di Primary (confluent-1)
# Lakukan DI NODE confluent-1:
sudo systemctl stop patroni@16-main

# 3. Dari node lain, pantau election (tunggu 30-60 detik)
watch -n 2 'patronictl -c /etc/patroni/16-main.yml list'

# 4. Test koneksi masih bisa via VIP
psql -h 10.100.13.248 -p 5000 -U postgres \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"

# 5. Hidupkan kembali confluent-1
# Lakukan DI NODE confluent-1:
sudo systemctl start patroni@16-main

# 6. Verifikasi confluent-1 join sebagai Replica
patronictl -c /etc/patroni/16-main.yml list
```

**Expected Result:**
- confluent-2 atau confluent-3 terpilih jadi Leader baru
- Koneksi via VIP 10.100.13.248:5000 tetap bisa setelah ~30-60 detik
- Saat confluent-1 hidup kembali, otomatis jadi Replica
- Data tidak hilang

**Actual Result:** 

| Yang Diukur | Expected | Actual |
|---|---|---|
| Waktu election | 30-60 detik | - |
| Leader baru | confluent-2 atau confluent-3 | confluent-3 |
| Koneksi setelah failover | Berhasil | Berhasil |
| confluent-1 kembali | Jadi Replica | - |

<img width="1213" height="456" alt="image" src="https://github.com/user-attachments/assets/d78e2ce8-07f7-40e4-b4f5-2b17a2225cc2" />


---

## Skenario 8.7 — Switchover Manual

**Tujuan:** Memindahkan Primary secara terkontrol ke node tertentu.

**Langkah:**

```bash
# 1. Catat siapa Primary saat ini
patronictl -c /etc/patroni/16-main.yml list

# 2. Lakukan switchover ke confluent-2
patronictl -c /etc/patroni/16-main.yml switchover \
  --master confluent-1 \
  --candidate confluent-2 \
  --force

# 3. Verifikasi
patronictl -c /etc/patroni/16-main.yml list

# 4. Test koneksi
psql -h 10.100.13.248 -p 5000 -U postgres \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"
```

**Expected Result:**
- confluent-2 jadi Leader baru
- confluent-1 jadi Replica
- Koneksi via VIP tetap berfungsi
- Data di testdb masih ada

**Actual Result:** *(diisi setelah testing)*

---

## Skenario 8.8 — Keepalived Failover (VIP Berpindah)

**Tujuan:** Membuktikan Floating IP berpindah ke node lain saat node MASTER mati.

**Langkah:**

```bash
# 1. Cek VIP ada di node mana
ip a show ens32 | grep 10.100.13.248

# 2. Stop Keepalived di node yang pegang VIP (misal confluent-1)
sudo systemctl stop keepalived

# 3. Cek VIP berpindah ke confluent-2
# Lakukan DI NODE confluent-2:
ip a show ens32 | grep 10.100.13.248
# VIP harus muncul di sini

# 4. Test koneksi masih bisa via VIP
psql -h 10.100.13.248 -p 5000 -U postgres -c "SELECT 1;"

# 5. Hidupkan kembali Keepalived di confluent-1
sudo systemctl start keepalived
# VIP kembali ke confluent-1 (priority lebih tinggi)

# 6. Verifikasi VIP kembali ke confluent-1
ip a show ens32 | grep 10.100.13.248
```

**Expected Result:**
- VIP berpindah ke confluent-2 dalam < 3 detik setelah Keepalived di confluent-1 stop
- Koneksi via VIP tetap bisa setelah perpindahan
- VIP kembali ke confluent-1 saat Keepalived di-start lagi

**Actual Result:** *(diisi setelah testing)*

---

## Ringkasan Hasil Testing

| Skenario | Status | Waktu Failover | Catatan |
|---|---|---|---|
| 8.6 Primary mati (failover otomatis) | 🔄 | - | - |
| 8.7 Switchover manual | 🔄 | - | - |
| 8.8 Keepalived VIP failover | 🔄 | - | - |

---

## Catatan & Temuan

*(diisi setelah testing)*

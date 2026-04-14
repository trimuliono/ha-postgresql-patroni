# Runbook: Failover Manual (Switchover)

Panduan melakukan switchover Primary secara terkontrol — misalnya untuk maintenance node.

---

## Kapan Pakai Ini?

- Mau maintenance/reboot node Primary
- Mau test apakah failover berfungsi
- Mau pindah Primary ke node tertentu

Berbeda dengan **failover otomatis** (terjadi sendiri saat Primary mati mendadak), **switchover** adalah proses terkontrol yang bisa direncanakan.

---

## Langkah-langkah

### 1. Cek kondisi cluster sekarang

```bash
patronictl -c /etc/patroni/16-main.yml list
```

Catat:
- Siapa yang saat ini jadi Leader?
- Apakah semua Replica statusnya `streaming` dan Lag = 0?

Jangan lakukan switchover jika ada Replica dengan Lag besar.

### 2. Lakukan switchover

```bash
# Pindahkan Primary dari confluent-1 ke confluent-2
patronictl -c /etc/patroni/16-main.yml switchover \
  --master confluent-1 \
  --candidate confluent-2 \
  --force
```

Tanpa `--force`, Patroni akan meminta konfirmasi interaktif.

### 3. Verifikasi hasil switchover

```bash
# Cek siapa leader baru
patronictl -c /etc/patroni/16-main.yml list

# Cek koneksi masih bisa via floating IP
psql -h 10.100.13.248 -p 5000 -U postgres \
  -c "SELECT inet_server_addr(), pg_is_in_recovery();"
# inet_server_addr harus berubah ke IP confluent-2
# pg_is_in_recovery harus = f (false = Primary)
```

### 4. Verifikasi bekas Primary join sebagai Replica

```bash
patronictl -c /etc/patroni/16-main.yml list
# confluent-1 harus muncul sebagai Replica dengan state streaming
```

---

## Troubleshooting

**Switchover gagal karena Replica tertinggal jauh:**
```bash
# Cek lag
patronictl -c /etc/patroni/16-main.yml list
# Tunggu hingga Lag = 0 sebelum switchover
```

**Bekas Primary tidak kembali sebagai Replica:**
```bash
# Cek log Patroni di bekas Primary
sudo journalctl -u patroni@16-main -n 50 --no-pager

# Coba restart Patroni di bekas Primary
sudo systemctl restart patroni@16-main
```

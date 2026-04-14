# Runbook: Troubleshooting

Panduan diagnosis dan perbaikan masalah umum di cluster ini.

---

## Cek Cepat Semua Service

```bash
# Jalankan di semua node
for svc in etcd patroni@16-main haproxy keepalived; do
  echo "=== $svc ==="
  sudo systemctl is-active $svc
done
```

---

## Masalah 1: Patroni Gagal Start — Tidak Bisa Konek ke etcd

**Gejala:** `sudo systemctl status patroni@16-main` menunjukkan error

**Diagnosis:**
```bash
# Cek apakah etcd running
sudo systemctl status etcd

# Cek log Patroni
sudo journalctl -u patroni@16-main -n 50 --no-pager

# Test koneksi ke etcd manual
curl http://127.0.0.1:2379/health
```

**Perbaikan:**
```bash
# Pastikan etcd start dulu sebelum Patroni
sudo systemctl start etcd
sudo systemctl start patroni@16-main
```

---

## Masalah 2: psql via VIP Gagal — "server closed the connection unexpectedly"

**Gejala:** Koneksi ke `10.100.13.248:5000` gagal meski HAProxy stats menunjukkan backend UP

**Ini masalah SELinux.** HAProxy stats bisa UP (health check ke Patroni berhasil) tapi HAProxy tidak bisa forward koneksi ke PostgreSQL karena SELinux memblokir.

**Diagnosis:**
```bash
# Cek SELinux denial
sudo ausearch -m avc -ts recent | grep haproxy | grep "dest=5432"
# Jika ada output → ini penyebabnya
```

**Perbaikan:**
```bash
sudo setsebool -P haproxy_connect_any 1
sudo systemctl restart haproxy

# Verifikasi
getsebool haproxy_connect_any
# Output: haproxy_connect_any --> on
```

---

## Masalah 3: HAProxy Menandai Semua Backend DOWN

**Kemungkinan 1: HAProxy tidak bisa bind ke port 5000/5001/7000**
```bash
# Cek log
sudo journalctl -u haproxy -n 20 --no-pager

# Fix
sudo semanage port -a -t haproxy_port_t -p tcp 5000
sudo semanage port -a -t haproxy_port_t -p tcp 5001
sudo semanage port -a -t haproxy_port_t -p tcp 7000
sudo systemctl restart haproxy
```

**Kemungkinan 2: Patroni REST API tidak merespons**
```bash
# Test health check manual
curl -o /dev/null -w "%{http_code}" http://10.100.13.241:8008/primary
# Harus 200 untuk Primary

curl -o /dev/null -w "%{http_code}" http://10.100.13.242:8008/replica
# Harus 200 untuk Replica
```

---

## Masalah 4: Floating IP Tidak Berpindah Saat Node Mati

**Diagnosis:**
```bash
# Cek log Keepalived
sudo journalctl -u keepalived -n 50 --no-pager

# Pastikan VRRP tidak diblokir firewall
sudo firewall-cmd --list-protocols | grep vrrp

# Cek priority antar node
sudo grep priority /etc/keepalived/keepalived.conf
```

**Perbaikan:**
```bash
# Izinkan protokol VRRP
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload

sudo systemctl restart keepalived
```

---

## Masalah 5: Replica Tidak Sync (Lag Terus Bertambah)

**Diagnosis:**
```bash
patronictl -c /etc/patroni/16-main.yml list
# Lihat kolom Replay LSN Lag — jika terus bertambah ada masalah

# Cek log PostgreSQL di Replica
sudo -u postgres cat /data/postgresql/16/main/log/postgresql-16-main.log | tail -50
```

**Perbaikan:**
```bash
# Restart Patroni di Replica
sudo systemctl restart patroni@16-main

# Jika masih tidak sync, reinitialize Replica
patronictl -c /etc/patroni/16-main.yml reinit 16-main confluent-2
```

---

## Masalah 6: etcd Tidak Healthy

**Diagnosis:**
```bash
etcdctl endpoint health
etcdctl member list

# Cek log
sudo journalctl -u etcd -n 50 --no-pager
```

**Penting:** etcd butuh quorum (minimal 2 dari 3 node). Jika hanya 1 node yang running, etcd akan berhenti menerima write dan Patroni tidak bisa update state cluster.

---

## Urutan Start Service yang Benar (Setelah Reboot)

```bash
# 1. etcd dulu di semua node
sudo systemctl start etcd

# 2. Verifikasi etcd sehat
etcdctl endpoint health

# 3. Patroni
sudo systemctl start patroni@16-main

# 4. HAProxy
sudo systemctl start haproxy

# 5. Keepalived
sudo systemctl start keepalived
```

Jangan start Patroni sebelum etcd ready — Patroni tidak akan bisa connect ke DCS.

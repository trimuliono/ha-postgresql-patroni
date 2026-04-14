# HAProxy

## Apa itu?

HAProxy (High Availability Proxy) adalah load balancer dan proxy server open source yang sangat populer di production environment. HAProxy menerima koneksi dari client dan mendistribusikannya ke server backend yang sesuai berdasarkan aturan yang dikonfigurasi.

## Kenapa dipakai di project ini?

HAProxy berperan sebagai "pintu masuk cerdas" ke cluster PostgreSQL. Tanpa HAProxy, aplikasi harus tahu sendiri IP mana yang Primary dan IP mana yang Replica — dan harus update config setiap kali terjadi failover. Dengan HAProxy:

- Aplikasi cukup konek ke **satu port** (`5000` untuk write, `5001` untuk read)
- HAProxy yang otomatis tahu mana Primary dan mana Replica (via health check ke Patroni)
- Saat failover terjadi, HAProxy detect dalam beberapa detik dan reroute traffic

## Cara kerjanya dalam konteks project ini

### Arsitektur Dua Listener

```
Client
  │
  ├── port 5000 (Primary) → cek /primary di Patroni API
  │     └── forward ke node yang response HTTP 200
  │
  └── port 5001 (Replica) → cek /replica di Patroni API
        └── forward ke node yang response HTTP 200
              (balance: leastconn — pilih yang paling sedikit koneksi)
```

### Mekanisme Health Check

HAProxy tidak langsung query PostgreSQL untuk tahu siapa Primary. Setiap `2 detik`, HAProxy mengirim HTTP request ke **Patroni REST API** di port 8008:

```
HAProxy → GET http://10.100.13.241:8008/primary → HTTP 200 = UP (Primary)
HAProxy → GET http://10.100.13.242:8008/primary → HTTP 503 = DOWN (ini Replica, bukan Primary)
HAProxy → GET http://10.100.13.243:8008/primary → HTTP 503 = DOWN (ini Replica, bukan Primary)
```

Hanya node yang return HTTP 200 yang akan menerima traffic.

### Konfigurasi Key

```
# Primary listener (port 5000)
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
```

Parameter penting:
- `inter 2s` — health check setiap 2 detik
- `fall 2` — node dianggap DOWN setelah 2 kali gagal check berturut-turut
- `rise 3` — node dianggap UP lagi setelah 3 kali berhasil check berturut-turut
- `on-marked-down shutdown-sessions` — koneksi aktif langsung diputus saat node DOWN

### Stats Page

HAProxy menyediakan web interface untuk monitoring di port 7000:

```
http://10.100.13.248:7000/
```

Dari sini bisa dilihat status setiap backend (UP/DOWN), jumlah koneksi, traffic, dll.

## Komponen penting yang perlu dipahami

**Frontend vs Backend vs Listen** — HAProxy punya konsep frontend (menerima koneksi), backend (server tujuan), dan `listen` (gabungan keduanya). Di config ini menggunakan `listen` karena lebih ringkas untuk setup sederhana.

**mode tcp** — HAProxy beroperasi di layer 4 (TCP), bukan layer 7 (HTTP). Artinya HAProxy tidak membaca isi query PostgreSQL, hanya meneruskan koneksi TCP. Health check tetap HTTP, tapi data forwarding-nya TCP.

**balance leastconn** — Algoritma load balancing untuk Replica: arahkan koneksi baru ke server dengan koneksi aktif paling sedikit. Cocok untuk koneksi database yang long-lived.

**httpchk** — Fitur HAProxy untuk melakukan health check via HTTP meski backend adalah TCP server (PostgreSQL). Ini yang memungkinkan integrasi dengan Patroni REST API.

## Hal yang saya pelajari

### Masalah SELinux di Rocky Linux 9

Ini adalah bagian yang paling tricky. Di Rocky Linux 9 dengan SELinux enforcing, ada **dua masalah terpisah** yang harus diselesaikan:

**Masalah 1: HAProxy tidak bisa bind ke port 5000/5001/7000**
SELinux tidak mengizinkan HAProxy mendengarkan di port non-standar. Fix:
```bash
sudo semanage port -a -t haproxy_port_t -p tcp 5000
sudo semanage port -a -t haproxy_port_t -p tcp 5001
sudo semanage port -a -t haproxy_port_t -p tcp 7000
```

**Masalah 2: HAProxy tidak bisa connect ke PostgreSQL port 5432**
Meski stats page menunjukkan backend UP, koneksi psql tetap gagal. Ini karena SELinux memblokir HAProxy connect ke PostgreSQL. Fix:
```bash
sudo setsebool -P haproxy_connect_any 1
```

Tanpa fix kedua ini, koneksi psql akan gagal dengan error "server closed the connection unexpectedly" meski HAProxy stats terlihat normal. Ini membingungkan karena error-nya tidak langsung menunjuk ke SELinux.

### Perbedaan Syntax HAProxy 2.x vs 1.x

HAProxy 2.8+ (yang diinstall dari repo Rocky 9) tidak lagi mendukung syntax lama:
```
# LAMA (tidak valid di HAProxy 2.8+)
option httpchk GET /primary HTTP/1.0

# BARU (yang benar)
option httpchk
http-check connect port 8008
http-check send meth GET uri /primary
http-check expect status 200
```

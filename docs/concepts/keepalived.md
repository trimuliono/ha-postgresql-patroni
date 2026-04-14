# Keepalived

## Apa itu?

Keepalived adalah tool Linux untuk menyediakan **high availability** di level jaringan menggunakan protokol VRRP (Virtual Router Redundancy Protocol). Fungsi utamanya dalam konteks ini: mengelola **Floating IP** (Virtual IP) yang bisa berpindah antar server secara otomatis.

## Kenapa dipakai di project ini?

Bayangkan tanpa Keepalived:
- HAProxy berjalan di confluent-1 (IP: 10.100.13.241)
- Aplikasi konek ke `10.100.13.241:5000`
- Kalau confluent-1 mati → aplikasi tidak bisa konek karena IP-nya mati

Dengan Keepalived:
- Ada satu **Floating IP**: `10.100.13.248` yang bisa berpindah ke node mana saja
- Aplikasi selalu konek ke `10.100.13.248:5000`
- Kalau confluent-1 mati → IP `10.100.13.248` pindah ke confluent-2 secara otomatis
- Aplikasi tidak perlu ganti config sama sekali

## Cara kerjanya dalam konteks project ini

### VRRP — Mekanisme Pemilihan Pemegang VIP

Ketiga node saling bertukar VRRP advertisement setiap 1 detik. Node dengan **priority tertinggi** yang memegang VIP:

| Node | Priority | State Awal |
|---|---|---|
| confluent-1 | 101 | MASTER — pegang VIP |
| confluent-2 | 100 | BACKUP |
| confluent-3 | 99 | BACKUP |

Jika confluent-1 mati, confluent-2 tidak menerima advertisement lagi → confluent-2 naik jadi MASTER dan mengambil VIP `10.100.13.248`.

Ketika confluent-1 hidup kembali → karena priority-nya lebih tinggi (101 > 100), confluent-1 merebut kembali VIP (preempt).

### Health Check HAProxy

Keepalived tidak hanya cek apakah node masih hidup, tapi juga cek apakah **HAProxy masih berjalan** via script:

```bash
# /etc/keepalived/chk_haproxy.sh
#!/bin/bash
if systemctl is-active --quiet haproxy; then
    exit 0   # HAProxy OK → node eligible pegang VIP
else
    exit 1   # HAProxy mati → lepas VIP ke node lain
fi
```

Script ini dijalankan setiap 2 detik. Jika HAProxy mati tapi node masih hidup, Keepalived tetap akan melepas VIP ke node lain. Ini penting agar tidak ada situasi di mana VIP ada di node tapi HAProxy-nya tidak berfungsi.

### Konfigurasi Berbeda Per Node

Hanya 2 hal yang berbeda antar node: `state` dan `priority`.

```
# confluent-1 (MASTER)
state MASTER
priority 101

# confluent-2 (BACKUP)
state BACKUP
priority 100

# confluent-3 (BACKUP)
state BACKUP
priority 99
```

Sisanya (interface, virtual_router_id, auth_pass, virtual_ipaddress) **harus sama** di semua node agar VRRP bisa berfungsi.

### Verifikasi VIP

```bash
# Cek VIP ada di node mana
ip a show ens32 | grep 10.100.13.248

# Di MASTER, harus muncul:
# inet 10.100.13.248/24 scope global secondary ens32

# Ping VIP dari luar
ping 10.100.13.248 -c 3
```

## Komponen penting yang perlu dipahami

**virtual_router_id** — ID grup VRRP. Semua node dalam satu grup harus pakai ID yang sama (`51` di setup ini). Jika ada dua grup VRRP berbeda di jaringan yang sama, pastikan ID-nya berbeda.

**auth_pass** — Password VRRP untuk mencegah node asing bergabung ke grup. Harus sama di semua node dalam satu grup.

**advert_int 1** — Interval pengiriman VRRP advertisement (detik). Semakin kecil, semakin cepat failover terdeteksi, tapi lebih banyak network traffic.

**preempt (default behavior)** — Secara default, jika MASTER yang tadinya mati hidup kembali, ia akan merebut kembali VIP (karena priority lebih tinggi). Ini bisa menyebabkan "flip" VIP bolak-balik. Bisa dinonaktifkan dengan `nopreempt` jika tidak diinginkan.

**track_script** — Keepalived menjalankan script eksternal untuk menentukan apakah node masih layak pegang VIP. Jika script return exit code non-zero, node akan turun prioritynya atau melepas VIP.

## Hal yang saya pelajari

- Keepalived menggunakan protokol VRRP yang perlu diizinkan di firewall: `firewall-cmd --permanent --add-protocol=vrrp`. Tanpa ini, VRRP advertisement diblokir dan node-node tidak bisa saling "ngobrol".
- Interface harus disesuaikan dengan nama interface jaringan di server. Di environment ini: `ens32`. Di server lain bisa `eth0`, `enp0s3`, `ens192`, dll. Cek dengan `ip a`.
- Saat testing, bisa stop Keepalived di MASTER dan cek apakah VIP berpindah ke node lain dengan `ip a show ens32 | grep 10.100.13.248` dari node berbeda.
- Keepalived adalah komponen yang "tidak kelihatan" dari sisi aplikasi — tapi kalau tidak ada, saat failover aplikasi tetap mencoba konek ke IP lama yang sudah tidak aktif.

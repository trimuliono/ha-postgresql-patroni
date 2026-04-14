# Glosarium — Istilah HA Cluster

Kumpulan istilah yang sering muncul di setup ini, dijelaskan dalam bahasa yang mudah dipahami.

---

## A

**Authentication (pg_hba.conf)**
File konfigurasi PostgreSQL yang menentukan siapa boleh konek dari mana dan pakai metode apa. `md5` = pakai password, `peer` = pakai OS user.

---

## B

**Bootstrap**
Proses inisialisasi pertama kali cluster Patroni. Hanya terjadi sekali — saat node pertama (Primary) dijalankan untuk pertama kalinya. Patroni yang menjalankan `initdb` untuk membuat direktori data PostgreSQL.

---

## C

**Cluster**
Sekumpulan node (server) yang bekerja bersama dan terlihat sebagai satu unit dari sisi aplikasi. Di sini: 3 node PostgreSQL yang dikelola Patroni.

---

## D

**DCS (Distributed Configuration Store)**
Tempat terpusat yang dipakai Patroni untuk menyimpan state cluster — siapa leader, konfigurasi, timeline. Di setup ini menggunakan etcd.

---

## E

**Election**
Proses pemilihan leader baru ketika Primary mati. Patroni di node-node Replica akan "voting" siapa yang layak jadi Primary berikutnya, berdasarkan node mana yang punya data paling lengkap (LSN tertinggi).

---

## F

**Failover**
Proses otomatis perpindahan Primary ke Replica ketika Primary mati mendadak. Tidak perlu intervensi manual.

**Floating IP / VIP (Virtual IP)**
Alamat IP yang bisa berpindah antar node. Di setup ini: `10.100.13.248`. Dikelola oleh Keepalived. Client selalu konek ke IP ini sehingga tidak perlu tahu node mana yang sedang aktif.

---

## H

**HA (High Availability)**
Kemampuan sistem untuk tetap beroperasi meskipun ada komponen yang gagal. Biasanya diukur dalam uptime percentage (99.9%, 99.99%, dll).

**Health Check**
Proses HAProxy untuk mengecek apakah setiap backend (node PostgreSQL) masih hidup dan punya role yang benar. Dilakukan setiap beberapa detik via HTTP ke Patroni REST API.

---

## L

**Leader**
Istilah Patroni untuk node Primary PostgreSQL — node yang menerima write.

**LSN (Log Sequence Number)**
Nomor urut posisi di WAL (Write-Ahead Log). Dipakai untuk mengukur seberapa jauh Replica sudah sync dengan Primary. Replica dengan LSN tertinggi = paling up-to-date.

---

## P

**pg_basebackup**
Tool bawaan PostgreSQL untuk mengambil backup penuh dari Primary, dipakai saat Replica baru bergabung atau saat node lama perlu sync ulang setelah mati.

**pg_rewind**
Tool PostgreSQL untuk "rewind" data directory yang diverge — biasanya dipakai saat bekas Primary hidup kembali dan perlu sync dengan Primary baru. Lebih cepat dari pg_basebackup karena tidak perlu copy semua data.

**Primary**
Node PostgreSQL yang menerima semua operasi read dan write. Hanya ada 1 Primary di satu waktu.

---

## Q

**Quorum**
Jumlah minimum node yang harus hidup agar cluster bisa berfungsi. Untuk 3-node cluster: minimal 2 node harus hidup (majority). Jika hanya 1 dari 3 yang hidup, cluster tidak bisa memutuskan siapa leader (split-brain prevention).

---

## R

**Replica / Standby**
Node PostgreSQL yang menerima replikasi dari Primary. Bisa menerima query read-only. Ada 2 Replica di setup ini.

**Replication Slot**
Mekanisme PostgreSQL untuk memastikan Primary tidak membuang WAL yang masih dibutuhkan Replica. Mencegah Replica tertinggal terlalu jauh.

**REST API**
Interface HTTP yang disediakan Patroni untuk monitoring dan management cluster. Endpoint penting: `/primary`, `/replica`, `/patroni`, `/switchover`.

**VRRP (Virtual Router Redundancy Protocol)**
Protokol jaringan yang dipakai Keepalived untuk mengkoordinasikan siapa yang "memegang" floating IP. Node dengan priority tertinggi yang menang.

---

## S

**SELinux (Security-Enhanced Linux)**
Sistem keamanan di Rocky Linux yang mengontrol apa yang boleh dilakukan setiap proses. Di setup ini perlu konfigurasi khusus agar HAProxy bisa bind ke port non-standar dan konek ke PostgreSQL.

**Split-brain**
Kondisi berbahaya di mana 2 node sama-sama mengklaim dirinya Primary dan menerima write secara bersamaan — menyebabkan data inconsistency. etcd + Patroni mencegah ini dengan memastikan hanya 1 leader yang valid di satu waktu.

**Streaming Replication**
Mekanisme replikasi PostgreSQL di mana Primary mengirim WAL ke Replica secara real-time (streaming), bukan batch. Membuat Replica selalu up-to-date.

**Switchover**
Versi manual dari failover — memindahkan Primary ke Replica tertentu secara terkontrol, biasanya untuk maintenance. Berbeda dengan failover yang otomatis dan emergency.

---

## T

**Timeline**
Nomor yang bertambah setiap kali terjadi failover/switchover di PostgreSQL. Penting untuk pg_rewind agar bisa menentukan titik diverge antara bekas Primary dan Primary baru.

**TTL (Time To Live)**
Waktu maksimum Patroni leader key tersimpan di etcd. Jika leader tidak memperbarui key dalam TTL (karena mati), key expired dan election dimulai. Di setup ini: 10 detik.

---

## W

**WAL (Write-Ahead Log)**
Log semua perubahan data di PostgreSQL sebelum ditulis ke disk. Dipakai untuk recovery dan replikasi. Replica menerima WAL dari Primary dan menerapkannya.

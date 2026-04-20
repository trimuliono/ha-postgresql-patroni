# Testing HA — Hasura dengan PostgreSQL Cluster

**Environment:** Kubernetes (Hasura) + confluent-1/2/3 (PostgreSQL)  
**Status:** ✅ DONE

Tujuan: Verifikasi bahwa Hasura tetap bisa beroperasi saat terjadi failover di PostgreSQL cluster, menggunakan Apache JMeter sebagai load testing tool.

---

## Arsitektur Setup Testing

```
JMeter (Windows lokal)
    |
    v
https://hasura-chm-tri.alldataint.com  (Ingress K8s)
    |
    v
hasura-chm-tri (Gateway — namespace: tri)
    |
    +-- Remote Schema (internal DNS) -->  hasura-chm-schema-tri
                                              |
                                              v
                                         PostgreSQL Cluster
                                         VIP: 10.100.13.248:5000
                                              |
                                    +---------+---------+
                                    |         |         |
                               confluent-1  confluent-2  confluent-3
```

### Kubernetes Resources

| Resource | Name | Namespace |
|---|---|---|
| Deployment | hasura-chm-tri | tri |
| Deployment | hasura-chm-schema-tri | tri |
| Service | hasura-chm-tri | tri |
| Service | hasura-chm-schema-tri | tri |
| Ingress | hasura-chm-tri | tri |
| Ingress | hasura-chm-schema-tri | tri |

### Database

| Database | Dipakai Oleh |
|---|---|
| hasura_metadata_tri | Metadata hasura-chm-tri |
| hasura_metadata_schema_tri | Metadata hasura-chm-schema-tri |
| hasura_datasource | Data source (shared) |

---

## Pre-condition

```bash
# 1. PostgreSQL cluster healthy
patronictl -c /etc/patroni/16-main.yml list

# 2. Hasura running di Kubernetes
kubectl get pods -n tri

# 3. Hasura bisa query database
# Akses https://hasura-chm-tri.alldataint.com/console dan test query
```

---

## Setup JMeter

### Konfigurasi Test Plan

| Parameter | Value |
|---|---|
| Tool | Apache JMeter 5.6.3 |
| Number of Threads | 50 |
| Ramp-up Period | 10 detik |
| Loop Count | Infinite |
| Endpoint | POST https://hasura-chm-tri.alldataint.com/v1/graphql |

### Query GraphQL yang Ditest

```json
{"query": "query { orders { id order_number customer_name status total_amount } }"}
```

### Listeners yang Digunakan

- View Results Tree
- Summary Report
- Aggregate Graph
- Response Time Graph
- Aggregate Report

---

## Skenario 1 — Kondisi Normal (Baseline)

**Tujuan:** Mendapat data baseline sebelum simulasi failover.

**Hasil:**

| Metric | Value |
|---|---|
| Error % | 0.00% |
| Average Response Time | ~62ms |
| Min Response Time | 2ms |
| Max Response Time | ~1114ms |
| Throughput | ~65-83 req/sec |

**Status:** ✅ PASS — Hasura berjalan normal, tidak ada error.

---

## Skenario 2 — Satu Node PostgreSQL Mati (Node 2)

**Tujuan:** Verifikasi failover otomatis Patroni saat Primary mati, dan dampaknya ke Hasura.

**Langkah:**
```bash
# Di confluent-2
sudo systemctl stop patroni@16-main
```

**Hasil:**

| Yang Diukur | Hasil |
|---|---|
| Waktu election | ~30-60 detik |
| Leader baru | confluent-3 |
| Error saat failover | Ada (sementara, ~30 detik) |
| Recovery Hasura | Otomatis, tanpa restart manual |
| Error % setelah recover | 0.00% |

**Status:** ✅ PASS — Hasura recover otomatis setelah failover selesai.

---

## Skenario 3 — Dua Node PostgreSQL Mati

**Langkah:**
```bash
# Di confluent-1
sudo systemctl stop patroni@16-main
```

**Hasil:** Cluster tetap berjalan dengan confluent-3 sebagai satu-satunya Primary.

**Status:** ✅ PASS

---

## Skenario 4 — Semua Node PostgreSQL Mati

**Langkah:**
```bash
# Di confluent-3
sudo systemctl stop patroni@16-main
```

**Hasil:**
- Pod Hasura masuk `CrashLoopBackOff`
- JMeter error rate 100%
- Log error: `connection to server at "10.100.13.248", port 5000 failed`

**Cek status:**
```bash
kubectl get pods -n tri
kubectl logs -n tri <nama-pod> --previous
kubectl get events -n tri --sort-by='.lastTimestamp'
```

**Status:** ✅ PASS (expected)

---

## Skenario 5 — Recovery

**Langkah:**
```bash
# Di confluent-1, confluent-2, confluent-3
sudo systemctl start patroni@16-main

# Verifikasi cluster sehat kembali
patronictl -c /etc/patroni/16-main.yml list
```

**Hasil:**
- PostgreSQL cluster kembali normal
- Pod Hasura di K8s restart otomatis oleh ReplicaSet tanpa intervensi manual
- JMeter kembali menunjukkan error 0%

**Estimasi waktu recovery:**
- PostgreSQL cluster up: ~30-60 detik
- Pod Hasura ready: ~1-2 menit (karena initialDelaySeconds: 60)
- CrashLoopBackOff backoff: bisa sampai 5 menit jika pod sudah crash berkali-kali sebelumnya

**Status:** ✅ PASS

---
Utilization saat tidak ada transaksi:
<img width="1600" height="722" alt="image" src="https://github.com/user-attachments/assets/5685c0e1-f35f-4a44-997c-47af1e6693a3" />

Utilization saat load testing:
<img width="1600" height="854" alt="image" src="https://github.com/user-attachments/assets/8e4d9218-f9f4-40d4-a8b2-0c214aa2943b" />

Kondisi saat cluster postgresql down:
<img width="1600" height="670" alt="image" src="https://github.com/user-attachments/assets/d45e9698-1a84-4403-97c3-09b15b68eac8" />
<img width="1600" height="847" alt="image" src="https://github.com/user-attachments/assets/ebc56806-c9dc-4116-8a01-8d137f19542c" />

<img width="1297" height="739" alt="image" src="https://github.com/user-attachments/assets/926ad1c7-6243-4abf-8109-c0403d3615c0" />

<img width="1294" height="256" alt="image" src="https://github.com/user-attachments/assets/65924da7-10ef-410f-8425-5e40be69b0fa" />

Grafik transaksi:
<img width="935" height="455" alt="Response Time Graph - Testing CHM" src="https://github.com/user-attachments/assets/11a13d68-a4cd-41d8-82f5-4c7ba5619866" />

## Analisis Response Time Graph — Simulasi Failover PostgreSQL (Hasura Metadata)

Grafik di bawah merekam keseluruhan sesi testing dari **17:05 hingga 18:21** (~76 menit), mencakup seluruh siklus: ramp-up, cluster degradasi bertahap hingga down, lalu recovery.

<img width="752" height="576" alt="image" src="https://github.com/user-attachments/assets/bf6a6190-e5c1-4491-a12b-61dffbc6f571" />


### Pembagian Fase pada Grafik

#### Fase 1 — Idle / Ramp-up (17:05 – 17:23)

Latensi dimulai dari **0–5 ms** dan naik secara linear hingga ~20 ms. Belum ada load signifikan. Ini mencerminkan kondisi JMeter baru mulai mengirim request dan koneksi ke PostgreSQL belum fully loaded. Tidak ada error.

---

#### Fase 2 — Node Mulai Dimatikan Bertahap (17:23 – 17:49)

Tepat di **17:23**, latensi langsung melompat ke ~65–70 ms dan **stabil di level tersebut** sepanjang fase ini. Lonjakan awal ini terjadi karena:

- JMeter mencapai jumlah thread penuh (50 threads)
- Patroni mulai melakukan election saat node pertama dimatikan
- Hasura berhasil melakukan reconnect ke node baru secara otomatis

Selama fase ini, **cluster masih berfungsi** — Hasura mampu failover ke node yang tersisa sehingga latensi kembali stabil meskipun ada node yang mati. Ini membuktikan bahwa mekanisme HA di lapisan PostgreSQL (Patroni) dan Hasura bekerja dengan baik untuk skenario partial failure.

---

#### Fase 3 — Latency Spikes saat Tiap Node Dimatikan (17:35, 17:42, 17:49)

Terdapat **3 lonjakan latensi tajam** yang berkorelasi langsung dengan momen setiap node dimatikan:

| Waktu | Estimasi Latensi | Node yang Dimatikan | Keterangan |
|---|---|---|---|
| ~17:35 | ~215 ms | confluent-2 (node pertama) | Patroni re-election, Hasura reconnect |
| ~17:42 | ~155 ms | confluent-1 (node kedua) | Cluster tinggal 1 node, reconnect lebih cepat |
| ~17:49 | ~240 ms | confluent-3 (node terakhir) | Node terakhir mati, spike tertinggi sebelum total failure |

**Mengapa spike terjadi?**

Setiap kali satu node dimatikan, Patroni membutuhkan waktu beberapa detik untuk mendeteksi kegagalan dan memilih node baru sebagai primary. Selama proses election berlangsung, koneksi dari Hasura ke VIP (`10.100.13.248:5000`) tertahan — tidak ada node yang menjawab. Akibatnya, request yang masuk mengantri dan latensi melonjak. Begitu election selesai dan primary baru tersedia, Hasura langsung reconnect dan latensi kembali normal.

Spike di **17:49 (~240 ms) adalah yang tertinggi** karena ini adalah momen kritis: node terakhir dimatikan dan Hasura berusaha sekeras mungkin mencari endpoint metadata yang valid. Tidak ada node yang tersisa, sehingga semua koneksi akhirnya gagal.

---

#### Fase 4 — Cluster Fully Down / Total Failure (17:49 – 18:02)

Setelah spike tertinggi di 17:49, latensi pada grafik **drop mendekati 0 ms**. Ini bukan berarti request berjalan lebih cepat — justru sebaliknya. Latensi nol mengindikasikan bahwa **semua request langsung gagal** (connection refused / timeout) tanpa sempat diproses oleh backend.

Perilaku sistem selama fase ini:
- Pod Hasura masuk `CrashLoopBackOff` karena liveness probe gagal
- JMeter mencatat error rate **100%**
- Log menunjukkan: `connection to server at "10.100.13.248", port 5000 failed`
- Tidak ada transaksi yang berhasil selama ~13 menit

Durasi fase ini (~13 menit) mencerminkan waktu yang dibutuhkan untuk menjalankan prosedur recovery manual (start ulang semua node Patroni) ditambah waktu Kubernetes untuk menstabilkan pod Hasura kembali.

---

#### Fase 5 — Recovery (18:02 – 18:04)

Setelah semua node Patroni di-start kembali, latensi **naik kembali ke ~65 ms dalam waktu sekitar 2 menit**. Transisi ini terjadi cukup mulus:

1. PostgreSQL cluster kembali online (~30–60 detik setelah `systemctl start`)
2. Patroni memilih primary baru dan VIP kembali aktif
3. Pod Hasura di-restart otomatis oleh Kubernetes ReplicaSet
4. Hasura berhasil konek ke metadata DB dan mulai melayani request kembali

Waktu recovery **~2 menit** terhitung dari cluster up hingga Hasura kembali menerima traffic — ini merupakan angka yang baik untuk infrastruktur berbasis Kubernetes + Patroni.

> **Catatan:** Jika pod sudah mengalami `CrashLoopBackOff` berkali-kali sebelumnya, Kubernetes akan menerapkan exponential backoff (bisa sampai 5 menit). Untuk mempercepat recovery, jalankan `kubectl rollout restart deployment/hasura-chm-tri -n tri` segera setelah database kembali up.

---

#### Fase 6 — Stabil Kembali (18:04 – 18:21)

Latensi kembali stabil di **~65 ms** — identik dengan baseline di Fase 2. Sistem berjalan normal hingga akhir sesi testing. Terdapat satu spike kecil sekitar **~80 ms di 18:12**, kemungkinan disebabkan oleh query pertama pasca warmup cache atau minor GC event di sisi JVM Hasura.

---

### Ringkasan Metrik Grafik

| Metrik | Nilai |
|---|---|
| Latensi baseline normal | ~65 ms |
| Latensi tertinggi (spike) | ~240 ms (17:49, node terakhir mati) |
| Durasi downtime total | ~13 menit (17:49 – 18:02) |
| Waktu recovery setelah cluster up | ~2 menit |
| Jumlah spike latensi signifikan | 3 kali (satu per node yang dimatikan) |
| Fase stabil pasca-recovery | Normal, tidak ada degradasi residual |

---

### Interpretasi & Kesimpulan Grafik

**Sistem terbukti resilient terhadap kegagalan bertahap (partial failure).** Selama ada minimal satu node PostgreSQL yang tersisa, Hasura mampu melanjutkan operasi dengan latensi yang stabil setelah periode reconnect singkat. Pola spike-then-recover yang konsisten di 17:35 dan 17:42 mengkonfirmasi bahwa Patroni election dan Hasura reconnect logic keduanya berfungsi dengan benar.

**Sistem tidak toleran terhadap total cluster failure**, yang merupakan perilaku yang expected — PostgreSQL HA dengan Patroni memang dirancang untuk menahan kegagalan sebagian node, bukan seluruh cluster. Ketika semua node mati, tidak ada mekanisme yang bisa menyelamatkan koneksi.

**Recovery cepat dan otomatis** setelah cluster kembali online membuktikan bahwa konfigurasi Kubernetes (ReplicaSet, liveness/readiness probe, `initialDelaySeconds`) sudah tepat. Tidak diperlukan intervensi manual pada lapisan Hasura maupun Kubernetes.

---

## Ringkasan Hasil

| Skenario | Status | Error Saat Failover | Recovery | Perlu Restart Manual? |
|---|---|---|---|---|
| 1 — Kondisi Normal | ✅ PASS | 0% | — | Tidak |
| 2 — 1 Node Mati | ✅ PASS | ~30 detik | Otomatis | Tidak |
| 3 — 2 Node Mati | ✅ PASS | Minimal | Otomatis | Tidak |
| 4 — Semua Node Mati | ✅ PASS (expected) | 100% | — | Tidak |
| 5 — Recovery | ✅ PASS | 0% setelah recover | Otomatis | Tidak |

---

## Temuan & Catatan

### 1. `initialDelaySeconds` Wajib Diset

Tanpa ini, liveness probe akan membunuh container sebelum Hasura selesai listen di port 8080 → `exitCode 137` → `CrashLoopBackOff`. Nilai minimal yang direkomendasikan: **60 detik**.

### 2. Resource Limits Wajib Diset

Tanpa `resources.limits`, pod rentan di-kill karena `OOMKilled` (`exitCode 137`). Memory limit minimal **1Gi** untuk Hasura dengan beban 50 concurrent threads.

### 3. Metadata DB Harus Terpisah per Instance

Jika dua instance Hasura berbagi metadata DB yang sama → inconsistent metadata dan field conflict. Setiap instance harus memiliki database metadata-nya sendiri (`hasura_metadata_tri` dan `hasura_metadata_schema_tri`).

### 4. Remote Schema Gunakan Internal DNS

`http://hasura-chm-schema-tri.tri.svc.cluster.local:8080/v1/graphql` lebih efisien dan reliable dibanding menggunakan domain Ingress eksternal, karena tidak melewati lapisan load balancer dan TLS termination yang tidak perlu untuk komunikasi internal cluster.

### 5. CrashLoopBackOff Backoff Bisa Panjang

Semakin sering pod crash, semakin lama Kubernetes menunggu sebelum restart berikutnya (exponential backoff, bisa sampai 5 menit). Solusi yang direkomendasikan: jalankan `kubectl rollout restart` segera setelah database kembali up untuk mereset backoff counter.

```bash
kubectl rollout restart deployment/hasura-chm-tri -n tri
kubectl rollout restart deployment/hasura-chm-schema-tri -n tri
```

---

## Konfigurasi Optimal

```yaml
livenessProbe:
  initialDelaySeconds: 60
  timeoutSeconds: 10
  periodSeconds: 15
  failureThreshold: 3

readinessProbe:
  initialDelaySeconds: 60
  timeoutSeconds: 10
  periodSeconds: 15
  failureThreshold: 3

resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

---

*Dokumen ini merupakan hasil testing HA Hasura dengan PostgreSQL Cluster menggunakan Patroni sebagai HA manager. Seluruh skenario telah diverifikasi dan hasilnya sesuai dengan ekspektasi desain sistem.*

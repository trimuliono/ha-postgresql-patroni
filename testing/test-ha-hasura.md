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

## Ringkasan

| Skenario | Status | Error Saat Failover | Recovery | Perlu Restart Manual? |
|---|---|---|---|---|
| 1 — Kondisi Normal | ✅ PASS | 0% | - | Tidak |
| 2 — 1 Node Mati | ✅ PASS | ~30 detik | Otomatis | Tidak |
| 3 — 2 Node Mati | ✅ PASS | Minimal | Otomatis | Tidak |
| 4 — Semua Node Mati | ✅ PASS (expected) | 100% | - | Tidak |
| 5 — Recovery | ✅ PASS | 0% setelah recover | Otomatis | Tidak |

---

## Temuan & Catatan

1. **`initialDelaySeconds` wajib diset** — Tanpa ini, liveness probe kill container sebelum Hasura listen di port 8080 → exitCode 137 → CrashLoopBackOff.

2. **Resource limits wajib diset** — Tanpa `resources.limits`, pod di-kill OOMKilled (exitCode 137).

3. **Metadata DB harus terpisah** — Jika dua instance Hasura berbagi metadata DB yang sama → inconsistent metadata dan field conflict.

4. **Remote Schema pakai internal DNS** — `http://hasura-chm-schema-tri.tri.svc.cluster.local:8080/v1/graphql` lebih efisien daripada domain ingress.

5. **CrashLoopBackOff backoff** — Semakin sering crash, semakin lama backoff (bisa sampai 5 menit). Solusi: `kubectl rollout restart` setelah database kembali up.

### Konfigurasi Optimal

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

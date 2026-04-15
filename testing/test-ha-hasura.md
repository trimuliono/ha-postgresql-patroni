# Testing HA — Hasura dengan PostgreSQL Cluster


**Environment:** Kubernetes (Hasura) + confluent-1/2/3 (PostgreSQL)  
**Status:** 🔄 PLANNED

Tujuan: Verifikasi bahwa Hasura tetap bisa beroperasi saat terjadi failover di PostgreSQL cluster.

---

## Pre-condition

```bash
# 1. PostgreSQL cluster healthy
patronictl -c /etc/patroni/16-main.yml list

# 2. Hasura running di Kubernetes
kubectl get pods -n <namespace>

# 3. Hasura bisa query database
# Akses Hasura console dan test query sederhana
```

---

## Skenario 1 — Failover PostgreSQL saat Hasura Aktif

**Tujuan:** Apakah Hasura reconnect otomatis ke Primary baru setelah failover?

**Langkah:**
1. Kirim query berkelanjutan ke Hasura (gunakan tool seperti `k6` atau loop sederhana)
2. Stop Patroni di Primary
3. Amati response Hasura selama dan setelah failover
4. Catat berapa lama error terjadi sebelum Hasura recover

**Yang Diukur:**
- Apakah Hasura error? Error apa?
- Berapa lama downtime dari sisi Hasura?
- Apakah Hasura perlu di-restart atau reconnect sendiri?

**Expected Result:** Hasura mengalami error sementara (~30-60 detik sesuai waktu failover PostgreSQL) lalu recover otomatis tanpa perlu restart.

**Actual Result:** *(diisi setelah testing)*

---

## Skenario 2 — Verifikasi Metadata Hasura Setelah Failover

**Tujuan:** Apakah konfigurasi Hasura (schema, permissions, relationships) tetap intact setelah failover?

**Langkah:**
1. Lakukan failover PostgreSQL (dari skenario 1)
2. Setelah cluster recover, akses Hasura console
3. Verifikasi semua table, relationships, dan permissions masih ada
4. Test query GraphQL yang melibatkan relasi antar table

**Expected Result:** Semua metadata Hasura intact — tidak ada yang hilang atau corrupt.

**Actual Result:** *(diisi setelah testing)*

---

## Ringkasan

| Skenario | Status | Downtime Hasura | Perlu Restart? | Catatan |
|---|---|---|---|---|
| Failover saat aktif | 🔄 | - | - | - |
| Metadata integrity | 🔄 | - | - | - |

---

## Catatan & Temuan

*(diisi setelah testing)*

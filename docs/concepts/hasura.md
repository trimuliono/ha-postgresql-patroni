# Hasura

## Apa itu?

Hasura adalah GraphQL engine yang secara otomatis membuat GraphQL API dari database PostgreSQL. Cukup hubungkan Hasura ke database PostgreSQL, dan Hasura langsung generate endpoint GraphQL untuk semua table — tanpa perlu menulis resolver manual.

## Kenapa dipakai di project ini?

Hasura adalah aplikasi yang menggunakan PostgreSQL cluster ini sebagai backend database. Testing HA di project ini bukan hanya memastikan PostgreSQL cluster tetap berjalan saat ada failover, tapi juga memastikan **Hasura tetap bisa beroperasi** saat terjadi failover di PostgreSQL.

Hasura punya dua jenis koneksi ke PostgreSQL:
1. **Metadata database** — tempat Hasura menyimpan konfigurasinya sendiri (table definitions, relationships, permissions, dll)
2. **Data source** — database yang di-expose via GraphQL API

Di project ini, keduanya menggunakan cluster PostgreSQL yang sama (via floating IP).

## Cara kerjanya dalam konteks project ini

### Deployment di Kubernetes

Hasura di-deploy di Kubernetes cluster, bukan langsung di server PostgreSQL. Ini umum di production — Hasura stateless, mudah di-scale, dan dikelola oleh Kubernetes.

```
Kubernetes Cluster
    └── Hasura Deployment
         └── HASURA_GRAPHQL_DATABASE_URL=
              postgresql://postgres:password@10.100.13.248:5000/hasura_metadata
                                              ↑
                                        Floating IP (VIP)
                                        Port 5000 = Primary via HAProxy
```

### Yang Diuji di Testing HA (16 April)

| Skenario | Yang Diverifikasi |
|---|---|
| Primary mati saat Hasura aktif | Apakah Hasura reconnect otomatis ke Primary baru? |
| Switchover manual | Apakah ada error di Hasura selama switchover? |
| Node kembali hidup | Apakah Hasura tetap stabil? |
| Query aktif saat failover | Berapa lama query in-flight gagal? |

### Connection String ke Cluster

```bash
# Format connection string Hasura ke cluster PostgreSQL
postgresql://postgres:YourSuperuserPassword@10.100.13.248:5000/hasura_metadata

# Breakdown:
# 10.100.13.248 = Floating IP (bukan IP node spesifik)
# 5000          = Port HAProxy Primary
# hasura_metadata = nama database khusus metadata Hasura
```

### Contoh Kubernetes Deployment (basic)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hasura
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hasura
  template:
    spec:
      containers:
      - name: hasura
        image: hasura/graphql-engine:latest
        env:
        - name: HASURA_GRAPHQL_DATABASE_URL
          value: "postgresql://postgres:YourSuperuserPassword@10.100.13.248:5000/hasura_metadata"
        - name: HASURA_GRAPHQL_ENABLE_CONSOLE
          value: "true"
        ports:
        - containerPort: 8080
```

## Komponen penting yang perlu dipahami

**Metadata database** — Database khusus yang dipakai Hasura untuk menyimpan semua konfigurasi (schema, permissions, relationships, remote schemas, dll). Jika database ini corrupt atau tidak bisa diakses, Hasura tidak bisa start.

**GraphQL Engine** — Hasura menerjemahkan GraphQL query menjadi SQL query secara otomatis. Developer tidak perlu menulis SQL — cukup kirim GraphQL query.

**Stateless** — Hasura sendiri tidak menyimpan data — semua state ada di PostgreSQL. Ini membuatnya mudah di-restart atau di-scale tanpa kehilangan data.

**Connection pooling** — Hasura mengelola pool koneksi ke PostgreSQL. Saat failover terjadi, koneksi dalam pool perlu di-refresh. Waktu recovery bergantung pada konfigurasi pool dan timeout.

## Hal yang akan dipelajari (Testing 16 April)

- Berapa lama Hasura mengalami downtime saat failover PostgreSQL?
- Apakah Hasura perlu di-restart setelah failover, atau bisa reconnect sendiri?
- Bagaimana behavior Hasura saat ada query yang sedang berjalan ketika failover terjadi?
- Apakah metadata Hasura tetap intact setelah failover?

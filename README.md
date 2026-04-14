# HA PostgreSQL Cluster — Patroni + etcd + HAProxy + Keepalived

> **OS:** Rocky Linux 9  
> **Stack:** PostgreSQL 16, Patroni, etcd v3.5.17, HAProxy, Keepalived  

Repo ini mendokumentasikan proses setup, konfigurasi, dan testing High Availability (HA) PostgreSQL cluster menggunakan Patroni sebagai cluster manager, etcd sebagai distributed configuration store, HAProxy sebagai load balancer, dan Keepalived untuk floating IP.

---

## Topologi Cluster

```
                        ┌─────────────────────────┐
                        │   Floating IP (VIP)      │
                        │   10.100.13.248          │
                        └────────────┬────────────┘
                                     │ Keepalived VRRP
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
   ┌──────────┴──────────┐ ┌────────┴────────┐ ┌──────────┴──────────┐
   │    confluent-1       │ │   confluent-2   │ │    confluent-3       │
   │   10.100.13.241      │ │  10.100.13.242  │ │   10.100.13.243      │
   │                      │ │                 │ │                      │
   │  PostgreSQL 16       │ │  PostgreSQL 16  │ │  PostgreSQL 16       │
   │  Patroni             │ │  Patroni        │ │  Patroni             │
   │  etcd                │ │  etcd           │ │  etcd                │
   │  HAProxy             │ │  HAProxy        │ │  HAProxy             │
   │  Keepalived (MASTER) │ │  Keepalived     │ │  Keepalived          │
   └──────────────────────┘ └─────────────────┘ └──────────────────────┘
```

| Node | Hostname | IP | Role |
|---|---|---|---|
| Node 1 | confluent-1 | 10.100.13.241 | Primary (awal) |
| Node 2 | confluent-2 | 10.100.13.242 | Replica |
| Node 3 | confluent-3 | 10.100.13.243 | Replica |
| VIP | - | 10.100.13.248 | Floating IP (Keepalived) |

---

## Port yang Digunakan

| Port | Service | Fungsi |
|---|---|---|
| 5432 | PostgreSQL | Koneksi database langsung |
| 8008 | Patroni REST API | Health check endpoint |
| 2379 | etcd client | DCS client communication |
| 2380 | etcd peer | DCS peer communication |
| 5000 | HAProxy | Primary — read/write |
| 5001 | HAProxy | Replica — read only |
| 7000 | HAProxy Stats | Monitoring page |

---

## Cara Konek ke Cluster

```bash
# Konek ke primary (read/write) via floating IP
psql -h 10.100.13.248 -p 5000 -U postgres

# Konek ke replica (read only) via floating IP
psql -h 10.100.13.248 -p 5001 -U postgres

# Cek status cluster
patronictl -c /etc/patroni/16-main.yml list

# HAProxy stats
curl http://10.100.13.248:7000/
```

---

## Struktur Repo

```
ha-postgresql-patroni/
│
├── README.md                          # File ini
│
├── docs/
│   ├── architecture.md                # Penjelasan arsitektur & keputusan desain
│   ├── glossary.md                    # Kamus istilah HA
│   │
│   ├── concepts/                      # Penjelasan tiap teknologi
│   │   ├── postgresql.md
│   │   ├── patroni.md
│   │   ├── etcd.md
│   │   ├── haproxy.md
│   │   ├── keepalived.md
│   │   └── hasura.md
│   │
│   └── runbooks/                      # Panduan operasional
│       ├── failover-manual.md
│       └── troubleshooting.md
│
├── setup/
│   └── ha-postgresql-patroni-rocky9.md   # Dokumentasi instalasi lengkap
│
└── testing/
    ├── test-cluster-postgresql.md     # Hasil testing cluster
    ├── test-ha-scenarios.md           # Skenario & hasil testing HA
    └── test-ha-hasura.md              # Testing HA dari sisi Hasura

```

---

## Referensi

| Dokumentasi | URL |
|---|---|
| Patroni | https://patroni.readthedocs.io |
| etcd | https://etcd.io/docs |
| HAProxy | https://www.haproxy.org |
| PostgreSQL 16 | https://www.postgresql.org/docs/16/ |
| Rocky Linux 9 | https://docs.rockylinux.org |

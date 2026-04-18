# Kubernetes Manifests — Hasura HA Testing

Folder ini berisi manifest Kubernetes untuk deploy Hasura yang digunakan dalam testing HA PostgreSQL cluster.

## Struktur

```
kubernetes/
├── README.md
├── hasura-chm-tri/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── hasura-chm-schema-tri/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

## Prerequisites

1. Buat database metadata terlebih dahulu:

```bash
PGPASSWORD=YourSuperuserPassword psql \
  -h 10.100.13.248 -U postgres -p 5000 \
  -c "CREATE DATABASE hasura_chm_metadata_k8s ;"

PGPASSWORD=YourSuperuserPassword psql \
  -h 10.100.13.248 -U postgres -p 5000 \
  -c "CREATE DATABASE hasura_chm_schema_metadata_k8s ;"
```

2. Buat namespace:

```bash
kubectl create namespace tri
```

3. Tambah /etc/hosts di client:

```
10.100.14.3  hasura-chm-tri.alldataint.com
10.100.14.3  hasura-chm-schema-tri.alldataint.com
```

## Deploy

```bash
# Deploy hasura-chm-schema-tri dulu (karena ini source-nya)
kubectl apply -f hasura-chm-schema-tri/

# Deploy hasura-chm-tri (gateway)
kubectl apply -f hasura-chm-tri/

# Cek status
kubectl get pods,svc,ingress -n tri
```

## Setup Remote Schema

Setelah kedua pod running, setup remote schema di console hasura-chm-tri:

1. Buka https://hasura-chm-tri.alldataint.com/console
2. Remote Schemas → Add
3. URL: `http://hasura-chm-schema-tri.tri.svc.cluster.local:8080/v1/graphql`
4. Root Fields Namespace: `chmGraphql`
5. Type Name Prefix: `chmGraphql`

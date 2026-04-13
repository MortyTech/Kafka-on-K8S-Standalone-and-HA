# Kafka on Kubernetes: Standalone & HA Deployment Guide

This guide explains how to deploy Apache Kafka (KRaft mode) on Kubernetes using the provided manifests – one for a **single‑node (standalone)** setup and one for a **three‑node high‑availability (HA)** cluster.

Both manifests use **Longhorn** for persistent storage and are configured for **KRaft** (no ZooKeeper).  

---

## Prerequisites

- A Kubernetes cluster (v1.19+)
- `kubectl` configured with cluster admin access
- **Longhorn** storage class installed (or replace `storageClassName` with your own)
- Two namespaces: `poc` and `prod` (will be created if missing)

---

## 1. Standalone Kafka (Namespace `poc`)

**File:** `kafka.yml`  
**Replicas:** 1  
**Use case:** Development, testing, or low‑traffic environments.

### Apply the manifest

```bash
# Create namespace if needed
kubectl create namespace poc --dry-run=client -o yaml | kubectl apply -f -

# Deploy Kafka
kubectl apply -f kafka.yml
```

### Verify

```bash
kubectl get all -n poc
kubectl exec -it -n poc kafka-0 -- /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### Configuration highlights

| Parameter          | Value                                      |
|--------------------|--------------------------------------------|
| Node ID            | 1 (fixed)                                  |
| Replication factor | 1 (topics, offsets, transactions)         |
| Min ISR            | 1                                          |
| Advertised listener| `kafka-0.kafka.poc.svc.cluster.local:9092`|

---

## 2. High‑Availability Kafka (Namespace `prod`)

**File:** `kafka-prod.yaml`  
**Replicas:** 3  
**Use case:** Production, fault‑tolerant messaging.

### Apply the manifest

```bash
# Create namespace if needed
kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -

# Deploy Kafka HA
kubectl apply -f kafka-prod.yaml
```

### Verify

```bash
# Check all three pods are running
kubectl get pods -n prod -l app=kafka

# Create a test topic with replication factor 3
kubectl exec -it -n prod kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --create --topic test --partitions 3 --replication-factor 3 \
  --bootstrap-server localhost:9092

# Describe the topic to see leader / ISR
kubectl exec -it -n prod kafka-0 -- /opt/kafka/bin/kafka-topics.sh \
  --describe --topic test --bootstrap-server localhost:9092
```

### Configuration highlights

| Parameter                          | Value                                                              |
|------------------------------------|--------------------------------------------------------------------|
| Node IDs                           | 1, 2, 3 (derived from pod index +1)                                |
| Controller quorum voters           | All three nodes (`1@kafka-0...:9093`, `2@kafka-1...`, `3@kafka-2...`) |
| Replication factor (topics, etc.)  | 3                                                                  |
| Min ISR                            | 2                                                                  |
| Advertised listener (dynamic)      | `${HOSTNAME}.kafka.${POD_NAMESPACE}.svc.cluster.local:9092`        |
| `fsGroup: 1000` in `securityContext` | Fixes data directory permissions automatically                  |
| Init container                     | Removes `lost+found` and ensures correct ownership                |

---

## Cleanup

Delete the deployments and associated PVCs (data will be lost):

```bash
# Standalone
kubectl delete -f kafka.yml

# HA
kubectl delete -f kafka-prod.yaml

# Optionally delete namespaces
kubectl delete namespace poc prod
```

---

## Important Notes

1. **KRaft mode** – both clusters run without ZooKeeper.  
2. **Storage class** – change `longhorn` to your default or preferred class (e.g., `standard`, `gp2`).  
3. **Node ID mapping** – the HA manifest uses `hostname` (e.g., `kafka-0` → node ID `1`). Do not change the number of replicas without adjusting the voter list.  
4. **Pod restart** – if a pod fails, Kubernetes will restart it. Persistent volumes keep the data.  
5. **External access** – the advertised listeners are only reachable from inside the cluster. For external clients, use a `LoadBalancer` or `NodePort` service.  

---

## Repository Structure

```
.
├── kafka.yml           # Standalone (1 node, namespace poc)
├── kafka-prod.yaml     # HA cluster (3 nodes, namespace prod)
└── README.md           # This guide
```

---

**Happy messaging!** 🚀

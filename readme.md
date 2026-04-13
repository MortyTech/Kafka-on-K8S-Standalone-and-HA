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

---

### 🩺 Initial Cluster Health Check (Are Nodes in Sync?)

Since you are using KRaft (without Zookeeper), metadata is distributed between the Controller and Brokers. To check the cluster status, access one of the Pods and run the following commands:

```bash
# Access the first Pod
kubectl exec -it kafka-0 -n prod -- /bin/bash

# Inside the Pod:
# 1. Check metadata status (Quorum)
/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status

# Output should look something like this and show that Leader and Followers are present:
# NodeId 1  ...  Leader
# NodeId 2  ...  Follower
# NodeId 3  ...  Follower

# 2. List active Brokers
/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
# Should display 3 Brokers with IDs 1,2,3.
```

If the output of these commands shows all 3 nodes, it means the cluster is in sync and its High Availability is guaranteed.

---

---

### ✅ High Availability (HA) Practical Test

Now that everything is stable, let's simulate a node failure to ensure the cluster survives.

#### 1. Create a Test Topic with Full Replication Factor

Open a terminal on `kafka-0` (or anywhere inside the cluster):

```bash
kubectl exec -it kafka-0 -n prod -- /bin/bash

# Create a Topic with 3 partitions and 3 replicas
/opt/kafka/bin/kafka-topics.sh --create \
  --topic ha-test \
  --partitions 3 \
  --replication-factor 3 \
  --bootstrap-server localhost:9092

# Check the distribution status (Replicas should be spread across 1,2,3)
/opt/kafka/bin/kafka-topics.sh --describe --topic ha-test --bootstrap-server localhost:9092
```

#### 2. Start a Consumer in Terminal 1 (Monitoring)

```bash
kubectl exec -it kafka-0 -n prod -- /bin/bash
/opt/kafka/bin/kafka-console-consumer.sh \
  --topic ha-test \
  --from-beginning \
  --bootstrap-server kafka.prod.svc.cluster.local:9092 \
  --group test-group-ha
```

#### 3. Start a Producer in Terminal 2 (Sending Messages)

```bash
kubectl exec -it kafka-0 -n prod -- /bin/bash

# A simple loop to send numbers
for i in $(seq 1 1000); do echo "Message $i"; sleep 1; done | \
/opt/kafka/bin/kafka-console-producer.sh \
  --topic ha-test \
  --bootstrap-server kafka.prod.svc.cluster.local:9092
```

#### 4. Simulate Node Failure (Terminal 3)

While the Producer is sending and the Consumer is receiving messages, delete the leader node (`kafka-1`):

```bash
kubectl delete pod kafka-1 -n prod
```

#### 5. Observe the Results

- **Consumer (Terminal 1):**  
  You may see a message like `Group coordinator ... is unavailable` or `Rebalancing`. After a few seconds (approximately 10-20 seconds), the Consumer resumes receiving messages **without losing a single message**.

- **Producer (Terminal 2):**  
  You may see a `TimeoutException` or `Broker not available` error. In a manual test with the console, message sending stops and you'll need to `Ctrl+C` and restart. However, in real libraries (like Java/Python), automatic **Retry** occurs and the Producer continues operating without errors.

- **New Quorum:**  
  After a few seconds, a new leader (e.g., `kafka-0` or `kafka-2`) is elected. You can verify this with the following command:

```bash
/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status
# Shows the new LeaderId
```

---

### 🖥️ Where Should Clients Connect?

#### From Inside Kubernetes (e.g., another Pod or service within the cluster)
The best and most standard address is the **Headless Service name**:
```
kafka.prod.svc.cluster.local:9092
```
Why? Because the Kafka library first fetches metadata using this address and then connects directly to the relevant brokers (using addresses like `kafka-0.kafka...:9092`).

#### From Outside Kubernetes (Developer Laptop)
The current StatefulSet does **not** have an `EXTERNAL` listener configured. To add external access, you must:
1. Create a **NodePort Service** for all three ports or use a **LoadBalancer**.
2. Alternatively, use `kubectl port-forward` for temporary testing:
   ```bash
   kubectl port-forward -n prod pod/kafka-0 9092:9092
   ```

### 📌 Additional Note on Sync Status (ISR)

In the `describe` output for the Topic, there is a column named **ISR (In-Sync Replicas)**. In a healthy cluster, this column should include all 3 brokers. When a node goes down, that node is removed from the ISR list, but `min.insync.replicas` set to `2` ensures that Producers with `acks=all` can still successfully write messages (since at least 2 replicas are alive).

---

### Current Status Summary

✅ 3-node KRaft cluster running successfully on `prod`.  
✅ Nodes are synchronized (`MaxFollowerLag=0`).  
✅ Current leader is `kafka-1`.  
✅ Ready for HA testing and client connections.

**Happy messaging!** 🚀

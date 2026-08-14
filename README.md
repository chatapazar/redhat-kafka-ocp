# Hands-On Lab: Red Hat Streams for Apache Kafka on OpenShift

> This lab walks through installing the Operator, deploying a Kafka cluster (KRaft mode), creating Topics/Users, producing/consuming real messages on OpenShift ad many other features.

---

## 📋 Table of Contents

- [Prerequisites](docs/1-prereq.md)
- [Part 1: Install the Streams for Apache Kafka Operator](#-part-1-install-the-streams-for-apache-kafka-operator)
- [Part 2: Deploy a Kafka Cluster](#-part-2-deploy-a-kafka-cluster)
- [Part 3: Verify Cluster Status](#-part-3-verify-cluster-status)
- [Part 4: Create a Topic](#-part-4-create-a-topic)
- [Part 5: Create a User and Authentication](#-part-5-create-a-user-and-authentication)
- [Part 6: Produce and Consume Messages](#-part-6-produce-and-consume-messages)
- [Part 7: Expose Kafka via External Route (Bonus)](#-part-7-expose-kafka-via-external-route-bonus)
- [Part 8: Cleanup](#-part-8-cleanup)
- [Troubleshooting](#-troubleshooting)
- [Reference](#-reference)

---

## 🚀 Part 1: Install the Streams for Apache Kafka Operator

### Step 1.1 — Create a Namespace

```bash
oc new-project kafka
```

### Step 1.2 — Install the Operator via OLM

```yaml
# 01-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: amq-streams
  namespace: kafka
spec:
  channel: stable
  name: amq-streams
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

```bash
oc apply -f 01-operator-subscription.yaml
```

### Step 1.3 — Wait for the Operator to Become Ready

```bash
# Wait until the operator pod status = Running
oc get pods -n kafka -w

# Press Ctrl+C once you see output like this:
# NAME                                        READY   STATUS    RESTARTS   AGE
# amq-streams-cluster-operator-xxxxxxxxx-xxxxx  1/1     Running   0          2m
```

Verify the CRDs were installed correctly:

```bash
oc get crd | grep kafka

# You should see at least these CRDs:
# kafkas.kafka.strimzi.io
# kafkatopics.kafka.strimzi.io
# kafkausers.kafka.strimzi.io
# kafkanodepools.kafka.strimzi.io
```

> ✅ **Checkpoint:** If the operator pod is `Running` and all CRDs are present, Part 1 is complete.

---

## 🏗️ Part 2: Deploy a Kafka Cluster

This lab uses **KRaft mode** (no ZooKeeper), which is the current standard approach.

### Step 2.1 — Create a KafkaNodePool for the Controller

```yaml
# 02-nodepool-controller.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: controller
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - controller
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1"
      memory: "2Gi"
```

### Step 2.2 — Create a KafkaNodePool for the Broker

```yaml
# 03-nodepool-broker.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: broker
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  replicas: 3
  roles:
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 20Gi
        deleteClaim: false
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1"
      memory: "2Gi"
```

### Step 2.3 — Create the Kafka Cluster

```yaml
# 04-kafka-cluster.yaml
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: my-cluster
  namespace: kafka
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
kind: Kafka
spec:
  kafka:
    version: 3.9.0
    metadataVersion: 3.9-IV0
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

### Step 2.4 — Apply Everything

```bash
oc apply -f 02-nodepool-controller.yaml
oc apply -f 03-nodepool-broker.yaml
oc apply -f 04-kafka-cluster.yaml
```

Full deployment takes roughly 2-5 minutes. Wait for it with:

```bash
oc wait kafka/my-cluster \
  --for=condition=Ready \
  --timeout=300s \
  -n kafka
```

---

## 🔍 Part 3: Verify Cluster Status

```bash
# List all pods
oc get pods -n kafka

# You should see:
# NAME                          READY   STATUS    RESTARTS   AGE
# my-cluster-broker-0           1/1     Running   0          3m
# my-cluster-broker-1           1/1     Running   0          3m
# my-cluster-broker-2           1/1     Running   0          3m
# my-cluster-controller-3       1/1     Running   0          3m
# my-cluster-controller-4       1/1     Running   0          3m
# my-cluster-controller-5       1/1     Running   0          3m
# my-cluster-entity-operator-xxxxxxxxx-xxxxx   2/2   Running   0   2m
```

```bash
# Check the Kafka CR status
oc get kafka my-cluster -n kafka

# View detailed conditions
oc describe kafka my-cluster -n kafka
```

```bash
# View the Services the Operator created
oc get svc -n kafka

# You should see the bootstrap service clients connect to:
# my-cluster-kafka-bootstrap   ClusterIP   ...   9092/TCP,9093/TCP
```

> ✅ **Checkpoint:** The Kafka CR status must show `Ready: True` before moving to the next part.

---

## 📨 Part 4: Create a Topic

```yaml
# 05-topic.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 604800000        # 7 days
    segment.bytes: 1073741824      # 1 GB
    min.insync.replicas: 2
```

```bash
oc apply -f 05-topic.yaml

# Verify
oc get kafkatopic -n kafka
oc describe kafkatopic my-topic -n kafka
```

---

## 🔐 Part 5: Create a User and Authentication

### Step 5.1 — Create a KafkaUser with SCRAM-SHA-512 Authentication

```yaml
# 06-user.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Read
          - Write
          - Describe
      - resource:
          type: group
          name: my-consumer-group
          patternType: literal
        operations:
          - Read
```

> ⚠️ **Note:** The `simple` authorization type also requires `authorization: type: simple` to be enabled on the Kafka CR itself (`spec.kafka.authorization`) for ACLs to actually be enforced — this lab focuses on teaching the user-creation syntax first.

```bash
oc apply -f 06-user.yaml

# Verify the User Operator automatically created a Secret
oc get kafkauser my-user -n kafka
oc get secret my-user -n kafka
```

### Step 5.2 — View the Generated Password

```bash
oc get secret my-user -n kafka \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## 💬 Part 6: Produce and Consume Messages

### Step 6.1 — Deploy a Test Client Pod

```bash
oc run kafka-client \
  --image=registry.redhat.io/amq-streams/kafka-39-rhel9:latest \
  --restart=Never \
  -n kafka \
  -- sleep infinity
```

Wait for the pod to become ready:

```bash
oc wait pod/kafka-client --for=condition=Ready -n kafka --timeout=60s
```

### Step 6.2 — Produce a Message (plain listener, no auth)

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-console-producer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic my-topic
```

Type a few test messages and press Enter after each (press Ctrl+D to exit):

```
Hello Kafka!
Test message 2
Test message 3
```

### Step 6.3 — Consume the Messages

Open a new terminal:

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-console-consumer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic my-topic \
  --from-beginning
```

You should see the messages you produced earlier appear.

> ✅ **Checkpoint:** If you can consume all the messages you produced, the cluster is working correctly end-to-end.

### Step 6.4 — Inspect Topic Details

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-topics.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe \
  --topic my-topic
```

### Step 6.5 — Inspect the Consumer Group

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --list

oc exec -it kafka-client -n kafka -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe \
  --group console-consumer-xxxxx   # replace with the group name from --list
```

---

## 🌐 Part 7: Expose Kafka via External Route (Bonus)

If you need clients outside the OpenShift cluster to connect:

```yaml
# 07-kafka-external-listener.yaml
# Add this listener into spec.kafka.listeners on the existing Kafka CR
- name: external
  port: 9094
  type: route
  tls: true
  authentication:
    type: scram-sha-512
```

```bash
# Edit the Kafka CR and apply
oc edit kafka my-cluster -n kafka

# View the Routes the Operator created (one route per broker)
oc get routes -n kafka
```

---

## 🧹 Part 8: Cleanup

```bash
# Delete all resources in the kafka namespace
oc delete kafkatopic --all -n kafka
oc delete kafkauser --all -n kafka
oc delete kafka --all -n kafka
oc delete kafkanodepool --all -n kafka
oc delete pod kafka-client -n kafka

# Remove the operator subscription too, if you want to uninstall the operator
oc delete subscription amq-streams -n kafka

# Or delete the entire namespace to remove everything at once
oc delete project kafka
```

---

## 🛠️ Troubleshooting

| Symptom | Possible Cause | Fix |
|---|---|---|
| Operator pod won't start | Wrong subscription channel, or catalog source not ready | `oc get subscription -n kafka -o yaml` and check status conditions |
| Kafka pod stuck `Pending` | Storage class not supported, or insufficient resources | `oc describe pod <pod-name> -n kafka` and check Events |
| Kafka CR never `Ready` | Broker still syncing, or misconfiguration | `oc describe kafka my-cluster -n kafka` and check conditions |
| `kafka-console-producer` can't connect | Wrong bootstrap server address, or listener config mismatch | `oc get svc -n kafka` and confirm the service name |
| Topic creation fails | `strimzi.io/cluster` label doesn't match the Kafka cluster name | Verify the label matches the Kafka CR's `metadata.name` |

### Useful Debug Commands

```bash
# View Cluster Operator logs
oc logs -f deployment/amq-streams-cluster-operator -n kafka

# View a specific broker's logs
oc logs -f my-cluster-broker-0 -n kafka

# View all events in the namespace
oc get events -n kafka --sort-by='.lastTimestamp'
```

---

## 📚 Reference

- [Red Hat Streams for Apache Kafka Documentation](https://access.redhat.com/documentation/en-us/red_hat_streams_for_apache_kafka)
- [Strimzi Documentation (Upstream)](https://strimzi.io/documentation/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

## 🎯 What You Learned in This Lab

- [x] Installed the Streams for Apache Kafka Operator via OLM
- [x] Deployed a Kafka Cluster in KRaft mode (no ZooKeeper)
- [x] Separated Controller and Broker roles using KafkaNodePool
- [x] Created and managed a Topic via the KafkaTopic CR
- [x] Created a User with authentication and ACLs via the KafkaUser CR
- [x] Produced and consumed messages via a CLI client pod
- [x] Exposed Kafka outside the cluster via a Route

---

*This lab was tested on OpenShift 4.16+ with Streams for Apache Kafka (`stable` channel) — operator/Kafka versions may change over time. Check the [compatibility matrix](https://access.redhat.com/articles/streams-supported-configurations) before using this in production.*

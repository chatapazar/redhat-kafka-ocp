# Hands-On Lab: Red Hat Streams for Apache Kafka บน OpenShift

> Lab นี้สอนตั้งแต่ติดตั้ง Operator, deploy Kafka cluster (KRaft mode), สร้าง Topic/User, ไปจนถึง produce/consume message จริงบน OpenShift

---

## 📋 สารบัญ

- [Prerequisites](#-prerequisites)
- [Part 1: ติดตั้ง Streams for Apache Kafka Operator](#-part-1-ติดตั้ง-streams-for-apache-kafka-operator)
- [Part 2: Deploy Kafka Cluster](#-part-2-deploy-kafka-cluster)
- [Part 3: ตรวจสอบสถานะ Cluster](#-part-3-ตรวจสอบสถานะ-cluster)
- [Part 4: สร้าง Topic](#-part-4-สร้าง-topic)
- [Part 5: สร้าง User และ Authentication](#-part-5-สร้าง-user-และ-authentication)
- [Part 6: Produce และ Consume Message](#-part-6-produce-และ-consume-message)
- [Part 7: Expose Kafka ผ่าน External Route (Bonus)](#-part-7-expose-kafka-ผ่าน-external-route-bonus)
- [Part 8: Cleanup](#-part-8-cleanup)
- [Troubleshooting](#-troubleshooting)
- [Reference](#-reference)

---

## ✅ Prerequisites

ก่อนเริ่ม lab ต้องมีสิ่งเหล่านี้:

- [ ] OpenShift cluster (4.14+) พร้อม `cluster-admin` หรือสิทธิ์ติดตั้ง Operator
- [ ] `oc` CLI ติดตั้งแล้วและ login เข้า cluster
- [ ] Storage class ที่รองรับ `ReadWriteOnce` (สำหรับ persistent storage ของ Kafka)

ตรวจสอบว่าพร้อมก่อนเริ่ม:

```bash
# ตรวจสอบว่า login แล้ว
oc whoami

# ตรวจสอบ storage class ที่มี
oc get storageclass

# ตรวจสอบสิทธิ์ติดตั้ง operator
oc auth can-i create subscriptions -n openshift-operators
```

---

## 🚀 Part 1: ติดตั้ง Streams for Apache Kafka Operator

### Step 1.1 — สร้าง Namespace

```bash
oc new-project kafka
```

### Step 1.2 — ติดตั้ง Operator ผ่าน OLM

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

### Step 1.3 — รอ Operator พร้อมใช้งาน

```bash
# รอจน Operator pod status = Running
oc get pods -n kafka -w

# กด Ctrl+C เมื่อเห็น output แบบนี้:
# NAME                                        READY   STATUS    RESTARTS   AGE
# amq-streams-cluster-operator-xxxxxxxxx-xxxxx  1/1     Running   0          2m
```

ตรวจสอบว่า CRD ถูกติดตั้งครบ:

```bash
oc get crd | grep kafka

# ควรเห็น CRD เหล่านี้เป็นอย่างน้อย:
# kafkas.kafka.strimzi.io
# kafkatopics.kafka.strimzi.io
# kafkausers.kafka.strimzi.io
# kafkanodepools.kafka.strimzi.io
```

> ✅ **Checkpoint:** ถ้าเห็น operator pod เป็น `Running` และ CRD ครบ แปลว่า Part 1 เสร็จสมบูรณ์

---

## 🏗️ Part 2: Deploy Kafka Cluster

Lab นี้ใช้ **KRaft mode** (ไม่ใช้ ZooKeeper) ซึ่งเป็นแนวทางมาตรฐานปัจจุบัน

### Step 2.1 — สร้าง KafkaNodePool สำหรับ Controller

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

### Step 2.2 — สร้าง KafkaNodePool สำหรับ Broker

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

### Step 2.3 — สร้าง Kafka Cluster

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

### Step 2.4 — Apply ทั้งหมด

```bash
oc apply -f 02-nodepool-controller.yaml
oc apply -f 03-nodepool-broker.yaml
oc apply -f 04-kafka-cluster.yaml
```

การ deploy เต็มรูปแบบใช้เวลาประมาณ 2-5 นาที ให้รอด้วยคำสั่งนี้:

```bash
oc wait kafka/my-cluster \
  --for=condition=Ready \
  --timeout=300s \
  -n kafka
```

---

## 🔍 Part 3: ตรวจสอบสถานะ Cluster

```bash
# ดู Pod ทั้งหมด
oc get pods -n kafka

# ควรเห็น:
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
# ดู Kafka CR status
oc get kafka my-cluster -n kafka

# ดู detail conditions
oc describe kafka my-cluster -n kafka
```

```bash
# ดู Service ที่ Operator สร้างให้
oc get svc -n kafka

# ควรเห็น bootstrap service สำหรับ client เชื่อมต่อ
# my-cluster-kafka-bootstrap   ClusterIP   ...   9092/TCP,9093/TCP
```

> ✅ **Checkpoint:** Kafka CR status ต้องเป็น `Ready: True` ก่อนไป Part ถัดไป

---

## 📨 Part 4: สร้าง Topic

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
    retention.ms: 604800000        # 7 วัน
    segment.bytes: 1073741824      # 1 GB
    min.insync.replicas: 2
```

```bash
oc apply -f 05-topic.yaml

# ตรวจสอบ
oc get kafkatopic -n kafka
oc describe kafkatopic my-topic -n kafka
```

---

## 🔐 Part 5: สร้าง User และ Authentication

### Step 5.1 — สร้าง KafkaUser พร้อม SCRAM-SHA-512 Authentication

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

> ⚠️ **หมายเหตุ:** Authorization type `simple` ต้องเปิด `authorization: type: simple` ใน Kafka CR ด้วย (`spec.kafka.authorization`) ถ้าต้องการบังคับ ACL จริง — lab นี้เน้นสอน syntax การสร้าง user ก่อน

```bash
oc apply -f 06-user.yaml

# ตรวจสอบว่า User Operator สร้าง Secret ให้อัตโนมัติ
oc get kafkauser my-user -n kafka
oc get secret my-user -n kafka
```

### Step 5.2 — ดู Password ที่ Generate ให้

```bash
oc get secret my-user -n kafka \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## 💬 Part 6: Produce และ Consume Message

### Step 6.1 — Deploy Client Pod สำหรับทดสอบ

```bash
oc run kafka-client \
  --image=registry.redhat.io/amq-streams/kafka-39-rhel9:latest \
  --restart=Never \
  -n kafka \
  -- sleep infinity
```

รอให้ pod พร้อม:

```bash
oc wait pod/kafka-client --for=condition=Ready -n kafka --timeout=60s
```

### Step 6.2 — Produce Message (Plain listener, ไม่มี auth)

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-console-producer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic my-topic
```

พิมพ์ข้อความทดสอบ แล้วกด Enter (กด Ctrl+D เพื่อออก):

```
สวัสดี Kafka!
ข้อความทดสอบที่ 2
ข้อความทดสอบที่ 3
```

### Step 6.3 — Consume Message

เปิด terminal ใหม่:

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-console-consumer.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --topic my-topic \
  --from-beginning
```

ควรเห็นข้อความที่ produce ไว้ก่อนหน้าปรากฏขึ้นมา

> ✅ **Checkpoint:** ถ้า consume เห็นข้อความที่ produce ไว้ครบ แปลว่า cluster ทำงานถูกต้องสมบูรณ์แบบ end-to-end

### Step 6.4 — ตรวจสอบ Topic Detail

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-topics.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe \
  --topic my-topic
```

### Step 6.5 — ตรวจสอบ Consumer Group

```bash
oc exec -it kafka-client -n kafka -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --list

oc exec -it kafka-client -n kafka -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe \
  --group console-consumer-xxxxx   # แทนด้วยชื่อ group ที่เห็นจาก --list
```

---

## 🌐 Part 7: Expose Kafka ผ่าน External Route (Bonus)

ถ้าต้องการให้ client จากนอก OpenShift cluster เชื่อมต่อได้:

```yaml
# 07-kafka-external-listener.yaml
# เพิ่ม listener นี้เข้าไปใน spec.kafka.listeners ของ Kafka CR เดิม
- name: external
  port: 9094
  type: route
  tls: true
  authentication:
    type: scram-sha-512
```

```bash
# แก้ Kafka CR แล้ว apply ใหม่
oc edit kafka my-cluster -n kafka

# ดู Route ที่ Operator สร้างให้ (1 route ต่อ broker)
oc get routes -n kafka
```

---

## 🧹 Part 8: Cleanup

```bash
# ลบ resource ทั้งหมดใน namespace kafka
oc delete kafkatopic --all -n kafka
oc delete kafkauser --all -n kafka
oc delete kafka --all -n kafka
oc delete kafkanodepool --all -n kafka
oc delete pod kafka-client -n kafka

# ลบ operator subscription (ถ้าต้องการถอนถอน operator ด้วย)
oc delete subscription amq-streams -n kafka

# ลบ namespace ทั้งหมด (ลบทุกอย่างในคราวเดียว)
oc delete project kafka
```

---

## 🛠️ Troubleshooting

| อาการ | สาเหตุที่เป็นไปได้ | วิธีแก้ |
|---|---|---|
| Operator pod ไม่ start | Subscription channel ผิด หรือ catalog source ไม่พร้อม | `oc get subscription -n kafka -o yaml` ดู status conditions |
| Kafka pod ค้าง `Pending` | Storage class ไม่รองรับ หรือ resource ไม่พอ | `oc describe pod <pod-name> -n kafka` ดู Events |
| Kafka CR ไม่ `Ready` | Broker ยัง sync ไม่เสร็จ หรือ config ผิด | `oc describe kafka my-cluster -n kafka` ดู conditions |
| `kafka-console-producer` ต่อไม่ได้ | Bootstrap server address ผิด หรือ listener config ไม่ตรง | `oc get svc -n kafka` เช็คชื่อ service ให้ตรง |
| Topic สร้างไม่สำเร็จ | `strimzi.io/cluster` label ไม่ตรงกับชื่อ Kafka cluster | ตรวจสอบ label ให้ตรงกับ `metadata.name` ของ Kafka CR |

### คำสั่ง Debug ที่มีประโยชน์

```bash
# ดู log ของ Cluster Operator
oc logs -f deployment/amq-streams-cluster-operator -n kafka

# ดู log ของ broker เฉพาะตัว
oc logs -f my-cluster-broker-0 -n kafka

# ดู event ทั้งหมดใน namespace
oc get events -n kafka --sort-by='.lastTimestamp'
```

---

## 📚 Reference

- [Red Hat Streams for Apache Kafka Documentation](https://access.redhat.com/documentation/en-us/red_hat_streams_for_apache_kafka)
- [Strimzi Documentation (Upstream)](https://strimzi.io/documentation/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

## 🎯 สิ่งที่เรียนรู้จาก Lab นี้

- [x] ติดตั้ง Streams for Apache Kafka Operator ผ่าน OLM
- [x] Deploy Kafka Cluster แบบ KRaft mode (ไม่ใช้ ZooKeeper)
- [x] แยก Controller และ Broker ผ่าน KafkaNodePool
- [x] สร้างและจัดการ Topic ผ่าน KafkaTopic CR
- [x] สร้าง User พร้อม Authentication และ ACL ผ่าน KafkaUser CR
- [x] Produce/Consume message ผ่าน CLI client pod
- [x] Expose Kafka ออกนอก cluster ผ่าน Route

---

*Lab นี้ทดสอบบน OpenShift 4.16+ และ Streams for Apache Kafka (channel `stable`) — เวอร์ชันของ operator/Kafka อาจเปลี่ยนตามเวลา ควรตรวจสอบ [compatibility matrix](https://access.redhat.com/articles/streams-supported-configurations) ก่อนใช้งานจริง*

# Scaling, Rebalance, Cruise Control

- [Scaling, Rebalance, Cruise Control](#scaling-rebalance-cruise-control)
  - [Scaling clusters by adding or removing brokers](#scaling-clusters-by-adding-or-removing-brokers)
  - [Reference](#reference)
  - [Back to Table of Content](#back-to-table-of-content)

---

## Scaling clusters by adding or removing brokers

- Scaling Kafka clusters by adding brokers can improve performance and reliability. Increasing the number of brokers provides more resources, enabling the cluster to handle larger workloads and process more messages. It also enhances fault tolerance by providing additional replicas. Conversely, removing underutilized brokers can reduce resource consumption and increase efficiency. Scaling must be done carefully to avoid disruption or data loss. Redistributing partitions across brokers reduces the load on individual brokers, increasing the overall throughput of the cluster.

  Adjusting the replicas configuration changes the number of brokers in a cluster. A replication factor of 3 means each partition is replicated across three brokers, ensuring fault tolerance in case of broker failure:

- create project `kafka-scale`

[](../manifests/examples/cruise-control/kafka-cruise-control-with-goals.yaml)

[](../manifests/examples/cruise-control/topic.yaml)

produce-job.jaml

  oc run kafka-util -ti \
  --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-topics.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe --topic my-topic

adjust replica of KafkaNodePool `broker` from 3 to 4 

  oc run kafka-util -ti \
  --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-topics.sh \
  --bootstrap-server my-cluster-kafka-bootstrap:9092 \
  --describe --topic my-topic

[](../manifests/examples/cruise-control/kafka-rebalance-add-brokers.yaml)


oc describe kafkarebalance my-rebalance

oc annotate kafkarebalance my-rebalance strimzi.io/rebalance="refresh"

oc annotate kafkarebalance my-rebalance strimzi.io/rebalance="approve"

---

## Reference

- [Using the partition reassignment tool (`kafka-reassign-partitions.sh`)](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.2/html-single/deploying_and_managing_streams_for_apache_kafka_on_openshift/index#assembly-reassign-tool-str)

- [Using Cruise Control for cluster rebalancing](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.2/html-single/deploying_and_managing_streams_for_apache_kafka_on_openshift/index#cruise-control-concepts-str)

---

## Back to Table of Content

- [Hands-On Lab: Red Hat Streams for Apache Kafka on OpenShift](../README.md)
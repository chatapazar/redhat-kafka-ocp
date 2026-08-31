# Upgrade Red Hat Streams for Apache Kafka

- [Upgrade Red Hat Streams for Apache Kafka](#upgrade-red-hat-streams-for-apache-kafka)
  - [Recommend Operator Installation](#recommend-operator-installation)
  - [Upgrading Streams for Apache Kafka](#upgrading-streams-for-apache-kafka)
  - [Upgrading OpenShift with minimal downtime](#upgrading-openshift-with-minimal-downtime)
  - [Reference](#reference)
  - [Back to Table of Content](#back-to-table-of-content)

---

## Recommend Operator Installation

- **Use Manual InstallPlan Approval**: Never set installPlanApproval: Automatic in production. Use Manual so your team can review, test, and approve update plans (InstallPlan) during designated maintenance windows.

- **Subscribe to Stable Channels**: Always select long-term supported channels (e.g., stable, stable-vX.Y) rather than fast, alpha, or candidate channels.

- **Isolate Blast Radius**: Prefer OwnNamespace or SingleNamespace installation modes for tenant-level operators. Use AllNamespaces (cluster-wide) strictly for cluster infrastructure operators (e.g., Logging, Mesh, Storage).

---

## Upgrading Streams for Apache Kafka

To upgrade brokers and clients without downtime, you must complete the Streams for Apache Kafka upgrade procedures in the following order:

1. Make sure your OpenShift cluster version is supported.

   Streams for Apache Kafka 3.2 requires OpenShift 4.16 to 4.21 (excluding 4.17).

  You can upgrade OpenShift with minimal downtime.

2. Ensure Kafka clusters are KRaft-based.

   If upgrading from a version earlier than 2.7, you must migrate from ZooKeeper to KRaft before upgrading the Cluster Operator. Upgrades from ZooKeeper-based clusters are not supported.

3. Prepare for v1 API conversion.

   If upgrading from a version prior to 3.2, this requirement includes a KafkaUser update that must be done before upgrading the Cluster Operator.

4. Upgrade the Cluster Operator.

5. Update the Kafka version and metadataVersion. (One round for the Kafka version and another for the metadata version.)

---

## Upgrading OpenShift with minimal downtime

- Default Pod Disruption Budgets (PDBs)

  - Kafka Brokers: Strimzi automatically creates a PDB with maxUnavailable: 1. This ensures that only one broker is allowed to be taken down at a time, preserving High Availability (HA) and preventing partition offline issues due to a loss of Quorum or In-Sync Replicas (ISR).

  - ZooKeeper / KRaft Controllers: Also configured with maxUnavailable: 1 by default to maintain quorum and prevent cluster outages.

- If you are upgrading OpenShift, refer to the OpenShift upgrade documentation to check the upgrade path and the steps to upgrade your nodes correctly. Before upgrading OpenShift, check the supported versions for your version of Streams for Apache Kafka.

  When performing your upgrade, ensure the availability of your Kafka clusters by following these steps:

  - Configure pod disruption budgets
  - Roll pods using one of these methods:
    - Apply an annotation to your pods to roll them manually
    - Use the Streams for Apache Kafka Drain Cleaner (recommended)
     
  For Kafka to stay operational, topics must also be replicated for high availability. **This requires topic configuration that specifies a replication factor of at least 3 and a minimum number of in-sync replicas to 1 less than the replication factor.**

---

## Reference

- [Upgrading Streams for Apache Kafka](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.2/html-single/deploying_and_managing_streams_for_apache_kafka_on_openshift/index#assembly-upgrade-str)

- Default Kafka Pod Deployment
  
  By default, Strimzi configures **Soft Pod Anti-Affinity** (`preferredDuringSchedulingIgnoredDuringExecution`) for Kafka broker pods using the topology key `kubernetes.io/hostname`.

  * **Best-Effort Node Distribution:** The Kubernetes scheduler attempts to place each Kafka broker pod on a different worker node to avoid single-node points of failure.

  * **Fallback Co-location:** Because it uses soft (preferred) anti-affinity, if the cluster has fewer worker nodes than brokers (e.g., 3 brokers on 2 nodes), Kubernetes will place multiple brokers on the same node instead of leaving pods in a `Pending` state.

  * **No Default Multi-AZ Awareness:** Unless you explicitly define the `rack` configuration, Strimzi will not automatically group or distribute brokers across Availability Zones (AZs).

  * **Unrestricted Node Placement:** There are no default `nodeSelector` or `tolerations` applied, allowing broker pods to land on any available general-purpose worker node with sufficient resources.

- [Rolling pods manually](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.2/html-single/deploying_and_managing_streams_for_apache_kafka_on_openshift/index#rolling_pods_manually_alternative_to_drain_cleaner)

- [Evicting pods with the Streams for Apache Kafka Drain Cleaner](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.2/html-single/deploying_and_managing_streams_for_apache_kafka_on_openshift/index#assembly-drain-cleaner-str)

---

## Back to Table of Content

- [Hands-On Lab: Red Hat Streams for Apache Kafka on OpenShift](../README.md)

- openshift update ไม่ให้กระทบ strimzi
- default strimzi poddistrubtionbudget


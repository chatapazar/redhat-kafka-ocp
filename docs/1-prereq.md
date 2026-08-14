# ✅ Prerequisites

> Before starting this lab, make sure you have:

---

- [✅ Prerequisites](#-prerequisites)
  - [OpenShift Container Platform](#openshift-container-platform)
  - [Streams for Apache Kafka Supported Lifecycles, LTS, Configuration](#streams-for-apache-kafka-supported-lifecycles-lts-configuration)
  - [Installing the Streams for Apache Kafka operator from the OperatorHub](#installing-the-streams-for-apache-kafka-operator-from-the-operatorhub)
  - [Download Red Hat Streams for Apache Kafka OpenShift Installation and Example Files](#download-red-hat-streams-for-apache-kafka-openshift-installation-and-example-files)

---

## OpenShift Container Platform

Before starting this lab, make sure you have:

- [ ] An OpenShift cluster (4.16+) with `cluster-admin` or sufficient permissions to install Operators
- [ ] `oc` CLI installed and logged in to the cluster
- [ ] A storage class that supports `ReadWriteOnce` (for Kafka persistent storage)

Verify readiness before you begin:

```bash
# Confirm you're logged in
oc whoami

# Check available storage classes
oc get storageclass

# Confirm you can install operators
oc auth can-i create subscriptions -n openshift-operators
```

---

## Streams for Apache Kafka Supported Lifecycles, LTS, Configuration

- [Streams for Apache Kafka Lifecycle](https://access.redhat.com/support/policy/updates/jboss_notes#p_Streams)
- [Streams for Apache Kafka LTS Support Policy](https://access.redhat.com/articles/6975608)
- [Supported Configurations](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/3.2/html/release_notes_for_streams_for_apache_kafka_3.2_on_openshift/ref-supported-configurations-str)

---

## Installing the Streams for Apache Kafka operator from the OperatorHub

- 

---

## Download Red Hat Streams for Apache Kafka OpenShift Installation and Example Files

- [Download Red Hat Streams for Apache Kafka](https://access.redhat.com/downloads/content/application-services/jboss.amq.streams/3.2.1/releases)

---
# Streams for Apache Kafka Proxy Record Encryption Filter Examples

The Record Encryption filter provides an Encryption-at-Rest solution for Apache Kafka(tm) which is transparent to both clients and brokers. 
The filter is responsible for encrypting messages that are sent by producing applications so the Kafka Broker never sees the plain text content of the messages.  
The filter also decrypts messages before they are returned to consuming applications.

In this directory, you'll find examples that help you deploy Streams for Apache Kafka Proxy with the Record Encryption filter to your OpenShift Cluster so that you may try out the feature together
with your own application.

The filter relies on a Key Management System (KMS). The role of the KMS is to provide cryptographic functions and act as a repository of key-material. The KMS is *not* part of Streams for Apache Kafka.  It is external and must be provided by the deployer of the system.
In this example  you can choose between HashiCorp Vault&#8482; or AWS KMS&#8482;. Please contact your Red Hat representative or support if you would like to use an alternative KMS.

# Prerequisites

* Administrative access to the OpenShift Cluster being used to evaluate Streams for Apache Kafka Proxy
* The following Operators must be installed in the OpenShift Cluster
  * Red Hat Streams for Apache Kafka Operator
  * Red Hat Streams for Apache Kafka Proxy Operator
  * Cert-manager Operator for Red Hat OpenShift
* CLI for your chosen KMS - either Vault CLI (`vault`) or AWS CLI (`aws`).
* Apache Kafka CLI tools (`kafka-console-producer.sh`, and `kafka-console-consumer.sh`) (found in the `bin` directory of the Streams for Apache Kafka on RHEL distribution).

For the `load-balancer` based example, you will also need the ability to configure DNS on the network used by the off-cluster applications.

In addition, you must follow the [KMS preparation instructions](./PREPARE_KMS.md).

# TLS certificates

The examples rely on TLS communication between the applications and the proxy. 
Cert-manager is used to create the required certificates.
In order to keep the examples simple, Cert-manager uses a self-signed issuer.
This is NOT suitable for production use.

# Running the examples

The examples in the [yaml](./yaml) directory demonstrate how you can configure the Proxy.
They also provide the commands you need to create keys in your KMS and produce and consume messages in order to see encryption in action.

* The [`cluster-ip`](./yaml/cluster-ip/README.md) example configures the proxy for on-cluster access.
* The [`load-balancer`](./yaml/load-balancer/README.md) example configures the proxy for off-cluster access.


# Streams for Apache Kafka Proxy Record Validation Filter Examples

The Record Validation filter ensures that all records produced to a kafka topic conform to an expected schema.
Records that don't conform to the expected schema will not be sent to the broker. Instead, the producer will
receive an error response. Record Validation filter currently support JSON format records.  It enforces JSON
schemas.

To use the Record Validation filter, you must have a JSON schema defined for records published to the topic.
The JSON schema must be stored in [Apicurio Registry](https://docs.redhat.com/en/documentation/red_hat_build_of_apicurio_registry/3.1).

In this directory, you'll find examples that help you deploy Streams for Apache Kafka with the Record Validation Filter
to your OpenShift Cluster so that you may try out the feature together with you own application.
The example provides a development instance of Apicurio Registry.

# Prerequisites

* Administrative access to the OpenShift Cluster being used to evaluate Streams for Apache Kafka Proxy
* The following Operators must be installed in the OpenShift Cluster
    * Red Hat Streams for Apache Kafka Operator
    * Red Hat Streams for Apache Kafka Proxy Operator
    * Red Hat build of Apicurio Registry Operator (3.x)
    * Cert-manager Operator for Red Hat OpenShift
* Apache Kafka CLI tools (`kafka-console-producer.sh`, and `kafka-console-consumer.sh`) (found in the `bin` directory of the Streams for Apache Kafka on RHEL distribution).
* `cURL`

For the `load-balancer` based example, you will also need the ability to configure DNS on the network used by the off-cluster applications.

# TLS certificates

The examples rely on TLS communication between the applications and the proxy.
Cert-manager is used to create the required certificates.
In order to keep the examples simple, Cert-manager uses a self-signed issuer.
This is NOT suitable for production use.

# Running the examples

The examples in the [yaml](./yaml) directory demonstrate how you can configure the Proxy.
They also provide the commands you need to create schemas and produce and consume messages in order to see record validation in action.

* The [`cluster-ip`](./yaml/cluster-ip/README.md) configures the proxy for on-cluster access.
* The [`load-balancer`](./yaml/load-balancer/README.md) configures the proxy for off-cluster access.



# Streams for Apache Kafka Proxy Record Encryption - exposed by off-cluster using an External Load Balancer Service

In this example, an instance of Apache Kafka is deployed using Streams for Apache Kafka.  The instance is proxied using
Streams for Apache Kafka Proxy with the proxy configured for Record Encryption.
The virtual cluster is exposed off cluster using a load-balancer service.

For this example, you must be able to configure DNS within the test environment so that the
Fully Qualified Domain Names (FQDNs) you choose resolve to the external address allocated to the load balancer.

Before you start, ensure that the [prerequisites](../../README.md) have been completed.

# Deploying the Example

1. Decide the FQDNs conventions that the host will use when it connects to the bootstrap and broker endpoints of the virtual cluster.
   In this example, we assume you'll be making use of wildcard DNS `*.myproxy.mycompany.com`.

2. Edit `yaml/load-balancer/proxy/01.KafkaProxyIngress-my-cluster.yaml` and adjust the suffixes of `bootstrapAddress` and
   `advertisedBrokerAddressPattern` to match the conventions you've just established.

3. Edit `yaml/load-balancer/proxy/02.Certificate.my-cluster-server-certificate.yaml` and adjust the `commonName` and `dnsNames` and
   to match the conventions you've just established.

4. Deploy the Example:
   ```sh
   oc apply -f yaml/core -f yaml/load-balancer --recursive
   ```
5. Wait for the ingress of the virtual cluster to have a bootstrap address assigned
   ```sh
   oc wait  -n my-proxy virtualkafkacluster my-cluster --for=jsonpath='{.status.ingresses[0].bootstrapServer}'
   ```
6. Get the bootstrap address
   ```sh
   BOOTSTRAP_SERVER=$(oc get -n my-proxy virtualkafkacluster my-cluster -o=jsonpath='{.status.ingresses[0].bootstrapServer}')
   ```

7. Get the self-signed certificate
   ```sh
   CA=$(oc get secret -n my-proxy my-cluster-server-certificate -o json | jq -r ".data.\"ca.crt\" | @base64d")
   ```
8. Get the external address of the virtual cluster:
   ```sh
   oc get -n my-proxy virtualkafkaclusters my-cluster -o=jsonpath='{.status.ingresses[0].loadBalancerIngressPoints}'
   ```

9. Now update DNS of the off-cluster network so that DNS requests for the bootstrap and broker endpoints resolve
   to the external address of the virtual cluster.


# Try out the example

1. Create a key for topic `trades` using the instructions applicable to your KMS provider:
   
   Vault:
   ```sh
   vault write -f transit/keys/KEK_trades
   ```
   AWS:
   ```sh
   aws kms create-alias --alias-name alias/KEK_trades --target-key-id $(aws kms create-key | jq -r '.KeyMetadata.KeyId')
   ```
2. Produce some messages to the topic:
   ```sh
   echo 'IBM:100\nAPPLE:99' | kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic trades --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```
3. Consume messages direct from the Kafka Cluster, showing that they are encrypted:
   ```sh
    oc run -n kafka cluster-consumer -qi --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic trades --from-beginning --timeout-ms 10000
   ```
4. Consume messages from the proxy showing they are decrypted:
   ```sh
    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic trades --from-beginning --timeout-ms 10000 --consumer-property ssl.truststore.type=PEM --consumer-property security.protocol=SSL --consumer-property ssl.truststore.certificates="${CA}"
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster like this:

```sh
oc delete -f yaml/load-balancer -f yaml/core --recursive
```

To remove the KMS configuration, see [the KMS cleanup instructions](../../PREPARE_KMS.md#cleaning-up).


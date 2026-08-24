# Streams for Apache Kafka Proxy Record Encryption - exposed on-cluster using a Cluster-IP service

In this example, an instance of Apache Kafka is deployed using Streams for Apache Kafka.  The instance is proxied using
Streams for Apache Kafka Proxy with the proxy configured for Record Encryption.
The virtual cluster is exposed within an OpenShift cluster by a cluster-ip service. 

Before you start, ensure that the [prerequisites](../../README.md) have been completed.

# Deploy the Example

1. Deploy the Example:
   ```sh
   oc apply -f yaml/core -f yaml/cluster-ip --recursive
   ```
2. Wait for the ingress of the virtual cluster to have a bootstrap address assigned
   ```sh
   oc wait  -n my-proxy virtualkafkacluster my-cluster --for=jsonpath='{.status.ingresses[0].bootstrapServer}'
   ```
3. Get the bootstrap address
   ```sh
   BOOTSTRAP_SERVER=$(oc get -n my-proxy virtualkafkacluster my-cluster -o=jsonpath='{.status.ingresses[0].bootstrapServer}')
   ```
4. Get the self-signed certificate
   ```sh
   CA=$(oc get secret -n my-proxy my-cluster-server-certificate -o json | jq -r ".data.\"ca.crt\" | @base64d")
   ```
5. Create a key for topic `trades` using the instructions applicable to your KMS provider:

   Vault:
   ```sh
   vault write -f transit/keys/KEK_trades
   ```
   AWS:
   ```sh
   aws kms create-alias --alias-name alias/KEK_trades --target-key-id $(aws kms create-key | jq -r '.KeyMetadata.KeyId')
   ```

# Run the Example

1. Produce some messages to the topic:
   ```sh
   echo 'IBM:100\nAPPLE:99' | oc run -n my-proxy -qi proxy-producer --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- ./bin/kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic trades --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```
2. Consume messages *direct* from the Kafka Cluster, showing that they are encrypted:
   ```sh
    oc run -n kafka cluster-consumer -qi --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic trades --from-beginning --timeout-ms 10000
   ```
3. Consume messages from the *proxy* showing they get decrypted automatically:
   ```sh
    oc run -n my-proxy proxy-consumer -qi --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server ${BOOTSTRAP_SERVER} --topic trades --from-beginning --timeout-ms 10000  --consumer-property ssl.truststore.type=PEM --consumer-property security.protocol=SSL --consumer-property ssl.truststore.certificates="${CA}"
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster like this:

```sh
oc delete -f yaml/cluster-ip -f yaml/core --recursive
```

To remove the KMS configuration, see [the KMS cleanup instructions](../../PREPARE_KMS.md#cleaning-up).


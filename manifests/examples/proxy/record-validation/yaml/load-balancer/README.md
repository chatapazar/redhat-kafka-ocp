# Streams for Apache Kafka Proxy Record Validation, exposed by External Load Balancer

In this example, an instance of Apache Kafka is deployed to an OpenShift cluster using Streams for Apache Kafka, alongside Apicurio Registry, which is also deployed to the cluster.

The Streams for Apache Kafka Proxy is deployed with configuration to perform record validation.  The configuration ensures that
any records sent to a topic called `people` adhere to a `person` schema.

Finally, kafka command line tools (run off cluster) are used to send valid and invalid record to the `people` topic
so that the effects of the validation can be observed.

For this example, you must be able to configure DNS within the test environment so that the
Fully Qualified Domain Names (FQDNs) you choose resolve to the external address allocated to the load balancer.

# Deploying the Example

1. Decide the FQDNs conventions that the host will use when it connects to the bootstrap and broker endpoints of the virtual cluster.
   In this example, we assume you'll be making use wildcard DNS `*.myproxy.mycompany.com`.

2. Edit `yaml/load-balancer/proxy/01.KafkaProxyIngress-my-ingress.yaml` and adjust the suffixes of `bootstrapAddress` and
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

10. Create the example schema in the registry:

    ```sh
    REGISTRY_URL=http://$(oc get apicurioregistries.registry.apicur.io -n schema-registry registry --template='{{.status.info.host}}')
    curl -X POST ${REGISTRY_URL}/apis/registry/v3/groups/default/artifacts -H "Content-Type: application/json; artifactType=JSON" -H "X-Registry-ArtifactId: Person" --data @schemas/person.schema.json
    ```

# Try out the example

1. Produce a record to the topic with a record value that matches the schema.
   ```sh
   cat record-examples/valid-person.json |  kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --sync --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```
2. Consume messages showing the record reached the broker.
   ```sh
    kafka-console-consumer.sh  --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --from-beginning --timeout-ms 10000  --consumer-property ssl.truststore.type=PEM --consumer-property security.protocol=SSL --consumer-property ssl.truststore.certificates="${CA}"
   ```   
3. Produce invalid records to the topic to be rejected by the filter.  The producer will report an exception.
   ```sh
   cat record-examples/invalid-person-invalid-age.json | kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --sync --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```

   ```sh
   cat record-examples/invalid-person-malformed.json | kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --sync --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```

4. Consume messages showing that no rejected records reached the broker.
   ```sh
    kafka-console-consumer.sh  --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --from-beginning --timeout-ms 10000  --consumer-property ssl.truststore.type=PEM --consumer-property security.protocol=SSL --consumer-property ssl.truststore.certificates="${CA}"
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster:

```sh
oc delete -f yaml/load-balancer -f yaml/core --recursive
```


# Streams for Apache Kafka Proxy Record Validation, exposed using Cluster-IP

In this example, an instance of Apache Kafka is deployed to an OpenShift cluster using Streams for Apache Kafka, alongside Apicurio Registry, which is also deployed to the cluster.

The Streams for Apache Kafka Proxy is deployed with configuration to perform record validation.  The configuration ensures that 
any records sent to a topic called `people` adhere to a `person` schema.

Finally, Kafka command line tools are used to send valid and invalid records to the `people` topic, so the effects of the validation can be observed.
so that the effects of the validation can be observed.

# Deploying the Example

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
5. Create the example schema in the registry:

   ```sh
   REGISTRY_URL=http://$(oc get apicurioregistries.registry.apicur.io -n schema-registry registry -o=jsonpath='{.status.info.host}')
   curl -X POST ${REGISTRY_URL}/apis/registry/v3/groups/default/artifacts -H "Content-Type: application/json; artifactType=JSON" -H "X-Registry-ArtifactId: Person" --data @schemas/person.schema.json
   ```

# Try out the example

1. Produce a record to the topic with a record value that matches the schema:
   ```sh
   cat record-examples/valid-person.json | oc run -n my-proxy -qi proxy-producer --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --sync --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```
2. Consume messages showing the record reached the broker:
   ```sh
    oc run -n my-proxy proxy-consumer -qi --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --from-beginning --timeout-ms 10000  --consumer-property ssl.truststore.type=PEM --consumer-property security.protocol=SSL --consumer-property ssl.truststore.certificates="${CA}"
   ```   
3. Produce invalid records to the topic to be rejected by the filter.  The producer will report an exception:
   ```sh
   cat record-examples/invalid-person-invalid-age.json | oc run -n my-proxy -qi proxy-producer --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --sync --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```
   
   ```sh
   cat record-examples/invalid-person-malformed.json | oc run -n my-proxy -qi proxy-producer --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --sync --producer-property ssl.truststore.type=PEM --producer-property security.protocol=SSL --producer-property ssl.truststore.certificates="${CA}"
   ```

4. Consume messages showing that no rejected records reached the broker:
   ```sh
    oc run -n my-proxy proxy-consumer -qi --image=registry.redhat.io/amq-streams/kafka-42-rhel9:3.2.1 --rm=true --restart=Never -- ./bin/kafka-console-consumer.sh  --bootstrap-server ${BOOTSTRAP_SERVER} --topic people --from-beginning --timeout-ms 10000 --consumer-property ssl.truststore.type=PEM --consumer-property security.protocol=SSL --consumer-property ssl.truststore.certificates="${CA}"
   ```   

# Cleaning up

When you have finished with this example, you can remove it from the OpenShift Cluster:

```sh
oc delete -f yaml/cluster-ip -f yaml/core --recursive
```

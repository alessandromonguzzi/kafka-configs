# Connecting Kafka with Elasticsearch using KafkaConnectS2I over SSL

(We'll use default namespace containing both Kafka cluster, KafkaConnect and Elasticsearch)
## Kafka cluster setup using AMQ Streams

1. Install AMQ Streams operator in default namespace
Search in in OperatorHub and install it in the default namespace.
2. Start a base Kafka cluster (1 zookeeper, 1 kafka broker) using the `kafka-my-cluster.yaml` file
```
oc apply -f kafka-my-cluster.yaml
```
3. Deploy a new topic `my-topic` using the kafkatopic-my-topic.yaml` file
```
oc apply -f kafkatopic-my-topic.yaml
```
4. Install Elasticsearch operator (https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-openshift-deploy-the-operator.html)
```
oc apply -f https://download.elastic.co/downloads/eck/1.1.2/all-in-one.yaml
```
5. Run Elasticsearch minimal setup using the `elasticsearch-elasticsearch-sample.yaml` file
```
oc apply -f elasticsearch-elasticsearch-sample.yaml
```
6. Create a truststore using the elasticsearch tls.crt included inside the `elasticsearch-sample-es-http-certs-public` secret
```
#extract certificate to tls.crt file
oc get secret "elasticsearch-sample-es-http-certs-public" -o go-template='{{index .data "tls.crt" | base64decode }}' > tls.crt
#generate a truststore file containing the tls.crt, note the es1234 password
keytool -import -file tls.crt -alias firstCA -keystore myTrustStore -storepass es1234 -noprompt
#create the es-truststore-secret
oc create secret generic es-truststore-secret --from-file=truststore.jks=myTrustStore --type=opaque
```
7. Start KafkaConnectS2I cluster using the `kafkaconnects2i-my-connect-cluster.yaml` file
```
oc apply -f kafkaconnects2i-my-connect-cluster.yaml
```
8. Load the `camel-elasticsearch-rest-kafka-connector` package into the my-connect-cluster-connect
```
mkdir connectors
curl https://repo1.maven.org/maven2/org/apache/camel/kafkaconnector/camel-elasticsearch-rest-kafka-connector/0.2.0/camel-elasticsearch-rest-kafka-connector-0.2.0-package.zip > connectors/file.zip
unzip -d connectors file.zip
rm connectors/file.zip
## Important note, it seems that log4j2 dependencies required by elasticsearch are missing. So download them manually
curl https://repo1.maven.org/maven2/org/apache/logging/log4j/log4j-core/2.13.3/log4j-core-2.13.3.jar > connectors/camel-elasticsearch-rest-kafka-connector/log4j-core-2.13.3.jar
curl https://repo1.maven.org/maven2/org/apache/logging/log4j/log4j-api/2.13.3/log4j-api-2.13.3.jar > connectors/camel-elasticsearch-rest-kafka-connector/log4j-api-2.13.3.jar
oc start-build my-connect-cluster-connect --from-dir=./connectors --follow
```
9. Create KafkaConnector inside the `my-connect-cluster-connect` using the `kafkaconnector-my-sink-connector.yaml` file
```
oc apply -f kafkaconnector-my-sink-connector.yaml
```
10. Test the connection between Kafka and Elasticsearch sending a test message
```
oc exec -it my-cluster-kafka-0 bash
echo '{"data":"Hello World"}' | bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic
```
Check that message is stored in Elasticsearch (it may take some time)
```
curl -k https://localhost:9200/my-topic/_search
```

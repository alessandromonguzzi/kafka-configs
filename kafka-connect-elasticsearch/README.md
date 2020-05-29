How to use kafka connect to push messages into elasticsearch

Requirements:
1) kafka_2.12-2.5.0
2) elasticsearch-7.7.0


Launch zookeeper
./bin/zookeeper-server-start.sh config/zookeeper.properties

Launch kakfa
./bin/kafka-server-start.sh config/server.properties


Important NOTE needed dns specification inside certificate generated by elasticsearch-certutil tool.


Generate certificates for elasticsearch and configure it:

bin/elasticsearch-certutil cert ca --pem --in instance.yml --dns localhost
unzip -l certificate-bundle.zip 
unzip -d certificates certificate-bundle.zip
mv certificates/ config/

vi config/elasticsearch.yml
Add the lines below to the elasticsearch.yml file.
discovery.seed_hosts: [ "kubernetes.docker.internal" ]
cluster.initial_master_nodes: [ "localhost" ]


# This turns on SSL for the HTTP (Rest) interface
xpack.security.http.ssl.enabled: true

# This configures the keystore to use for SSL on HTTP
xpack.security.http.ssl.key: certificates/localhost/localhost.key
xpack.security.http.ssl.certificate: certificates/localhost/localhost.crt
xpack.security.http.ssl.certificate_authorities: certificates/ca/ca.crt
xpack.security.http.ssl.client_authentication: optional

Launch elasticsearch
bin/elasticsearch

go to kafka installation and generate a truststore to use in the kafka-connect to connect with elasticsearch. Watchout, you need only the .crt file to do this step, but move from PKCS12 to JKS

 
keytool -import -alias localhost -file elasticsearch-7.7.0/config/certificates/localhost/localhost.crt -storetype JKS -keystore server.truststore


create a plugin directory inside kakfa installation
mkdir plugin
and copy there camel-elasticsearch-rest-kafka-connector folder

Launch kafka-connect that is listening on port 8083 (or change it)
EXTRA_ARGS="-Djavax.net.ssl.trustStore=server.truststore -Djavax.net.ssl.trustStorePassword=es1234" bin/connect-distributed.sh connect-0.properties

Register camel-es-connector to kafka-connect
curl -sX POST -H "Content-Type: application/json" localhost:8083/connectors -d@elasticSinkTaskLocalSSL.json

Generate a message using kafka-console-producer.sh (NOTE: message must be in JSON format, normal String does not work)
./bin/kafka-console-producer.sh --topic my-topic --broker-list localhost:9092

Message example:
{"data":"My message"}

command to verify content of elasticsearch on topic my-topic (it can take some time (<1min for kafka-connect to work)
curl -k https://localhost:9200/my-topic/_search


Some useful links:
https://www.elastic.co/blog/configuring-ssl-tls-and-https-to-secure-elasticsearch-kibana-beats-and-logstash
https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-tls.html
https://www.elastic.co/blog/elasticsearch-security-configure-tls-ssl-pki-authentication
https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/_encrypted_communication.html
https://camel.apache.org/camel-kafka-connector/latest/connectors/camel-elasticsearch-rest-kafka-sink-connector.html
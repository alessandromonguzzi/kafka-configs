Kafka configuration with SASL and SCRAM-SHA-512

The following procedure creates a single broker, single zookeeper setup with two clients: alice as producer and bob as consumer.

Start zookeeper in the normal way.

Kafka broker configuration:

- sasl-server.properties
- jaas-kafka-server.conf
- sasl-kafka-server-start.sh

Configure users in zookeeper

./bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[iterations=8192,password=alice],SCRAM-SHA-512=[password=alice]' --entity-type users --entity-name alice

./bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password=admin],SCRAM-SHA-512=[password=admin]' --entity-type users --entity-name admin

The shell script is a copy of the original launch script with an additional login property that can be also configured as system property from outside.

./bin/sasl-kafka-server-start.sh config/sasl-server.properties

Clients use different sasl.properties and jaas.conf files

Alice producer:

configuration file:
- sasl-producer-alice.properties
- jaas-kafka-client-alice.conf

To allow the producer to connect to the broker it is necessary to configure the ACL

bin/kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:alice --operation Write --topic test

Launch string:

KAFKA_OPTS="-Djava.security.auth.login.config=./config/jaas-kafka-client-alice.conf" ./bin/kafka-console-producer.sh --topic test --broker-list localhost:9092 --producer.config config/sasl-producer-alice.properties


Bob consumer:

configuration file:
- sasl-consumer-bob.properties
- jaas-kafka-client-bob.conf

Add the configuration to zookeeper (SCRAM-SHA-512 credentials are stored in zookeeper)

bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password=bob]' --entity-type users --entity-name bob

bin/kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:bob --operation Read --group bob-group

bin/kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:bob --operation Read --topic test

Launch string:

KAFKA_OPTS="-Djava.security.auth.login.config=config/jaas-kafka-client-bob.conf" ./bin/kafka-console-consumer.sh --topic test --bootstrap-server localhost:9092 --group bob-group --consumer.config config/sasl-consumer-bob.properties

Useful reference for Kafka authentication and authorization:
https://developer.ibm.com/technologies/messaging/tutorials/kafka-authn-authz/

## odoo
- initial config in odoo.conf, initialize database instead of using web interface.

## database
- database is postgresql 13
- config is customized according to debezium postgresql: https://hub.docker.com/r/debezium/postgres
  notably wal_level: logical

# Clusters
zookeeper, kafka, and kafka-connect are each in cluster of 3 workers.

## Kafka Cluster
unresolved issue:
- after some starting and terminating, this error terminates one of the brokers:
```
cdc-kf1   | [2023-03-03 09:00:47,042] ERROR Exiting Kafka due to fatal exception during startup. (kafka.Kafka$)
cdc-kf1   | org.apache.zookeeper.KeeperException$NodeExistsException: KeeperErrorCode = NodeExists
cdc-kf1   | 	at org.apache.zookeeper.KeeperException.create(KeeperException.java:126)
cdc-kf1   | 	at kafka.zk.KafkaZkClient$CheckedEphemeral.getAfterNodeExists(KafkaZkClient.scala:1903)
cdc-kf1   | 	at kafka.zk.KafkaZkClient$CheckedEphemeral.create(KafkaZkClient.scala:1841)
cdc-kf1   | 	at kafka.zk.KafkaZkClient.checkedEphemeralCreate(KafkaZkClient.scala:1808)
cdc-kf1   | 	at kafka.zk.KafkaZkClient.registerBroker(KafkaZkClient.scala:96)
cdc-kf1   | 	at kafka.server.KafkaServer.startup(KafkaServer.scala:335)
cdc-kf1   | 	at kafka.Kafka$.main(Kafka.scala:109)
cdc-kf1   | 	at kafka.Kafka.main(Kafka.scala)
cdc-kf1   | [2023-03-03 09:00:47,042] INFO [KafkaServer id=1] shutting down (kafka.server.KafkaServer)
cdc-kf1 exited with code 1
```

## kafka connect
- replication done by https://debezium.io/blog/2020/09/15/debezium-auto-create-topics/
- refer to notes to register connector after worker is started

## some useful HTTP requests for kafka connect
- list connectors: `curl -s http://localhost:8083/connectors`
- register debezium source connector for odoo database:
```
curl -H 'Content-Type: application/json' localhost:8083/connectors --data '
{
  "name": "odoo.source",  
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector", 
    "plugin.name": "pgoutput",
    "database.hostname": "db", 
    "database.port": "5432", 
    "database.user": "odoo", 
    "database.password": "odoo", 
    "database.dbname" : "odoo", 
    "database.server.name": "postgres", 
    "topic.prefix": "x",
    "topic.creation.default.replication.factor": 3,
    "topic.creation.default.partitions": 1
  }
}'
```
- remove debezium source connector from kafka connect: `curl -X DELETE http://localhost:8083/connectors/odoo.source`
- register lambda sink connector:
```
curl -H 'Content-Type: application/json' localhost:8083/connectors --data '
{
  "name": "lambda.sink",  
  "config": {
    "connector.class": "io.confluent.connect.aws.lambda.AwsLambdaSinkConnector", 
    "aws.lambda.function.name": "cdclambdasink-dev",
    "aws.lambda.region": "ap-southeast-1",
    "aws.access.key.id": "YOUR AWS ACCOUNT ACCESS KEY",
    "aws.secret.access.key": "YOUR AWS ACCOUNT SECRET",
    "aws.lambda.invocation.type": "sync",
    "aws.lambda.batch.size": 1,
    "topics": "x.public.product_template",
    "confluent.topic.bootstrap.servers": "kf1:9092,kf2:9092,kf3:9092",
    "reporter.bootstrap.servers": "kf1:9092,kf2:9092,kf3:9092"
  }
}'
```
- remove lambda sink connector: `curl -X DELETE http://localhost:8083/connectors/lambda.sink`

## how to start
- `docker compose -f dc.o.yaml up` (make sure you start database before kafka.  debezium could not recognize database after its own startup)
- `docker compose -f dc.k.yaml up`
- use kafka connect RESTful API to register connectors

## TODO
- [ ] condense the config in kafka docker-compose file.  too repeatitive on environment variables.
- [ ] expose kafka connect to localhost.  for now I still need to enter the connect docker container to perform the RESTful requests.
- [ ] find out why kafka cluster dies randomly.
- [ ] how to restart lambda function after a lambda error, without removing and creating the connector (i.e. use the connect restart API?)
- [ ] how to reset message offset to particular point to re-run messages?
- [ ] how to start odoo database before debezium and the system still work.  (this is less important because usually database are always running)

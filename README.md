# Kafka KRaft & REST Proxy Docker Setup

## Create Docker Network
```sh
docker network create kafka-network
```

## Start Kafka KRaft Mode
```sh
docker run -d --name kafka-kraft --network kafka-network -h kafka-kraft \
  -p 9101:9101 -p 9092:9092 -p 29092:29092 \
  -v kafka-data:/var/lib/kafka/data \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES="broker,controller" \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP="CONTROLLER:PLAINTEXT,PLAINTEXT_EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT" \
  -e KAFKA_ADVERTISED_LISTENERS="PLAINTEXT://kafka-kraft:29092,PLAINTEXT_EXTERNAL://<PUBLIC_IP>:9092" \
  -e KAFKA_LISTENERS="PLAINTEXT://kafka-kraft:29092,PLAINTEXT_EXTERNAL://0.0.0.0:9092,CONTROLLER://kafka-kraft:29093" \
  -e KAFKA_CONTROLLER_LISTENER_NAMES="CONTROLLER" \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS="1@kafka-kraft:29093" \
  -e KAFKA_INTER_BROKER_LISTENER_NAME="PLAINTEXT" \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_JMX_PORT=9101 \
  -e KAFKA_JMX_HOSTNAME=localhost \
  -e CLUSTER_ID=<CLUSTER_ID> \
  confluentinc/cp-kafka:7.9.0
```

### Generate a Cluster ID (If not available)
```sh
docker run --rm confluentinc/cp-kafka:7.9.0 kafka-storage random-uuid
```

## Start Kafka REST Proxy
```sh
docker run -d --name kafka-rest --network kafka-network -h kafka-rest \
  -p 8082:8082 \
  -e KAFKA_REST_HOST_NAME=kafka-rest \
  -e KAFKA_REST_BOOTSTRAP_SERVERS="PLAINTEXT://kafka-kraft:29092,PLAINTEXT_EXTERNAL://<PUBLIC_IP>:9092" \
  -e KAFKA_REST_LISTENERS="http://0.0.0.0:8082" \
  confluentinc/cp-kafka-rest:7.9.0
```

## Kafka REST API Commands

### List Clusters
```sh
curl -X GET http://localhost:8082/v3/clusters
```

### List Topics
```sh
curl -X GET http://localhost:8082/v3/clusters/<CLUSTER_ID>/topics
```

### Create a Topic
```sh
curl -X POST http://localhost:8082/v3/clusters/<CLUSTER_ID>/topics \
  -H "Content-Type: application/json" \
  -d '{
        "topic_name": "test-topic",
        "partitions_count": 1,
        "replication_factor": 1
      }'
```

### Produce Messages
```sh
curl --location 'http://localhost:8082/v3/clusters/<CLUSTER_ID>/topics/test-topic/records' \
--header 'Content-Type: application/json' \
--data '{
    "key": {
        "type": "STRING",
        "data": "key1"
    },
    "value": {
        "type": "STRING",
        "data": "Hello Kafka!"
    }
}'
```

### Consume Messages
```sh
curl -X GET http://localhost:8082/v3/clusters/<CLUSTER_ID>/topics/test-topic/records
```

Replace `<CLUSTER_ID>` with the generated Cluster ID and `<PUBLIC_IP>` with your server's public IP address.


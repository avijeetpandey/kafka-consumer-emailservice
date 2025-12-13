# Email Notification Service (Kafka Consumer) ✅

A small Spring Boot service that consumes product-created events from a Kafka topic and logs (or handles) email notifications. This repository demonstrates a simple Kafka consumer using Spring for Apache Kafka.

---

## Table of contents

- Overview
- Architecture & key components
- Configuration
- Running locally (with Docker + Kafka)
- Sending sample events
- Troubleshooting
- Extending the consumer

---

## Overview ✉️

This service listens to the Kafka topic `product-created-events-topic` and handles incoming JSON events of type `ProductCreatedEvent`.

It uses Spring Boot and Spring Kafka and currently the handler logs the received event; this is where you would integrate an email-sending service (SMTP, third-party, etc.).

## Architecture & key components 🔧

- `com.consumer.emailservice.EmailserviceApplication` — Spring Boot entry point.
- `com.consumer.emailservice.model.ProductCreatedEvent` — POJO representing the event payload (productId, title, price, quantity).
- `com.consumer.emailservice.handler.ProductCreatedEventHandler` — Kafka listener component annotated with `@KafkaListener(topics = "product-created-events-topic")` and `@KafkaHandler` that receives `ProductCreatedEvent` objects and logs them.
- `src/main/resources/application.properties` — Kafka consumer configuration (bootstrap servers, deserializers, group id).

## Configuration ⚙️

Relevant properties live in `src/main/resources/application.properties`:

```properties
spring.application.name=emailservice
server.port=3000
spring.kafka.consumer.bootstrap-servers=localhost:9092,localhost:9094
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.group-id=product-created-events
```

Notes and recommendations:
- The project uses `JsonDeserializer` to map incoming JSON to `ProductCreatedEvent` — to allow deserialization to your package classes, add:

```properties
spring.kafka.consumer.properties.spring.json.trusted.packages=com.consumer.emailservice,*
```

- Ensure `spring.kafka.consumer.bootstrap-servers` matches your Kafka broker addresses.

## Run locally (recommended: Docker Compose) 🐳

Prerequisites:
- Java (see `pom.xml` `java.version`) — use the JDK version you have available.
- Maven
- Docker & Docker Compose (if using the included instructions below)

Example `docker-compose.yml` to run Zookeeper + Kafka (quick start):

```yaml
version: '3.8'
services:
  zookeeper:
    image: bitnami/zookeeper:latest
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    ports:
      - 2181:2181

  kafka:
    image: bitnami/kafka:latest
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_LISTENERS=PLAINTEXT://:9092
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
    ports:
      - 9092:9092
    depends_on:
      - zookeeper
```

Start Kafka:

```bash
docker compose up -d
```

Build and run the service locally:

```bash
mvn clean package
mvn spring-boot:run
# or if you prefer the fat jar
# java -jar target/emailservice-0.0.1-SNAPSHOT.jar
```

The application will connect to Kafka as configured and subscribe to `product-created-events-topic`.

## Sending sample events to the topic 📝

Create the topic (if you need to):

```bash
# Using kafka-topics from inside the Kafka container
docker exec -it <kafka-container-name> kafka-topics --create --topic product-created-events-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

Produce a sample JSON message (console producer):

```bash
# Example JSON payload
# {"productId":"p-123","title":"New Widget","price":9.99,"quantity":5}

docker exec -i <kafka-container-name> kafka-console-producer --broker-list localhost:9092 --topic product-created-events-topic <<'EOF'
{"productId":"p-123","title":"New Widget","price":9.99,"quantity":5}
EOF
```

Or use `kafkacat` / `kcat`:

```bash
echo '{"productId":"p-123","title":"New Widget","price":9.99,"quantity":5}' | kcat -P -b localhost:9092 -t product-created-events-topic
```

You should see the consumer log the event in the application logs, e.g.: 

```
INFO  Received a new event: New Widget
```

## Troubleshooting & tips ⚠️

- If deserialization fails, verify `spring.kafka.consumer.properties.spring.json.trusted.packages` is set correctly (or set to `*` during development).
- If your service cannot connect to Kafka, double-check `bootstrap-servers` and network binding/adverts from your Docker/Kafka setup.
- Verify the topic name matches: `product-created-events-topic` in both the producer and the listener.
---
title: "3.2 Event driven architecture with Apache Kafka"
linkTitle: "3.2 Event driven architecture with Apache Kafka"
weight: 32
sectionnumber: 3.2
description: >
   Event driven architecture with Apache Kafka.
---


## {{% param sectionnumber %}}.1: Apache Kafka

In this section we are going to deploy a Apache Kafka cluster with the Strimzi Operator and use it to distribute our events between the microservices. In modern large scale applications messages must be processed, reprocessed, analyzed and handled - often in real time. The key design principles of Kafka were formed based on the need of high-throughput architectures that are easily scalable and provide key features to store and process streamed data.


### {{% param sectionnumber %}}.1.1: Publish-subscribe durable messaging system

Apache Kafka is a durable messaging system which uses the publish-subscribe pattern for data exchange. Components publish events to a *topic* and subscribed components will get notified with said event whenever a new message is published. *Topics* represent a datastream which holds a stream of data in temporal order. Applications can send and process records to or from a *topic*. A *record* is a byte array that can store any object in any format. A *record* has four attributes, *key* and *value* are mandatory, and the other attributes, *timestamp* and *headers* are optional. The value can be whatever needs to be sent.

There are four important parts of any Kafka system:

* *Broker*: The broker handles all requests from clients (produce, consume and metadata) and keeps data replicated within the cluster. There can be one or more brokers in a cluster.
* *Zookeeper*: The Zookeeper organizes the state of the cluster (brokers, topics and users).
* *Producer*: The producer sends records to the broker.
* *Consumer*: The consumer subscribes to a topic and consumes records from the broker.

If you want to dive deeper into the Kafka world take a look at the officail [documentation](https://kafka.apache.org/documentation/).


## {{% param sectionnumber %}}.2: Hands-on


### {{% param sectionnumber %}}.2.1: Local

Let's try and get a data producer with a Kafka cluster up and running on your local machine.

Create a new docker-compose file with the following content:

```Dockerfile

version: '2'

services:

  zookeeper:
    image: strimzi/kafka:0.11.3-kafka-2.1.0
    command: [
      "sh", "-c",
      "bin/zookeeper-server-start.sh config/zookeeper.properties"
    ]
    ports:
      - "2181:2181"
    environment:
      LOG_DIR: /tmp/logs

  kafka:
    image: strimzi/kafka:0.11.3-kafka-2.1.0
    command: [
      "sh", "-c",
      "bin/kafka-server-start.sh config/server.properties --override listeners=$${KAFKA_LISTENERS} --override advertised.listeners=$${KAFKA_ADVERTISED_LISTENERS} --override zookeeper.connect=$${KAFKA_ZOOKEEPER_CONNECT}"
    ]
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      LOG_DIR: "/tmp/logs"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

```

With this docker-compose file you can start your own local kafka cluster.
Start this kafka cluster with:

```s

docker-compose -f ${PATH_TO_DOCKER_COMPOSE_FILE} up

```

We can now start the data-producer Quarkus microservice which will produce data and send it to a topic called 'manual'.

```s

docker run --network host -e QUARKUS_PROFILE=dev -p 8080:8080 g1raffi/quarkus-techlab-data-producer:kafka

```

As soon as you start your microservice you will see that he will start to produce data to the kafka cluster. Let's verify this and consume our kafka topic manually.

The kafka cluster comes with some handy binaries that will allow you to do certain operations directly on the cluster. You can produce messages with the `kafka-console-producer` binary and consume records with the `kafka-console-consumer` binary.

We log into our kafka cluster with the docker interactive shell. Find your kafka clusters container name with `docker ps` (usually it's name is `kafka_kafka_1`).

```s

docker exec -it kafka_kafka_1 /bin/bash

```

When logged into your docker container you can start to consume data:

```s

[kafka@b6061841d63c kafka]$ ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic manual --from-beginning

```

Additional we can start the consumer microservice as well which will start to consume the data produced by our data-producer microservice.

```s

docker run --network host -e QUARKUS_PROFILE=dev -p 8080:8080 g1raffi/quarkus-techlab-data-consumer:kafka

```

You will see the console logs with the data consumed from the kafka topic. It works!


### {{% param sectionnumber %}}.2.2: OpenShift

We have seen how to get our kafka instance, producer and consumers up and running locally. Let's step our game up and move it to the cloud.

Create a file called kafka-cluster.yaml with the following content:

```yml

apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: amm-techlab
  labels:
    application: amm-techlab
spec:
  kafka:
    version: 2.5.0
    replicas: 1
    listeners:
      plain: {}
      tls: {}
    config:
      auto.create.topics.enable: false
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      log.message.format.version: "2.5"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 10Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}

```

With the Strimzi operator we can manage our Kafka cluster with Kubernetes-native custom resource definitions. The operator will set up your broker tailored to your needs and configuration. We will also manage our topics with the Strimzi operator.

To create a new topic in our Kafka cluster create a file called `manual-topic.yaml` with the following content:

```yml

apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: manual
  labels:
    application: quarkus-techlab
    strimzi.io/cluster: quarkus-techlab
spec:
  partitions: 1
  replicas: 1
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824

```

This will create the 'manual' topic which allows our microservices to communicate.

Update the data-consumer / data-producers ImageStreams to pull the Kafka tag and see how they communicate through the Kafka broker in your very own project:

```yml

# [...]

  - apiVersion: v1
    kind: ImageStream
    metadata:
      labels:
        application: quarkus-techlab
      name: data-producer
    spec:
      lookupPolicy:
        local: false
      tags:
        - annotations:
            openshift.io/imported-from: g1raffi/quarkus-techlab-data-producer:kafka
          from:
            kind: DockerImage
            name: g1raffi/quarkus-techlab-data-producer:kafka
          name: kafka
          referencePolicy:
            type: Source

# [...]
```

```yml

# [...]

  - apiVersion: v1
    kind: ImageStream
    metadata:
      labels:
        application: quarkus-techlab
      name: data-consumer
    spec:
      lookupPolicy:
        local: false
      tags:
        - annotations:
            openshift.io/imported-from: g1raffi/quarkus-techlab-data-consumer:kafka
          from:
            kind: DockerImage
            name: g1raffi/quarkus-techlab-data-consumer:kafka
          name: kafka
          referencePolicy:
            type: Source

# [...]
```
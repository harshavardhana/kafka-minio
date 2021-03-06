# minio-kafka

## TL;DR
How to setup testbed Minio + Apache Kafka (using Docker) for testing audit logging.

## Prerequisites
- Docker installed (Docker version 18.09.6 on Linux)
- docker-compose installed (1.23.0)

## Setup Minio and Apache Kafka with docker-compose
The setup will contain two containers:

- Minio server latest using local volumes for persistence
- Apache Kafka container (with Zookeeper built-in)

In order to join those two containers, we will use docker-compose. Create `docker-compose.yml` file and put the following content:

```yml
version: '3.3'

services:
  kafka:
    image: spotify/kafka
    hostname: kafka
    environment:
      - ADVERTISED_HOST=kafka
      - ADVERTISED_PORT=9092
    ports:
      - "2181:2181"
      - "9092:9092"

  http-kafka:
    image: y4m4/http-kafka
    hostname: http-kakfa
    environment:
      DEBUG: "on"
      MINIO_AUDIT_KAFKA_ENABLE: "on"
      MINIO_AUDIT_KAFKA_BROKERS: "kafka:9092"
      MINIO_AUDIT_KAFKA_TOPIC: "topic"
    entrypoint: >
      /bin/sh -c "
      curl https://raw.githubusercontent.com/harshavardhana/minio-kafka/master/wait-for.sh -o wait-for.sh;
      chmod +x wait-for.sh;
      ./wait-for.sh kafka:9092 -- /http-kafka;
      "

  minio:
    image: minio/minio:RELEASE.2021-06-14T01-29-23Z
    depends_on:
      - kafka
      - http-kafka
    ports:
      - "9000:9000"
    environment:
       MINIO_AUDIT_WEBHOOK_ENABLE_target1: "on"
       MINIO_AUDIT_WEBHOOK_ENDPOINT_target1: "http://http-kafka:4222/rest/kafka"
    volumes:
      - minio-data:/export
    entrypoint: >
      /bin/sh -c "
      curl https://raw.githubusercontent.com/harshavardhana/minio-kafka/master/wait-for.sh -o wait-for.sh;
      chmod +x wait-for.sh;
      microdnf install netcat;
      ./wait-for.sh minio-kafka:4222 -- /usr/bin/docker-entrypoint.sh minio server /export;
      "

volumes:
  minio-data:
```

This docker-compose file declares two containers. First one, `kafka` runs Kafka image developed by Spotify team (it includes Zookeeper). It exposes itself under `kafka` hostname. Two environment variables need to be set:

```
ADVERTISED_HOST: kafka
ADVERTISED_PORT: 9092
```

Those two describe to which hostname the Kafka server should respond from within the container.

Second container, minio runs newest (at the time of writing this post) Minio server. It needs to connect to the running Kafka server, that's why there is the `depends_on` entry in docker-compose:

```yml
depends_on:
    - kafka
    - http-kafka
```

## Starting the environment
Once the `docker-compose.yml` file is ready, open your favorite terminal in the folder which contains `docker-compose.yml` and run:
```
docker-compose up
```

After running this command, you can check status of the containers by invoking:
```
docker-compose ps
```

## Stopping the environment
When the work is done, you can easily turn off running containers, by invoking following command in the folder where you have your `docker-compose.yml` file.
```
docker-compose down
```

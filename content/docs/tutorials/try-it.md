---
title: "Try it in 5 minutes"
date: 2026-06-10
weight: 2
description: "Clone the demo and run 106 integration tests against a real Java microservice stack in under 5 minutes"
---

Clone the demo project and run the full test suite against a real stack — no configuration needed beyond Docker.

## Requirements

- Java 21+
- Maven 3.8+
- Docker

## Run

```bash
git clone https://github.com/garlandframework/garland-demo
cd garland-demo
docker-compose up -d
mvn test -pl tests
```

`docker-compose up -d` starts PostgreSQL, Kafka, and MongoDB. Maven runs the full suite against two Spring Boot microservices — user-service and projection-service — that are bundled in the repo.

## What runs

106 tests across four layers:

- **Endpoint tests** — HTTP request, assert response status and body, verify entity persisted in Postgres
- **Component tests** — assert Kafka events published after an HTTP call; assert MongoDB projections updated after a Kafka event
- **End-to-end tests** — single pipeline verifying HTTP response, Postgres, Kafka, and MongoDB in one assertion using `Verify.allOf`
- **Example classes** — `PostgresExamples`, `KafkaExamples`, `MongoExamples` — one test per API method, annotated with explanations

## Explore individual examples

Run a single example class to explore a specific client:

```bash
# all Postgres patterns
mvn test -pl tests -Dtest=PostgresExamples

# all Kafka patterns
mvn test -pl tests -Dtest=KafkaExamples

# all MongoDB patterns
mvn test -pl tests -Dtest=MongoExamples
```

## Next step

Read the [Quick Start](/docs/tutorials/quickstart/) to understand the setup and apply the same patterns to your own project.

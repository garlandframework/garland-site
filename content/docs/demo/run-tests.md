---
title: "How to Run Tests"
date: 2026-06-10
description: "Start the demo stack and run the test suite"
build:
  list: always
---

## Prerequisites

- Java 17+
- Maven 3.8+
- Docker and Docker Compose

## Start the stack

```bash
git clone https://github.com/garlandframework/garland-demo
cd garland-demo
docker-compose up -d --build
```

This builds and starts both Spring Boot services alongside PostgreSQL, Kafka, and MongoDB. Wait about 15 seconds for services to be ready.

**Swagger UI** — explore the APIs once the stack is running:
- user-service: http://localhost:8080/swagger-ui/index.html
- projection-service: http://localhost:8081/swagger-ui/index.html

---

## Run the full suite

```bash
mvn test -pl tests
```

106 tests run against the live stack. No mocking. The suite acquires an auth token automatically in `@BeforeSuite` and runs a smoke probe before the first test — if the probe fails (infrastructure not ready), the suite aborts immediately with a clear message.

---

## Run a single test class

```bash
mvn test -pl tests -Dtest=CreateUserApiTest
mvn test -pl tests -Dtest=UserEndToEndTest
mvn test -pl tests -Dtest=UserApiToKafkaTest
```

---

## Run example classes

Each Garland client has a dedicated example class with one test per API method and inline explanations. Run them to explore the API against a real stack:

```bash
# HTTP client patterns
mvn test -pl tests -Dtest=HttpExamples

# Postgres patterns
mvn test -pl tests -Dtest=PostgresExamples

# Kafka patterns
mvn test -pl tests -Dtest=KafkaExamples

# MongoDB patterns
mvn test -pl tests -Dtest=MongoExamples
```

---

## Rebuild after code changes

Two scripts are included for resetting the stack without destroying infrastructure manually:

```bash
# Full reset — rebuilds service images, wipes all volumes and data
./restart-clean.sh

# Soft reset — rebuilds service images, keeps database and Kafka data
./restart-clean.sh --soft
```

Use the full reset when you change database schema or Kafka topic configuration. Use the soft reset when you only change application code and want to preserve existing data.

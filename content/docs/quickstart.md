---
title: "Quick Start"
weight: 1
---

## Requirements

- Java 21+
- Maven 3.8+

## Add dependencies

Add the modules you need to your test `pom.xml`:

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-http</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-postgres</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-kafka</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-mongodb</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
```

## Configure clients

```java
HttpTestClient http = new HttpTestClient(
    HttpConfig.builder()
        .baseUrl("http://localhost:8080")
        .build()
);

PostgresTestClient postgres = new PostgresTestClient(
    DbConfig.builder()
        .url("jdbc:postgresql://localhost:5432/mydb")
        .username("user").password("pass")
        .entityPackage("com.example.entities")
        .build()
);
```

## Write a pipeline test

```java
@Test
public void createUser_verifiedInDbAndKafka() {
    UserDto expected = new UserDto("Alice", "alice@example.com");

    Pipeline.given(HttpCallRequest.post("/api/users").withBody(expected))
            .then(http.makeCall(201, UserDto.class))
            .then(Verify.matching(expected))
            .then(UserMapper.toEntity().andThen(postgres.findByFields()))
            .execute();
}
```

## See a full example

Clone the [demo project](https://github.com/garlandframework/garland-demo) — two Spring Boot
microservices with a complete test suite covering HTTP, Postgres, Kafka, and MongoDB.

```bash
git clone https://github.com/garlandframework/garland-demo
cd garland-demo
docker-compose up -d
mvn test -pl tests
```

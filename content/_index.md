---
title: "Garland"
description: "Garland is a Java library for writing integration tests as type-safe pipelines — assert HTTP, PostgreSQL, Kafka, and MongoDB in a single chain, with automatic retry and no mocks"
---

Verifying a Java microservice end-to-end means checking the HTTP response, the database row, the Kafka event, and the MongoDB projection — for real, without mocking any of it. Usually that means four different clients, four assertion styles, and `Thread.sleep` holding it all together. Garland puts all of it in a single type-safe pipeline, with built-in retry instead of sleep calls.

If you know Citrus, think of Garland as the same goal — testing real systems instead of mocks — with a fluent, type-safe Java API instead of XML configuration, and no separate DSL to learn.

{{< button href="/docs/tutorials/try-it/" >}}Try the demo in 5 minutes{{< /button >}}

```java
UserDto expected = TestUsers.defaultUser();

Pipeline.given(TestUserRequests.createUser(expected))
        .then(httpClient.makeCall(201, UserDto.class))   // assert 201, deserialize response
        .then(Verify.matching(expected))                 // assert response body
        .then(Verify.allOf(
                UserMapper.toEntity().andThen(postgresClient.findByFields()),                      // Postgres ✓
                UserMapper.toEvent().andThen(kafkaClient.consumeMatching(UserCreatedEvent.class)), // Kafka ✓
                UserMapper.toDoc().andThen(mongoClient.findByFields())                             // MongoDB ✓
        ))
        .execute();
```

One pipeline. One test. HTTP response, Postgres, Kafka, and MongoDB — all verified.

{{< youtube bN6O8ek2TjQ >}}

## Add to your project

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-http</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>
```

Available modules: `garland-http`, `garland-postgres`, `garland-kafka`, `garland-mongodb`, `garland-testng`

## Links

- [GitHub](https://github.com/garlandframework/garland)
- [Demo project](https://github.com/garlandframework/garland-demo)
- [Maven Central](https://central.sonatype.com/search?q=g:dev.garlandframework)
- [Quick start →](/docs/tutorials/quickstart/)
- [Contact](mailto:garland.test.tool@gmail.com)

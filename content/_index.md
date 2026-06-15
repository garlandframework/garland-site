---
title: "Garland"
description: "Garland is a Java library for writing integration tests as type-safe pipelines — assert HTTP, PostgreSQL, Kafka, and MongoDB in a single chain, with automatic retry and no mocks"
---

Mocking the database, the event broker, and the projection store gives you test coverage that may not survive contact with production. Garland verifies the real stack: HTTP, PostgreSQL, Kafka, and MongoDB in a single type-safe pipeline, no mocks needed.

{{< button href="/docs/tutorials/try-it/" >}}Try the demo in 5 minutes{{< /button >}}
&nbsp;
{{< button href="/docs/tutorials/quickstart/" >}}Quick Start{{< /button >}}

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

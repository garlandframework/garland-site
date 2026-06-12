---
title: "Garland"
description: "Type-safe pipeline composition for Java integration testing"
---

## What is Garland?

Garland is a Java library for writing integration tests as type-safe pipelines.
Each step transforms and passes data to the next. The compiler enforces the chain —
if a step returns the wrong type, the build fails.

```java
UserDto expected = TestUsers.defaultUser();

Pipeline.given(TestUserRequests.createUser(expected))
        .then(httpClient.makeCall(201, UserDto.class))
        .then(Verify.matching(expected))
        .then(Verify.allOf(
                UserMapper.toEntity().andThen(postgresClient.findByFields()),
                UserMapper.toEvent().andThen(kafkaClient.consumeMatching(UserCreatedEvent.class)),
                UserMapper.toDoc().andThen(mongoClient.findByFields())
        ))
        .execute();
```

One pipeline. One test. HTTP response, Postgres, Kafka, and MongoDB — all verified.

{{< button href="/docs/tutorials/try-it/" >}}Try the demo in 5 minutes{{< /button >}}
&nbsp;
{{< button href="/docs/tutorials/quickstart/" >}}Quick Start{{< /button >}}

## Add to your project

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-http</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
```

Available modules: `garland-http`, `garland-postgres`, `garland-kafka`, `garland-mongodb`, `garland-testng`

## Links

- [GitHub](https://github.com/garlandframework/garland)
- [Demo project](https://github.com/garlandframework/garland-demo)
- [Maven Central](https://central.sonatype.com/search?q=g:dev.garlandframework)
- [Quick start →](/docs/tutorials/quickstart/)

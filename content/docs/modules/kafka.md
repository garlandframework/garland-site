---
title: "garland-kafka — Assert Kafka Events in Java"
date: 2026-06-10
description: "garland-kafka reference — assert Kafka events in Java integration tests without Thread.sleep, with automatic retry and message field matching"
build:
  list: local
---

`garland-kafka` provides `KafkaTestClient` — a Kafka consumer and producer wrapped into pipeline steps. It subscribes to one or more topics, retries until the expected event arrives, and asserts the deserialized content. A `publish` step is included for tests that drive the system via Kafka rather than HTTP.

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-kafka</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>
```

---

## How consuming works

{{< mermaid >}}
flowchart TD
    expected["Expected event\n(input from pipeline)"]
    poll["Poll Kafka topic"]
    deser["Deserialize record\nJSON → EventDto"]
    match["Assert event matches expected\nnull fields ignored"]
    retry["Retry — poll next record"]
    event["Event\n✓ returned to pipeline"]

    expected -->|"consumeMatching()"| poll
    poll --> deser
    deser --> match
    match -->|"no match"| retry
    retry --> poll
    match -->|"matched"| event

    style expected fill:#f5f5f5,stroke:#999
    style event fill:#d4edda,stroke:#28a745
    style retry fill:#fff3cd,stroke:#ffc107
{{< /mermaid >}}

---

## Setup

```java
KafkaTestClient kafka = new KafkaTestClient(
        KafkaConfig.builder()
                .bootstrapServers("localhost:9092")
                .topic("user.created")
                .topic("user.updated")
                .topic("user.deleted")
                .groupId(UUID.randomUUID().toString())
                .build(),
        RetryConfig.of(5, Duration.ofSeconds(2))
).withTemporalTolerance(Duration.ofMillis(1));
```

Always use a random UUID for `groupId` — reusing a group ID across suite runs causes the consumer to resume from a previous offset and pick up stale events.

The **first topic** registered via `.topic()` becomes the default topic for `publish()`. Additional topics are only subscribed for consumption.

Call `kafka.close()` in your after-suite hook to close the consumer and producer connections.

---

## warmup

Call `warmup()` once before each test section starts consuming. It seeks all subscribed partitions to their current end so the section reads only events produced after the seek — without it the consumer may start at an earlier offset and pick up events from previous sections.

```java
kafka.warmup();
```

The typical place is a `@BeforeTest` (or equivalent) hook that runs before each group of related tests:

```java
@BeforeTest
public void seekKafkaToLatest() {
    kafka.warmup();
}
```

You do not need to call `warmup()` before every individual `consume` step — one call per test section is enough.

---

## consumeMatching

The standard Kafka assertion pattern. The step input is the expected value. Records are polled with retry until one matches — null fields in the expected value are ignored.

```java
Pipeline.given(createUserRequest())
        .then(http.makeCall(201, UserDto.class))
        .then(UserMapper.toCreatedEvent())          // UserDto → UserCreatedEvent
        .then(kafka.consumeMatching(UserCreatedEvent.class))
        .execute();
```

Use `consumeMatching` rather than `consume(type, expected)` on shared topics — it tolerates interleaved messages by continuing to poll and compare until the retry budget is exhausted. `consume(type, expected)` asserts only the very next record and fails immediately if it does not match.

**Temporal tolerance override for a single call:**

```java
.then(kafka.consumeMatching(UserCreatedEvent.class, Duration.ofSeconds(2)))
```

Use this for SLA-style assertions — asserting a service-generated timestamp is within a 2-second processing window, not just absorbing truncation noise.

---

## consume

**No assertion** — reads the next record and passes it downstream. Use when you want to receive the event and inspect or transform it in a subsequent step:

```java
Pipeline.given((Void) null)
        .then(kafka.consume(UserCreatedEvent.class))
        .then(Verify.matching(new UserCreatedEvent(null, null, null, null, null, "user-service")))
        .execute();
```

**With a fixed expected value** — reads the next record and asserts it matches. Use only when the topic is clean and you know which event arrives next:

```java
UserCreatedEvent expected = new UserCreatedEvent(null, null, null, null, null, "user-service");

Pipeline.given((Void) null)
        .then(kafka.consume(UserCreatedEvent.class, expected))
        .execute();
```

Prefer `consumeMatching` on topics where other events may arrive before the one you expect.

---

## publish

Sends a `KafkaMessage` to the default topic (the first topic in `KafkaConfig`). Use for component tests that drive the system via Kafka rather than HTTP:

```java
UserCreatedEvent event = TestUserEvents.defaultUserCreatedEvent();

Pipeline.given(new KafkaMessage<>(event.userId().toString(), event))
        .then(kafka.publish())
        .execute();
```

`KafkaMessage<T>` takes a string key used for Kafka partitioning — typically the entity ID — and the event value, which is serialized to JSON automatically.

**Publish and assert downstream:**

```java
Pipeline.given(new KafkaMessage<>(event.userId().toString(), event))
        .then(kafka.publish())
        .execute();

Pipeline.given(expectedProjectionDoc)
        .then(mongoClient.findById())
        .execute();
```

Use two separate pipelines for publish and downstream assertion — they are distinct operations with different input types.

---

## Timestamp tolerance

Events often carry timestamps that originated in a database — already truncated to microseconds or milliseconds before reaching Kafka. Set a global tolerance via `withTemporalTolerance` to absorb this without annotating every assertion site:

```java
KafkaTestClient kafka = new KafkaTestClient(config, retryConfig)
        .withTemporalTolerance(Duration.ofMillis(1));
```

Use the `consumeMatching(Class, Duration)` overload to override the tolerance for a specific call — for example, when the tolerance represents an SLA window rather than a precision correction.

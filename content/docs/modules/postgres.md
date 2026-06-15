---
title: "garland-postgres"
date: 2026-06-10
description: "Assert PostgreSQL state in Java integration tests — query, retry, and test data setup via Hibernate"
build:
  list: never
---

`garland-postgres` provides `PostgresTestClient` — a Hibernate-backed client that produces steps for asserting database state inside pipelines. It holds connection configuration and retry policy, and exposes methods like `findById`, `findByFields`, and `countByFields` that each return a `Step` ready to be passed to `.then()`.

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-postgres</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>
```

---

## How a query works

`findById` and `findByFields` do everything in one step: look up the row, wait for it to appear (retry), and assert the entity matches the expected value.

{{< mermaid >}}
flowchart TD
    input["Entity template\n(non-null fields used as criteria)"]
    query["Query Postgres\nHibernate Criteria API"]
    retry["Retry on absence\nuntil found or attempts exhausted"]
    match["Assert entity matches template\nnull fields ignored"]
    entity["Entity\n✓ returned to pipeline"]

    input -->|"findById / findByFields"| query
    query -->|"not found yet"| retry
    retry --> query
    query -->|"found"| match
    match --> entity

    style input fill:#f5f5f5,stroke:#999
    style entity fill:#d4edda,stroke:#28a745
    style retry fill:#fff3cd,stroke:#ffc107
{{< /mermaid >}}

---

## Setup

```java
PostgresWrapper postgres = new PostgresWrapper(
        PostgresConfig.builder()
                .url("jdbc:postgresql://localhost:5432/mydb")
                .username("user")
                .password("pass")
                .entity(UserEntity.class)
                .entity(AddressEntity.class)
                .build()
);

PostgresTestClient db = new PostgresTestClient(postgres, RetryConfig.of(5, Duration.ofSeconds(2)))
        .withTemporalTolerance(Duration.ofNanos(1000));
```

`PostgresConfig` validates at build time — `url`, `username`, and at least one entity class are required. Hibernate validates the live schema against the registered entity list at construction time.

Call `postgres.close()` in your test framework's after-suite hook to release the connection pool.

---

## findById

Looks up the entity by its `@Id` field value and asserts it matches the input. Null fields in the input are ignored during comparison. Retries until the row appears — safe after async writes.

```java
Pipeline.given(createUserRequest())
        .then(http.makeCall(201, UserDto.class))
        .then(UserMapper.toEntity())
        .then(db.findById())
        .execute();
```

**Temporal tolerance override for a single call:**

```java
.then(db.findById(Duration.ofSeconds(1)))
```

Use this when asserting a service-generated timestamp is within an SLA window, rather than just absorbing truncation noise.

---

## findByFields

Builds a `WHERE` clause from all non-null fields of the input, finds the single matching row, and asserts it matches the input. Use when the entity ID is not known before insertion, or when you want to assert by business key.

```java
UserEntity template = UserEntity.builder()
        .name("Alice")
        .surname("Smith")
        .build();

Pipeline.given(template)
        .then(db.findByFields())
        .execute();
```

Throws `IllegalStateException` if more than one row matches — narrow your criteria if needed.

---

## countByFields

Returns the count of rows matching all non-null fields of the input as a `Long`. No assertion is performed by the step itself — chain `Verify.equalTo` to assert the count:

```java
UserEntity template = UserEntity.builder()
        .name("Alice")
        .build();

Pipeline.given(template)
        .then(db.countByFields())
        .then(Verify.equalTo(1L))
        .execute();
```

---

## existsById / notExistsById

Lighter presence and absence checks — no content comparison:

```java
// assert the row is there (retries until found)
.then(db.existsById())

// assert the row is gone (no retry — deletion is synchronous)
.then(db.notExistsById())
```

Both steps return the input entity unchanged, so the pipeline can continue after the check.

For `notExistsById`, pre-compute the entity before the deletion call — you need the ID after the resource is gone:

```java
UserDto created = Pipeline.given(createUserRequest())
        .then(http.makeCall(201, UserDto.class))
        .execute();

UserEntity entity = UserMapper.INSTANCE.toEntity(created); // capture ID before deletion

Pipeline.given(deleteRequest(created.getId()))
        .then(http.makeCall(204, Void.class))
        .execute();

Pipeline.given(entity)
        .then(db.notExistsById())
        .execute();
```

---

## persist

Inserts an entity directly via Hibernate and asserts the stored result matches the expected value. Use for test setup when you need a specific record in the database before the system under test runs.

```java
UserEntity entity = UserEntity.builder()
        .id(UUID.randomUUID())
        .name("Setup")
        .surname("User")
        .build();

Pipeline.given(PostgresRequest.persist(entity))
        .then(db.persist(entity))
        .execute();
```

Direct inserts bypass application logic — no validation, no domain events. Use `findById` after an HTTP call to assert application behaviour; use `persist` only to establish preconditions.

---

## delete

Deletes an entity and asserts it is no longer present:

```java
Pipeline.given(PostgresRequest.delete(UserEntity.class, id))
        .then(db.delete())
        .execute();
```

---

## Timestamp precision

PostgreSQL stores `Instant` with microsecond resolution. Java 9+ `Instant.now()` can carry nanosecond precision — the extra nanoseconds are truncated on write and cause assertion failures on read.

Apply a 1 µs tolerance globally via `withTemporalTolerance`:

```java
PostgresTestClient db = new PostgresTestClient(postgres, retryConfig)
        .withTemporalTolerance(Duration.ofNanos(1000));
```

This covers the truncation difference on every `findById` call without annotating each assertion site. Use the `findById(Duration)` overload to override the tolerance for a specific call.

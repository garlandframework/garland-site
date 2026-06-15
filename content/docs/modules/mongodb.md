---
title: "garland-mongodb"
date: 2026-06-10
description: "Assert MongoDB documents in Java integration tests — query, retry, and test data setup"
build:
  list: local
---

`garland-mongodb` provides `MongoTestClient` — a MongoDB-backed client that produces steps for asserting document state inside pipelines. Its API mirrors `garland-postgres` exactly: the same methods (`findById`, `findByFields`, `countByFields`, `existsById`, `notExistsById`, `persist`, `delete`) with the same semantics — only the store and configuration differ.

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-mongodb</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>
```

---

## Setup

```java
MongoWrapper mongo = new MongoWrapper(
        MongoConfig.builder()
                .connectionString("mongodb://localhost:27017")
                .database("mydb")
                .collection(UserProjectionDoc.class, "users")
                .collection(OrderProjectionDoc.class, "order_projections")
                .build()
);

MongoTestClient db = new MongoTestClient(mongo, RetryConfig.of(10, Duration.ofSeconds(2)))
        .withTemporalTolerance(Duration.ofMillis(1));
```

Each document class must be mapped to its MongoDB collection name via `.collection(Class, String)`. An unmapped class causes an `IllegalArgumentException` at query time, not at construction. The ID field is resolved by field name `"id"` or any field annotated `@Id`.

Call `mongo.close()` in your after-suite hook to release the driver connection.

---

## findById

Finds a document by its ID field and asserts it matches the input. Null fields in the input are ignored. Retries until the document appears — essential for projection documents written asynchronously after a Kafka event.

```java
Pipeline.given(createUserRequest())
        .then(http.makeCall(201, UserDto.class))
        .then(UserMapper.toProjectionDoc())          // UserDto → UserProjectionDoc (expected)
        .then(db.findById())
        .execute();
```

**Temporal tolerance override for a single call:**

```java
.then(db.findById(Duration.ofSeconds(2)))
```

Use this to assert a service-generated timestamp is within an SLA window rather than just absorbing truncation noise.

---

## findByFields

Queries by all non-null fields of the input, finds the single matching document, and asserts it matches. Use when the document ID is not known before the query, or when asserting by a business field:

```java
UserProjectionDoc template = UserProjectionDoc.builder()
        .fullName("Alice Smith")
        .sourceSystem("user-service")
        .build();

Pipeline.given(template)
        .then(db.findByFields())
        .execute();
```

Throws `IllegalStateException` if more than one document matches — narrow your criteria if needed.

---

## countByFields

Returns the count of documents matching all non-null fields as a `Long`. No assertion is performed by the step — chain `Verify.equalTo` to assert the count:

```java
UserProjectionDoc template = UserProjectionDoc.builder()
        .id(userId)
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
// assert the document is there (retries until found)
.then(db.existsById())

// assert the document is gone (no retry — deletion is synchronous)
.then(db.notExistsById())
```

Both steps return the input document unchanged so the pipeline can continue after the check.

For `notExistsById`, pre-compute the document before the operation that removes it:

```java
UserDto created = Pipeline.given(createUserRequest())
        .then(http.makeCall(201, UserDto.class))
        .execute();

UserProjectionDoc doc = UserMapper.INSTANCE.toProjectionDoc(created); // capture ID before deletion

Pipeline.given(deleteRequest(created.getId()))
        .then(http.makeCall(204, Void.class))
        .execute();

Pipeline.given(doc)
        .then(db.notExistsById())
        .execute();
```

---

## persist

Inserts a document directly and asserts the stored result matches the expected value. Use for test setup when you need a specific document in the collection before the system under test runs.

```java
UserProjectionDoc doc = UserProjectionDoc.builder()
        .id(UUID.randomUUID())
        .fullName("Setup User")
        .sourceSystem("test")
        .build();

Pipeline.given(MongoRequest.persist(doc))
        .then(db.persist(doc))
        .execute();
```

Direct inserts bypass the projection pipeline — no Kafka event is produced, no downstream processing occurs. Use `findById` after publishing a Kafka event to assert application behaviour; use `persist` only to establish preconditions.

---

## delete

Deletes a document and asserts it is no longer present:

```java
Pipeline.given(MongoRequest.delete(UserProjectionDoc.class, id))
        .then(db.delete())
        .execute();
```

---

## Timestamp precision

MongoDB stores `Instant` with millisecond resolution. Java 9+ `Instant.now()` can carry nanosecond precision — the sub-millisecond part is truncated on write and causes assertion failures on read.

Apply a 1 ms tolerance globally via `withTemporalTolerance`:

```java
MongoTestClient db = new MongoTestClient(mongo, retryConfig)
        .withTemporalTolerance(Duration.ofMillis(1));
```

Use the `findById(Duration)` overload to override for a specific call when the tolerance represents an SLA window rather than a precision correction.

---
title: "Main Concepts"
description: "Core ideas behind Garland — Pipeline, Step, Verify, Retry, and ResourceTracker"
type: "page"
---

Garland is built on a small number of composable primitives. Understanding them is enough to write any test.

## Pipeline

`Pipeline<I, O>` is an immutable, type-safe step chain. Each call to `.then(step)` returns a new pipeline with an updated output type — the previous instance is never modified.

```java
Pipeline.given(input)
        .then(stepA)   // Pipeline<Input, A>
        .then(stepB)   // Pipeline<Input, B>
        .execute();    // returns B
```

Execution is eager and sequential. The output of each step becomes the input of the next.

→ [Full reference: Pipeline](/docs/modules/core/#pipeline)

---

## Step

`Step<I, O>` is a `@FunctionalInterface`:

```java
(I input, PipelineContext ctx) -> O
```

Steps can transform data (via return value) and share state sideways via the context. Pass method references, lambdas, or adapt a plain function with `Step.lift()`:

```java
static Step<UserDto, UserEntity> toEntity() {
    return Step.lift(mapper::toEntity);
}
```

→ [Full reference: Step](/docs/modules/core/#step)

---


## Verify

`Verify` provides ready-made assertion steps that slot directly into a pipeline.

**`matching` vs `equalTo`** — the most important distinction:

```java
// null fields in expected are ignored — only non-null fields are checked
// right choice for most happy-path assertions
.then(Verify.matching(expected))

// strict — all fields compared including nulls
// use when the expected object is complete and no tolerance is acceptable
.then(Verify.equalTo(expected))
```

**Temporal tolerance** — absorbs timestamp precision loss (e.g. MongoDB truncates nanoseconds to milliseconds):

```java
.then(Verify.matching(expected, Duration.ofMillis(1)))
```

**List assertions:**

```java
// actual list must contain all expected elements (may contain more)
.then(Verify.containsAll(expectedList))

// actual list must not contain any of these elements
.then(Verify.doesNotContain(unexpectedList))
```

**Fan-out** — `allOf` runs every branch against the same input and collects all failures before throwing:

```java
.then(Verify.allOf(
        UserMapper.toEntity().andThen(postgresClient.findByFields()),
        UserMapper.toEvent().andThen(kafkaClient.consumeMatching(UserCreatedEvent.class)),
        UserMapper.toDoc().andThen(mongoClient.findByFields())
))
```

Unlike chaining `.then()` calls — which stop at the first failure — `allOf` always reports every failing branch in a single error message.

→ [Full reference: Verify](/docs/modules/core/#verify)

---

## Retry

`RetryConfig` controls retry behaviour for all test clients:

```java
RetryConfig.of(10, Duration.ofSeconds(1));  // 10 attempts, 1s delay
RetryConfig.attempts(5);                    // 5 attempts, no delay
```

Retry is applied automatically by each test client. It exists to absorb asynchronous propagation delays — Kafka consumers, read-model projections, eventually-consistent endpoints — without manual `Thread.sleep` calls.

→ [Full reference: RetryConfig](/docs/modules/core/#retryconfig)

---

## ResourceTracker

`ResourceTracker<ID>` solves test isolation: it records every resource created during a test and deletes them all after the test completes, regardless of whether the test passed or failed.

Without cleanup, a failed test leaves data in the database that can cause the next test to fail for the wrong reason.

```java
// declare once per test class
ResourceTracker<UUID> tracker = new ResourceTracker<>(id -> userApi.delete(id));

// inside the pipeline — track() is a passthrough step that records the ID
Pipeline.given(createUserRequest())
        .then(httpClient.makeCall(201, UserDto.class))
        .then(tracker.track(UserDto::id))
        .execute();

// in @AfterMethod — deletes all tracked resources via the API
@AfterMethod
public void cleanup() {
    tracker.cleanupAll(log);
}
```

`track(extractor)` is a passthrough step — it extracts the ID from the pipeline value and registers it, then passes the value through unchanged so the chain continues normally.

→ [Full reference: ResourceTracker](/docs/modules/core/#resourcetracker)

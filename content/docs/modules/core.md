---
title: "garland-core — Pipeline, Verify & Retry"
date: 2026-06-09
description: "garland-core reference — type-safe Pipeline DSL, Step, Verify, Retry, and ResourceTracker for Java integration testing without mocks"
build:
  list: always
---

`garland-core` defines the primitives every other module is built on. It has no external dependencies beyond SLF4J for logging.

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-core</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>
```

---

## Pipeline as a chain of typed functions

A pipeline is a sequence of functions where the output type of each function becomes the input type of the next. The compiler enforces that adjacent steps are compatible — if types don't connect, the code doesn't compile.

{{< mermaid >}}
flowchart LR
    I["I\n(input)"]
    A["A"]
    B["B"]
    C["C\n(result)"]

    I -->|"step1: I → A"| A
    A -->|"step2: A → B"| B
    B -->|"step3: B → C"| C

    style I fill:#f5f5f5,stroke:#999
    style C fill:#d4edda,stroke:#28a745
{{< /mermaid >}}

**Fan-out** — `Verify.allOf` runs multiple independent functions against the same input and collects all failures:

{{< mermaid >}}
flowchart LR
    I["I"] -->|"step1: I → A"| A["A"]
    A -->|"step2: A → B"| B["B"]
    B --> allOf{"allOf"}
    allOf -->|"f1: B → ?"| r1["✓"]
    allOf -->|"f2: B → ?"| r2["✓"]
    allOf -->|"f3: B → ?"| r3["✓"]

    style allOf fill:#fff3cd,stroke:#ffc107
    style r1 fill:#d4edda,stroke:#28a745
    style r2 fill:#d4edda,stroke:#28a745
    style r3 fill:#d4edda,stroke:#28a745
{{< /mermaid >}}

---

## Pipeline

`Pipeline<I, O>` is an immutable, type-safe step chain. Each `.then(step)` call returns a new pipeline with an updated output type — the current instance is never modified. Execution is eager and sequential via `.execute()`.

```java
UserDto user = Pipeline.given(createUserRequest())
        .then(httpClient.makeCall(201, UserDto.class))
        .execute();
```

| Method | Description |
|---|---|
| `Pipeline.given(input)` | Creates a new pipeline with the given input |
| `.then(step)` | Appends a step, returns a new `Pipeline` with updated output type |
| `.withContext(ctx)` | Replaces the default context — use when pipelines need to share state |
| `.execute()` | Runs all steps in order, returns the final output |

---

## Step

`Step<I, O>` is a `@FunctionalInterface`: `(I input, PipelineContext ctx) -> O`. Pass method references, lambdas, or use the static factory methods.

A step receives an input and returns an output. The context flows through all steps sideways — available to every step but not part of the main data chain:

{{< mermaid >}}
flowchart LR
    ctx["PipelineContext\n(shared)"]
    input["input\nI"] --> step["step"]
    step --> output["output\nO"]
    ctx -..->|read / write| step

    style ctx fill:#f5f5f5,stroke:#999
    style step fill:#fff3cd,stroke:#ffc107
    style input fill:#f5f5f5,stroke:#999
    style output fill:#d4edda,stroke:#28a745
{{< /mermaid >}}

`andThen` composes two steps into one — the output type of the first must match the input type of the second:

{{< mermaid >}}
flowchart LR
    subgraph composed["f1.andThen(f2) — Step#lt;I, B#gt;"]
        A["f1: I → A"] -->|"A"| B["f2: A → B"]
    end
    I["I"] --> composed
    composed --> O["B"]

    style composed fill:#f0f4ff,stroke:#4a6cf7
    style I fill:#f5f5f5,stroke:#999
    style O fill:#d4edda,stroke:#28a745
{{< /mermaid >}}

| Method | Description |
|---|---|
| `Step.lift(fn)` | Adapts a plain `Function<I, O>` — use when the step doesn't need the context |
| `Step.of(fn)` | Identity wrapper — helps the compiler infer types when starting an `andThen` chain |
| `Step.saveToContext(key)` | Passthrough step that stores the value in context under `key` |
| `.andThen(next)` | Composes two steps into one |

```java
// adapt a plain mapper method into a Step
static Step<UserDto, UserEntity> toEntity() {
    return Step.lift(mapper::toEntity);
}

// compose two steps into one
Step<UserDto, UserEntity> composed = toEntity().andThen(db.findByFields());
```

---

## Verify

Static factory for assertion steps. Each method returns a `Step` that asserts a condition and passes the value through unchanged on success.

| Method | Description |
|---|---|
| `Verify.matching(expected)` | Non-null fields of `expected` must match actual — null fields are ignored |
| `Verify.matching(expected, duration)` | Same, with temporal tolerance for timestamp fields |
| `Verify.equalTo(expected)` | Strict — all fields compared including nulls |
| `Verify.containsAll(list)` | Actual list must contain all expected elements |
| `Verify.doesNotContain(list)` | Actual list must not contain any of the unexpected elements |
| `Verify.allOf(branches...)` | Runs all branches, collects all failures, throws a combined error |

**`matching` vs `equalTo`:** use `matching` for most assertions — it lets you build a partial expected object with only the fields you care about. Use `equalTo` only when you need to assert that certain fields are `null`.

---

## RetryConfig

Controls how test clients retry failing operations. Retry is transparent to the pipeline — the chain looks identical whether a step succeeds on the first attempt or the fifth.

```java
RetryConfig.of(10, Duration.ofSeconds(1));  // 10 attempts, 1s delay between each
RetryConfig.attempts(5);                    // 5 attempts, no delay
```

{{< mermaid >}}
flowchart LR
    I["I\n(input)"]
    A["A"]
    B["B"]
    C["C\n(result)"]

    I -->|"step1: I → A"| A
    A -->|"step2: A → B"| B
    B -->|"step3: B → C"| C

    B -->|"retry on failure"| B

    style I fill:#f5f5f5,stroke:#999
    style C fill:#d4edda,stroke:#28a745
    style B fill:#fff3cd,stroke:#ffc107
{{< /mermaid >}}

The retrying step is highlighted — from the pipeline's perspective it is just another `step2: A → B`. Whether it attempts once or ten times is an implementation detail of the step itself, configured via `RetryConfig`.

---

## ResourceTracker

Tracks resource IDs created during a test and deletes them all after the test completes, regardless of whether the test passed or failed. Without cleanup, a failed test leaves data behind that can cause subsequent tests to fail for the wrong reason.

```java
// declare once per test class — provide the delete function
ResourceTracker<UUID> tracker = new ResourceTracker<>(id -> userApi.delete(id));

// inside the pipeline — records the ID, passes the value through unchanged
.then(tracker.track(UserDto::id))

// call in your test framework's after-each hook to clean up all tracked resources
tracker.cleanupAll(log);
```

`track(extractor)` is a passthrough step — it extracts the ID from the current pipeline value, registers it for later deletion, and returns the value unchanged so the chain continues normally. `cleanupAll` silently skips any resource that fails to delete and logs a warning instead of failing the cleanup.

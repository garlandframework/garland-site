---
title: "Demo Project Overview"
description: "garland-demo — two Spring Boot microservices with a complete Garland test suite"
type: "page"
---

[garland-demo](https://github.com/garlandframework/garland-demo) is a reference project that shows what a real Garland test suite looks like end-to-end. It is not a toy — it covers a realistic microservice architecture with authentication, async event propagation, and read-model projections.

{{< button href="https://github.com/garlandframework/garland-demo" target="_blank" >}}Download Demo{{< /button >}}

---

## System architecture

```
                     ┌─────────────────────────────┐
                     │      user-service :8080      │
                     │                              │
  HTTP client ─────► │  /api/users    PostgreSQL    │
                     │  /api/orders   (userdb)      │
                     │  /api/auth                   │
                     └─────────────┬────────────────┘
                                   │
                           Kafka topics:
                           user.created / user.updated / user.deleted
                           order.placed / order.cancelled
                                   │
                     ┌─────────────▼──────────────────┐
                     │    projection-service :8081     │
                     │                                 │
                     │  /api/projections   MongoDB     │
                     │                  (projectiondb) │
                     └─────────────────────────────────┘
```

**user-service** owns the write side: it persists users and orders to PostgreSQL and publishes domain events to Kafka after every mutation.

**projection-service** owns the read side: it consumes those events and builds denormalized documents in MongoDB that the UI would query.

The test suite covers both services. Tests target the system from the outside — no mocks, no internal state, no direct database setup through the application layer.

---

## What's in the suite

- **Users domain** — a complete reference test suite: endpoint tests, flow tests, component tests, and end-to-end tests. This is the domain to read and learn from.
- **Orders domain** — the API and services are fully implemented but no tests exist. This is intentional — it is the blank canvas for [generating tests with Claude Code](/docs/demo/generate-tests/).
- **Example classes** — one file per Garland client (`PostgresExamples`, `KafkaExamples`, `MongoExamples`, `HttpExamples`), each containing one test per API method with inline explanations.

---

## Explore further

- [How to Run Tests](/docs/demo/run-tests/) — start infrastructure, run the full suite, run individual classes
- [How to Generate Tests](/docs/demo/generate-tests/) — generate tests for the orders domain using Claude Code
- [Test layers](/docs/demo/test-layers/) — how the suite is structured across endpoint, flow, component, and end-to-end tests
- [Support layer](/docs/demo/support-layer/) — factories, mappers, shared base test, and connection config

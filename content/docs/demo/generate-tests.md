---
title: "How to Generate Tests"
date: 2026-06-10
description: "Generate tests for the orders domain using Claude Code slash commands"
build:
  list: never
---

The demo ships with Claude Code slash commands that generate Garland tests. The orders domain — a fully implemented API with no tests — is the blank canvas to practice with.

## What you need

- [Claude Code](https://claude.ai/code) CLI or IDE extension (VS Code / JetBrains)
- The demo project open as your working directory

The generation commands are pre-installed in the repo at `.claude/commands/`. Claude Code picks them up automatically.

---

## The orders domain

`user-service` has a complete orders API:

| Endpoint | Description |
|---|---|
| `POST /api/orders` | Place an order |
| `GET /api/orders/{id}` | Get order by ID |
| `PUT /api/orders/{id}/cancel` | Cancel an order |

`projection-service` consumes `order.placed` and `order.cancelled` Kafka events and builds order projection documents in MongoDB. The full stack is running — it just has no tests.

The users test suite is the reference for what good tests look like. The generation commands know the project layout, factories, mappers, clients, and rules — they produce output that matches the existing style.

---

## Generation commands

### `/gen-endpoint-test`

Generates a TestNG endpoint test class. One class per endpoint, covering the happy path and all validation and error cases.

```
/gen-endpoint-test PlaceOrderApiTest — full suite
/gen-endpoint-test CancelOrderApiTest — happy path and 404
/gen-endpoint-test GetOrderApiTest — happy path and not found
```

### `/gen-flow-test`

Generates a flow test class covering state transitions across a sequence of endpoint calls within one service.

```
/gen-flow-test OrderFlowTest — place then cancel lifecycle
/gen-flow-test OrderFlowTest — full CRUD sequence
```

### `/gen-component-test`

Generates component tests that verify a single service boundary end-to-end.

```
/gen-component-test OrderApiToKafkaTest — place order publishes event and persists to Postgres
/gen-component-test KafkaToOrderProjectionTest — order event projected to MongoDB
```

### `/gen-e2e-test`

Generates an end-to-end test class that walks the full chain: HTTP → Postgres → Kafka → MongoDB.

```
/gen-e2e-test OrderEndToEndTest — place and cancel flows
```

---

## How to run

Open the demo project in Claude Code and type a command with a description:

```
/gen-endpoint-test PlaceOrderApiTest — full suite
```

Claude Code reads the existing users test suite as a reference, applies the project-specific rules baked into the command, and generates a complete test class in the correct package with the correct imports, factories, and cleanup.

Run the generated tests immediately:

```bash
mvn test -pl tests -Dtest=PlaceOrderApiTest
```

---

## What the commands know

Each command comes pre-loaded with the project's conventions:

- Package structure and class naming
- Which clients are available (`httpClient`, `postgresClient`, `kafkaClient`, `orderKafkaClient`, `mongoClient`)
- All factory methods (`TestOrderRequests`, `TestOrders`, `TestOrderItems`, `TestOrderEvents`)
- Mapper bridge methods for the orders domain
- Cleanup rules (`trackOrder()` after every `makeCall(201, OrderDto.class)`)
- Error DTO shapes (`ValidationErrorDto` for 400, `ErrorDto` for everything else)
- When to use `Verify.allOf` vs sequential `.then()` calls
- Cross-domain FK handling (orders reference a user — create the user first in happy-path tests)

The generated output requires no manual editing for structure or imports — it compiles and runs as-is.

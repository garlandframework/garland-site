---
title: "Module Reference"
description: "Java integration testing modules — HTTP, PostgreSQL, Kafka, MongoDB, TestNG reference"
type: "page"
---

| Module | Description |
|---|---|
| [garland-core](/docs/modules/core/) | The foundation. Defines how tests are structured — pipelines, steps, assertions, and retry. Every other module builds on top of this. |
| [garland-http](/docs/modules/http/) | Make HTTP calls, assert status codes, deserialize responses, and handle auth — all in a single step. |
| [garland-postgres](/docs/modules/postgres/) | Query and assert rows in PostgreSQL. Find by ID, find by field values, check absence, count matches. |
| [garland-kafka](/docs/modules/kafka/) | Consume and publish Kafka messages in tests. Handles async delays automatically — no Thread.sleep needed. |
| [garland-mongodb](/docs/modules/mongodb/) | Query and assert MongoDB documents. Same API shape as the Postgres module. |
| [garland-testng](/docs/modules/testng/) | Optional add-on for TestNG users. Provides a base test class with structured lifecycle logging and resource cleanup. |

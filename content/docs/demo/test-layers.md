---
title: "Test layers"
date: 2026-06-10
description: "How the demo test suite is structured across endpoint, flow, component, and end-to-end tests"
build:
  list: local
---

The demo suite is organized into four layers. Each layer tests a different slice of the system and answers a different question. Tests live in `tests/src/test/java/dev/garlandframework/demo/tests/`.

| Layer | Package | Question |
|---|---|---|
| Endpoint | `users/endpoint/` | Does this endpoint behave correctly in isolation? |
| Flow | `users/flow/` | Are state transitions within one service consistent? |
| Component | `users/component/` | Does this service boundary work end-to-end? |
| E2E | `UserEndToEndTest` | Does the full system chain hold together? |

---

## Endpoint tests

One test class per endpoint. Each class covers the happy path plus all validation and error cases for that endpoint. Fast to run, narrow in scope — a failure here points directly to the endpoint and nothing else.

```java
@Test(description = "POST /api/users with valid data returns 201 and user is persisted in Postgres")
public void createUser_persistedInDb() {
    HttpCallRequest<UserDto> request = TestUserRequests.createUser();

    Pipeline.given(request)
            .then(httpClient.makeCall(201, UserDto.class))
            .then(trackUser())
            .then(Verify.matching(request.dto()))
            .then(UserTestMapper.toEntity())
            .then(postgresClient.findByFields())
            .execute();
}

@Test(description = "Blank name is rejected with 400 and error pointing to 'name' field")
public void createUser_blankName_returns400() {
    Pipeline.given(TestUserRequests.createUser(TestUsers.builder().name("").build()))
            .then(httpClient.makeCall(400, ValidationErrorDto.class))
            .then(Verify.matching(ValidationErrorDto.forField("name")))
            .execute();
}
```

---

## Flow tests

Tests that exercise state transitions across a sequence of calls within one service, using only the HTTP API. No Kafka, no MongoDB — the boundary is the service itself.

Flows answer questions like: does a created user appear in the list? Does deleting a user remove it from subsequent GET calls? Does updating one user leave another unchanged?

```java
@Test(description = "Created user appears in the full user list")
public void createThenGetAll_listContainsUser() {
    UserDto created = Pipeline.given(TestUserRequests.createUser())
            .then(httpClient.makeCall(201, UserDto.class))
            .then(trackUser())
            .execute();

    Pipeline.given(TestUserRequests.getAllUsers())
            .then(httpClient.makeCall(200, new TypeReference<List<UserDto>>() {}))
            .then(Verify.containsAll(List.of(created)))
            .execute();
}

@Test(description = "Deleting one user does not remove other users from the list")
public void createTwo_deleteOne_thenGetAll_otherStillPresent() {
    UserDto userA = Pipeline.given(TestUserRequests.createUser())
            .then(httpClient.makeCall(201, UserDto.class))
            .then(trackUser())
            .execute();
    UserDto userB = Pipeline.given(TestUserRequests.createUser())
            .then(httpClient.makeCall(201, UserDto.class))
            .then(trackUser())
            .execute();

    Pipeline.given(TestUserRequests.deleteUser(userA.getUuid()))
            .then(httpClient.makeCall(204, Void.class))
            .execute();

    Pipeline.given(TestUserRequests.getAllUsers())
            .then(httpClient.makeCall(200, new TypeReference<List<UserDto>>() {}))
            .then(Verify.containsAll(List.of(userB)))
            .execute();
}
```

---

## Component tests

Component tests verify a single service boundary end-to-end. They split cleanly along team ownership — each test class covers exactly one service's responsibility.

**`UserApiToKafkaTest`** — verifies user-service: HTTP request in, Postgres persisted and Kafka event published out:

```java
@Test
public void createUser_persistedInDb_andPublishesKafkaEvent() {
    UserDto expected = TestUsers.defaultUser();

    Pipeline.given(TestUserRequests.createUser(expected))
            .then(httpClient.makeCall(201, UserDto.class))
            .then(Verify.matching(expected))
            .then(trackUser())
            .then(Verify.allOf(
                    UserTestMapper.toEntity().andThen(postgresClient.findByFields()),
                    UserTestMapper.toCreatedEvent().andThen(kafkaClient.consumeMatching(UserCreatedEvent.class))
            ))
            .execute();
}
```

**`KafkaToProjectionTest`** — verifies projection-service: Kafka event in, MongoDB projection out. user-service is not involved:

```java
@Test
public void userCreatedEvent_projectedToMongo() {
    UserCreatedEvent event = TestUserEvents.defaultUserCreatedEvent();

    Pipeline.given(new KafkaMessage<>(event.userId().toString(), event))
            .then(kafkaClient.publish())
            .execute();

    Pipeline.given(UserTestMapper.INSTANCE.toProjectionDoc(event))
            .then(mongoClient.findById())
            .execute();
}
```

This split means a failing `KafkaToProjectionTest` points at projection-service; a failing `UserApiToKafkaTest` points at user-service. Neither class can produce a false positive for the other service.

---

## End-to-end tests

A single pipeline that drives the full system — HTTP call through user-service, Kafka event, MongoDB projection — and verifies all three in one assertion using `Verify.allOf`:

```java
@Test
public void createUser_fullSystemFlow() {
    UserDto expected = TestUsers.defaultUser();

    Pipeline.given(TestUserRequests.createUser(expected))
            .then(httpClient.makeCall(201, UserDto.class))
            .then(Verify.matching(expected))
            .then(trackUser())
            .then(Verify.allOf(
                    UserTestMapper.toEntity().andThen(postgresClient.findByFields()),
                    UserTestMapper.toCreatedEvent().andThen(kafkaClient.consumeMatching(UserCreatedEvent.class)),
                    UserTestMapper.dtoToCreatedProjectionDoc().andThen(mongoClient.findByFields())
            ))
            .execute();
}
```

`Verify.allOf` runs all three branches against the same input and collects every failure before throwing — a single test run surfaces all broken assertions at once.

---

## The orders domain

The orders API is fully implemented in user-service (`POST /api/orders`, `GET /{id}`, `PUT /{id}/cancel`), and projection-service projects order events to MongoDB. No tests exist for this domain.

This is intentional — the orders domain is the blank canvas for generating tests. The users test suite is the reference for what the generated output should look like.

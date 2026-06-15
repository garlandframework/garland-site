---
title: "How to test HTTP endpoints with Garland"
date: 2026-06-10
weight: 2
description: "How to write happy-path and negative HTTP tests in Java with Garland — status assertion, body matching, validation errors, auth failures"
build:
  list: local
---

This page covers how to test HTTP endpoints with `HttpTestClient` — from a basic status assertion to auth failures and validation error checking. All examples are taken from [garland-demo](https://github.com/garlandframework/garland-demo) and run against the real stack — clone it to see them execute.

These examples assume test DTOs are already in place. If you are starting from scratch, read [How to create test DTOs](/docs/how-to/test-dtos/) first.

---

## Assert status and deserialize the response

The most common pattern. `makeCall` asserts the status code and deserializes the response body in one step:

```java
UserDto created = Pipeline.given(TestUserRequests.createUser())
        .then(httpClient.makeCall(201, UserDto.class))
        .execute();
```

The pipeline returns a fully typed `UserDto`. Use it directly in follow-up assertions or requests.

## Assert response body shape in the same step

Pass an `HttpCallResponse` when you want to assert status and body together. Null fields in the expected object are ignored — only the non-null ones are compared:

```java
UserDto request = TestUsers.defaultUser();
UserDto expected = UserDto.builder()
        .name(request.getName())
        .surname(request.getSurname())
        .build(); // uuid, createdAt left null → not compared

Pipeline.given(TestUserRequests.createUser(request))
        .then(httpClient.makeCall(new HttpCallResponse<>(201, Map.of(), expected)))
        .execute();
```

This keeps the assertion in one place without a separate `Verify.matching` step.

## Deserialize a generic response type

Use `TypeReference` when the response type is generic. A plain `Class<List>` erases the element type and breaks deserialization at runtime:

```java
List<UserDto> users = Pipeline.given(TestUserRequests.getAllUsers())
        .then(httpClient.makeCall(200, new TypeReference<List<UserDto>>() {}))
        .execute();
```

---

## Negative tests

### 404 — resource not found

Same `makeCall` pattern, different status code and error DTO:

```java
Pipeline.given(TestUserRequests.getUser(UUID.randomUUID()))
        .then(httpClient.makeCall(404, ErrorDto.class))
        .then(Verify.matching(ErrorDto.withStatus(404)))
        .execute();
```

`ErrorDto.withStatus(404)` builds a partial expected object — `Verify.matching` skips null fields, so only the status code is compared.

### 401 — missing or invalid token

`withoutHeader` returns a new client with the named header removed. The original client is unchanged — every other test in the suite keeps its token:

```java
// no token
Pipeline.given(TestUserRequests.createUser())
        .then(httpClient.withoutHeader("Authorization")
                .makeCall(new HttpCallResponse<>(401, Map.of(), ErrorDto.withStatus(401))))
        .execute();

// invalid token
Pipeline.given(TestUserRequests.createUser())
        .then(httpClient.withBearer("not-a-valid-jwt")
                .makeCall(new HttpCallResponse<>(401, Map.of(), ErrorDto.withStatus(401))))
        .execute();
```

Both tests share the suite-wide `httpClient` without modifying it.

### 400 — validation errors

The key is a `ValidationErrorDto` that mirrors your service's error shape, and a `forField` factory that builds a partial expected object pointing at a specific field:

```java
// blank required field
Pipeline.given(TestUserRequests.createUser(TestUsers.builder().name("").build()))
        .then(httpClient.makeCall(400, ValidationErrorDto.class))
        .then(Verify.matching(ValidationErrorDto.forField("name")))
        .execute();

// field too long
Pipeline.given(TestUserRequests.createUser(TestUsers.builder().name("a".repeat(101)).build()))
        .then(httpClient.makeCall(400, ValidationErrorDto.class))
        .then(Verify.matching(ValidationErrorDto.forField("name")))
        .execute();

// nested field — address.street
UserDto user = TestUsers.builder()
        .address(TestAddresses.builder().street("").build())
        .build();
Pipeline.given(TestUserRequests.createUser(user))
        .then(httpClient.makeCall(400, ValidationErrorDto.class))
        .then(Verify.matching(ValidationErrorDto.forField("address.street")))
        .execute();
```

`ValidationErrorDto.forField("name")` produces an object with `status=400` and a single violation pointing at `name`. The message field is null, so `Verify.matching` ignores it — the test asserts the field was rejected without coupling to the exact message text.

If you need to assert the full error including the message, construct the expected object with all fields populated:

```java
Pipeline.given(TestUserRequests.createUser(TestUsers.builder().name("").build()))
        .then(httpClient.makeCall(400, ValidationErrorDto.class))
        .then(Verify.matching(new ValidationErrorDto(400,
                List.of(new FieldViolationDto("name", "must not be blank")))))
        .execute();
```

`Verify.matching` compares every non-null field — including the message — so the test will fail if the wording changes. Use the full form when the message content is part of the contract; use `forField` when only the field path matters.

### 400 — invalid query parameter

Pass bad values directly at the call site. The factory produces the valid base request; the test overrides a single parameter:

```java
Pipeline.given(TestUserRequests.getAllUsers().withQueryParam("page", "-1"))
        .then(httpClient.makeCall(400, ErrorDto.class))
        .execute();
```

---

## Continue the pipeline after a void step

DELETE returns `Void`, which would sever the type chain. Capture the value you need before the void step, then reinject it with a lambda:

```java
UserDto created = Pipeline.given(TestUserRequests.createUser())
        .then(httpClient.makeCall(201, UserDto.class))
        .execute();

Pipeline.given(TestUserRequests.deleteUser(created.getUuid()))
        .then(httpClient.makeCall(204, Void.class))
        .then((ignored, ctx) -> TestUserRequests.getUser(created.getUuid()))
        .then(httpClient.makeCall(404, ErrorDto.class))
        .execute();
```

The second pipeline deletes the user and immediately verifies it is gone — two related assertions in one logical sequence.

---

## Further reading

- [garland-http module reference](/docs/modules/http/) — full API including polling, auth, multipart, file download, and timeout
- [Demo project](https://github.com/garlandframework/garland-demo) — `HttpExamples`, `CreateUserApiTest`, `AuthTest` contain all patterns shown here as runnable tests

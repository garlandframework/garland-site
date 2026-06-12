---
title: "garland-http"
date: 2026-06-09
description: "Test HTTP APIs in Java integration tests — status assertion, deserialization, auth, polling, and retry"
build:
  list: never
---

`garland-http` provides `HttpTestClient` — a configured HTTP client that produces steps for use in pipelines. It holds configuration (base URL, default headers, retry policy) and exposes methods like `makeCall`, `pollingCall`, and `downloadFile` that each return a `Step` ready to be passed to `.then()`.

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-http</artifactId>
    <version>1.0.0</version>
    <scope>test</scope>
</dependency>
```

---

## How a call works

A single `makeCall` step does everything: sends the request, checks the status, deserializes the body, and optionally asserts the response matches an expected value.

{{< mermaid >}}
flowchart TD
    req["HttpCallRequest\n───────────────\nurl · method · headers\nbody · query params"]
    send["Send HTTP request"]
    status["Assert status code\n e.g. 201 == 201 ✓"]
    deser["Deserialize response body\nJSON → ResponseDto"]
    match["Match against expected\nnull fields ignored · optional"]
    dto["ResponseDto\n✓ returned to pipeline"]

    req -->|"makeCall()"| send
    send --> status
    status --> deser
    deser --> match
    match --> dto

    style req fill:#f5f5f5,stroke:#999
    style dto fill:#d4edda,stroke:#28a745
    style match fill:#fff3cd,stroke:#ffc107
{{< /mermaid >}}

---

## HttpCallRequest

`HttpCallRequest<T>` carries all parameters for a single HTTP call. Build it via static factory methods — not inline in tests:

```java
HttpCallRequest.get("/api/users/123")
HttpCallRequest.post("/api/users", userDto)
HttpCallRequest.put("/api/users/123", updateDto)
HttpCallRequest.delete("/api/users/123")
```

**Query parameters:**

```java
// add a single param
request.withQueryParam("page", "0")

// add several at once
request.withQueryParams(Map.of("page", "0", "size", "10"))
```

---

## HttpCallResponse

`HttpCallResponse<T>` describes the expected response for assertion:

```java
// status + body assertion (no header assertions)
new HttpCallResponse<>(201, Map.of(), expectedDto)

// status + specific response header assertion
new HttpCallResponse<>(201, Map.of("Content-Type", List.of("application/json")), expectedDto)
```

- **status** — asserted as exact match
- **headers** — asserted as subset; actual response may contain additional headers
- **dto** — null fields ignored during comparison (same as `Verify.matching`)

---

## makeCall

`makeCall` has two main forms:

```java
// assert status + match response body (null fields in expected are ignored)
.then(http.makeCall(new HttpCallResponse<>(201, Map.of(), expectedDto)))

// assert status only, return deserialized body
.then(http.makeCall(400, ErrorDto.class))
```

**Generic response types** (e.g. `List<UserDto>`) use `TypeReference` to avoid type erasure:

```java
.then(http.makeCall(200, new TypeReference<List<UserDto>>() {}))
```

**Temporal tolerance** for timestamp fields in the response:

```java
.then(http.makeCall(new HttpCallResponse<>(201, Map.of(), expectedDto), Duration.ofMillis(1)))
```

---

## pollingCall

Retries until the response body matches. Use for endpoints that expose eventually-consistent state populated by async consumers — no `Thread.sleep` needed:

```java
.then(http.pollingCall(200, expectedDto, RetryConfig.of(10, Duration.ofSeconds(1))))
```

---

## Auth

**Client-level** — token known before the pipeline. Returns a new client instance; the original is not modified:

```java
HttpTestClient authed = http.withBearer(token);
HttpTestClient withKey = http.withApiKey("X-Api-Key", key);
HttpTestClient withCookie = http.withCookie("session", value);
```

**Context-level** — token fetched inside the pipeline:

```java
Pipeline.given(loginRequest())
        .then(http.makeCall(200, TokenDto.class))
        .then(HttpTestClient.storeBearer(TokenDto::accessToken))
        .then(__ -> protectedRequest())
        .then(http.makeCall(201, ResultDto.class))  // Authorization injected automatically
        .execute();
```

Client-level headers always take precedence over context — a suite-wide token set via `withBearer` cannot be overridden mid-pipeline.

**Negative auth tests** — send a request without the suite-wide token:

```java
.then(http.withoutHeader("Authorization").makeCall(401, ErrorDto.class))
```

---

## Client configuration

All `with*` methods return a new client instance — the original is never modified:

| Method | Description |
|---|---|
| `withBaseUrl(url)` | Prepends `url` to any request path starting with `/` |
| `withTimeout(duration)` | Per-request timeout — throws if server does not respond in time |
| `withBearer(token)` | Sets `Authorization: Bearer <token>` on all requests |
| `withHeader(name, value)` | Sets a custom default header |
| `withoutHeader(name)` | Removes a header — useful for negative auth tests |
| `withApiKey(header, key)` | Sets an API key header |
| `withCookie(name, value)` | Sets a `Cookie` header |

---

## FormBody

For `application/x-www-form-urlencoded` requests — typically OAuth2 token endpoints and legacy APIs. `Content-Type` is set automatically:

```java
HttpCallRequest<FormBody> request = new HttpCallRequest<>(
        "/oauth/token",
        "POST",
        List.of(),
        new FormBody()
                .field("grant_type", "client_credentials")
                .field("client_id", "my-client")
                .field("client_secret", "secret")
);

Pipeline.given(request)
        .then(http.makeCall(200, TokenDto.class))
        .execute();
```

---

## Raw string body

Pass a `String` directly as the body to skip Jackson serialization. `Content-Type: application/json` is set automatically. Use when you need to send pre-built JSON or test malformed payloads:

```java
HttpCallRequest<String> request = new HttpCallRequest<>(
        "/api/users",
        "POST",
        List.of(),
        "{\"name\": \"Alice\", \"email\": \"alice@example.com\"}"
);
```

---

## uploadFile

File uploads use `MultipartBody` as the request body. `Content-Type: multipart/form-data` is set automatically:

```java
HttpCallRequest<MultipartBody> request = new HttpCallRequest<>(
        "/api/files/upload",
        "POST",
        List.of(),
        new MultipartBody()
                .field("description", "profile photo")
                .file("photo", Path.of("/tmp/photo.jpg"), "image/jpeg")
);

Pipeline.given(request)
        .then(http.makeCall(200, UploadResultDto.class))
        .execute();
```

Multiple files and fields can be chained — `MultipartBody` is immutable, each call returns a new instance:

```java
new MultipartBody()
        .field("type", "report")
        .file("file1", Path.of("/tmp/a.pdf"), "application/pdf")
        .file("file2", Path.of("/tmp/b.pdf"), "application/pdf")
```

---

## downloadFile

Downloads binary response to a local path:

```java
Path file = Pipeline.given(TestFileRequests.downloadReport("123"))
        .then(http.downloadFile(200, Path.of("/tmp/report.pdf")))
        .execute();
```

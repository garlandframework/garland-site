---
title: "Support layer"
date: 2026-06-10
description: "Factories, mappers, shared base test, and connection config in the demo test suite"
build:
  list: local
---

The support layer lives in `tests/src/test/java/dev/garlandframework/demo/tests/support/`. It contains everything the test classes depend on but does not itself contain any test methods.

---

## Connections

`Connections.java` is the single place that holds all infrastructure coordinates — service URLs, database credentials, Kafka topic names. Test classes never hardcode URLs or credentials; they reference constants from this class.

```java
public static final String USER_SERVICE_URL       = "http://localhost:8080";
public static final String PROJECTION_SERVICE_URL = "http://localhost:8081";

public static final String PG_URL      = "jdbc:postgresql://localhost:5432/userdb?TimeZone=UTC";
public static final String PG_USERNAME = "user";

public static final String KAFKA_BOOTSTRAP_SERVERS  = "localhost:9092";
public static final String KAFKA_TOPIC_USER_CREATED = "user.created";
public static final String KAFKA_TOPIC_USER_UPDATED = "user.updated";
public static final String KAFKA_TOPIC_USER_DELETED = "user.deleted";

public static final String MONGO_CONNECTION_STRING = "mongodb://localhost:27017";
public static final String MONGO_DATABASE          = "projectiondb";
```

Centralizing this means changing the target environment is a one-file edit.

---

## Factories (Object Mother pattern)

Each domain has a set of factory classes — `TestUsers`, `TestUserRequests`, `TestAddresses`, `TestCars`, `TestUserEvents` — that produce test data and HTTP requests. Test classes call these factories rather than constructing objects inline.

**`TestUsers`** generates random but valid domain objects using [Datafaker](https://www.datafaker.net/):

```java
public static UserDto defaultUser() {
    return builder().build();
}

public static UserDto.UserDtoBuilder builder() {
    return UserDto.builder()
            .name(faker.name().firstName())
            .surname(faker.name().lastName())
            .address(TestAddresses.defaultAddress())
            .cars(List.of(TestCars.defaultCar()));
}
```

Every test gets a unique random user — no hardcoded names that could collide across parallel runs or leaked test data from previous runs.

**`TestUserRequests`** wraps `HttpCallRequest` construction so test classes never build request objects inline:

```java
public static HttpCallRequest<UserDto> createUser() {
    return createUser(TestUsers.defaultUser());
}

public static HttpCallRequest<UserDto> createUser(UserDto dto) {
    return HttpCallRequest.post(Connections.USER_SERVICE_URL + "/api/users", dto);
}

public static HttpCallRequest<Void> deleteUser(UUID id) {
    return HttpCallRequest.delete(Connections.USER_SERVICE_URL + "/api/users/" + id);
}
```

Keeping request construction in one place means a URL or method change is a single-file edit, not a search-and-replace across the test suite.

---

## Mappers

`UserTestMapper` is a MapStruct `@Mapper` that converts between the models each system uses. A single `UserDto` response from the HTTP layer needs to become a `UserEntity` for Postgres, a `UserCreatedEvent` for Kafka, and a `UserProjectionDoc` for MongoDB.

The mapper interface declares the field-level conversions:

```java
@Mapping(source = "uuid", target = "id")
UserEntity toEntity(UserDto dto);

@Mapping(source = "uuid", target = "userId")
@Mapping(target = "fullName", expression = "java(dto.getName() + \" \" + dto.getSurname())")
@Mapping(target = "eventTimestamp", ignore = true)
@Mapping(target = "sourceSystem", constant = "user-service")
UserCreatedEvent toCreatedEvent(UserDto dto);
```

Fields set by the production system — `eventTimestamp`, computed fields like `fullName` — are either ignored or declared as constants. `Verify.matching` handles the rest: null fields in the expected object are skipped during comparison, so only the fields the test explicitly set are checked.

**Pipeline bridge methods** wrap each mapper method in `Step.lift` so it plugs directly into `.then()`:

```java
static Step<UserDto, UserEntity> toEntity() {
    return Step.lift(INSTANCE::toEntity);
}

static Step<UserDto, UserCreatedEvent> toCreatedEvent() {
    return Step.lift(INSTANCE::toCreatedEvent);
}
```

---

## BaseTest

`BaseTest` is the abstract class all test classes extend. It wires together all clients, handles suite-level setup, and connects `ResourceTracker` to TestNG's `@AfterMethod`.

**Client setup** happens once in `@BeforeSuite`. The suite authenticates first and stores the token on the HTTP client — all subsequent requests carry the Bearer header automatically:

```java
httpClient = new HttpTestClient(RetryConfig.of(3, Duration.ofSeconds(2)));
TokenDto token = Pipeline.given(TestAuthRequests.login())
        .then(httpClient.makeCall(200, TokenDto.class))
        .execute();
httpClient = httpClient.withBearer(token.token());
```

**Resource cleanup** uses `AbstractGarlandBaseTest` from `garland-testng`. Two trackers — `userTracker` and `orderTracker` — are registered in the constructor. TestNG calls `cleanupAll` after every test method via the `@AfterMethod` wired in the base class:

```java
protected final ResourceTracker<UUID> userTracker = new ResourceTracker<>(
        id -> Pipeline.given(TestUserRequests.deleteUser(id))
                      .then(httpClient.makeCall(204, Void.class))
                      .execute()
);

protected BaseTest() {
    registerTrackers(orderTracker, userTracker);
}
```

Cleanup runs through the API — not by direct database deletion — so it exercises the delete endpoint and keeps test data honest.

**Smoke probe** runs at the end of `@BeforeSuite`. It creates a user, waits for the MongoDB projection to appear with a generous retry budget, then deletes the user. If the probe fails, the suite aborts immediately with a clear message rather than running hundreds of tests that will all fail for infrastructure reasons:

```java
private static void runSmokeProbe() {
    UserDto user = Pipeline.given(TestUserRequests.createUser())
            .then(httpClient.makeCall(201, UserDto.class))
            .execute();

    Pipeline.given(user)
            .then(UserTestMapper.dtoToCreatedProjectionDoc())
            .then(probeMongoClient.findById())  // 30 retries × 3s
            .execute();
}
```

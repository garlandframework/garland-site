---
title: "How to test HTTP → PostgreSQL with Garland"
date: 2026-06-10
weight: 3
description: "End-to-end guide: create test DTOs and entities, write a mapper, configure PostgresTestClient, and verify database state after an HTTP call in a single pipeline"
build:
  list: never
---

This page walks through the full setup for asserting that an HTTP call persisted the correct data in PostgreSQL — from reading production classes through to a working pipeline test. All examples are taken from [garland-demo](https://github.com/garlandframework/garland-demo); clone it to run them.

This guide assumes test DTOs are already set up. If not, read [How to create test DTOs](/docs/how-to/test-dtos/) first.

---

## Step 1 — Locate the entity

Find the JPA entity that maps to the table your service writes to:

```java
@Entity
@Table(name = "users")
public class UserEntity {

    @Id
    private UUID id;            // note: "id", not "uuid"

    private String name;
    private String surname;

    @OneToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "address_id")
    private AddressEntity address;

    @OneToMany(fetch = FetchType.EAGER)
    @JoinColumn(name = "user_id")
    private List<CarEntity> cars;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "modified_at")
    private LocalDateTime modifiedAt;
}
```

Notice the field name mismatch: the DTO calls it `uuid`, the entity calls it `id`. The mapper handles this. Also note related entities — `AddressEntity` and `CarEntity` — these must all be registered with `PostgresConfig`.

---

## Step 2 — Configure PostgresTestClient

Register every entity class that Hibernate needs to know about. Missing a related entity causes a `MappingException` at construction time, not at query time:

```java
PostgresWrapper postgres = new PostgresWrapper(
        PostgresConfig.builder()
                .url("jdbc:postgresql://localhost:5432/userdb?TimeZone=UTC")
                .username("user")
                .password("password")
                .entity(UserEntity.class)
                .entity(AddressEntity.class)   // register relations too
                .entity(CarEntity.class)
                .build()
);

PostgresTestClient postgresClient = new PostgresTestClient(
        postgres,
        RetryConfig.of(5, Duration.ofSeconds(2))   // wait up to 10s for async writes
).withTemporalTolerance(Duration.ofNanos(1000));   // PostgreSQL truncates µs → absorb drift
```

`RetryConfig.of(5, Duration.ofSeconds(2))` gives the service up to 10 seconds to commit. `withTemporalTolerance(Duration.ofNanos(1000))` absorbs the microsecond truncation PostgreSQL applies to `TIMESTAMP` columns — without it, timestamp assertions fail intermittently.

Configure this once in a `@BeforeSuite` method in a shared base class and reuse across all tests.

---

## Step 3 — Write the mapper

The mapper converts `UserDto` (what the HTTP step returns) into `UserEntity` (what `findByFields` expects as input). Use MapStruct to generate the conversion at compile time.

The mapper interface declares field-level decisions — renames, ignored fields, and computed values:

```java
@Mapper
public interface UserTestMapper {

    UserTestMapper INSTANCE = Mappers.getMapper(UserTestMapper.class);

    @Mapping(source = "uuid", target = "id")   // rename: DTO "uuid" → entity "id"
    UserEntity toEntity(UserDto dto);
}
```

Fields set by the production system — `createdAt`, `modifiedAt` — are left unmapped (they will be null in the expected entity). `Verify.matching` inside `findByFields` skips null fields, so the assertion ignores timestamps it cannot predict.

If a field name on the entity does not match any field on the DTO, MapStruct will warn at compile time. Fix with `@Mapping` or `@Mapping(target = "fieldName", ignore = true)`.

**Pipeline bridge method** — wrap the mapper in `Step.lift` so it can be passed directly to `.then()`:

```java
static Step<UserDto, UserEntity> toEntity() {
    return Step.lift(INSTANCE::toEntity);
}
```

This is the only way to connect the HTTP step output (`UserDto`) to the database step input (`UserEntity`). Write one bridge method per conversion you need in tests.

---

## Step 4 — The full pipeline

With clients, mapper, and bridge in place, the test is a straight chain:

```java
@Test
public void createUser_persistedInDb() {
    HttpCallRequest<UserDto> request = TestUserRequests.createUser();

    Pipeline.given(request)
            .then(httpClient.makeCall(201, UserDto.class))   // POST → assert 201, return UserDto
            .then(trackUser())                               // register for cleanup
            .then(Verify.matching(request.dto()))            // assert response matches request
            .then(UserTestMapper.toEntity())                 // UserDto → UserEntity
            .then(postgresClient.findByFields())             // query DB, assert match
            .execute();
}
```

Each step's output type is the next step's input type — enforced at compile time. If the mapper or client types do not align, the code does not compile.

What each step does:

| Step | Input | Output | Purpose |
|------|-------|--------|---------|
| `makeCall(201, UserDto.class)` | `HttpCallRequest<UserDto>` | `UserDto` | POST, assert 201, return deserialized body |
| `trackUser()` | `UserDto` | `UserDto` | Register UUID for cleanup in `@AfterMethod` |
| `Verify.matching(request.dto())` | `UserDto` | `UserDto` | Assert response matches what was sent |
| `UserTestMapper.toEntity()` | `UserDto` | `UserEntity` | Convert for database lookup |
| `postgresClient.findByFields()` | `UserEntity` | `UserEntity` | Query DB by non-null fields, assert match |

---

## Skipping the response assertion

If you only want to verify persistence without asserting the HTTP response body, drop `Verify.matching`:

```java
Pipeline.given(TestUserRequests.createUser())
        .then(httpClient.makeCall(201, UserDto.class))
        .then(UserTestMapper.toEntity())
        .then(postgresClient.findByFields())
        .execute();
```

---

## Further reading

- [garland-postgres module reference](/docs/modules/postgres/) — `findById`, `findByFields`, `countByFields`, `notExistsById`, timestamp tolerance
- [garland-http module reference](/docs/modules/http/) — full `HttpTestClient` API
- [Demo project](https://github.com/garlandframework/garland-demo) — `CreateUserApiTest`, `UserTestMapper`, `BaseTest` contain the full working setup shown here

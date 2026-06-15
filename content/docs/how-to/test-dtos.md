---
title: "How to create test DTOs"
date: 2026-06-10
weight: 1
description: "How to create test DTOs for Garland — recreate production types as mutable, builder-enabled classes with nullable fields and Jackson tolerance"
build:
  list: local
---

Before writing pipeline tests you need test DTOs — classes that mirror your production types but are shaped for test use. This page explains what to create and why.

## Step 1 — Locate the production DTO

Start in the service source. Find the DTO your endpoint accepts and returns:

```java
// user-service/src/main/java/.../dto/UserDto.java
public record UserDto(
        UUID uuid,
        String name,
        String surname,
        AddressDto address,
        List<CarDto> cars,
        LocalDateTime createdAt,
        LocalDateTime modifiedAt
) {}
```

Note the field types — you will recreate this as a test DTO with the same fields but a different class form.

## Step 2 — Create the test DTO

Do not use the production DTO directly in your test module. The production DTO is typically an immutable record — it has no builder and no Jackson annotations. Recreate it as a mutable class in the test module with three additions:

- **`@Builder`.** Tests build partial expected objects — only the fields they control. `Verify.matching` skips null fields, so leaving a field unset means "don't assert this." A builder makes that pattern clean.
- **All fields nullable.** The production record may have fields that are never null in practice, but the test DTO must allow null to support partial assertions.
- **`@JsonIgnoreProperties(ignoreUnknown = true)`.** The server may add fields over time. Without this annotation, Jackson throws on any field the test DTO does not declare.

Field types are usually identical to the production type — `UUID`, `String`, `LocalDateTime`. Only the class form changes:

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
public class UserDto {
    private UUID uuid;
    private String name;
    private String surname;
    private AddressDto address;
    private List<CarDto> cars;
    private LocalDateTime createdAt;
    private LocalDateTime modifiedAt;
}
```

One test DTO per domain object. Apply the same pattern to nested types (`AddressDto`, `CarDto`) — each needs its own builder-enabled class.

## Widening field types

In some cases you may need to widen a field type beyond what the production DTO uses. For example, if the production DTO has `int count` and you want to test what the server does when it receives `2.4` or `"abc"`, changing that field to `Object` in the test DTO lets Jackson serialize whatever value the test passes. Jackson serializes based on runtime type, so `Object count = 2.4` sends a JSON float even though the production field is an integer. Apply this selectively — only to fields where you need to send values outside the production type's range.

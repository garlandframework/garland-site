---
title: "garland-testng — TestNG Base Class for Integration Tests"
date: 2026-06-10
description: "garland-testng reference — TestNG base class with automatic resource cleanup, suite lifecycle management, and structured test logging for Java integration tests"
build:
  list: local
---

`garland-testng` provides two classes that wire Garland into TestNG: `AbstractGarlandBaseTest` hooks `ResourceTracker` cleanup into `@AfterMethod` automatically, and `TestNGLogger` replaces TestNG's default output with a structured, colored log that shows class hierarchy, pass/fail/skip indicators, and inline failure messages.

```xml
<dependency>
    <groupId>dev.garlandframework</groupId>
    <artifactId>garland-testng</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>
```

---

## AbstractGarlandBaseTest

Extend it in your base test class. It exposes `registerTrackers(...)` and attaches an `@AfterMethod(alwaysRun = true)` hook that calls `cleanupAll` on every registered tracker after each test, regardless of pass or fail.

```java
public abstract class BaseTest extends AbstractGarlandBaseTest {

    protected final ResourceTracker<UUID> userTracker = new ResourceTracker<>(
            id -> Pipeline.given(TestUserRequests.deleteUser(id))
                          .then(http.makeCall(204, Void.class))
                          .execute()
    );

    protected BaseTest() {
        registerTrackers(userTracker);
    }
}
```

Any number of trackers can be registered — all are cleaned up in the same `@AfterMethod` call. Cleanup failures are logged as warnings and do not fail the test.

The `@AfterMethod` is declared with `alwaysRun = true`, so it runs even when the test method itself threw an exception. Resources created in a failing test are deleted as reliably as in a passing one.

---

## TestNGLogger

A TestNG listener that formats test progress as a structured log. Attach it with `@Listeners` on your base class:

```java
@Listeners(TestNGLogger.class)
public abstract class BaseTest extends AbstractGarlandBaseTest { ... }
```

Output for a passing class:

```
┌─ CLASS CreateUserApiTest
   DESCRIPTION Create user — validation, success cases, duplicate handling

   ▶ METHOD createUser_returns201_andPersistsToDb
     DESCRIPTION POST /api/users with valid body — expect 201 and entity in Postgres
   ✓ METHOD createUser_returns201_andPersistsToDb

   ▶ METHOD createUser_duplicateEmail_returns409
     DESCRIPTION POST /api/users with existing email — expect 409
   ✓ METHOD createUser_duplicateEmail_returns409

└─ CLASS ✓ CreateUserApiTest
```

Output when a method fails:

```
   ▶ METHOD createUser_returns201_andPersistsToDb
   ✗ METHOD createUser_returns201_andPersistsToDb
     expected: <Alice> but was: <null>

└─ CLASS ✗ CreateUserApiTest
```

The failure message is printed inline — no need to scroll past the stack trace to see what went wrong. Skipped methods are marked with `⊘`. Class descriptions come from the `description` attribute of the class-level `@Test` annotation; method descriptions come from the method-level `@Test(description = "...")`.

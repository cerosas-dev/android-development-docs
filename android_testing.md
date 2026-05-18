# Android Testing Guide

> A comprehensive reference for testing Android applications using industry-standard tools and practices — covering the testing pyramid, TDD, unit tests, integration tests, UI tests with Jetpack Compose Test, and screenshot tests with the Jetpack Screenshot Testing plugin.

## Table of Contents

1. [The Testing Pyramid](#1-the-testing-pyramid)
2. [Test-Driven Development (TDD)](#2-test-driven-development-tdd)
3. [Unit Tests](#3-unit-tests)
4. [Integration Tests](#4-integration-tests)
5. [UI Tests with Jetpack Compose](#5-ui-tests-with-jetpack-compose)
6. [Screenshot Tests](#6-screenshot-tests)

---

## 1. The Testing Pyramid

### What is the Testing Pyramid?

The Testing Pyramid is a framework for thinking about how many tests to write at each level of abstraction. Coined by Mike Cohn and later adapted for mobile by Google, it arranges tests into three tiers — Unit, Integration, and UI — ordered from fastest/cheapest at the bottom to slowest/most expensive at the top.

```
              ╔══════════════════╗
              ║    UI Tests      ║  ~10% — slow, brittle, expensive
              ║  (End-to-End)    ║
          ╔═══╩══════════════════╩═══╗
          ║   Integration Tests     ║  ~20% — moderate speed and cost
          ║  (Components together)  ║
      ╔═══╩═════════════════════════╩═══╗
      ║          Unit Tests             ║  ~70% — fast, cheap, precise
      ║   (Single class / function)     ║
      ╚═════════════════════════════════╝
```

Each tier has a different purpose, speed, and cost profile. Misbalancing the pyramid — writing too many UI tests and too few unit tests — leads to a slow, unreliable test suite that developers stop trusting.

### Why Follow the Pyramid?

The pyramid reflects an economic reality: **unit tests are orders of magnitude cheaper to write, run, and maintain than UI tests.** A unit test runs in milliseconds on the JVM with no emulator required. A UI test takes seconds (or minutes) and requires an emulator or physical device. A suite of 500 unit tests can run in under 30 seconds; 500 UI tests might take an hour.

The pyramid also reflects **failure locality**. When a unit test fails, you know exactly which class broke. When a UI test fails, it could be caused by any layer in the stack — network, database, ViewModel, or the UI itself.

### When to Write Which Test?

| Test Level | When to write it | Tools |
|-----------|-----------------|-------|
| Unit | For every non-trivial function, class, or business rule | JUnit 4, MockK, Coroutines Test |
| Integration | When two or more components must work together correctly | Hilt Testing, Room in-memory, MockWebServer |
| UI | For critical user journeys and screen-level behaviour | Jetpack Compose Test |
| Screenshot | For UI component visual regressions | Jetpack Screenshot Testing |

> **Note:** The pyramid is a guide, not a law. A pure UI app with no business logic may need proportionally more UI tests. A pure library with no UI may need zero. Apply the rationale, not the percentages mechanically.

### Android-Specific Pyramid

In Android, the tiers map to two test source sets:

| Source set | Runs on | Speed | Examples |
|-----------|---------|-------|---------|
| `src/test/` | JVM only — no Android framework | Milliseconds | Unit tests, use case tests, ViewModel tests |
| `src/androidTest/` | Emulator or real device | Seconds to minutes | UI tests, integration tests with Room/DB |
| `src/screenshotTest/` | JVM (rendered via Compose tooling) | Seconds | Screenshot / visual regression tests |

Always prefer `src/test/` (JVM) over `src/androidTest/` (device) when possible. The fewer tests that require a device, the faster your CI pipeline.

---

## 2. Test-Driven Development (TDD)

### What is TDD?

Test-Driven Development is a software development practice in which you write a failing test **before** writing any production code, then write the minimum code needed to make the test pass, then refactor. This cycle repeats for every new behaviour:

```
  ┌─────────────────────────────────────────────────┐
  │                                                 │
  │   1. RED   — Write a failing test               │
  │       ↓                                         │
  │   2. GREEN — Write the minimum code to pass it  │
  │       ↓                                         │
  │   3. REFACTOR — Clean up without breaking tests │
  │       ↓                                         │
  │   (repeat for next behaviour)                   │
  │                                                 │
  └─────────────────────────────────────────────────┘
```

The test defines the contract. The implementation fulfils it. The refactor improves its quality. This order is intentional and non-negotiable in TDD — writing the test first forces you to think about the API before you implement it.

### Why Use TDD?

- **Forces good design.** Hard-to-test code is hard-to-use code. If writing a test for a class is painful, the class probably has too many dependencies or too many responsibilities.
- **Prevents over-engineering.** You only write code that is needed to pass the current test. No speculative features.
- **Provides a safety net.** A comprehensive test suite written alongside the code catches regressions immediately.
- **Produces living documentation.** Tests describe exactly what the code does, in plain test names.

### When to Use TDD?

TDD is most valuable for **business logic** — use cases, domain models, validation rules, state machines. It is less practical for pure UI layout code, framework plumbing (Hilt modules, Room entities), and exploratory prototyping. A pragmatic approach: apply TDD rigorously to the domain and data layers; write tests before or alongside UI and integration layers.

### The TDD Cycle in Android — Full Example

We'll implement a `LoginUseCase` using TDD.

#### Step 1 — RED: Write the failing test first

```kotlin
// src/test/kotlin/com/example/app/domain/usecase/LoginUseCaseTest.kt
class LoginUseCaseTest {

    private val fakeAuthRepository = FakeAuthRepository()
    private val useCase = LoginUseCase(fakeAuthRepository)

    @Test
    fun `invoke returns AuthToken on valid credentials`() = runTest {
        fakeAuthRepository.stubbedToken = AuthToken("access123", "refresh456")

        val result = useCase(LoginUseCase.Params("alice@example.com", "Password1"))

        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrThrow().accessToken).isEqualTo("access123")
    }

    @Test
    fun `invoke returns failure when email is blank`() = runTest {
        val result = useCase(LoginUseCase.Params("", "Password1"))

        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()?.message).contains("Email")
    }

    @Test
    fun `invoke returns failure when password is shorter than 8 characters`() = runTest {
        val result = useCase(LoginUseCase.Params("alice@example.com", "Pass1"))

        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()?.message).contains("8 characters")
    }

    @Test
    fun `invoke returns failure when repository throws`() = runTest {
        fakeAuthRepository.shouldThrow = IOException("No network")

        val result = useCase(LoginUseCase.Params("alice@example.com", "Password1"))

        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isInstanceOf(IOException::class.java)
    }
}

// Fake used across tests — no mocking needed for simple stubs
class FakeAuthRepository : AuthRepository {
    var stubbedToken: AuthToken? = null
    var shouldThrow: Throwable? = null

    override suspend fun login(email: String, password: String): Result<AuthToken> {
        shouldThrow?.let { return Result.failure(it) }
        return Result.success(stubbedToken ?: AuthToken("default", "default"))
    }
}
```

Run: `./gradlew :app:testDebugUnitTest --tests "*LoginUseCaseTest*"`
Expected: **FAIL** — `LoginUseCase` does not exist yet.

#### Step 2 — GREEN: Write the minimum implementation

```kotlin
// src/main/kotlin/com/example/app/domain/usecase/LoginUseCase.kt
class LoginUseCase(private val authRepository: AuthRepository) {

    data class Params(val email: String, val password: String)

    suspend operator fun invoke(params: Params): Result<AuthToken> = runCatching {
        require(params.email.isNotBlank()) { "Email cannot be blank" }
        require(params.password.length >= 8) { "Password must be at least 8 characters" }
        authRepository.login(params.email, params.password).getOrThrow()
    }
}
```

Run: `./gradlew :app:testDebugUnitTest --tests "*LoginUseCaseTest*"`
Expected: **PASS** — all 4 tests green.

#### Step 3 — REFACTOR: Improve without breaking

```kotlin
// Extract validation constants to improve readability
class LoginUseCase(private val authRepository: AuthRepository) {

    data class Params(val email: String, val password: String)

    suspend operator fun invoke(params: Params): Result<AuthToken> = runCatching {
        validate(params)
        authRepository.login(params.email, params.password).getOrThrow()
    }

    private fun validate(params: Params) {
        require(params.email.isNotBlank())                         { "Email cannot be blank" }
        require(params.password.length >= MIN_PASSWORD_LENGTH) {
            "Password must be at least $MIN_PASSWORD_LENGTH characters"
        }
    }

    companion object {
        private const val MIN_PASSWORD_LENGTH = 8
    }
}
```

Run again: `./gradlew :app:testDebugUnitTest --tests "*LoginUseCaseTest*"`
Expected: **PASS** — all tests still green after refactor.

---

## 3. Unit Tests

### What are Unit Tests?

A unit test verifies the behaviour of a **single, isolated unit of code** — typically one class or one function. All external dependencies (repositories, data sources, APIs, databases) are replaced with fakes or mocks so the test only exercises the code under test.

Unit tests live in `src/test/` and run entirely on the JVM without a device or emulator. They have no access to the Android framework (`Context`, `Activity`, `View`). This is intentional: if your business logic requires Android framework classes, it belongs in the wrong layer.

### Why Write Unit Tests?

Unit tests are the foundation of confidence in your codebase. They are:
- **Fast:** A suite of 500 unit tests typically runs in under 10 seconds.
- **Precise:** A failing unit test points directly at the broken class.
- **Cheap to maintain:** No emulator setup, no flakiness from async timing or network state.
- **Design feedback:** If a class is hard to unit test, it is too tightly coupled.

### When to Write Unit Tests?

Write a unit test for every non-trivial behaviour: use cases, ViewModel state transitions, repository cache strategies, mappers, validators, and pure functions. Skip unit tests only for trivial wiring code (Hilt modules, data class definitions) that has no logic of its own.

---

### Gradle Setup

```kotlin
// app/build.gradle.kts
dependencies {
    // JUnit 4 runner
    testImplementation("junit:junit:4.13.2")

    // MockK — Kotlin-first mocking library
    testImplementation("io.mockk:mockk:1.13.10")

    // Kotlin Coroutines test utilities
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")

    // Google Truth — readable assertions
    testImplementation("com.google.truth:truth:1.4.2")

    // Turbine — Flow testing
    testImplementation("app.cash.turbine:turbine:1.1.0")
}
```

---

### MainDispatcherRule — Required for Every ViewModel Test

`viewModelScope` defaults to `Dispatchers.Main`. In tests there is no Android main looper, so you must replace it with a test dispatcher:

```kotlin
// src/test/kotlin/com/example/app/util/MainDispatcherRule.kt
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    private val dispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {

    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
        dispatcher.cleanupTestCoroutines()
    }
}
```

Apply it to every ViewModel test class:

```kotlin
@get:Rule val mainDispatcherRule = MainDispatcherRule()
```

---

### Fakes vs Mocks

Prefer **fakes** (handwritten implementations) for stable interfaces; use **mocks** (MockK) for complex interactions that are expensive to fake or where call verification matters.

| | Fake | Mock (MockK) |
|---|---|---|
| **What** | Handwritten implementation of an interface | Auto-generated proxy that records and verifies calls |
| **When** | Stable interface, multiple test reuse | Verifying call count / order, complex return sequences |
| **Readability** | High — reads like real code | Medium — DSL can be verbose |
| **Fragility** | Low — ignores irrelevant calls | Medium — every unexpected call may fail |

```kotlin
// Fake — simple, reusable across many tests
class FakeUserRepository : UserRepository {
    var stubbedUser: User? = null
    var savedUsers = mutableListOf<User>()

    override suspend fun getUser(id: String): Result<User> =
        stubbedUser?.let { Result.success(it) }
            ?: Result.failure(NoSuchElementException("User $id not found"))

    override suspend fun saveUser(user: User): Result<Unit> {
        savedUsers.add(user)
        return Result.success(Unit)
    }

    override fun observeUsers(): Flow<List<User>> = flowOf(savedUsers.toList())
}

// Mock — for verifying interactions
val mockAnalytics = mockk<AnalyticsTracker>(relaxed = true)
viewModel.trackEvent("login")
verify(exactly = 1) { mockAnalytics.track("login") }
```

---


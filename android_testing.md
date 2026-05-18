# Android Testing Guide

> A comprehensive reference for testing Android applications using industry-standard tools and practices — covering the testing pyramid, TDD, unit tests, integration tests, UI tests with Jetpack Compose Test, and screenshot tests with the Jetpack Screenshot Testing plugin.

## Table of Contents

1. [The Testing Pyramid](#1-the-testing-pyramid)
2. [Test-Driven Development (TDD)](#2-test-driven-development-tdd)
3. [Stubbing and Mocking](#3-stubbing-and-mocking)
4. [Robot Pattern](#4-robot-pattern)
5. [Test Fixtures](#5-test-fixtures)
6. [Unit Tests](#6-unit-tests)
7. [Integration Tests](#7-integration-tests)
8. [UI Tests with Jetpack Compose](#8-ui-tests-with-jetpack-compose)
9. [Screenshot Tests](#9-screenshot-tests)

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

## 3. Stubbing and Mocking

### What are Stubbing and Mocking?

Stubbing and mocking are two related testing techniques used to control dependencies so the test can focus on the behaviour of the unit under test.

- **Stubbing** means defining what a dependency should return or throw during a test.
- **Mocking** means replacing a dependency with a test double that can also verify how it was used.

In practice, a single library such as MockK can do both. You can *stub* `sessionManager.getToken()` to return a value, then *verify* that `sessionManager.clearSession()` was called exactly once.

### Why Use Them?

Real dependencies are often slow, nondeterministic, or hard to control. A repository that hits the network, a clock that returns the current time, or an analytics tracker that writes events externally all make tests harder to reason about.

Stubs and mocks give the test complete control over those dependencies. That control produces tests that are faster, more repeatable, and more precise when failures occur.

### Stubbing vs Mocking vs Faking

These terms are often mixed together, but they solve slightly different problems:

| Technique | Main purpose | Best use case |
|----------|--------------|---------------|
| Stub | Control returned data or thrown errors | “When this dependency is called, return X” |
| Mock | Verify interactions with a dependency | “Make sure this method was called once with these arguments” |
| Fake | Provide a lightweight working implementation | Reusable in-memory repository, DAO, cache, or service |

A simple rule of thumb:
- Use a **fake** when you want realistic behaviour and reuse across many tests.
- Use a **stub** when you only need a dependency to return one controlled answer.
- Use a **mock** when the important assertion is about the interaction itself.

### When to Stub

Use stubbing when the return value of a dependency is part of the test setup, not the test assertion. Common examples include:

- Returning a user from a repository.
- Throwing an `IOException` from an API client.
- Returning `true` from a feature flag service.
- Returning a specific timestamp from a clock abstraction.

In these cases, the dependency is only there to support the test scenario. The test usually asserts on the resulting state, output, or UI.

### When to Mock

Use mocking when the interaction with the dependency is itself the behaviour you care about. Common examples include:

- Verifying an analytics event was sent.
- Verifying a repository called `save()` exactly once.
- Verifying a retry policy triggered three attempts.
- Verifying an interceptor cleared the session after a 401 response.

If the test would still be meaningful without checking the call, you may not need a mock.

---

### Stubbing with MockK

MockK uses `every { ... } returns ...` for regular functions and `coEvery { ... } returns ...` for suspending functions.

```kotlin
class GetUserUseCaseTest {

    private val userRepository = mockk<UserRepository>()
    private val useCase = GetUserUseCase(userRepository)

    @Test
    fun `returns user from repository`() = runTest {
        val user = User("u1", "Alice", "alice@example.com", null)
        coEvery { userRepository.getUser("u1") } returns Result.success(user)

        val result = useCase(GetUserUseCase.Params("u1"))

        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrThrow()).isEqualTo(user)
    }

    @Test
    fun `returns failure when repository throws`() = runTest {
        coEvery { userRepository.getUser("u1") } returns Result.failure(IOException("timeout"))

        val result = useCase(GetUserUseCase.Params("u1"))

        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isInstanceOf(IOException::class.java)
    }
}
```

### Throwing Errors from Stubs

Stubs are especially useful when you need to force an error path that is difficult to reproduce reliably with a real dependency:

```kotlin
@Test
fun `emits Error state when refresh fails`() = runTest {
    coEvery { userRepository.refreshUsers() } throws IOException("No network")

    viewModel.refresh()

    assertThat(viewModel.uiState.value).isInstanceOf(UserListUiState.Error::class.java)
}
```

This is one of the strongest uses of stubbing: driving code into rare branches such as timeouts, server errors, or invalid data responses.

---

### Mocking with Verification

Mocking becomes useful when the test needs to prove that a dependency was used in a particular way.

```kotlin
class LoginAnalyticsTest {

    private val analytics = mockk<AnalyticsTracker>(relaxed = true)
    private val authRepository = mockk<AuthRepository>()
    private val useCase = LoginUseCase(authRepository, analytics)

    @Test
    fun `tracks success event after login succeeds`() = runTest {
        coEvery { authRepository.login("alice@example.com", "Password1") } returns
            Result.success(AuthToken("access123", "refresh456"))

        useCase(LoginUseCase.Params("alice@example.com", "Password1"))

        verify(exactly = 1) {
            analytics.track("login_success")
        }
    }

    @Test
    fun `tracks failure event after login fails`() = runTest {
        coEvery { authRepository.login(any(), any()) } returns
            Result.failure(IOException("network error"))

        useCase(LoginUseCase.Params("alice@example.com", "Password1"))

        verify(exactly = 1) {
            analytics.track("login_failure")
        }
    }
}
```

A **relaxed** mock automatically returns simple default values for functions you do not care about, which reduces setup noise for dependencies used only for verification.

### Verifying Arguments and Call Count

```kotlin
@Test
fun `saves updated user exactly once`() = runTest {
    val repository = mockk<UserRepository>()
    coEvery { repository.saveUser(any()) } returns Result.success(Unit)

    val user = User("u1", "Bob", "bob@example.com", null)
    val useCase = SaveUserUseCase(repository)

    useCase(SaveUserUseCase.Params(user))

    coVerify(exactly = 1) {
        repository.saveUser(match { it.id == "u1" && it.name == "Bob" })
    }
}
```

This style is useful when the correctness of the outgoing command matters more than the returned result.

---

### Common MockK Patterns

```kotlin
// Regular function
val clock = mockk<AppClock>()
every { clock.now() } returns Instant.parse("2026-05-18T12:00:00Z")

// Suspending function
coEvery { userRepository.getUser("u1") } returns Result.success(user)

// Function returns Unit
val analytics = mockk<AnalyticsTracker>(relaxed = true)
every { analytics.track(any()) } just Runs

// Verify regular function
verify(exactly = 1) { analytics.track("screen_opened") }

// Verify suspending function
coVerify(exactly = 1) { userRepository.refreshUsers() }

// Verify nothing happened
verify(exactly = 0) { analytics.track(any()) }
coVerify(exactly = 0) { userRepository.deleteUser(any()) }
```

### Mocking Flows

When a dependency exposes a `Flow`, stub the returned stream directly:

```kotlin
@Test
fun `observeUsers emits repository values`() = runTest {
    val users = listOf(User("u1", "Alice", "alice@example.com", null))
    every { userRepository.observeUsers() } returns flowOf(users)

    viewModel.users.test {
        assertThat(awaitItem()).isEqualTo(users)
        cancelAndConsumeRemainingEvents()
    }
}
```

The key point is that you are not mocking Flow internals. You are stubbing the dependency method to return a known Flow.

---

### Prefer State Assertions over Interaction Assertions

A common mistake is over-verifying everything. That produces brittle tests tightly coupled to implementation details.

```kotlin
// Too coupled to internals
coVerify { repository.getUser("u1") }
coVerify { analytics.track("load_user") }
coVerify { cache.put(any()) }

// Better when outcome is what matters
assertThat(viewModel.uiState.value).isInstanceOf(UserDetailUiState.Success::class.java)
```

Prefer interaction verification only when the interaction is the contract. For example, analytics, logging, retries, navigation callbacks, and side-effect boundaries are good candidates. Pure data transformation usually is not.

### Avoid Mocking What You Own Too Early

For stable app interfaces such as `UserRepository`, `SettingsRepository`, or `NetworkMonitor`, a fake is often easier to read and maintain than a heavily stubbed mock. Mocks shine when:

- The dependency has many hard-to-model branches.
- The test needs strict call verification.
- The behaviour is incidental and not worth building a fake for.

If a test needs five `coEvery` calls just to construct a basic scenario, that may be a sign a fake would be clearer.

---

### Example: Fake vs Stub vs Mock

```kotlin
// Fake: reusable implementation with state
class FakeFeatureFlagRepository : FeatureFlagRepository {
    private val flags = mutableMapOf<String, Boolean>()

    fun setFlag(key: String, enabled: Boolean) {
        flags[key] = enabled
    }

    override fun isEnabled(key: String): Boolean = flags[key] ?: false
}

// Stub: one-off controlled answer
val featureFlags = mockk<FeatureFlagRepository>()
every { featureFlags.isEnabled("new_home") } returns true

// Mock: verify interaction
val analytics = mockk<AnalyticsTracker>(relaxed = true)
verify { analytics.track("new_home_opened") }
```

All three are valid. The right choice depends on whether the test needs reusable behaviour, controlled output, or interaction verification.

### Common Mistakes

- **Mocking everything by default.** This makes tests verbose and fragile.
- **Verifying implementation details instead of outcomes.** Tests break during harmless refactors.
- **Using mocks where a fake would be simpler.** The test becomes harder to read.
- **Forgetting to clear mocks between tests.** State leakage causes confusing failures.
- **Stubbing methods the code never calls.** Extra setup creates noise and hides what matters.

### Best Practices

| Practice | Why it helps |
|---------|---------------|
| Stub only what the scenario needs | Less setup makes the test intent clearer |
| Verify only meaningful interactions | Reduces brittleness during refactoring |
| Prefer fakes for stable domain interfaces | Tests read more like real code |
| Use `coEvery` / `coVerify` for suspend functions | Matches coroutine-based APIs correctly |
| Reset or recreate mocks per test | Prevents cross-test contamination |
| Use relaxed mocks for fire-and-forget collaborators | Cuts boilerplate for analytics, loggers, and trackers |

---

---

---

## 4. Robot Pattern

### What is the Robot Pattern?

The Robot Pattern is a UI test design pattern popularised by Jake Wharton (Square) that creates a **typed DSL** over raw Compose Test (or Espresso) calls. Each screen or component gets a corresponding *robot* class that exposes human-readable actions and assertions. The test body calls the robot; the robot calls the test framework. Tests never touch `composeTestRule` directly.

The pattern splits two responsibilities that are normally tangled together:

- **What the test does** — expressed in domain terms (`loginRobot.enterEmail("alice@example.com").clickSignIn()`).
- **How the framework finds elements** — hidden inside the robot (`onNodeWithTag("email_field").performTextInput(email)`).

### Why Use the Robot Pattern?

Without the pattern, every test method duplicates raw selectors. A single tag rename (`"email_field"` → `"login_email_input"`) breaks dozens of tests scattered across the file. With the pattern, the same rename requires one change in the robot, and all tests continue to pass.

The pattern also makes tests readable to non-engineers. A product manager can read `loginRobot.assertErrorDisplayed("Invalid password")` and immediately understand what is being verified — without needing to know anything about Compose semantics nodes.

Additionally, the fluent `apply`-based DSL encourages writing tests as **step-by-step scenarios** that mirror real user behaviour, making the intent of each test immediately obvious from its name and body alone.

### When to Use the Robot Pattern?

Use the Robot Pattern whenever:
- More than one test method performs the same sequence of UI interactions.
- A screen has complex forms, multi-step flows, or conditional states.
- The test suite is large enough that maintenance cost of raw selectors becomes significant.

For a single-screen throwaway test, a plain `composeTestRule` call is fine. Once the second test starts duplicating the same `onNodeWithTag(...)` calls, extract a robot.

### How to Design a Good Robot

A useful robot hides framework details but does **not** hide the scenario. Keep these rules in mind:

- **One robot per screen or component.** A `LoginRobot` should know the login screen only; navigation can return the next robot.
- **Expose domain language.** Prefer `enterEmail`, `submitLogin`, `assertLoginFailed` over `tapButton1` or `typeTextInField`.
- **Use stable selectors internally.** Prefer `testTag` over visible text when possible because tags change less often than UI copy.
- **Keep actions and assertions explicit.** Tests should still read like behaviour specifications, not like a magical black box.
- **Return `this` for fluent steps.** Chaining keeps the scenario compact and readable.

A robot is not a page object clone with dozens of generic helpers. It should model behaviour that matters to the screen under test.

### How to Introduce Robots Incrementally

You do not need to refactor the whole suite at once. A practical migration path is:

1. Start with one flaky or duplicated screen test.
2. Move repeated selectors and actions into a robot.
3. Leave one or two raw assertions in the test if they are unique.
4. Extract more methods only when a second test needs them.

This keeps the abstraction lean. If a method is used once and does not improve readability, it probably does not belong in the robot yet.

---

### Defining a Robot

A robot is a plain Kotlin class that wraps a `ComposeContentTestRule`. Every method returns `this` via `apply` so calls chain fluently. Group action methods and assertion methods clearly — or split into an *action robot* and an *assertion robot* if the class grows large.

```kotlin
// src/androidTest/kotlin/com/example/app/robot/LoginRobot.kt
class LoginRobot(private val rule: ComposeContentTestRule) {

    // --- Actions ---

    fun enterEmail(email: String) = apply {
        rule.onNodeWithTag("login_email_field").performTextInput(email)
    }

    fun clearEmail() = apply {
        rule.onNodeWithTag("login_email_field").performTextClearance()
    }

    fun enterPassword(password: String) = apply {
        rule.onNodeWithTag("login_password_field").performTextInput(password)
    }

    fun clickSignIn() = apply {
        rule.onNodeWithText("Sign In").performClick()
    }

    fun clickForgotPassword() = apply {
        rule.onNodeWithText("Forgot password?").performClick()
    }

    // --- Assertions ---

    fun assertSignInButtonEnabled() = apply {
        rule.onNodeWithText("Sign In").assertIsEnabled()
    }

    fun assertSignInButtonDisabled() = apply {
        rule.onNodeWithText("Sign In").assertIsNotEnabled()
    }

    fun assertErrorDisplayed(message: String) = apply {
        rule.onNodeWithText(message).assertIsDisplayed()
    }

    fun assertErrorDoesNotExist() = apply {
        rule.onNodeWithTag("login_error_message").assertDoesNotExist()
    }

    fun assertLoadingIndicatorVisible() = apply {
        rule.onNodeWithTag("login_loading").assertIsDisplayed()
    }
}
```

### Using a Robot in Tests

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeTestRule = createAndroidComposeRule<HiltTestActivity>()

    @Inject lateinit var fakeAuthRepository: FakeAuthRepository

    private val robot by lazy { LoginRobot(composeTestRule) }

    @Before
    fun setUp() {
        hiltRule.inject()
        composeTestRule.setContent { AppTheme { LoginScreen(onLoginSuccess = {}) } }
    }

    @Test
    fun signInButton_isDisabled_whenFieldsAreEmpty() {
        robot.assertSignInButtonDisabled()
    }

    @Test
    fun signInButton_becomesEnabled_whenBothFieldsAreFilled() {
        robot
            .enterEmail("alice@example.com")
            .enterPassword("Password1")
            .assertSignInButtonEnabled()
    }

    @Test
    fun validCredentials_dismissLoadingAndNavigatesToHome() {
        fakeAuthRepository.stubbedToken = AuthToken("access123", "refresh456")

        robot
            .enterEmail("alice@example.com")
            .enterPassword("Password1")
            .clickSignIn()
            .assertLoadingIndicatorVisible()

        composeTestRule.waitUntil(5_000) {
            composeTestRule.onAllNodesWithTag("home_screen").fetchSemanticsNodes().isNotEmpty()
        }
    }

    @Test
    fun invalidCredentials_showsErrorMessage() {
        fakeAuthRepository.shouldThrow = InvalidCredentialsException("Invalid password")

        robot
            .enterEmail("alice@example.com")
            .enterPassword("WrongPass1")
            .clickSignIn()
            .assertErrorDisplayed("Invalid password")
    }

    @Test
    fun clearingEmail_disablesSignInButton() {
        robot
            .enterEmail("alice@example.com")
            .enterPassword("Password1")
            .clearEmail()
            .assertSignInButtonDisabled()
    }
}
```

### Chaining Robots for Multi-Screen Flows

When a flow spans multiple screens, each robot can return the next robot to keep the scenario readable end-to-end:

```kotlin
// src/androidTest/kotlin/com/example/app/robot/LoginRobot.kt
fun clickSignIn(nextRobot: HomeRobot): HomeRobot {
    rule.onNodeWithText("Sign In").performClick()
    return nextRobot
}

// src/androidTest/kotlin/com/example/app/robot/HomeRobot.kt
class HomeRobot(private val rule: ComposeContentTestRule) {

    fun assertWelcomeMessageDisplayed(name: String) = apply {
        rule.onNodeWithText("Welcome, $name").assertIsDisplayed()
    }

    fun tapFirstUser(): UserDetailRobot {
        rule.onAllNodesWithTag("user_card")[0].performClick()
        return UserDetailRobot(rule)
    }
}

// Multi-screen test reads like a scenario
@Test
fun loginFlow_navigatesToHomeAndShowsUserList() {
    LoginRobot(composeTestRule)
        .enterEmail("alice@example.com")
        .enterPassword("Password1")
        .clickSignIn(HomeRobot(composeTestRule))
        .assertWelcomeMessageDisplayed("Alice")
        .tapFirstUser()
        .assertUserDetailVisible()
}
```

### Example: Raw Test vs Robot Test

Compare the same intent written both ways:

```kotlin
// Without robot
@Test
fun invalidCredentials_showsError() {
    composeTestRule.onNodeWithTag("login_email_field").performTextInput("alice@example.com")
    composeTestRule.onNodeWithTag("login_password_field").performTextInput("WrongPass1")
    composeTestRule.onNodeWithText("Sign In").performClick()
    composeTestRule.onNodeWithText("Invalid password").assertIsDisplayed()
}

// With robot
@Test
fun invalidCredentials_showsError() {
    LoginRobot(composeTestRule)
        .enterEmail("alice@example.com")
        .enterPassword("WrongPass1")
        .clickSignIn()
        .assertErrorDisplayed("Invalid password")
}
```

The second version is shorter, easier to scan, and resilient to selector changes because the selectors live in one place.

### Common Mistakes

- **Over-abstracting.** Do not create methods so generic that they hide what the user actually does.
- **Leaking test framework calls into tests.** If every test still uses `onNodeWithTag`, the robot is not buying much.
- **Putting business logic in robots.** Robots should orchestrate UI interaction, not calculate test expectations.
- **Creating one mega-robot.** Huge robots become hard to maintain and harder to trust.

> **Tip:** Keep each robot focused on a single screen or component. A robot that knows about two screens is a sign the screens have been tested as a coupled unit, which makes failures harder to diagnose.

---

---

## 5. Test Fixtures

### What are Test Fixtures?

A test fixture is any piece of **reusable code that sets up the known, consistent state** a test needs to run. Fixtures eliminate duplicated setup logic scattered across test classes and ensure tests start from a predictable baseline. The term covers several related patterns:

- **Data factories (Object Mother)** — functions or objects that produce domain model instances with sensible defaults and named overrides.
- **JUnit Rules** — `@get:Rule` helpers that run setup and teardown around each test method.
- **Fake implementations** — handwritten interface implementations that return controlled data (covered in Unit Tests, referenced here in context).
- **Database seeders** — helpers that pre-populate a Room in-memory database before each test.

### Why Use Test Fixtures?

Without fixtures, every test file defines its own `val user = User("u1", "Alice", "alice@example.com", null)`. When the `User` constructor gains a new required parameter, every one of those definitions breaks. With a factory, you change one line and all tests still compile.

Fixtures also make the **signal-to-noise ratio** of each test body much higher. The test body should describe what is being tested, not how to construct the domain objects needed to test it. A test that reads `val user = UserFixture.aUser(name = "Bob")` communicates its intent far better than five lines of constructor calls.

### When to Use Test Fixtures?

Introduce a fixture as soon as:
- Two or more tests construct the same domain object, even with slight variations.
- Test `@Before` blocks grow beyond 5–10 lines of setup that is identical across test classes.
- A schema change (new field, renamed parameter) causes compilation failures in more than one test file.

### How Fixtures Help in Practice

Fixtures solve three recurring problems in Android test suites:

- **Duplication:** the same `User`, `AuthToken`, or DAO setup appears in many files.
- **Fragility:** model changes ripple through dozens of tests.
- **Unreadable tests:** setup becomes longer than the assertion.

A good fixture moves repetitive construction into one place while still letting each test override the one field it cares about. That balance is the real goal: less boilerplate without losing clarity.

### How to Choose the Right Fixture Type

Use the smallest fixture that solves the problem:

| Problem | Best fixture |
|---------|--------------|
| Repeating model creation | Data factory / Object Mother |
| Shared dispatcher or cleanup logic | JUnit Rule |
| Repeated database prepopulation | Database fixture |
| Repeated dependency behaviour | Fake implementation |

Not every repeated line deserves a fixture. Create one when it improves readability or reduces maintenance cost, not just to hide code.

---

### Data Factory — Object Mother Pattern

The Object Mother pattern centralises all test data creation in one place. Each `a*` method provides an instance with sensible defaults; named parameters let individual tests override only what matters for them.

```kotlin
// src/test/kotlin/com/example/app/fixtures/UserFixture.kt
object UserFixture {

    fun aUser(
        id: String = "user-1",
        name: String = "Alice Smith",
        email: String = "alice@example.com",
        avatarUrl: String? = null
    ): User = User(id, name, email, avatarUrl)

    fun aListOfUsers(count: Int = 3): List<User> =
        (1..count).map { i ->
            aUser(
                id       = "user-$i",
                name     = "User $i",
                email    = "user$i@example.com"
            )
        }

    fun anAdminUser(): User = aUser(id = "admin-1", name = "Admin User", email = "admin@example.com")
}

object UserEntityFixture {

    fun aUserEntity(
        id: String = "user-1",
        name: String = "Alice Smith",
        email: String = "alice@example.com",
        avatarUrl: String? = null
    ): UserEntity = UserEntity(id, name, email, avatarUrl)

    fun aListOfUserEntities(count: Int = 3): List<UserEntity> =
        (1..count).map { i ->
            aUserEntity(id = "user-$i", name = "User $i", email = "user$i@example.com")
        }
}

object AuthFixture {

    fun anAuthToken(
        accessToken: String  = "access-token-abc",
        refreshToken: String = "refresh-token-xyz"
    ): AuthToken = AuthToken(accessToken, refreshToken)
}
```

Usage in tests — the test body states only what differs from the default:

```kotlin
@Test
fun `getUser returns user by id`() = runTest {
    val user = UserFixture.aUser(id = "target-user")
    fakeUserRepository.stubbedUser = user

    val result = getUserUseCase(GetUserUseCase.Params("target-user"))

    assertThat(result.getOrThrow().id).isEqualTo("target-user")
}

@Test
fun `user list shows all loaded users`() = runTest {
    fakeUserRepository.stubbedUsers = UserFixture.aListOfUsers(count = 5)

    viewModel.loadUsers()

    assertThat((viewModel.uiState.value as UserListUiState.Success).users).hasSize(5)
}
```

### Fixture Naming and Structure

A few naming conventions make fixtures easier to scan:

- Prefix builders with `a` or `an`: `aUser()`, `anAuthToken()`.
- Keep defaults valid and realistic so a fixture represents a normal state first.
- Use named parameters for overrides so tests remain self-documenting.
- Place fixtures in `src/test/fixtures` or `src/androidTest/fixtures` close to the tests that use them.

Avoid putting fixtures in `src/main/`. They are test support code, not production code.

---

### JUnit Rule as a Fixture

A `TestWatcher`-based JUnit Rule runs code before and after every test method in the class, giving you guaranteed setup and teardown without relying on `@Before` / `@After` methods across every test class that needs it.

```kotlin
// src/test/kotlin/com/example/app/fixtures/MainDispatcherRule.kt
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    val dispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {

    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)

    override fun finished(description: Description) {
        Dispatchers.resetMain()
        dispatcher.cleanupTestCoroutines()
    }
}

// src/test/kotlin/com/example/app/fixtures/MockkClearRule.kt
// Automatically resets all MockK state between tests — prevents leakage
class MockkClearRule : TestWatcher() {
    override fun finished(description: Description) = clearAllMocks()
}
```

Combine multiple fixture rules in one test class:

```kotlin
class UserDetailViewModelTest {

    @get:Rule val mainDispatcherRule = MainDispatcherRule()
    @get:Rule val mockkClearRule     = MockkClearRule()

    private val mockAnalytics = mockk<AnalyticsTracker>(relaxed = true)
    private val fakeRepo      = FakeUserRepository()
    private val viewModel     = UserDetailViewModel(fakeRepo, mockAnalytics)

    @Test
    fun `loads user on init`() = runTest {
        fakeRepo.stubbedUser = UserFixture.aUser()
        // mockk state is clean — no leakage from previous test
        val state = viewModel.uiState.value
        assertThat(state).isInstanceOf(UserDetailUiState.Success::class.java)
    }
}
```

### Why Rules Are Useful Fixtures

Rules are ideal when setup and teardown must always happen together. Replacing `Dispatchers.Main`, clearing mocks, creating temp files, or resetting clocks are all examples where a rule is safer than remembering to duplicate code in every `@Before` and `@After` pair.

They also reduce accidental omissions. If a test class needs the main dispatcher replaced, a rule makes that dependency visible at the top of the class instead of buried in setup code.

---

### Database Fixture — Seeding Room for Integration Tests

For DAO and repository integration tests, a fixture helper pre-populates the in-memory database in one call:

```kotlin
// src/androidTest/kotlin/com/example/app/fixtures/DatabaseFixture.kt
class DatabaseFixture(private val database: AppDatabase) {

    suspend fun seedUser(vararg users: UserEntity = arrayOf(UserEntityFixture.aUserEntity())) {
        database.userDao().upsertAll(users.toList())
    }

    suspend fun seedUsers(count: Int) {
        database.userDao().upsertAll(UserEntityFixture.aListOfUserEntities(count))
    }

    suspend fun clear() {
        database.clearAllTables()
    }
}
```

Usage in a Room DAO test:

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var db: DatabaseFixture

    @Before
    fun setUp() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        db = DatabaseFixture(database)
    }

    @After
    fun tearDown() = database.close()

    @Test
    fun search_returnsMatchingUsers_afterSeed() = runTest {
        db.seedUser(
            UserEntityFixture.aUserEntity(id = "u1", name = "Alice Smith"),
            UserEntityFixture.aUserEntity(id = "u2", name = "Bob Jones")
        )

        val results = database.userDao().search("alice").first()

        assertThat(results).hasSize(1)
        assertThat(results[0].name).isEqualTo("Alice Smith")
    }

    @Test
    fun findById_returnsNull_onEmptyDb() = runTest {
        // db.clear() not needed — inMemoryDatabase starts empty
        assertThat(database.userDao().findById("nonexistent")).isNull()
    }

    @Test
    fun upsertAll_thenSearch_returnsAllMatches() = runTest {
        db.seedUsers(count = 10)

        val all = database.userDao().search("user").first()

        assertThat(all).hasSize(10)
    }
}
```

### How to Keep Fixtures Safe

Fixtures should increase determinism, not hide shared mutable state. Follow these rules:

- Return new objects by default instead of reusing mutable singletons.
- Reset fake state between tests.
- Avoid random values unless the test explicitly needs them.
- Keep default dates, IDs, and strings stable so failures are reproducible.
- Prefer immutable fixture data for unit tests.

If a fixture stores mutable lists or counters globally, tests can start influencing each other. That is one of the fastest ways to create order-dependent failures.

---

### Combining Fixtures and Robots

Fixtures and robots work well together. Fixtures handle data and state setup; robots handle UI interaction. The test body reads as a clean scenario:

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class UserListScreenTest {

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeTestRule = createAndroidComposeRule<HiltTestActivity>()

    @Inject lateinit var fakeUserRepository: FakeUserRepository

    private val robot by lazy { UserListRobot(composeTestRule) }

    @Before
    fun setUp() {
        hiltRule.inject()
        // Fixture provides the data; the test doesn't build User objects inline
        fakeUserRepository.stubbedUsers = UserFixture.aListOfUsers(count = 3)
        composeTestRule.setContent { AppTheme { UserListScreen(onNavigateToDetail = {}) } }
    }

    @Test
    fun displaysAllLoadedUsers() {
        robot
            .assertUserCount(3)
            .assertUserNameVisible("User 1")
            .assertUserNameVisible("User 2")
            .assertUserNameVisible("User 3")
    }

    @Test
    fun tappingUser_opensDetailScreen() {
        robot
            .tapUserAtIndex(0)
            .assertDetailScreenVisible()
    }
}
```

> **Note:** The fixture (`UserFixture.aListOfUsers`) decouples the test from the shape of `User`. If a new field is added to `User`, only `UserFixture` needs updating — the test body is unchanged.

### Common Mistakes

- **Fixture bloat:** giant factories that know every edge case in the app.
- **Hidden intent:** helpers with vague names like `createData1()`.
- **Shared mutable defaults:** a default `mutableListOf()` reused across tests.
- **Over-centralisation:** one global fixture file for the whole project becomes hard to navigate.

A good fixture should make a test easier to understand after one quick read. If the helper makes the setup more mysterious, simplify it.

---

### Best Practices for Fixtures

| Practice | Reason |
|----------|--------|
| Keep factory defaults realistic | Tests that use a valid default catch more real-world bugs than tests seeded with dummy strings like `"test"` |
| Override only what the test cares about | `UserFixture.aUser(email = "")` states the intention clearly — the blank email is what matters, everything else is noise |
| Name factories with `a` / `an` prefix | `aUser()`, `anAuthToken()` reads naturally in test bodies |
| Never share mutable fixture state between tests | Shared mutable state causes test-order dependencies; reset in `@Before` or use `TestWatcher` |
| Co-locate fixtures with tests, not with production code | Fixture classes belong in `src/test/` or `src/androidTest/`, never `src/main/` |
| Use `@After` / `TestWatcher.finished` to clean up resources | Unclosed databases and uncleared MockK state cause false positives in subsequent tests |

---

---

## 6. Unit Tests

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

### Testing ViewModels

ViewModels are the most important unit to test thoroughly. Test every state transition and every event.

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserDetailViewModelTest {

    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val fakeGetUserUseCase = FakeGetUserUseCase()
    private val fakeUpdateUserUseCase = FakeUpdateUserUseCase()
    private val savedStateHandle = SavedStateHandle(mapOf("userId" to "user-1"))

    private val viewModel by lazy {
        UserDetailViewModel(savedStateHandle, fakeGetUserUseCase, fakeUpdateUserUseCase)
    }

    @Test
    fun `init triggers load and emits Success when user is found`() = runTest {
        val user = User("user-1", "Alice", "alice@example.com", null)
        fakeGetUserUseCase.result = Result.success(user)

        val state = viewModel.uiState.value

        assertThat(state).isInstanceOf(UserDetailUiState.Success::class.java)
        assertThat((state as UserDetailUiState.Success).user).isEqualTo(user)
    }

    @Test
    fun `init emits Loading then Success`() = runTest {
        val user = User("user-1", "Alice", "alice@example.com", null)
        fakeGetUserUseCase.result = Result.success(user)

        viewModel.uiState.test {
            assertThat(awaitItem()).isInstanceOf(UserDetailUiState.Loading::class.java)
            assertThat(awaitItem()).isInstanceOf(UserDetailUiState.Success::class.java)
            cancelAndConsumeRemainingEvents()
        }
    }

    @Test
    fun `init emits Error with retryable=true when IOException thrown`() = runTest {
        fakeGetUserUseCase.result = Result.failure(IOException("timeout"))

        val state = viewModel.uiState.value

        assertThat(state).isInstanceOf(UserDetailUiState.Error::class.java)
        assertThat((state as UserDetailUiState.Error).isRetryable).isTrue()
    }

    @Test
    fun `updateName emits ShowSnackbar event on success`() = runTest {
        val user = User("user-1", "Alice", "alice@example.com", null)
        fakeGetUserUseCase.result = Result.success(user)
        fakeUpdateUserUseCase.result = Result.success(Unit)

        viewModel.events.test {
            viewModel.updateName("Bob")
            assertThat(awaitItem()).isEqualTo(UserDetailEvent.ShowSnackbar("Name updated"))
            cancelAndConsumeRemainingEvents()
        }
    }

    @Test
    fun `updateName does nothing when state is not Success`() = runTest {
        fakeGetUserUseCase.result = Result.failure(RuntimeException("error"))
        // State is Error, not Success

        viewModel.events.test {
            viewModel.updateName("Bob")
            expectNoEvents()
            cancelAndConsumeRemainingEvents()
        }
    }
}

// Fakes
class FakeGetUserUseCase : GetUserUseCase(FakeUserRepository()) {
    var result: Result<User> = Result.failure(NotImplementedError())
    override suspend fun execute(params: Params): User = result.getOrThrow()
}

class FakeUpdateUserUseCase : UpdateUserUseCase(FakeUserRepository()) {
    var result: Result<Unit> = Result.success(Unit)
    override suspend fun execute(params: Params): Unit = result.getOrThrow()
}
```

### Testing Use Cases

Use case tests are the simplest: construct with fakes, invoke, assert.

```kotlin
class RegisterUserUseCaseTest {

    private val fakeUserRepository = FakeUserRepository()
    private val fakeAuthRepository = FakeAuthRepository()
    private val emailValidator = RealEmailValidator()

    private val useCase = RegisterUserUseCase(
        fakeUserRepository, fakeAuthRepository, emailValidator
    )

    @Test fun `success with valid input`() = runTest {
        val result = useCase(RegisterUserUseCase.Params("Alice", "alice@example.com", "Password1"))
        assertThat(result.isSuccess).isTrue()
    }

    @Test fun `fails when name is blank`() = runTest {
        val result = useCase(RegisterUserUseCase.Params("  ", "alice@example.com", "Password1"))
        assertThat(result.exceptionOrNull()?.message).ignoringCase().contains("name")
    }

    @Test fun `fails when email is invalid`() = runTest {
        val result = useCase(RegisterUserUseCase.Params("Alice", "not-an-email", "Password1"))
        assertThat(result.exceptionOrNull()?.message).ignoringCase().contains("email")
    }

    @Test fun `fails when password is too short`() = runTest {
        val result = useCase(RegisterUserUseCase.Params("Alice", "alice@example.com", "Pass1"))
        assertThat(result.exceptionOrNull()?.message).contains("8 characters")
    }

    @Test fun `fails when email is already registered`() = runTest {
        fakeUserRepository.existingEmail = "alice@example.com"
        val result = useCase(RegisterUserUseCase.Params("Alice", "alice@example.com", "Password1"))
        assertThat(result.exceptionOrNull()?.message).ignoringCase().contains("already")
    }
}
```

### Testing with MockK

Use MockK when you need to verify call interactions or return sequences that are impractical to fake:

```kotlin
class AuthInterceptorTest {

    private val sessionManager = mockk<SessionManager>()
    private val interceptor = AuthInterceptor(sessionManager)

    @Test
    fun `adds Authorization header when token is present`() {
        every { sessionManager.getToken() } returns "my-token"

        val chain = buildFakeChain("https://api.example.com/users")
        interceptor.intercept(chain)

        val captured = slot<Request>()
        verify { chain.proceed(capture(captured)) }
        assertThat(captured.captured.header("Authorization")).isEqualTo("Bearer my-token")
    }

    @Test
    fun `clears session when server returns 401`() {
        every { sessionManager.getToken() } returns "expired-token"
        every { sessionManager.clearSession() } just Runs

        val chain = buildFakeChainWithResponse(401)
        interceptor.intercept(chain)

        verify(exactly = 1) { sessionManager.clearSession() }
    }

    @Test
    fun `does not clear session for 403 responses`() {
        every { sessionManager.getToken() } returns "valid-token"
        every { sessionManager.clearSession() } just Runs

        val chain = buildFakeChainWithResponse(403)
        interceptor.intercept(chain)

        verify(exactly = 0) { sessionManager.clearSession() }
    }

    private fun buildFakeChain(url: String): Interceptor.Chain {
        val request = Request.Builder().url(url).build()
        val chain = mockk<Interceptor.Chain>()
        every { chain.request() } returns request
        every { chain.proceed(any()) } returns buildResponse(200, chain)
        return chain
    }

    private fun buildFakeChainWithResponse(code: Int): Interceptor.Chain {
        val chain = mockk<Interceptor.Chain>()
        every { chain.request() } returns Request.Builder().url("https://api.example.com/me").build()
        every { chain.proceed(any()) } returns buildResponse(code, chain)
        return chain
    }

    private fun buildResponse(code: Int, chain: Interceptor.Chain): Response =
        Response.Builder()
            .request(chain.request())
            .protocol(Protocol.HTTP_1_1)
            .code(code)
            .message("")
            .body(ResponseBody.create(null, ""))
            .build()
}
```

### Testing Flows with Turbine

Turbine provides a clean DSL for asserting on `Flow` emissions in sequence:

```kotlin
class SearchViewModelTest {

    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val fakeSearchUseCase = FakeSearchUsersUseCase()
    private val viewModel = SearchViewModel(fakeSearchUseCase)

    @Test
    fun `results emit empty list before query reaches minimum length`() = runTest {
        viewModel.results.test {
            assertThat(awaitItem()).isEmpty()  // initial value
            viewModel.onQueryChanged("a")     // too short
            expectNoEvents()                  // debounce + length filter
            cancelAndConsumeRemainingEvents()
        }
    }

    @Test
    fun `results update after debounce with query of 2+ chars`() = runTest {
        val users = listOf(User("1", "Alice", "alice@example.com", null))
        fakeSearchUseCase.stubbedResults = users

        viewModel.results.test {
            assertThat(awaitItem()).isEmpty()
            viewModel.onQueryChanged("al")
            mainDispatcherRule.dispatcher.advanceTimeBy(350) // past debounce
            assertThat(awaitItem()).isEqualTo(users)
            cancelAndConsumeRemainingEvents()
        }
    }
}
```

---

---

## 7. Integration Tests

### What are Integration Tests?

Integration tests verify that **two or more components work correctly together** — usually at least one of which is a real Android framework dependency (Room database, Hilt injection graph, or a real HTTP server via MockWebServer). They live in `src/androidTest/` and require an emulator or device, unless the framework component can be replaced with an in-process substitute (Room in-memory database can run on the JVM for example via `src/test/` with Robolectric, but the `androidTest` approach is more reliable).

### Why Write Integration Tests?

Unit tests verify individual classes in isolation, but they cannot catch **contract mismatches** between layers: a DAO query that compiles but returns the wrong columns; a Retrofit interface that maps a 404 to the wrong exception type; a Room migration that silently drops a column. Integration tests catch exactly these problems.

### When to Write Integration Tests?

Write an integration test when you need to verify:
- A Room DAO query returns the correct data for a given schema.
- A repository correctly coordinates its local and remote data sources.
- A Hilt module wires dependencies in the correct order.
- A network layer handles specific HTTP response codes correctly.

---

### Gradle Setup

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        testInstrumentationRunner = "com.example.app.HiltTestRunner"
    }
}

dependencies {
    // Hilt testing
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.51.1")
    kaptAndroidTest("com.google.dagger:hilt-android-compiler:2.51.1")

    // Room in-memory database
    androidTestImplementation("androidx.room:room-testing:2.6.1")

    // MockWebServer — in-process HTTP server for Retrofit/OkHttp tests
    androidTestImplementation("com.squareup.okhttp3:mockwebserver:4.12.0")

    // AndroidX Test runner and rules
    androidTestImplementation("androidx.test:runner:1.6.1")
    androidTestImplementation("androidx.test:rules:1.6.1")
    androidTestImplementation("androidx.test.ext:junit:1.2.1")

    // MockK for Android (instrumented tests)
    androidTestImplementation("io.mockk:mockk-android:1.13.10")

    // Truth
    androidTestImplementation("com.google.truth:truth:1.4.2")

    // Coroutines test
    androidTestImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
}
```

### Custom Hilt Test Runner

```kotlin
// src/androidTest/kotlin/com/example/app/HiltTestRunner.kt
class HiltTestRunner : AndroidJUnitRunner() {
    override fun newApplication(
        classLoader: ClassLoader,
        className: String,
        context: Context
    ): Application = super.newApplication(classLoader, HiltTestApplication::class.java.name, context)
}
```

---

### Integration Test: Room DAO

The most common integration test: verify DAO queries against a real (in-memory) Room database. These run on device but are the single most reliable way to catch SQL mistakes.

```kotlin
// src/androidTest/kotlin/com/example/app/data/local/dao/UserDaoTest.kt
@RunWith(AndroidJUnit4::class)
@SmallTest
class UserDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao

    @Before
    fun setUp() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        )
            .allowMainThreadQueries()   // allowed in tests only
            .build()
        userDao = database.userDao()
    }

    @After
    fun tearDown() = database.close()

    @Test
    fun upsert_andFindById_returnsInsertedEntity() = runTest {
        val entity = UserEntity("u1", "Alice", "alice@example.com", null)

        userDao.upsert(entity)
        val retrieved = userDao.findById("u1")

        assertThat(retrieved).isEqualTo(entity)
    }

    @Test
    fun findById_returnsNull_whenNotFound() = runTest {
        assertThat(userDao.findById("nonexistent")).isNull()
    }

    @Test
    fun upsert_updatesExistingRow_whenIdMatches() = runTest {
        userDao.upsert(UserEntity("u1", "Alice", "alice@example.com", null))
        userDao.upsert(UserEntity("u1", "Alice Updated", "alice@example.com", null))

        assertThat(userDao.findById("u1")?.name).isEqualTo("Alice Updated")
    }

    @Test
    fun observeAll_emitsNewList_afterUpsert() = runTest {
        val emissions = mutableListOf<List<UserEntity>>()
        val job = launch { userDao.observeAll().take(2).toList(emissions) }

        userDao.upsert(UserEntity("u1", "Alice", "alice@example.com", null))

        job.join()
        assertThat(emissions).hasSize(2)          // initial empty + after insert
        assertThat(emissions[1]).hasSize(1)
        assertThat(emissions[1][0].name).isEqualTo("Alice")
    }

    @Test
    fun deleteById_removesRow() = runTest {
        userDao.upsert(UserEntity("u1", "Alice", "alice@example.com", null))
        userDao.deleteById("u1")
        assertThat(userDao.findById("u1")).isNull()
    }

    @Test
    fun search_returnMatchingByName() = runTest {
        userDao.upsertAll(listOf(
            UserEntity("u1", "Alice Smith", "a@example.com", null),
            UserEntity("u2", "Bob Jones",  "b@example.com", null)
        ))

        val results = userDao.search("alice").first()

        assertThat(results).hasSize(1)
        assertThat(results[0].id).isEqualTo("u1")
    }
}
```

---

### Integration Test: Repository + Room

Test the repository's cache strategy with a real in-memory DAO and a fake remote data source:

```kotlin
// src/androidTest/kotlin/com/example/app/data/repository/UserRepositoryIntegrationTest.kt
@RunWith(AndroidJUnit4::class)
class UserRepositoryIntegrationTest {

    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao
    private lateinit var fakeRemote: FakeUserRemoteDataSource
    private lateinit var repository: UserRepositoryImpl

    @Before
    fun setUp() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).build()
        userDao    = database.userDao()
        fakeRemote = FakeUserRemoteDataSource()

        repository = UserRepositoryImpl(
            remoteDataSource = fakeRemote,
            localDataSource  = UserLocalDataSourceImpl(userDao),
            inMemoryCache    = UserInMemoryCache(),
            networkMonitor   = FakeNetworkMonitor(isConnected = true)
        )
    }

    @After fun tearDown() = database.close()

    @Test
    fun getUser_fetchesFromRemote_andCachesLocally_whenDbIsEmpty() = runTest {
        fakeRemote.stubbedUser = UserDto("u1", "Alice", "alice@example.com", null)

        val result = repository.getUser("u1")

        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrThrow().name).isEqualTo("Alice")
        assertThat(userDao.findById("u1")).isNotNull()   // persisted to local DB
    }

    @Test
    fun getUser_returnsFromLocalDb_withoutHittingNetwork_whenCached() = runTest {
        userDao.upsert(UserEntity("u1", "Alice", "alice@example.com", null))

        val result = repository.getUser("u1")

        assertThat(result.isSuccess).isTrue()
        assertThat(fakeRemote.fetchUserCallCount).isEqualTo(0)
    }

    @Test
    fun observeUsers_emitsUpdate_afterRefresh() = runTest {
        fakeRemote.stubbedUsers = listOf(
            UserDto("u1", "Alice", "alice@example.com", null),
            UserDto("u2", "Bob",   "bob@example.com",   null)
        )

        val results = mutableListOf<List<User>>()
        val job = launch { repository.observeUsers().take(2).toList(results) }

        repository.refreshUsers()
        job.join()

        assertThat(results.last()).hasSize(2)
    }
}
```

---

### Integration Test: Network Layer with MockWebServer

```kotlin
// src/androidTest/kotlin/com/example/app/data/remote/UserApiServiceTest.kt
@RunWith(AndroidJUnit4::class)
class UserApiServiceTest {

    private lateinit var mockWebServer: MockWebServer
    private lateinit var apiService: UserApiService

    @Before
    fun setUp() {
        mockWebServer = MockWebServer()
        mockWebServer.start()

        val retrofit = Retrofit.Builder()
            .baseUrl(mockWebServer.url("/"))
            .addConverterFactory(
                Json { ignoreUnknownKeys = true }
                    .asConverterFactory("application/json".toMediaType())
            )
            .build()

        apiService = retrofit.create(UserApiService::class.java)
    }

    @After fun tearDown() = mockWebServer.shutdown()

    @Test
    fun getUser_parsesResponseCorrectly() = runTest {
        mockWebServer.enqueue(
            MockResponse()
                .setResponseCode(200)
                .setHeader("Content-Type", "application/json")
                .setBody("""{"user_id":"u1","full_name":"Alice","email_address":"alice@example.com"}""")
        )

        val dto = apiService.getUser("u1")

        assertThat(dto.userId).isEqualTo("u1")
        assertThat(dto.fullName).isEqualTo("Alice")
    }

    @Test
    fun getUser_throwsHttpException_on404() = runTest {
        mockWebServer.enqueue(MockResponse().setResponseCode(404))

        val exception = assertThrows(HttpException::class.java) {
            runBlocking { apiService.getUser("nonexistent") }
        }
        assertThat(exception.code()).isEqualTo(404)
    }

    @Test
    fun requestIncludesCorrectPath() = runTest {
        mockWebServer.enqueue(
            MockResponse().setResponseCode(200).setBody(
                """{"user_id":"u1","full_name":"Alice","email_address":"alice@example.com"}"""
            )
        )

        apiService.getUser("u1")

        val request = mockWebServer.takeRequest()
        assertThat(request.path).isEqualTo("/users/u1")
        assertThat(request.method).isEqualTo("GET")
    }
}
```

### Integration Test: Hilt Component Graph

```kotlin
// src/androidTest/kotlin/com/example/app/di/DependencyGraphTest.kt
@HiltAndroidTest
class DependencyGraphTest {

    @get:Rule val hiltRule = HiltAndroidRule(this)

    @Inject lateinit var userRepository: UserRepository
    @Inject lateinit var getUserUseCase: GetUserUseCase

    @Before fun inject() = hiltRule.inject()

    @Test
    fun userRepository_isInjected() {
        assertThat(userRepository).isNotNull()
    }

    @Test
    fun getUserUseCase_isInjected() {
        assertThat(getUserUseCase).isNotNull()
    }
}
```

---

---

## 8. UI Tests with Jetpack Compose

### What are UI Tests?

UI tests interact with your app's interface the way a real user would — they find elements, tap them, type text, scroll lists, and verify what is displayed. In a Compose app, Jetpack Compose Test replaces Espresso as the primary UI testing framework. Tests run against a real rendered Compose tree using a test `Activity` that hosts the composable under test.

### Why Write UI Tests?

Unit tests cannot verify that the UI reacts correctly to state changes. A ViewModel test can verify that `_uiState.value = UiState.Success(user)` is reached, but only a UI test can verify that the correct name appears on screen, that the loading spinner disappears, and that clicking the button triggers the right action. UI tests are the only tests that catch wiring errors between the ViewModel and the composable.

### When to Write UI Tests?

Write UI tests for:
- **Critical user journeys** (login flow, checkout, onboarding).
- **Screen-level state rendering** (loading → success → error transitions).
- **Interactive components** (forms, dialogs, bottom sheets).
- **Navigation** (tapping an item navigates to the detail screen).

Do not write UI tests for visual styling (use screenshot tests instead) or pure logic (use unit tests instead).

---

### Gradle Setup

```kotlin
// app/build.gradle.kts
dependencies {
    // Compose UI test — core library
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.6.8")

    // Needed to host composables in a test Activity
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.6.8")

    // Hilt for instrumented tests
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.51.1")
    kaptAndroidTest("com.google.dagger:hilt-android-compiler:2.51.1")

    // AndroidX Test
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test:rules:1.6.1")

    // Truth
    androidTestImplementation("com.google.truth:truth:1.4.2")
}
```

---

### ComposeTestRule — The Entry Point

`ComposeTestRule` hosts a composable inside a test `Activity` and provides the API for finding nodes, performing actions, and making assertions.

```kotlin
// For testing a Composable in isolation — no Activity needed
@get:Rule val composeTestRule = createComposeRule()

// For testing with a real Activity (navigation, system back, etc.)
@get:Rule val activityRule = createAndroidComposeRule<MainActivity>()
```

---

### Finders, Actions, and Assertions

#### Finders — Locate a node in the composition

```kotlin
// By text content (case-sensitive by default)
composeTestRule.onNodeWithText("Sign In")
composeTestRule.onNodeWithText("Sign In", ignoreCase = true)

// By content description (for icons / images)
composeTestRule.onNodeWithContentDescription("Back")

// By test tag — the most reliable selector
composeTestRule.onNodeWithTag("email_field")

// By role
composeTestRule.onNode(hasClickAction())

// Combine matchers
composeTestRule.onNode(hasText("Alice") and hasClickAction())

// All matching nodes
composeTestRule.onAllNodesWithText("Delete")
```

Set test tags in production code:
```kotlin
OutlinedTextField(
    value = email,
    onValueChange = onEmailChanged,
    modifier = Modifier.testTag("email_field")
)
```

#### Actions — Interact with a node

```kotlin
composeTestRule.onNodeWithText("Sign In").performClick()
composeTestRule.onNodeWithTag("email_field").performTextInput("alice@example.com")
composeTestRule.onNodeWithTag("email_field").performTextClearance()
composeTestRule.onNodeWithTag("password_field").performTextInput("Password1")
composeTestRule.onNodeWithTag("user_list").performScrollToIndex(50)
composeTestRule.onNodeWithTag("user_card_0").performTouchInput { swipeLeft() }
```

#### Assertions — Verify node state

```kotlin
composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
composeTestRule.onNodeWithText("Alice").assertExists()
composeTestRule.onNodeWithText("DeletedItem").assertDoesNotExist()
composeTestRule.onNodeWithTag("submit_button").assertIsEnabled()
composeTestRule.onNodeWithTag("submit_button").assertIsNotEnabled()
composeTestRule.onNodeWithTag("email_field").assertTextContains("alice@example.com")
composeTestRule.onAllNodesWithTag("user_card").assertCountEquals(3)
```

---

### Testing a Stateless Composable in Isolation

Always prefer testing stateless composables directly — no ViewModel needed:

```kotlin
@RunWith(AndroidJUnit4::class)
class UserCardTest {

    @get:Rule val composeTestRule = createComposeRule()

    private val testUser = User("u1", "Alice Smith", "alice@example.com", null)

    @Test
    fun displaysUserNameAndEmail() {
        composeTestRule.setContent {
            AppTheme { UserCard(user = testUser, onUserClick = {}) }
        }

        composeTestRule.onNodeWithText("Alice Smith").assertIsDisplayed()
        composeTestRule.onNodeWithText("alice@example.com").assertIsDisplayed()
    }

    @Test
    fun clickInvokesOnUserClickWithCorrectId() {
        var clickedId = ""
        composeTestRule.setContent {
            AppTheme { UserCard(user = testUser, onUserClick = { clickedId = it }) }
        }

        composeTestRule.onNodeWithText("Alice Smith").performClick()

        assertThat(clickedId).isEqualTo("u1")
    }

    @Test
    fun errorStateDisplaysRetryButton() {
        composeTestRule.setContent {
            AppTheme {
                ErrorScreen(
                    message = "Something went wrong",
                    onRetry = {}
                )
            }
        }

        composeTestRule.onNodeWithText("Something went wrong").assertIsDisplayed()
        composeTestRule.onNodeWithText("Retry").assertIsDisplayed()
    }
}
```

---

### Testing a Screen with a ViewModel (Hilt)

Use `@HiltAndroidTest` for screens that get their ViewModel via `hiltViewModel()`:

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class UserListScreenTest {

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeTestRule = createAndroidComposeRule<HiltTestActivity>()

    @Inject lateinit var fakeUserRepository: FakeUserRepository

    @Before fun setUp() { hiltRule.inject() }

    @Test
    fun showsLoadingIndicator_initially() {
        fakeUserRepository.isLoading = true
        composeTestRule.setContent { AppTheme { UserListScreen(onNavigateToDetail = {}) } }

        composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }

    @Test
    fun showsUserList_whenDataLoads() {
        fakeUserRepository.stubbedUsers = listOf(
            User("u1", "Alice", "alice@example.com", null),
            User("u2", "Bob",   "bob@example.com",   null)
        )
        composeTestRule.setContent { AppTheme { UserListScreen(onNavigateToDetail = {}) } }

        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
        composeTestRule.onNodeWithText("Bob").assertIsDisplayed()
    }

    @Test
    fun showsErrorMessage_whenRepositoryFails() {
        fakeUserRepository.shouldThrow = IOException("Network unavailable")
        composeTestRule.setContent { AppTheme { UserListScreen(onNavigateToDetail = {}) } }

        composeTestRule.onNodeWithText("Network unavailable", substring = true).assertIsDisplayed()
    }
}

// Replacement module for tests
@TestInstallIn(components = [SingletonComponent::class], replaces = [RepositoryModule::class])
@Module
abstract class FakeRepositoryModule {
    @Binds @Singleton
    abstract fun bindUserRepository(fake: FakeUserRepository): UserRepository
}
```

---

### Testing Navigation

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class NavigationTest {

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeTestRule = createAndroidComposeRule<HiltTestActivity>()

    @Inject lateinit var fakeUserRepository: FakeUserRepository

    @Before
    fun setUp() {
        hiltRule.inject()
        fakeUserRepository.stubbedUsers = listOf(User("u1", "Alice", "alice@example.com", null))
    }

    @Test
    fun tappingUser_navigatesToDetailScreen() {
        composeTestRule.setContent { AppTheme { AppNavHost() } }

        // We're on the list screen
        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()

        // Tap the user card
        composeTestRule.onNodeWithText("Alice").performClick()

        // We should now be on the detail screen
        composeTestRule.onNodeWithTag("user_detail_screen").assertIsDisplayed()
        composeTestRule.onNodeWithText("alice@example.com").assertIsDisplayed()
    }
}
```

---

### Handling Asynchronous State

Compose Test has built-in idle synchronisation — it waits for the Compose rendering and coroutines to settle before checking assertions. For explicit waiting, use `waitUntil`:

```kotlin
// Wait up to 5 seconds for a condition to become true
composeTestRule.waitUntil(timeoutMillis = 5_000) {
    composeTestRule.onAllNodesWithText("Alice").fetchSemanticsNodes().isNotEmpty()
}

composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
```

---

---

## 9. Screenshot Tests

### What are Screenshot Tests?

Screenshot tests (also called visual regression tests or pixel tests) capture the rendered output of a composable or screen as a reference image — a **golden** — and then compare every subsequent test run against that golden. If any pixel changes, the test fails and shows a diff. This catches accidental visual regressions that would otherwise slip through unit and UI tests.

### Why Write Screenshot Tests?

Unit and UI tests verify **behaviour**, not **appearance**. A UI test that asserts `onNodeWithText("Alice").assertIsDisplayed()` does not notice if the text colour changed from black to white, the card padding doubled, or a shadow disappeared. Screenshot tests fill this gap — they are your safeguard against unintended visual changes during refactoring, dependency upgrades, or theme changes.

### When to Write Screenshot Tests?

Write screenshot tests for:
- **Design-system components** (buttons, cards, text fields, dialogs) — the building blocks used everywhere.
- **Complex composables** with conditional rendering (loading / error / empty / success states).
- **Screens that must match a designer-approved specification** exactly.

Do not write screenshot tests for screens whose visual output changes frequently (e.g., feeds with live data) — these would require constant golden updates.

---

### Jetpack Screenshot Testing (Official Approach — AGP 8.3+)

Google's official screenshot testing toolchain uses the `screenshotTest` Gradle source set introduced in Android Gradle Plugin 8.3. Tests live in `src/screenshotTest/kotlin/`, render Compose previews on the JVM (no emulator required), and produce PNG golden images checked into version control.

#### Gradle Setup

```kotlin
// project-level build.gradle.kts
plugins {
    id("com.android.application") version "8.5.0" apply false
}

// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    compileSdk = 35

    testOptions {
        screenshotTest {
            enabled = true
        }
    }
}

dependencies {
    // Required for rendering @Preview composables in screenshotTest source set
    screenshotTestImplementation("androidx.compose.ui:ui-tooling:1.6.8")
}
```

#### Source Set

Screenshot tests go in a dedicated source set that is separate from both `test/` and `androidTest/`:

```
app/src/
├── main/kotlin/                   # Production code
├── test/kotlin/                   # JVM unit tests
├── androidTest/kotlin/            # Device instrumented tests
└── screenshotTest/kotlin/         # Screenshot tests (JVM, no emulator)
    └── com/example/app/
        ├── UserCardScreenshotTest.kt
        └── UserListScreenScreenshotTest.kt
```

#### Writing Screenshot Tests

Each test class is annotated with `@PreviewScreenshotTest` and calls `captureScreenshot()` on composables decorated with `@Preview`:

```kotlin
// src/screenshotTest/kotlin/com/example/app/UserCardScreenshotTest.kt
@RunWith(ParameterizedRobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class UserCardScreenshotTest(private val name: String, private val uiMode: Int) {

    companion object {
        @JvmStatic
        @ParameterizedRobolectricTestRunner.Parameters(name = "{0}")
        fun parameters() = listOf(
            arrayOf("light", Configuration.UI_MODE_NIGHT_NO),
            arrayOf("dark",  Configuration.UI_MODE_NIGHT_YES)
        )
    }

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun userCard_default() {
        composeTestRule.captureScreenshot(uiMode = uiMode, fileName = "user_card_$name")
    }

    private fun ComposeContentTestRule.captureScreenshot(uiMode: Int, fileName: String) {
        setContent {
            CompositionLocalProvider(LocalConfiguration provides Configuration().apply {
                this.uiMode = uiMode
            }) {
                AppTheme(darkTheme = uiMode and Configuration.UI_MODE_NIGHT_MASK == Configuration.UI_MODE_NIGHT_YES) {
                    UserCard(
                        user = User("u1", "Alice Smith", "alice@example.com", null),
                        onUserClick = {}
                    )
                }
            }
        }
        onRoot().captureToImage().saveTo(File("src/screenshotTest/screenshots/$fileName.png"))
    }
}
```

#### Using `@PreviewScreenshotTest` Annotation (Simpler Approach)

For composables that already have `@Preview` annotations, the toolchain can auto-generate screenshot tests:

```kotlin
// Production code — in the main source set
@Preview(name = "Light Mode", showBackground = true)
@Preview(name = "Dark Mode", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Composable
fun UserCardPreview() {
    AppTheme {
        UserCard(
            user = User("u1", "Alice Smith", "alice@example.com", null),
            onUserClick = {}
        )
    }
}

// Screenshot test — in screenshotTest source set
class UserCardScreenshotTest {
    @get:Rule val screenshotRule = AndroidComposableScreenshotRule()

    @Test
    @PreviewScreenshotTest
    fun userCardPreviews() = screenshotRule.assertPreviews(UserCardPreview::class)
}
```

#### Running Screenshot Tests

```bash
# Generate / update goldens (first run, or after intentional UI changes)
./gradlew :app:updateDebugScreenshotTest

# Verify against existing goldens (CI — fails on any pixel difference)
./gradlew :app:verifyDebugScreenshotTest

# View the HTML report with diffs
open app/build/reports/screenshotTest/debug/verify/index.html
```

#### Checking Goldens into Version Control

Golden files are PNG images stored in `src/screenshotTest/screenshots/`. They must be committed to version control so CI can compare against them:

```bash
git add src/screenshotTest/screenshots/
git commit -m "test: add golden screenshots for UserCard"
```

---

### What Happens When a Test Fails

When `verifyDebugScreenshotTest` detects a pixel difference, it:
1. Fails the build.
2. Generates a diff image in `app/build/outputs/screenshotTest/`.
3. Produces an HTML report showing the expected, actual, and diff side-by-side.

To accept the change as intentional, re-run `updateDebugScreenshotTest`, review the new goldens, and commit them.

---

### Full Multi-State Screenshot Test Example

```kotlin
// src/screenshotTest/kotlin/com/example/app/UserDetailScreenshotTest.kt
class UserDetailScreenshotTest {

    @get:Rule val composeTestRule = createComposeRule()

    @Test
    fun userDetailSuccess_lightTheme() {
        composeTestRule.setContent {
            AppTheme(darkTheme = false) {
                UserDetailContent(
                    user = User("u1", "Alice Smith", "alice@example.com", "https://example.com/avatar.jpg")
                )
            }
        }
        composeTestRule.onRoot().captureToImage()
            .assertAgainstGolden(screenshotRule, "user_detail_success_light")
    }

    @Test
    fun userDetailError_lightTheme() {
        composeTestRule.setContent {
            AppTheme(darkTheme = false) {
                ErrorScreen(message = "User not found", onRetry = {})
            }
        }
        composeTestRule.onRoot().captureToImage()
            .assertAgainstGolden(screenshotRule, "user_detail_error_light")
    }

    @Test
    fun userDetailLoading_lightTheme() {
        composeTestRule.setContent {
            AppTheme(darkTheme = false) {
                LoadingIndicator()
            }
        }
        composeTestRule.onRoot().captureToImage()
            .assertAgainstGolden(screenshotRule, "user_detail_loading_light")
    }
}
```

---

### Best Practices for Screenshot Tests

| Practice | Reason |
|----------|--------|
| Test all visual states (loading, error, empty, success) | Each state is a distinct visual component |
| Test both light and dark themes | Theme bugs are common and easy to miss |
| Use fixed preview data — never random or date-based | Flaky goldens from non-deterministic data are useless |
| Keep composables under test narrow (component, not full screen) | Smaller goldens fail for smaller, more actionable reasons |
| Store goldens in version control, never in `.gitignore` | CI needs them to detect regressions |
| Review golden diffs in PRs like you review code changes | Unexpected visual changes should require explicit approval |

---

## License

This document is distributed under the **GNU General Public License v3.0 (GPL-3.0)**.

```
Android Testing Guide Documentation
Copyright (C) 2024 — Contributors

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <https://www.gnu.org/licenses/>.
```

For the full license text, see the [LICENSE](./LICENSE) file in this repository or visit
<https://www.gnu.org/licenses/gpl-3.0.html>.

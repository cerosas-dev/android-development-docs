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

## 4. Integration Tests

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


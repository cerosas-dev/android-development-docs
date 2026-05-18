# Android Clean Architecture & Best Practices

> A comprehensive reference guide for professional Android development using Kotlin, Jetpack libraries, and industry-standard architectural patterns.

## Table of Contents

1. [MVVM Architecture](#1-mvvm-architecture)
2. [Recommended Codebase Structure](#2-recommended-codebase-structure)
3. [SOLID Principles](#3-solid-principles)
4. [Clean Code](#4-clean-code)
5. [Threading, Coroutines & Dispatchers](#5-threading-coroutines--dispatchers)
6. [Data Sources](#6-data-sources)
7. [Repositories](#7-repositories)
8. [Use Cases](#8-use-cases)
9. [ViewModel](#9-viewmodel)
10. [Jetpack Compose](#10-jetpack-compose)
11. [Network Operations with OkHttp](#11-network-operations-with-okhttp)
12. [Dependency Injection with Hilt](#12-dependency-injection-with-hilt)
13. [Data Storage](#13-data-storage)
    - [Room](#room)
    - [SharedPreferences & SecureSharedPreferences](#sharedpreferences--securesharedpreferences)
14. [Image Loading with COIL](#14-image-loading-with-coil)

---

## 1. MVVM Architecture

### What is MVVM?

Model-View-ViewModel (MVVM) is a software architectural pattern that separates an application into three interconnected layers:

- **Model** — the data and business logic layer (repositories, data sources, domain models).
- **View** — the UI layer (Composables, Activities, Fragments) that displays data and captures user input.
- **ViewModel** — the mediator that holds and exposes UI state, handles user events, and communicates with the Model.

MVVM is the officially recommended architecture for Android by Google and forms the backbone of the modern Android development stack (Jetpack, Compose, Hilt).

### Why MVVM?

Before MVVM, developers used MVC (where Activity was the Controller) or MVP (where a Presenter held a direct View reference). Both caused serious problems:

| Pattern | Core Problem |
|---------|-------------|
| MVC | Activity mixes UI, input handling, and business logic. Untestable without a device. |
| MVP | Presenter holds View reference → memory leaks on rotation. Manual lifecycle management required. |
| MVVM | ViewModel has no reference to the View. Survives configuration changes. All logic is JVM unit-testable. |

### When to Use MVVM?

Always — for any screen in your app, no matter how simple. Even a single-screen app benefits from MVVM because it separates concerns and makes the screen testable from day one.

### Unidirectional Data Flow (UDF)

MVVM in Android follows a strict UDF model. Events flow *up* from View to ViewModel; state flows *down* from ViewModel to View. State never travels backwards.

```
┌─────────────────────────────────────┐
│               View                  │  ← Activity / Fragment / Composable
│   Renders state. Forwards events.   │
└──────────┬────────────▲─────────────┘
           │ Events     │ UiState
           ▼            │
┌──────────────────────────────────────┐
│             ViewModel                │  ← Holds UI state, calls UseCases
│   No Android framework dependencies  │
└──────────┬────────────▲──────────────┘
           │ Calls      │ Result<T>
           ▼            │
┌──────────────────────────────────────┐
│             Use Case                 │  ← Single business operation
└──────────┬────────────▲──────────────┘
           │ Reads/     │ Domain
           ▼ Writes     │ Models
┌──────────────────────────────────────┐
│            Repository                │  ← Abstracts data origin
└──────────┬────────────▲──────────────┘
           │            │
           ▼            │
┌──────────────────────────────────────┐
│           Data Sources               │  ← Remote API / Local DB / Cache
└──────────────────────────────────────┘
```

### Key Principles

- **View** knows nothing about business logic. It only renders state and forwards events.
- **ViewModel** survives configuration changes. It holds and exposes `StateFlow<UiState>`.
- **Repository** is the single source of truth. It decides whether to fetch from network or cache.
- **Use Cases** encapsulate one and only one business rule.

### Complete MVVM Example

**Domain model — pure Kotlin, zero Android imports:**

```kotlin
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?
)
```

**UI State — models every possible screen state explicitly:**

```kotlin
sealed class UserUiState {
    object Loading : UserUiState()
    data class Success(val user: User) : UserUiState()
    data class Error(val message: String, val canRetry: Boolean = true) : UserUiState()
}
```

**ViewModel — the single source of truth for the UI:**

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            getUserUseCase(GetUserUseCase.Params(userId))
                .onSuccess { user -> _uiState.value = UserUiState.Success(user) }
                .onFailure { e ->
                    _uiState.value = UserUiState.Error(
                        message  = e.message ?: "Unknown error",
                        canRetry = e is IOException
                    )
                }
        }
    }
}
```

**Composable View — reacts to state, forwards events:**

```kotlin
@Composable
fun UserScreen(
    viewModel: UserViewModel = hiltViewModel(),
    userId: String
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(userId) { viewModel.loadUser(userId) }

    when (val state = uiState) {
        is UserUiState.Loading -> CircularProgressIndicator()
        is UserUiState.Success -> UserContent(state.user)
        is UserUiState.Error   -> ErrorMessage(
            message = state.message,
            onRetry = if (state.canRetry) ({ viewModel.loadUser(userId) }) else null
        )
    }
}
```

### Unit Testing the ViewModel

The ViewModel has zero Android framework dependencies, so it can be tested with plain JUnit + coroutines:

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserViewModelTest {

    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val fakeGetUserUseCase = FakeGetUserUseCase()
    private lateinit var viewModel: UserViewModel

    @Before
    fun setUp() { viewModel = UserViewModel(fakeGetUserUseCase) }

    @Test
    fun `loadUser emits Success when use case returns user`() = runTest {
        val user = User("1", "Alice", "alice@example.com", null)
        fakeGetUserUseCase.result = Result.success(user)

        viewModel.loadUser("1")

        assertThat(viewModel.uiState.value).isEqualTo(UserUiState.Success(user))
    }

    @Test
    fun `loadUser emits Error when use case fails`() = runTest {
        fakeGetUserUseCase.result = Result.failure(IOException("No network"))

        viewModel.loadUser("1")

        val state = viewModel.uiState.value as UserUiState.Error
        assertThat(state.canRetry).isTrue()
    }
}
```

---

## 2. Recommended Codebase Structure

### What is It?

A well-defined codebase structure is a consistent, agreed-upon arrangement of files and packages that reflects the architectural boundaries of the application. It is not just a folder convention — it is a physical enforcement of the dependency rules in Clean Architecture.

### Why Structure Matters

Without a clear structure:
- Developers place logic in the wrong layer (business rules inside the ViewModel or Composable).
- Cross-layer dependencies accumulate silently and become impossible to untangle.
- Onboarding new developers takes days instead of hours.
- Testing becomes fragile because tightly coupled layers cannot be isolated.

A well-structured project makes the architecture self-documenting — you can open any folder and instantly understand what lives there and what it depends on.

### When to Apply It

Apply the structure from the very first commit. Retrofitting it onto a large, flat project is expensive. The cost of getting structure right at the start is zero; the cost of ignoring it grows with every new feature.

### Layer Dependency Rules

The core rule: **dependencies only point inward**.

```
presentation  ──►  domain  ◄──  data
```

- `domain` knows nothing about `data` or `presentation`.
- `data` implements interfaces defined in `domain`.
- `presentation` calls use cases from `domain` and never accesses `data` directly.

### Single-Module Package Structure

For most apps, a single Gradle module with feature-based sub-packages is the right starting point:

```
com.example.app/
│
├── MyApplication.kt                  # @HiltAndroidApp entry point
│
├── di/                               # Hilt dependency injection modules
│   ├── NetworkModule.kt
│   ├── DatabaseModule.kt
│   ├── RepositoryModule.kt
│   └── DataSourceModule.kt
│
├── domain/                           # Pure Kotlin — zero Android/framework imports
│   ├── model/                        # Domain entities (User, Product, Order…)
│   │   └── User.kt
│   ├── repository/                   # Repository interfaces (contracts)
│   │   └── UserRepository.kt
│   └── usecase/                      # One class per business operation
│       ├── GetUserUseCase.kt
│       ├── LoginUseCase.kt
│       └── ObserveUsersUseCase.kt
│
├── data/                             # Implements domain interfaces
│   ├── remote/
│   │   ├── api/
│   │   │   └── UserApiService.kt     # Retrofit interface
│   │   ├── dto/
│   │   │   └── UserDto.kt            # JSON-mapped data class
│   │   └── datasource/
│   │       └── UserRemoteDataSourceImpl.kt
│   ├── local/
│   │   ├── database/
│   │   │   └── AppDatabase.kt        # Room database
│   │   ├── dao/
│   │   │   └── UserDao.kt
│   │   ├── entity/
│   │   │   └── UserEntity.kt
│   │   └── datasource/
│   │       └── UserLocalDataSourceImpl.kt
│   ├── preferences/
│   │   ├── AppPreferences.kt         # SharedPreferences wrapper
│   │   └── SecurePreferences.kt      # EncryptedSharedPreferences wrapper
│   ├── mapper/
│   │   └── UserMappers.kt            # Extension functions: Dto→Domain, Entity→Domain
│   └── repository/
│       └── UserRepositoryImpl.kt
│
└── presentation/                     # UI layer — Compose screens + ViewModels
    ├── navigation/
    │   └── AppNavHost.kt             # NavHost + all route definitions
    ├── theme/
    │   ├── AppTheme.kt
    │   ├── Color.kt
    │   └── Type.kt
    └── feature/
        ├── userlist/
        │   ├── UserListScreen.kt
        │   ├── UserListViewModel.kt
        │   └── UserListUiState.kt
        └── userdetail/
            ├── UserDetailScreen.kt
            ├── UserDetailViewModel.kt
            └── UserDetailUiState.kt
```

### Multi-Module Structure (Large Teams / Apps)

For apps with multiple teams or features that need strict isolation, split into Gradle modules:

```
:app                    # Entry point — ties everything together
:core:domain            # Pure Kotlin — interfaces, models, use case bases
:core:data              # Implements :core:domain interfaces
:core:network           # OkHttp, Retrofit, interceptors
:core:database          # Room database, DAOs, entities
:core:ui                # Shared Composables, theme, design system
:feature:userlist       # Self-contained feature: screen + ViewModel
:feature:userdetail     # Self-contained feature: screen + ViewModel
:feature:auth           # Login / registration flow
```

Each `:feature` module depends only on `:core:domain` (not on `:core:data`), enforcing the clean architecture boundary at the build level.

### File Naming Conventions

| Layer | Suffix | Example |
|-------|--------|---------|
| Domain model | *(none)* | `User.kt` |
| Remote DTO | `Dto` | `UserDto.kt` |
| Room Entity | `Entity` | `UserEntity.kt` |
| Room DAO | `Dao` | `UserDao.kt` |
| Remote data source | `RemoteDataSourceImpl` | `UserRemoteDataSourceImpl.kt` |
| Local data source | `LocalDataSourceImpl` | `UserLocalDataSourceImpl.kt` |
| Repository interface | `Repository` | `UserRepository.kt` |
| Repository impl | `RepositoryImpl` | `UserRepositoryImpl.kt` |
| Use case | `UseCase` | `GetUserUseCase.kt` |
| ViewModel | `ViewModel` | `UserListViewModel.kt` |
| Composable screen | `Screen` | `UserListScreen.kt` |
| UI state | `UiState` | `UserListUiState.kt` |
| Hilt module | `Module` | `NetworkModule.kt` |

---

## 3. SOLID Principles

### What are SOLID Principles?

SOLID is an acronym for five object-oriented design principles introduced by Robert C. Martin. They guide developers toward writing code that is maintainable, extensible, and robust against change.

### Why Use SOLID?

Code that violates SOLID principles becomes a "big ball of mud" — a system where changing one thing unexpectedly breaks another, where classes grow without limit, and where adding a new feature requires touching dozens of files. SOLID principles are preventive medicine for these problems.

### When to Apply SOLID?

Always, but especially when you feel the urge to add "just one more thing" to an existing class. Each violation feels harmless in isolation; the damage accumulates over months.

---

### S — Single Responsibility Principle (SRP)

> A class should have only one reason to change.

**What it means:** Each class should do exactly one job. If it needs to change for two different reasons, it violates SRP.

**Why it matters:** Small, focused classes are easy to understand, test, and change in isolation.

**Android violation — the God Activity:**

```kotlin
class UserActivity : AppCompatActivity() {
    fun loadUser(id: String) {
        val response = URL("https://api.example.com/users/$id").readText()
        val json = JSONObject(response)
        val user = User(json.getString("id"), json.getString("name"))
        database.save(user)
        nameTextView.text = "${user.name.uppercase()} (${user.id})"
    }
}
```

**Corrected — each class has one job:**

```kotlin
class UserRemoteDataSourceImpl @Inject constructor(
    private val apiService: UserApiService
) : UserRemoteDataSource {
    override suspend fun fetchUser(id: String): UserDto = apiService.getUser(id)
}

class UserLocalDataSourceImpl @Inject constructor(
    private val userDao: UserDao
) : UserLocalDataSource {
    override suspend fun saveUser(user: UserEntity) = userDao.upsert(user)
}

class UserFormatter {
    fun formatDisplayName(user: User): String = "${user.name.uppercase()} (${user.id})"
}
```

---

### O — Open/Closed Principle (OCP)

> Software entities should be open for extension, but closed for modification.

**What it means:** Add new behavior by adding new classes, not by editing existing ones.

**Why it matters:** Every time you modify tested, working code, you risk regressions.

```kotlin
interface PaymentProcessor {
    suspend fun process(amount: Double, currency: String): PaymentResult
}

class StripePaymentProcessor @Inject constructor(private val api: StripeApi) : PaymentProcessor {
    override suspend fun process(amount: Double, currency: String) = api.charge(amount, currency).toResult()
}

class PayPalPaymentProcessor @Inject constructor(private val api: PayPalApi) : PaymentProcessor {
    override suspend fun process(amount: Double, currency: String) = api.createPayment(amount, currency).toResult()
}

// Adding Google Pay requires zero changes to CheckoutUseCase
class CheckoutUseCase @Inject constructor(
    private val paymentProcessor: PaymentProcessor
) {
    suspend fun execute(order: Order): Result<PaymentResult> = runCatching {
        paymentProcessor.process(order.total, order.currency)
    }
}
```

---

### L — Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types without altering program correctness.

**What it means:** A subtype must honour the contract of its base type — same expectations, no surprise exceptions.

**Why it matters:** Violations break polymorphism. If callers must check `is Square` before using a `Shape`, the abstraction is broken.

```kotlin
// Violation: Penguin breaks the Bird contract
open class Bird { open fun fly() {} }
class Penguin : Bird() { override fun fly() = throw UnsupportedOperationException() }

// Corrected: segregate capabilities into separate interfaces
interface FlyingBird { fun fly() }
interface SwimmingBird { fun swim() }

class Eagle : FlyingBird { override fun fly() { /* soar */ } }
class Penguin : SwimmingBird { override fun swim() { /* dive */ } }
```

---

### I — Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they do not use.

**What it means:** Prefer many small, focused interfaces over one large general-purpose interface.

**Why it matters:** Fat interfaces create unnecessary coupling and force implementations to provide methods they don't need.

```kotlin
// Bad — every client depends on all four capabilities
interface UserRepository {
    suspend fun getUser(id: String): User
    suspend fun saveUser(user: User)
    suspend fun exportToCsv(): ByteArray
    suspend fun generateReport(): Report
}

// Good — fine-grained contracts, composed by the implementation
interface UserReader  { suspend fun getUser(id: String): Result<User> }
interface UserWriter  { suspend fun saveUser(user: User): Result<Unit> }
interface UserExporter { suspend fun exportToCsv(): Result<ByteArray> }

class UserRepositoryImpl : UserReader, UserWriter, UserExporter { /* ... */ }

class UserListViewModel @Inject constructor(private val reader: UserReader) : ViewModel()
class AdminViewModel @Inject constructor(
    private val reader: UserReader,
    private val exporter: UserExporter
) : ViewModel()
```

---

### D — Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

**What it means:** Business logic depends on interfaces, never on concrete implementations like Room DAOs or Retrofit services.

**Why it matters:** Swapping a database or network library becomes a one-class change, not a rewrite.

```kotlin
// Domain layer — the abstraction
interface UserRepository { suspend fun getUser(id: String): Result<User> }

// Domain layer — depends only on the abstraction
class GetUserUseCase @Inject constructor(private val repo: UserRepository) {
    suspend operator fun invoke(id: String) = repo.getUser(id)
}

// Data layer — the detail, implements the abstraction
class UserRepositoryImpl @Inject constructor(
    private val remote: UserRemoteDataSource,
    private val local: UserLocalDataSource
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> = runCatching {
        local.getUser(id)?.toDomain() ?: remote.fetchUser(id).toDomain()
    }
}

// Hilt wires them together — the domain never knows about the impl
@Binds abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
```

---

## 4. Clean Code

### What is Clean Code?

Clean code is code that is easy to read, easy to change, and easy to verify. It is written for the next developer, not just for the compiler. Robert C. Martin defines it as: *"Clean code reads like well-written prose."*

### Why Write Clean Code?

Software spends 80% of its lifetime in maintenance. Every minute saved reading confusing code multiplies across an entire team over months. Clean code is not cosmetic — it is economic.

### When to Write Clean Code?

Always, from the first line. "We'll clean it up later" is almost always false.

---

### Naming

```kotlin
// Bad
fun calc(a: Int, b: Int, t: Double): Double = (a + b) * t
val d = 86400
val usrs = mutableListOf<User>()

// Good
fun calculateOrderTotal(basePrice: Int, taxAmount: Int, discountRate: Double): Double =
    (basePrice + taxAmount) * (1.0 - discountRate)
val secondsInADay = 86_400
val activeUsers = mutableListOf<User>()
```

Booleans always use `is`, `has`, `can`, `should` prefixes:

```kotlin
val isEmailValid: Boolean
val hasNetworkError: Boolean
val canUserEdit: Boolean
val shouldShowOnboarding: Boolean
```

---

### Functions

A function should do exactly one thing. If you need "and" in its name, it's doing two things.

```kotlin
// Bad — validates, persists, and notifies in one function
fun processUser(u: User): Boolean {
    if (!u.email.contains("@")) return false
    db.save(u)
    emailService.send(u.email, "Welcome!")
    return true
}

// Good — each step is independently named, readable, and testable
fun registerUser(user: User): Result<Unit> = runCatching {
    validateUser(user)
    userRepository.save(user)
    notificationService.sendWelcomeEmail(user)
}

private fun validateUser(user: User) {
    require(user.email.isValidEmail()) { "Invalid email: ${user.email}" }
    require(user.name.isNotBlank()) { "Name cannot be blank" }
}
```

---

### Avoid Magic Numbers and Strings

```kotlin
// Bad
if (response.code == 401) logout()
delay(3000)

// Good
private const val HTTP_UNAUTHORIZED = 401
private const val LOGOUT_DELAY_MS   = 3_000L

if (response.code == HTTP_UNAUTHORIZED) logout()
delay(LOGOUT_DELAY_MS)
```

---

### Null Safety

```kotlin
// Bad — verbose Java-style null guards
fun getDisplayName(user: User?): String {
    if (user != null && user.name != null && user.name.isNotEmpty()) return user.name
    return "Guest"
}

// Good — idiomatic Kotlin
fun getDisplayName(user: User?): String =
    user?.name?.takeIf { it.isNotEmpty() } ?: "Guest"
```

---

### Error Handling with Result

```kotlin
suspend fun loadProfile(userId: String): Result<UserProfile> = runCatching {
    val user  = userRepository.getUser(userId).getOrThrow()
    val posts = postRepository.getPosts(userId).getOrThrow()
    UserProfile(user, posts)
}

viewModelScope.launch {
    loadProfile(userId)
        .onSuccess { _state.value = State.Success(it) }
        .onFailure { e ->
            val message = when (e) {
                is IOException   -> "Network error. Check your connection."
                is HttpException -> "Server error (${e.code()})."
                else             -> "Unexpected error."
            }
            _state.value = State.Error(message)
        }
}
```

---

### Kotlin Scope Functions

| Function | Context | Returns | Use when |
|----------|---------|---------|----------|
| `let` | `it` | Lambda result | Null-safe transformation |
| `run` | `this` | Lambda result | Initialize + compute |
| `with` | `this` | Lambda result | Multiple calls on same object |
| `apply` | `this` | The object | Object configuration |
| `also` | `it` | The object | Side effects (logging) |

```kotlin
// apply — configure an object
val intent = Intent(context, UserDetailActivity::class.java).apply {
    putExtra("userId", user.id)
    flags = Intent.FLAG_ACTIVITY_SINGLE_TOP
}

// let — null-safe transform
val displayName = user?.name?.let { it.trim().uppercase() } ?: "UNKNOWN"

// also — log without breaking the chain
val user = userRepository.getUser(id)
    .also { log.d("Fetched: $it") }

// with — call multiple methods on the same object
val summary = with(user) { "$name | $email | Joined: ${createdAt.format()}" }
```

---

### Sealed Classes and When Expressions

```kotlin
sealed class AuthState {
    object Unauthenticated                                    : AuthState()
    object Authenticating                                     : AuthState()
    data class Authenticated(val user: User, val token: String) : AuthState()
    data class Error(val cause: Throwable)                    : AuthState()
}

// Compiler enforces exhaustive handling — no else needed
fun handleAuthState(state: AuthState) = when (state) {
    is AuthState.Unauthenticated  -> showLoginScreen()
    is AuthState.Authenticating   -> showLoadingSpinner()
    is AuthState.Authenticated    -> navigateToHome(state.user)
    is AuthState.Error            -> showError(state.cause.message)
}
```

---

### Coroutines Best Practices

```kotlin
// Never use GlobalScope in production — use viewModelScope or lifecycleScope
viewModelScope.launch { /* tied to ViewModel lifecycle */ }

// Switch dispatchers explicitly with withContext
suspend fun compressImage(bitmap: Bitmap): Bitmap = withContext(Dispatchers.Default) {
    bitmap.compress()
}

// supervisorScope: independent children — one failure doesn't cancel siblings
suspend fun loadDashboard(): Dashboard = supervisorScope {
    val users  = async { userRepository.getUsers() }
    val stats  = async { statsRepository.getStats() }
    Dashboard(
        users  = users.await().getOrDefault(emptyList()),
        stats  = stats.await().getOrDefault(Stats.empty())
    )
}
```

---

## 5. Threading, Coroutines & Dispatchers

### What is Threading?

A **thread** is an independent path of execution within a process. Android apps start with one thread — the **Main thread** (also called the **UI thread**). All UI rendering, touch event handling, and view updates happen here. The system enforces a strict rule: **any operation that blocks the main thread for more than ~16 ms causes a dropped frame; any block over ~5 seconds triggers an Application Not Responding (ANR) dialog.**

The consequence is that all slow work — network calls, database queries, file I/O, complex computations — must be moved off the main thread. Kotlin Coroutines are the modern, idiomatic way to do this on Android.

### What are Kotlin Coroutines?

Coroutines are **lightweight, suspendable units of work** that can pause execution at suspension points (marked with the `suspend` keyword) without blocking the underlying thread. While a coroutine is suspended, its thread is free to do other work. When the suspended operation completes, the coroutine resumes — possibly on a different thread.

**Why not just use threads directly?**

| Problem | Threads | Coroutines |
|---------|---------|------------|
| Memory cost | ~1 MB per thread | ~few KB per coroutine |
| Blocking | Block the OS thread | Suspend — thread is freed |
| Cancellation | Manual, error-prone | Structured, automatic |
| Error propagation | Uncaught exceptions crash the app | Propagated through the hierarchy |
| Android integration | Manual lifecycle management | `viewModelScope`, `lifecycleScope` |

You can run **millions** of coroutines simultaneously where only thousands of threads would exhaust available memory.

### Gradle Dependencies

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
}
```

---

### Suspend Functions

A `suspend` function is one that can pause its execution without blocking a thread. It can only be called from another `suspend` function or from a coroutine builder.

```kotlin
// A regular function — blocks whichever thread calls it
fun loadUserBlocking(id: String): User = api.getUser(id)  // blocks!

// A suspend function — pauses the coroutine, frees the thread
suspend fun loadUser(id: String): User = api.getUser(id)  // suspends ✓
```

Under the hood, the Kotlin compiler transforms `suspend` functions into state machines using **Continuation Passing Style (CPS)** — no new thread is created. The suspension point is just a scheduled callback managed by the coroutine runtime.

```kotlin
// How you write it
suspend fun fetchAndSave(id: String): User {
    val user = api.getUser(id)    // suspend point ①
    db.save(user)                 // suspend point ②
    return user
}

// What the compiler generates (simplified)
fun fetchAndSave(id: String, continuation: Continuation<User>): Any {
    // State machine with labels for each suspension point
}
```

---

### CoroutineScope and Structured Concurrency

Every coroutine runs inside a **CoroutineScope**. The scope establishes the **lifetime** of coroutines — when the scope is cancelled, all coroutines inside it are cancelled automatically. This is **structured concurrency**: child coroutines cannot outlive their parent.

```
CoroutineScope (viewModelScope)
├── launch { fetchUser() }           ← child coroutine
│   └── withContext(IO) { db.query() } ← nested
└── launch { fetchFeed() }            ← sibling coroutine
```

When `viewModelScope` is cancelled (ViewModel cleared), every child coroutine is cancelled immediately — no leaks, no stale callbacks.

**Android-provided scopes:**

```kotlin
// ViewModel — cancelled when the ViewModel is cleared
viewModelScope.launch { /* ... */ }

// Fragment/Activity — cancelled when the lifecycle reaches DESTROYED
lifecycleScope.launch { /* ... */ }

// Fragment — cancelled when the view is DESTROYED (use for UI-bound work)
viewLifecycleOwner.lifecycleScope.launch { /* ... */ }

// Repeat while lifecycle is in a given state (e.g., started)
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> render(state) }
    }
}
```

---

### CoroutineContext

A `CoroutineContext` is an indexed set of elements that configure a coroutine's behaviour. The most important elements are:

| Element | Interface | Determines |
|---------|-----------|-----------|
| `Job` | `Job` / `SupervisorJob` | Lifecycle and cancellation |
| `CoroutineDispatcher` | `CoroutineDispatcher` | Which thread(s) the coroutine runs on |
| `CoroutineName` | `CoroutineName` | Debug name (shows in logs/debugger) |
| `CoroutineExceptionHandler` | `CoroutineExceptionHandler` | Handles uncaught exceptions |

Contexts combine with `+`:

```kotlin
val context = Dispatchers.IO + SupervisorJob() + CoroutineName("UserLoader") +
    CoroutineExceptionHandler { _, throwable -> log.e("Uncaught", throwable) }

val scope = CoroutineScope(context)
```

Child coroutines **inherit** context elements from their parent, but can override individual ones:

```kotlin
viewModelScope.launch {                        // inherits Dispatchers.Main
    withContext(Dispatchers.IO) { db.query() } // overrides dispatcher locally
    // back on Main here
}
```

---

### Dispatchers

A **Dispatcher** determines which thread or thread pool a coroutine runs on. Always be explicit — never assume what thread you're on.

#### `Dispatchers.Main`

Runs on the Android **UI thread**. Use for updating UI state, collecting Flows in composables, or any work that must run on the main thread.

```kotlin
// viewModelScope uses Main by default
viewModelScope.launch {    // running on Main
    _uiState.value = UiState.Loading
    val result = withContext(Dispatchers.IO) { repository.getUser(id) }
    _uiState.value = UiState.Success(result)  // back on Main
}
```

#### `Dispatchers.IO`

Uses a shared thread pool optimised for **blocking I/O operations**: network calls, database reads/writes, file access. The pool can grow up to 64 threads (or the number of CPU cores, whichever is larger).

```kotlin
suspend fun loadUser(id: String): User = withContext(Dispatchers.IO) {
    // Safe to block here — IO pool thread, not Main
    apiService.getUser(id)
}

suspend fun readFile(path: String): String = withContext(Dispatchers.IO) {
    File(path).readText()
}
```

> **Note:** Room and Retrofit automatically handle their own dispatcher switching internally. You do not need to wrap them in `withContext(Dispatchers.IO)` when using `suspend` functions — but doing so is harmless and makes intent explicit.

#### `Dispatchers.Default`

Uses a thread pool sized to the **number of CPU cores**. Use for **CPU-intensive computations**: sorting large lists, parsing JSON, image processing, complex algorithms.

```kotlin
suspend fun sortUsers(users: List<User>): List<User> = withContext(Dispatchers.Default) {
    users.sortedWith(compareBy({ it.name }, { it.email }))
}

suspend fun compressImage(bitmap: Bitmap): Bitmap = withContext(Dispatchers.Default) {
    // CPU-bound work
    bitmap.scale(0.5f).applyFilter(BlurFilter(radius = 8f))
}
```

#### `Dispatchers.Unconfined`

Runs in the calling thread until the first suspension point, then resumes in whichever thread the suspending function uses. **Avoid in production** — the thread it runs on is unpredictable, making it unsuitable for UI work or thread-sensitive operations. It is occasionally useful in unit tests.

#### `Dispatchers.Main.immediate`

Same as `Dispatchers.Main` but, if you are already on the main thread, the coroutine executes immediately without re-dispatching (no frame delay). Useful when you need to update the UI synchronously from main-thread code.

```kotlin
fun onButtonClick() {
    viewModelScope.launch(Dispatchers.Main.immediate) {
        _uiState.value = UiState.Loading   // immediate, no delay
    }
}
```

#### Summary Table

| Dispatcher | Thread pool | Use for |
|------------|------------|---------|
| `Dispatchers.Main` | UI thread (1 thread) | UI updates, state emission |
| `Dispatchers.Main.immediate` | UI thread (immediate) | Synchronous UI updates from main thread |
| `Dispatchers.IO` | Up to 64 threads | Network, DB, file I/O |
| `Dispatchers.Default` | CPU-core threads | Heavy computation, sorting, parsing |
| `Dispatchers.Unconfined` | Caller's thread | Tests only — avoid in production |

---

### Coroutine Builders

Builders are extension functions on `CoroutineScope` that launch new coroutines.

#### `launch` — Fire and Forget

Returns a `Job`. Use when you don't need a return value.

```kotlin
viewModelScope.launch {
    repository.syncData()   // runs concurrently, result not used
}

// Store the Job to cancel it manually
val job = viewModelScope.launch {
    while (isActive) {
        delay(5_000)
        repository.refresh()
    }
}

// Later:
job.cancel()
```

#### `async` / `await` — Parallel Decomposition

Returns a `Deferred<T>`. Use when you need a result and want to run multiple operations **concurrently**.

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    // Both launch in parallel
    val usersDeferred = async(Dispatchers.IO) { userRepository.getUsers() }
    val statsDeferred = async(Dispatchers.IO) { statsRepository.getStats() }

    // Suspend until both complete
    Dashboard(
        users = usersDeferred.await(),
        stats = statsDeferred.await()
    )
}
// Total time ≈ max(users time, stats time) — not sum
```

#### `withContext` — Switch Dispatcher

Suspends the current coroutine, runs the block on a different dispatcher, then resumes on the original dispatcher. Does **not** create a new coroutine — it reuses the current one.

```kotlin
suspend fun getUserWithMapping(id: String): UserProfile {
    val dto = withContext(Dispatchers.IO) { apiService.getUser(id) }  // IO
    val profile = withContext(Dispatchers.Default) { dto.toProfile() } // CPU
    return profile
    // implicit: we're back on whatever dispatcher called this function
}
```

#### `runBlocking` — Bridge to Blocking World

**Blocks** the calling thread until the coroutine completes. **Never use in production Android code** — it will freeze the main thread. It exists for unit tests and `main()` functions.

```kotlin
// ✓ OK in tests
@Test
fun `user is loaded correctly`() = runBlocking {
    val user = repository.getUser("1")
    assertThat(user.name).isEqualTo("Alice")
}

// ✗ Never do this in an Activity, Fragment, or ViewModel
runBlocking { repository.getUser("1") }  // freezes UI thread!
```

---

### Job and SupervisorJob

A `Job` represents the lifecycle of a coroutine and defines its parent-child relationships. When a `Job` is cancelled, all its children are cancelled.

```kotlin
val job = viewModelScope.launch {
    launch { task1() }  // child 1
    launch { task2() }  // child 2
}

job.cancel()   // cancels both task1 and task2
job.join()     // suspends until job and all children complete
```

**`Job` vs `SupervisorJob`:**

With a regular `Job`, if any child fails, the failure propagates to the parent and cancels all siblings:

```kotlin
// Regular Job — one failure cancels all
coroutineScope {
    launch { task1() }           // if task2 throws, task1 is cancelled
    launch { task2() /* fails */ }
}
```

With `SupervisorJob`, a child's failure is isolated — siblings continue running:

```kotlin
// SupervisorJob — failures are isolated
supervisorScope {
    launch { task1() }           // continues even if task2 throws
    launch { task2() /* fails */ }
}

// viewModelScope and lifecycleScope use SupervisorJob internally
```

---

### Coroutine Cancellation

Cancellation in coroutines is **cooperative** — a coroutine must check for cancellation at suspension points. Built-in suspension functions (`delay`, `withContext`, Room queries, Retrofit calls) check for cancellation automatically. Long-running CPU-bound loops must check manually.

```kotlin
// Suspension points check automatically
launch {
    while (true) {
        delay(1_000)     // checks isActive; throws CancellationException if cancelled
        processData()
    }
}

// CPU loop must check manually
launch(Dispatchers.Default) {
    for (item in massiveList) {
        ensureActive()   // throws CancellationException if cancelled
        processItem(item)
    }
}

// isActive — non-throwing alternative
launch(Dispatchers.Default) {
    for (item in massiveList) {
        if (!isActive) return@launch
        processItem(item)
    }
}
```

**`CancellationException` must never be caught silently:**

```kotlin
// ✗ Bad — hides cancellation, coroutine never actually stops
try {
    someWork()
} catch (e: Exception) {
    log.e("error", e)  // swallows CancellationException!
}

// ✓ Good — re-throw CancellationException
try {
    someWork()
} catch (e: CancellationException) {
    throw e            // let structured concurrency handle it
} catch (e: Exception) {
    log.e("error", e)
}

// ✓ Or handle only non-cancellation failures
try {
    someWork()
} catch (e: IOException) {
    handleNetworkError(e)
}
```

---

### Error Handling

#### `try/catch` inside coroutines

```kotlin
viewModelScope.launch {
    try {
        val user = repository.getUser(id)
        _uiState.value = UiState.Success(user)
    } catch (e: IOException) {
        _uiState.value = UiState.Error("Network unavailable")
    } catch (e: HttpException) {
        _uiState.value = UiState.Error("Server error: ${e.code()}")
    }
}
```

#### `CoroutineExceptionHandler`

Catches **uncaught exceptions** from `launch` blocks (not from `async` — those surface via `await`). Install it on the scope or the individual `launch` call:

```kotlin
val handler = CoroutineExceptionHandler { context, throwable ->
    log.e("Coroutine", "Unhandled exception in ${context[CoroutineName]}", throwable)
    _events.tryEmit(AppEvent.ShowCrashBanner)
}

viewModelScope.launch(handler) {
    riskyOperation()   // exception here is caught by handler, not propagated
}
```

#### `supervisorScope` for parallel tasks with independent failure

```kotlin
suspend fun loadDashboard(): Dashboard = supervisorScope {
    val usersDeferred  = async { userRepository.getUsers() }
    val statsDeferred  = async { statsRepository.getStats() }   // may fail
    val feedDeferred   = async { feedRepository.getFeed() }

    Dashboard(
        users  = runCatching { usersDeferred.await() }.getOrDefault(emptyList()),
        stats  = runCatching { statsDeferred.await() }.getOrDefault(Stats.empty()),
        feed   = runCatching { feedDeferred.await() }.getOrDefault(emptyList())
    )
    // If stats fails, users and feed still complete normally
}
```

---

### Flow — Reactive Streams with Coroutines

`Flow<T>` is the coroutine-native reactive streams API. A Flow is a **cold** stream — it only starts emitting when collected, and each collector gets its own independent execution.

#### Building a Flow

```kotlin
// flow builder — emit values lazily
fun observePrices(): Flow<Double> = flow {
    while (true) {
        val price = api.getPrice()  // suspend call
        emit(price)
        delay(5_000)
    }
}

// flowOf — emit a fixed set of values
val singleFlow: Flow<Int> = flowOf(1, 2, 3)

// asFlow — convert a collection
val listFlow: Flow<String> = listOf("a", "b", "c").asFlow()

// channelFlow — emit from multiple coroutines
fun observeMultipleSources(): Flow<Event> = channelFlow {
    launch { api.observeEvents().collect { send(it) } }
    launch { db.observeLocalEvents().collect { send(it) } }
}
```

#### Operators

```kotlin
observePrices()
    .filter { it > 0.0 }                            // discard negatives
    .map { price -> "$${String.format("%.2f", price)}" } // transform
    .distinctUntilChanged()                          // skip duplicate emissions
    .debounce(300)                                   // wait 300ms before emitting
    .onEach { log.d("Price updated: $it") }          // side effect
    .catch { e -> emit("Error: ${e.message}") }      // handle errors
    .collect { displayPrice(it) }                    // terminal — starts the flow
```

#### `flowOn` — Switch the Upstream Dispatcher

`flowOn` changes the dispatcher of everything **above** it in the chain, while everything below runs on the original dispatcher:

```kotlin
fun observeUsers(): Flow<List<User>> = userDao.observeAll()   // Room produces on IO
    .map { entities -> entities.map { it.toDomain() } }       // mapping on IO
    .flowOn(Dispatchers.IO)                                    // ← IO applies to lines above
    .map { users -> users.sortedBy { it.name } }              // this runs on Default (caller)
    .flowOn(Dispatchers.Default)
// Collector receives sorted users on its own dispatcher (usually Main)
```

#### `StateFlow` — Hot, Always Has a Value

`StateFlow` is a hot observable state holder. It always has a current value, replays the latest value to new collectors, and never completes.

```kotlin
// In ViewModel
private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

// Update from any coroutine
viewModelScope.launch { _uiState.value = UserUiState.Success(user) }

// Collect in Compose (lifecycle-aware)
val uiState by viewModel.uiState.collectAsStateWithLifecycle()

// Convert a cold Flow into StateFlow
val sortedUsers: StateFlow<List<User>> = repository.observeUsers()
    .map { it.sortedBy(User::name) }
    .stateIn(
        scope         = viewModelScope,
        started       = SharingStarted.WhileSubscribed(5_000), // keep active 5s after last collector
        initialValue  = emptyList()
    )
```

#### `SharedFlow` — Hot, No Value, Multiple Subscribers

`SharedFlow` is a hot stream with no current-value concept. It broadcasts events to all active collectors. Used for one-shot events (navigation, snackbars) where caching the last value would be wrong.

```kotlin
private val _events = MutableSharedFlow<UiEvent>(extraBufferCapacity = 1)
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

// Emit from any coroutine
viewModelScope.launch { _events.emit(UiEvent.ShowSnackbar("Saved!")) }
// or non-suspending:
_events.tryEmit(UiEvent.ShowSnackbar("Saved!"))

// Collect in Compose
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
            is UiEvent.Navigate     -> navController.navigate(event.route)
        }
    }
}
```

#### Cold vs Hot Flows at a Glance

| | `Flow` (cold) | `StateFlow` (hot) | `SharedFlow` (hot) |
|---|---|---|---|
| Starts on | First collect | Creation | Creation |
| Replays | Nothing | Latest value | Configurable (default 0) |
| Completes | Yes | Never | Never |
| Multiple collectors | Independent executions | Share the same value | Share the same events |
| Use for | Data pipelines, DB streams | UI state | One-shot events |

---

### Practical Patterns

#### Pattern 1: Repository emitting data from Room

```kotlin
// Repository — Room runs on IO, we map on Default, caller receives on its dispatcher
fun observeUsers(): Flow<List<User>> = localDataSource.observeAllUsers()
    .map { entities -> entities.map { it.toDomain() } }
    .flowOn(Dispatchers.Default)
```

#### Pattern 2: ViewModel converting a cold Flow to StateFlow

```kotlin
val users: StateFlow<List<User>> = userRepository.observeUsers()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

#### Pattern 3: Parallel network calls with independent error handling

```kotlin
suspend fun loadScreen(): ScreenData = supervisorScope {
    val profileDeferred = async(Dispatchers.IO) { profileRepository.get() }
    val feedDeferred    = async(Dispatchers.IO) { feedRepository.get() }

    ScreenData(
        profile = profileDeferred.await(),
        feed    = runCatching { feedDeferred.await() }.getOrDefault(emptyList())
    )
}
```

#### Pattern 4: Debounced search with `flatMapLatest`

```kotlin
val results: StateFlow<List<User>> = searchQuery
    .debounce(300)
    .filter { it.length >= 2 }
    .distinctUntilChanged()
    .flatMapLatest { query ->               // cancels previous search on new input
        userRepository.search(query)
            .catch { emit(emptyList()) }
    }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

#### Pattern 5: Timeout and retry

```kotlin
suspend fun fetchWithRetry(id: String): User = withTimeout(10_000) {
    retry(times = 3, delayMs = 1_000) {
        repository.getUser(id).getOrThrow()
    }
}

suspend fun <T> retry(times: Int, delayMs: Long, block: suspend () -> T): T {
    repeat(times - 1) { attempt ->
        try { return block() } catch (e: IOException) { delay(delayMs * (attempt + 1)) }
    }
    return block()   // last attempt — let the exception propagate
}
```

---

## 6. Data Sources

### What are Data Sources?

Data sources are the lowest layer in Clean Architecture. Each is responsible for accessing exactly one origin of data — a remote API, a local database, shared preferences, an in-memory cache, or the file system. They are the only layer that knows how data is stored or fetched.

### Why Separate Data Sources?

Without separation, the repository contains both network calls and database queries mixed together, making it impossible to test either in isolation. Splitting them means you can fake a remote data source in tests without touching the database, and vice versa.

### When to Create a Data Source?

Create one data source class per data origin. One for the API, one for Room, one for preferences. Never let a data source access two origins — that belongs in the repository.

### Types of Data Sources

| Type | Responsibility | Technology |
|------|---------------|------------|
| Remote | Fetch data from a server | Retrofit + OkHttp |
| Local | Persist structured data on-device | Room (SQLite) |
| Preferences | Store simple key-value settings | SharedPreferences / DataStore |
| In-Memory Cache | Hold recently-accessed data in RAM | `LinkedHashMap`, `LruCache` |
| File | Read/write files to disk | `File`, `ContentResolver` |

### Remote Data Source

```kotlin
interface UserRemoteDataSource {
    suspend fun fetchUser(id: String): UserDto
    suspend fun fetchAllUsers(page: Int, pageSize: Int): List<UserDto>
    suspend fun createUser(dto: CreateUserDto): UserDto
    suspend fun updateUser(id: String, dto: UpdateUserDto): UserDto
    suspend fun deleteUser(id: String)
}

class UserRemoteDataSourceImpl @Inject constructor(
    private val apiService: UserApiService
) : UserRemoteDataSource {
    override suspend fun fetchUser(id: String)           = apiService.getUser(id)
    override suspend fun fetchAllUsers(page: Int, pageSize: Int) = apiService.getAllUsers(page, pageSize)
    override suspend fun createUser(dto: CreateUserDto)  = apiService.createUser(dto)
    override suspend fun updateUser(id: String, dto: UpdateUserDto) = apiService.updateUser(id, dto)
    override suspend fun deleteUser(id: String)          { apiService.deleteUser(id) }
}
```

### Local Data Source

```kotlin
interface UserLocalDataSource {
    suspend fun getUser(id: String): UserEntity?
    suspend fun getAllUsers(): List<UserEntity>
    suspend fun saveUser(user: UserEntity)
    suspend fun saveUsers(users: List<UserEntity>)
    suspend fun deleteUser(id: String)
    suspend fun deleteAll()
    fun observeAllUsers(): Flow<List<UserEntity>>
    fun observeUser(id: String): Flow<UserEntity?>
}

class UserLocalDataSourceImpl @Inject constructor(
    private val userDao: UserDao
) : UserLocalDataSource {
    override suspend fun getUser(id: String)              = userDao.findById(id)
    override suspend fun getAllUsers()                     = userDao.getAll()
    override suspend fun saveUser(user: UserEntity)       = userDao.upsert(user)
    override suspend fun saveUsers(users: List<UserEntity>) = userDao.upsertAll(users)
    override suspend fun deleteUser(id: String)           = userDao.deleteById(id)
    override suspend fun deleteAll()                      = userDao.deleteAll()
    override fun observeAllUsers()                        = userDao.observeAll()
    override fun observeUser(id: String)                  = userDao.observeById(id)
}
```

### In-Memory Cache

```kotlin
class UserInMemoryCache @Inject constructor() {
    private val cache = LinkedHashMap<String, Pair<User, Long>>()
    private val ttlMs = 5 * 60 * 1_000L

    fun get(id: String): User? {
        val (user, timestamp) = cache[id] ?: return null
        if (System.currentTimeMillis() - timestamp > ttlMs) { cache.remove(id); return null }
        return user
    }

    fun put(user: User) {
        if (cache.size >= 100) cache.remove(cache.keys.first())
        cache[user.id] = user to System.currentTimeMillis()
    }

    fun invalidate(id: String) = cache.remove(id)
    fun clear() = cache.clear()
}
```

### DTOs vs Entities vs Domain Models

```kotlin
// DTO — shaped to match the API JSON. Uses serialization annotations. data/remote/dto/
@Serializable
data class UserDto(
    @SerialName("user_id")    val userId: String,
    @SerialName("full_name")  val fullName: String,
    @SerialName("email_address") val email: String,
    @SerialName("avatar_url") val avatarUrl: String? = null
)

// Entity — shaped to match the Room table. Uses Room annotations. data/local/entity/
@Entity(tableName = "users", indices = [Index(value = ["email"], unique = true)])
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name")  val name: String,
    val email: String,
    @ColumnInfo(name = "avatar_url") val avatarUrl: String?,
    @ColumnInfo(name = "cached_at")  val cachedAt: Long = System.currentTimeMillis()
)

// Domain model — pure Kotlin, zero annotations. domain/model/
data class User(val id: String, val name: String, val email: String, val avatarUrl: String?)

// Mappers — data/mapper/
fun UserDto.toDomain()    = User(userId, fullName, email, avatarUrl)
fun UserDto.toEntity()    = UserEntity(userId, fullName, email, avatarUrl)
fun UserEntity.toDomain() = User(id, name, email, avatarUrl)
fun User.toEntity()       = UserEntity(id, name, email, avatarUrl)
```

---

## 7. Repositories

### What is a Repository?

A repository is the single source of truth for a given data domain. It is the only component that knows where data comes from (network, database, cache) and decides which source to use at any given moment.

### Why Use a Repository?

Without a repository, every ViewModel or Use Case would need to know whether to call the network or the database, handle caching logic, and map between DTOs and domain models. This logic would be duplicated across the app. The repository centralises all of that in one place.

### When to Create a Repository?

Create one per major domain entity or aggregate root (e.g., `UserRepository`, `ProductRepository`, `OrderRepository`). Never span unrelated domains in one repository — that violates SRP.

### Repository Interface (Domain Layer)

```kotlin
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
    suspend fun deleteUser(id: String): Result<Unit>
    suspend fun refreshUsers(): Result<Unit>
    fun observeUsers(): Flow<List<User>>
    fun observeUser(id: String): Flow<Result<User>>
}
```

### Repository Implementation

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val inMemoryCache: UserInMemoryCache,
    private val networkMonitor: NetworkMonitor
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        inMemoryCache.get(id)?.let { return Result.success(it) }

        val cached = localDataSource.getUser(id)
        if (cached != null) {
            inMemoryCache.put(cached.toDomain())
            return Result.success(cached.toDomain())
        }

        val dto = remoteDataSource.fetchUser(id)
        val user = dto.toDomain()
        localDataSource.saveUser(dto.toEntity())
        inMemoryCache.put(user)
        user
    }

    override suspend fun saveUser(user: User): Result<Unit> = runCatching {
        localDataSource.saveUser(user.toEntity())
        inMemoryCache.put(user)
        if (networkMonitor.isConnected) remoteDataSource.updateUser(user.id, user.toUpdateDto())
    }

    override suspend fun refreshUsers(): Result<Unit> = runCatching {
        val dtos = remoteDataSource.fetchAllUsers(page = 1, pageSize = 100)
        localDataSource.saveUsers(dtos.map { it.toEntity() })
        inMemoryCache.clear()
    }

    override fun observeUsers(): Flow<List<User>> =
        localDataSource.observeAllUsers().map { it.map(UserEntity::toDomain) }

    override fun observeUser(id: String): Flow<Result<User>> =
        localDataSource.observeUser(id).map { entity ->
            entity?.let { Result.success(it.toDomain()) }
                ?: Result.failure(NoSuchElementException("User $id not found"))
        }
}
```

### Cache Strategies

```kotlin
// Network-first: best for frequently-changing data (prices, live scores)
suspend fun getUserNetworkFirst(id: String): Result<User> = runCatching {
    try {
        val dto = remoteDataSource.fetchUser(id)
        localDataSource.saveUser(dto.toEntity())
        dto.toDomain()
    } catch (e: IOException) {
        localDataSource.getUser(id)?.toDomain()
            ?: throw NoSuchElementException("No cached data for user $id")
    }
}

// Cache-first: best for stable data (user profiles, product catalogs)
fun getUserCacheFirst(id: String): Flow<Result<User>> = flow {
    val cached = localDataSource.getUser(id)
    if (cached != null) emit(Result.success(cached.toDomain()))
    try {
        val fresh = remoteDataSource.fetchUser(id)
        localDataSource.saveUser(fresh.toEntity())
        emit(Result.success(fresh.toDomain()))
    } catch (e: Exception) {
        if (cached == null) emit(Result.failure(e))
    }
}
```

---

## 8. Use Cases

### What is a Use Case?

A use case (Interactor) encapsulates exactly one business operation. It orchestrates data from one or more repositories to fulfil a specific user intention: log in, place an order, follow another user.

### Why Use Cases?

Without use cases, business logic leaks into ViewModels. A ViewModel that calls three repositories, applies validation, and formats data is fragile, hard to test, and impossible to reuse. Use cases extract this logic into standalone, independently testable units.

### When to Create a Use Case?

Create one whenever a discrete action requires logic beyond a single repository call. If the ViewModel just calls `repository.getUser(id)`, a use case is overkill. If it needs to validate, coordinate multiple repositories, or apply a business rule, extract a use case.

### Base Abstractions

```kotlin
abstract class UseCase<in Params, out Result> {
    suspend operator fun invoke(params: Params): kotlin.Result<Result> =
        runCatching { execute(params) }
    protected abstract suspend fun execute(params: Params): Result
}

abstract class FlowUseCase<in Params, out Result> {
    operator fun invoke(params: Params): Flow<kotlin.Result<Result>> =
        execute(params).map { kotlin.Result.success(it) }.catch { emit(kotlin.Result.failure(it)) }
    protected abstract fun execute(params: Params): Flow<Result>
}

abstract class NoParamsUseCase<out Result> {
    suspend operator fun invoke(): kotlin.Result<Result> = runCatching { execute() }
    protected abstract suspend fun execute(): Result
}
```

### Concrete Use Cases

```kotlin
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) : UseCase<GetUserUseCase.Params, User>() {
    data class Params(val userId: String)
    override suspend fun execute(params: Params): User =
        userRepository.getUser(params.userId).getOrThrow()
}

class RegisterUserUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val authRepository: AuthRepository,
    private val emailValidator: EmailValidator
) : UseCase<RegisterUserUseCase.Params, User>() {

    data class Params(val name: String, val email: String, val password: String)

    override suspend fun execute(params: Params): User {
        require(params.name.isNotBlank())                  { "Name is required" }
        require(emailValidator.isValid(params.email))      { "Invalid email" }
        require(params.password.length >= 8)               { "Password must be at least 8 characters" }
        require(params.password.any { it.isUpperCase() })  { "Password must contain an uppercase letter" }
        check(userRepository.findByEmail(params.email) == null) { "Email already registered" }
        return authRepository.register(params.name, params.email, params.password)
    }
}

class ObserveUsersUseCase @Inject constructor(
    private val userRepository: UserRepository
) : FlowUseCase<Unit, List<User>>() {
    override fun execute(params: Unit): Flow<List<User>> = userRepository.observeUsers()
}
```

### Composing Use Cases

```kotlin
class RefreshAndGetUserUseCase @Inject constructor(
    private val refreshUsersUseCase: RefreshUsersUseCase,
    private val getUserUseCase: GetUserUseCase
) : UseCase<RefreshAndGetUserUseCase.Params, User>() {
    data class Params(val userId: String)
    override suspend fun execute(params: Params): User {
        refreshUsersUseCase()
        return getUserUseCase(GetUserUseCase.Params(params.userId)).getOrThrow()
    }
}
```

### Unit Testing Use Cases

```kotlin
class RegisterUserUseCaseTest {
    private val fakeUserRepo = FakeUserRepository()
    private val fakeAuthRepo = FakeAuthRepository()
    private val useCase = RegisterUserUseCase(fakeUserRepo, fakeAuthRepo, RealEmailValidator())

    @Test fun `succeeds with valid input`() = runTest {
        assertThat(useCase(RegisterUserUseCase.Params("Alice", "alice@example.com", "Password1")).isSuccess).isTrue()
    }

    @Test fun `fails when password is too short`() = runTest {
        val result = useCase(RegisterUserUseCase.Params("Alice", "alice@example.com", "Pass1"))
        assertThat(result.exceptionOrNull()?.message).contains("8 characters")
    }

    @Test fun `fails when email already registered`() = runTest {
        fakeUserRepo.existingEmail = "alice@example.com"
        val result = useCase(RegisterUserUseCase.Params("Alice", "alice@example.com", "Password1"))
        assertThat(result.exceptionOrNull()?.message).contains("already registered")
    }
}
```

---

## 9. ViewModel

### What is a ViewModel?

ViewModel is an Android Jetpack component that stores and manages UI-related data in a lifecycle-aware way. It survives configuration changes (screen rotations, theme changes) without losing state, and clears automatically when the associated screen is permanently destroyed.

### Why Use ViewModel?

Without ViewModel, data fetched by an Activity is destroyed on every rotation, forcing redundant network calls. ViewModel solves this by living in a scope that outlasts the configuration change. Combined with `StateFlow`, it makes UI state observable, predictable, and fully testable on the JVM.

### When to Use ViewModel?

Every screen needs a ViewModel. For cross-screen shared state (shopping cart, session), use a ViewModel scoped to the Activity, or a shared repository injected into each ViewModel.

### UI State Design

```kotlin
// Sealed class for clear loading/success/error lifecycle
sealed class UserDetailUiState {
    object Idle    : UserDetailUiState()
    object Loading : UserDetailUiState()
    data class Success(val user: User) : UserDetailUiState()
    data class Error(val message: String, val isRetryable: Boolean) : UserDetailUiState()
}

// Data class for complex screens with multiple independent sections
data class DashboardUiState(
    val profile:      AsyncData<UserProfile>    = AsyncData.Loading,
    val feed:         AsyncData<List<Post>>     = AsyncData.Loading,
    val isRefreshing: Boolean                   = false
)

sealed class AsyncData<out T> {
    object Loading                        : AsyncData<Nothing>()
    data class Success<T>(val data: T)    : AsyncData<T>()
    data class Error(val message: String) : AsyncData<Nothing>()
}
```

### Full ViewModel with Events

```kotlin
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getUserUseCase: GetUserUseCase,
    private val updateUserUseCase: UpdateUserUseCase,
    private val deleteUserUseCase: DeleteUserUseCase
) : ViewModel() {

    private val userId: String = checkNotNull(savedStateHandle["userId"])

    private val _uiState = MutableStateFlow<UserDetailUiState>(UserDetailUiState.Idle)
    val uiState: StateFlow<UserDetailUiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<UserDetailEvent>(extraBufferCapacity = 1)
    val events: SharedFlow<UserDetailEvent> = _events.asSharedFlow()

    init { loadUser() }

    fun loadUser() {
        viewModelScope.launch {
            _uiState.value = UserDetailUiState.Loading
            getUserUseCase(GetUserUseCase.Params(userId))
                .onSuccess { _uiState.value = UserDetailUiState.Success(it) }
                .onFailure { e ->
                    _uiState.value = UserDetailUiState.Error(
                        message      = e.localizedMessage ?: "Unknown error",
                        isRetryable  = e is IOException
                    )
                }
        }
    }

    fun updateName(newName: String) {
        val user = (_uiState.value as? UserDetailUiState.Success)?.user ?: return
        viewModelScope.launch {
            updateUserUseCase(UpdateUserUseCase.Params(user.copy(name = newName)))
                .onSuccess { _events.tryEmit(UserDetailEvent.ShowSnackbar("Name updated")) }
                .onFailure { _events.tryEmit(UserDetailEvent.ShowSnackbar("Update failed")) }
        }
    }
}

sealed class UserDetailEvent {
    data class ShowSnackbar(val message: String) : UserDetailEvent()
    object NavigateBack : UserDetailEvent()
}
```

### Debouncing User Input

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchUseCase: SearchUsersUseCase
) : ViewModel() {

    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    val results: StateFlow<List<User>> = _query
        .debounce(300)
        .filter { it.length >= 2 }
        .distinctUntilChanged()
        .flatMapLatest { q -> searchUseCase(SearchUsersUseCase.Params(q)).catch { emit(emptyList()) } }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun onQueryChanged(query: String) { _query.value = query }
}
```

### Combining Multiple Flows

```kotlin
val uiState: StateFlow<DashboardUiState> = combine(
    observeUserUseCase(Unit),
    observeNotificationsUseCase(Unit)
) { userResult, notificationsResult ->
    DashboardUiState(
        user          = userResult.getOrNull(),
        notifications = notificationsResult.getOrDefault(emptyList()),
        hasError      = userResult.isFailure
    )
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), DashboardUiState())
```

---

## 10. Jetpack Compose

### What is Jetpack Compose?

Jetpack Compose is Android's modern, fully declarative UI toolkit. Instead of defining views in XML and imperatively mutating them (`textView.text = "Hello"`), you write Composable functions that *describe* what the UI should look like for a given state. When state changes, Compose automatically re-renders only the affected composables.

### Why Use Compose?

- **Less code:** No XML, no `findViewById`, no view binding boilerplate.
- **Reactive by default:** UI automatically updates when state changes.
- **Kotlin-first:** Full language features available directly in UI code.
- **Powerful previews:** Render composables in Android Studio without running the app.
- **Better testing:** Composables are functions that compose and test cleanly.

### When to Use Compose?

For all new Android UI development. Google has recommended Compose as the default UI framework since 2022. For legacy projects, Compose interoperates with XML views through `ComposeView` and `AndroidView`.

### Core Concepts

| Concept | Description |
|---------|-------------|
| `@Composable` | Marks a UI-emitting function |
| State | Data that drives UI; changes trigger recomposition |
| Recomposition | Compose re-runs composables that read changed state |
| State hoisting | Moving state up so children are stateless and reusable |
| `remember` | Memoizes across recompositions (not configuration changes) |
| `rememberSaveable` | Memoizes across recompositions AND configuration changes |
| Side effects | `LaunchedEffect`, `DisposableEffect`, `SideEffect` |

### Composable Functions

```kotlin
@Composable
fun UserCard(
    user: User,
    onUserClick: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .clickable { onUserClick(user.id) },
        shape = RoundedCornerShape(12.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            AsyncImage(
                model = user.avatarUrl,
                contentDescription = "Avatar of ${user.name}",
                contentScale = ContentScale.Crop,
                modifier = Modifier.size(48.dp).clip(CircleShape)
            )
            Column(verticalArrangement = Arrangement.spacedBy(2.dp)) {
                Text(text = user.name,  style = MaterialTheme.typography.titleMedium)
                Text(
                    text  = user.email,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }
    }
}
```

### State Hoisting

```kotlin
// Stateful parent — owns state, talks to ViewModel
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query   by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.results.collectAsStateWithLifecycle()
    SearchContent(query, results, viewModel::onQueryChanged)
}

// Stateless child — pure function of its inputs, easy to test and preview
@Composable
fun SearchContent(
    query: String,
    results: List<User>,
    onQueryChanged: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier) {
        OutlinedTextField(value = query, onValueChange = onQueryChanged, modifier = Modifier.fillMaxWidth())
        LazyColumn { items(results, key = { it.id }) { UserCard(it, onUserClick = {}) } }
    }
}
```

### Side Effects

```kotlin
// LaunchedEffect — runs a suspend block; cancelled and re-launched when the key changes
LaunchedEffect(userId) { viewModel.loadUser(userId) }

// DisposableEffect — setup and teardown tied to composition lifetime
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event ->
        if (event == Lifecycle.Event.ON_RESUME) viewModel.onResume()
    }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}

// SideEffect — runs on every successful recomposition (sync Compose → non-Compose)
SideEffect { systemUiController.setStatusBarColor(MaterialTheme.colorScheme.primary) }
```

### Material 3 Theming

```kotlin
private val LightColorScheme = lightColorScheme(
    primary    = Color(0xFF6650A4),
    onPrimary  = Color(0xFFFFFFFF),
    background = Color(0xFFFFFBFE),
    surface    = Color(0xFFFFFBFE)
)

@Composable
fun AppTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = if (darkTheme) darkColorScheme() else LightColorScheme,
        typography  = AppTypography,
        content     = content
    )
}
```

### Animations

```kotlin
// Animated visibility
AnimatedVisibility(visible = isLoading, enter = fadeIn(), exit = fadeOut()) {
    LinearProgressIndicator(modifier = Modifier.fillMaxWidth())
}

// Animated value
val alpha by animateFloatAsState(targetValue = if (isEnabled) 1f else 0.38f, label = "alpha")

// Crossfade between states
Crossfade(targetState = uiState, label = "screen") { state ->
    when (state) {
        is Loading -> LoadingScreen()
        is Success -> ContentScreen(state.data)
        is Error   -> ErrorScreen(state.message)
    }
}
```

### Navigation

```kotlin
sealed class AppRoute(val path: String) {
    object UserList   : AppRoute("user_list")
    object UserDetail : AppRoute("user_detail/{userId}") {
        fun buildRoute(id: String) = "user_detail/$id"
    }
}

@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController, startDestination = AppRoute.UserList.path) {
        composable(AppRoute.UserList.path) {
            UserListScreen(onNavigateToDetail = { navController.navigate(AppRoute.UserDetail.buildRoute(it)) })
        }
        composable(
            route = AppRoute.UserDetail.path,
            arguments = listOf(navArgument("userId") { type = NavType.StringType })
        ) {
            UserDetailScreen(onNavigateBack = navController::popBackStack)
        }
    }
}
```

### Performance Best Practices

```kotlin
LazyColumn {
    items(users, key = { it.id }) { user -> UserCard(user, onUserClick) }
}

val sortedUsers = remember(users) { users.sortedBy { it.name } }
val showScrollToTop by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }

@Immutable data class UserCardModel(val id: String, val name: String)
```

### Compose Testing

```kotlin
@get:Rule val composeRule = createComposeRule()

@Test fun `UserCard displays name and email`() {
    val user = User("1", "Alice", "alice@example.com", null)
    composeRule.setContent { AppTheme { UserCard(user, onUserClick = {}) } }
    composeRule.onNodeWithText("Alice").assertIsDisplayed()
    composeRule.onNodeWithText("alice@example.com").assertIsDisplayed()
}

@Test fun `clicking UserCard invokes onUserClick`() {
    var clicked = ""
    val user = User("1", "Alice", "alice@example.com", null)
    composeRule.setContent { AppTheme { UserCard(user, onUserClick = { clicked = it }) } }
    composeRule.onNodeWithText("Alice").performClick()
    assertThat(clicked).isEqualTo("1")
}
```

---

## 11. Network Operations with OkHttp

### What is OkHttp?

OkHttp is an efficient, open-source HTTP client for Android and Java developed by Square. It handles connection pooling, transparent GZIP, HTTP/2, response caching, and retry logic out of the box. Retrofit uses OkHttp as its transport layer.

### Why OkHttp?

OkHttp gives you fine-grained control over the HTTP stack through its interceptor chain — a composable pipeline where each interceptor can inspect, modify, retry, or short-circuit requests and responses. Cross-cutting concerns like authentication, caching, and logging are trivially pluggable.

### When to Configure OkHttp?

Always, even when using Retrofit. Configure authentication headers, timeouts, logging, and caching at the OkHttp level so all Retrofit services share the same configuration automatically.

### Gradle Dependencies

```kotlin
dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
}
```

### OkHttp Client Configuration

```kotlin
@Provides @Singleton
fun provideOkHttpClient(
    authInterceptor: AuthInterceptor,
    @ApplicationContext context: Context
): OkHttpClient = OkHttpClient.Builder()
    .cache(Cache(context.cacheDir.resolve("http_cache"), 10L * 1024 * 1024))
    .connectTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .retryOnConnectionFailure(true)
    .addInterceptor(authInterceptor)
    .addInterceptor(CacheInterceptor())
    .addNetworkInterceptor(
        HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                    else HttpLoggingInterceptor.Level.NONE
        }
    )
    .build()
```

### Authentication Interceptor

```kotlin
class AuthInterceptor @Inject constructor(
    private val sessionManager: SessionManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = sessionManager.getToken()
        val request = chain.request().newBuilder()
            .apply { if (token != null) header("Authorization", "Bearer $token") }
            .header("Accept", "application/json")
            .header("X-App-Version", BuildConfig.VERSION_NAME)
            .build()
        val response = chain.proceed(request)
        if (response.code == 401) sessionManager.clearSession()
        return response
    }
}
```

### Automatic Token Refresh (Authenticator)

```kotlin
class TokenRefreshAuthenticator @Inject constructor(
    private val sessionManager: SessionManager,
    private val authApiService: AuthApiService
) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.request.header("X-Retry-After-Refresh") != null) return null
        synchronized(this) {
            val refreshToken = sessionManager.getRefreshToken() ?: return null
            val newToken = runBlocking {
                runCatching { authApiService.refresh(RefreshRequest(refreshToken)) }.getOrNull()
            } ?: run { sessionManager.clearSession(); return null }
            sessionManager.saveSession(newToken)
            return response.request.newBuilder()
                .header("Authorization", "Bearer ${newToken.accessToken}")
                .header("X-Retry-After-Refresh", "true")
                .build()
        }
    }
}
```

### Cache Interceptor

```kotlin
class CacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        return response.newBuilder()
            .removeHeader("Pragma")
            .removeHeader("Cache-Control")
            .header("Cache-Control", CacheControl.Builder().maxAge(5, TimeUnit.MINUTES).build().toString())
            .build()
    }
}
```

### Certificate Pinning

```kotlin
val certificatePinner = CertificatePinner.Builder()
    .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=") // backup
    .build()

OkHttpClient.Builder().certificatePinner(certificatePinner).build()
```

### Retrofit API Service

```kotlin
interface UserApiService {
    @GET("users/{id}") suspend fun getUser(@Path("id") id: String): UserDto
    @GET("users") suspend fun getAllUsers(@Query("page") page: Int, @Query("per_page") perPage: Int): List<UserDto>
    @POST("users") suspend fun createUser(@Body request: CreateUserRequest): UserDto
    @PUT("users/{id}") suspend fun updateUser(@Path("id") id: String, @Body request: UpdateUserRequest): UserDto
    @DELETE("users/{id}") suspend fun deleteUser(@Path("id") id: String): Response<Unit>
    @Multipart @POST("users/{id}/avatar")
    suspend fun uploadAvatar(@Path("id") id: String, @Part avatar: MultipartBody.Part): UserDto
}
```

### Retrofit Instance

```kotlin
@Provides @Singleton
fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/v1/")
    .client(okHttpClient)
    .addConverterFactory(
        Json { ignoreUnknownKeys = true }.asConverterFactory("application/json".toMediaType())
    )
    .build()
```

### Structured Error Handling

```kotlin
sealed class ApiError : Exception() {
    data class HttpError(val code: Int, val body: String?) : ApiError()
    data class NetworkError(override val cause: Throwable) : ApiError()
    data class SerializationError(override val cause: Throwable) : ApiError()
    object UnknownError : ApiError()
}

suspend fun <T> safeApiCall(call: suspend () -> T): Result<T> =
    runCatching { call() }.fold(
        onSuccess = { Result.success(it) },
        onFailure = { e ->
            Result.failure(when (e) {
                is HttpException     -> ApiError.HttpError(e.code(), e.response()?.errorBody()?.string())
                is IOException       -> ApiError.NetworkError(e)
                is JsonDataException -> ApiError.SerializationError(e)
                else                 -> ApiError.UnknownError
            })
        }
    )
```

### Paging 3 Integration

```kotlin
class UserPagingSource @Inject constructor(
    private val apiService: UserApiService
) : PagingSource<Int, UserDto>() {

    override fun getRefreshKey(state: PagingState<Int, UserDto>): Int? =
        state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, UserDto> {
        val page = params.key ?: 1
        return try {
            val items = apiService.getAllUsers(page = page, perPage = params.loadSize)
            LoadResult.Page(items, prevKey = if (page == 1) null else page - 1, nextKey = if (items.isEmpty()) null else page + 1)
        } catch (e: Exception) { LoadResult.Error(e) }
    }
}
```

---

## 12. Dependency Injection with Hilt

### What is Hilt?

Hilt is Google's recommended dependency injection (DI) framework for Android, built on top of Dagger 2. Dependency injection is a pattern where objects receive their dependencies from the outside rather than creating them internally. Hilt automates the wiring of these dependencies using annotations and code generation, eliminating the boilerplate of manual Dagger setup.

### Why Use Hilt?

Without DI, classes create their own dependencies (`val repo = UserRepositoryImpl()`), making them impossible to swap in tests. With Hilt:
- Dependencies are declared via constructor injection — classes are oblivious to how they are created.
- The dependency graph is verified at **compile time** — no runtime crashes from missing bindings.
- Swapping a real implementation for a fake in tests requires only one annotation.
- Android Jetpack components (ViewModel, Worker, Activity) are supported first-class.

### When to Use Hilt?

In every Android app that has more than one class. If a class has dependencies, inject them. Never let classes `new` up their own dependencies; that defeats testability and makes swapping implementations for different environments (debug, staging, production) painful.

### Gradle Setup

```kotlin
// project-level build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android") version "2.51.1" apply false
}

// app-level build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    id("kotlin-kapt")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.51.1")
    kapt("com.google.dagger:hilt-android-compiler:2.51.1")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
    // For WorkManager integration
    implementation("androidx.hilt:hilt-work:1.2.0")
    kapt("androidx.hilt:hilt-compiler:1.2.0")
}

kapt { correctErrorTypes = true }
```

### Application Class

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Hilt initialises the component graph here
    }
}
```

### Hilt Modules — `@Provides` vs `@Binds`

Use `@Provides` for types you don't own (third-party classes like `Retrofit`, `OkHttpClient`, `Room`) and `@Binds` to map an interface to its implementation (always prefer `@Binds` — it generates less code):

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides @Singleton
    fun provideOkHttpClient(authInterceptor: AuthInterceptor): OkHttpClient =
        OkHttpClient.Builder().addInterceptor(authInterceptor).build()

    @Provides @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/v1/")
        .client(client)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    @Provides @Singleton
    fun provideUserApiService(retrofit: Retrofit): UserApiService =
        retrofit.create(UserApiService::class.java)
}

// Use abstract class + @Binds for interface-to-implementation binding
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds @Singleton abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
    @Binds @Singleton abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
}

@Module
@InstallIn(SingletonComponent::class)
abstract class DataSourceModule {
    @Binds @Singleton abstract fun bindUserRemoteDataSource(impl: UserRemoteDataSourceImpl): UserRemoteDataSource
    @Binds @Singleton abstract fun bindUserLocalDataSource(impl: UserLocalDataSourceImpl): UserLocalDataSource
}
```

### Hilt Scopes

Scopes control how long a provided instance lives. Always use the narrowest scope that satisfies your requirements.

| Scope | Component | Lifetime | Use for |
|-------|-----------|----------|---------|
| `@Singleton` | `SingletonComponent` | App lifetime | Network, DB, repositories |
| `@ActivityRetainedScoped` | `ActivityRetainedComponent` | Survives config changes | Shared state across fragments |
| `@ActivityScoped` | `ActivityComponent` | Activity lifetime | Activity-local helpers |
| `@ViewModelScoped` | `ViewModelComponent` | ViewModel lifetime | Use case instances scoped to one VM |
| `@FragmentScoped` | `FragmentComponent` | Fragment lifetime | Fragment-local helpers |

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    @Provides @ViewModelScoped
    fun provideAnalyticsTracker(): AnalyticsTracker = AnalyticsTracker()
}
```

### Injecting into Android Components

```kotlin
// Activity
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    @Inject lateinit var sessionManager: SessionManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { AppTheme { AppNavHost() } }
    }
}

// Fragment
@AndroidEntryPoint
class UserListFragment : Fragment() {
    private val viewModel: UserListViewModel by viewModels()
    @Inject lateinit var analytics: AnalyticsTracker
}

// ViewModel
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val observeUsersUseCase: ObserveUsersUseCase
) : ViewModel()

// Composable
@Composable fun UserListScreen() {
    val viewModel: UserListViewModel = hiltViewModel()
}
```

### Custom Qualifiers

When you need two instances of the same type with different configurations:

```kotlin
@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class AuthenticatedOkHttpClient

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class UnauthenticatedOkHttpClient

@Module @InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton @AuthenticatedOkHttpClient
    fun provideAuthenticatedClient(authInterceptor: AuthInterceptor): OkHttpClient =
        OkHttpClient.Builder().addInterceptor(authInterceptor).build()

    @Provides @Singleton @UnauthenticatedOkHttpClient
    fun provideUnauthenticatedClient(): OkHttpClient = OkHttpClient.Builder().build()
}

// Usage
class SomeDataSource @Inject constructor(
    @AuthenticatedOkHttpClient private val authenticatedClient: OkHttpClient,
    @UnauthenticatedOkHttpClient private val publicClient: OkHttpClient
)
```

### Assisted Injection

For classes that need both injected and runtime parameters:

```kotlin
class UserDetailViewModel @AssistedInject constructor(
    @Assisted val userId: String,
    private val getUserUseCase: GetUserUseCase
) : ViewModel() {

    @AssistedFactory
    interface Factory {
        fun create(userId: String): UserDetailViewModel
    }
}

// In Compose — use hiltViewModel with the assisted factory
@Composable
fun UserDetailScreen(userId: String) {
    val factory = hiltViewModel<UserDetailViewModelFactory>()
    val viewModel = remember(userId) { factory.create(userId) }
}
```

### Hilt Entry Points

For code that cannot use constructor injection (e.g., `ContentProvider`, custom views):

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface UserRepositoryEntryPoint {
    fun userRepository(): UserRepository
}

// Usage
val entryPoint = EntryPointAccessors.fromApplication(
    context.applicationContext,
    UserRepositoryEntryPoint::class.java
)
val userRepository = entryPoint.userRepository()
```

### WorkManager Integration

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val userRepository: UserRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = try {
        userRepository.refreshUsers().getOrThrow()
        Result.success()
    } catch (e: Exception) {
        if (runAttemptCount < 3) Result.retry() else Result.failure()
    }
}
```

### Testing with Hilt

```kotlin
@HiltAndroidTest
class UserRepositoryTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)
    @Inject lateinit var userRepository: UserRepository

    @Before fun setUp() { hiltRule.inject() }

    @Test fun `getUser returns user`() = runTest {
        assertThat(userRepository.getUser("1").isSuccess).isTrue()
    }
}

// Replace production module in tests
@TestInstallIn(components = [SingletonComponent::class], replaces = [NetworkModule::class])
@Module
object FakeNetworkModule {
    @Provides @Singleton
    fun provideUserApiService(): UserApiService = FakeUserApiService()
}
```

---

## 13. Data Storage

### Room

### What is Room?

Room is Android's recommended SQLite abstraction library. It provides compile-time SQL query verification, coroutine and Flow support, and a clean annotation-based API for defining tables, queries, and migrations. Room eliminates the error-prone boilerplate of raw SQLite while giving you full access to all SQLite features.

### Why Use Room?

- **Compile-time safety:** SQL query errors are caught at build time, not at runtime.
- **Flow support:** DAOs can return `Flow<T>`, so the UI automatically updates when the database changes.
- **Migration support:** Structured API for managing schema changes without losing user data.
- **Testing:** Swap `Room.databaseBuilder` for `Room.inMemoryDatabaseBuilder` in tests — zero I/O, instant.

### When to Use Room?

Whenever you need to store structured, relational data locally. Prefer Room over raw SQLite always, and over `SharedPreferences`/`DataStore` when data has relationships, needs querying, or needs to be observed reactively.

### Gradle Setup

```kotlin
dependencies {
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")          // Coroutines + Flow support
    kapt("androidx.room:room-compiler:2.6.1")
    testImplementation("androidx.room:room-testing:2.6.1")
}
```

### Entities

```kotlin
@Entity(
    tableName = "users",
    indices = [
        Index(value = ["email"], unique = true),
        Index(value = ["full_name"])
    ]
)
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name")  val name: String,
    val email: String,
    @ColumnInfo(name = "avatar_url") val avatarUrl: String?,
    @ColumnInfo(name = "is_active")  val isActive: Boolean = true,
    @ColumnInfo(name = "cached_at")  val cachedAt: Long = System.currentTimeMillis()
)

@Entity(
    tableName = "posts",
    foreignKeys = [
        ForeignKey(
            entity        = UserEntity::class,
            parentColumns = ["id"],
            childColumns  = ["user_id"],
            onDelete      = ForeignKey.CASCADE
        )
    ],
    indices = [Index(value = ["user_id"])]
)
data class PostEntity(
    @PrimaryKey val id: String,
    val title: String,
    val body: String,
    @ColumnInfo(name = "user_id")    val userId: String,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)
```

### Relationships

```kotlin
// One-to-many: User has many Posts
data class UserWithPosts(
    @Embedded val user: UserEntity,
    @Relation(parentColumn = "id", entityColumn = "user_id")
    val posts: List<PostEntity>
)

// Many-to-many: Users belong to many Groups
@Entity(tableName = "user_group_cross_ref", primaryKeys = ["user_id", "group_id"])
data class UserGroupCrossRef(
    @ColumnInfo(name = "user_id")  val userId: String,
    @ColumnInfo(name = "group_id") val groupId: String
)

data class UserWithGroups(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn    = "id",
        entityColumn    = "group_id",
        associateBy     = Junction(UserGroupCrossRef::class, parentColumn = "user_id", entityColumn = "group_id")
    )
    val groups: List<GroupEntity>
)
```

### DAO

```kotlin
@Dao
interface UserDao {

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun findById(id: String): UserEntity?

    @Query("SELECT * FROM users WHERE id = :id")
    fun observeById(id: String): Flow<UserEntity?>

    @Query("SELECT * FROM users WHERE is_active = 1 ORDER BY full_name ASC")
    fun observeAll(): Flow<List<UserEntity>>

    @Query("SELECT * FROM users WHERE is_active = 1 ORDER BY full_name ASC")
    suspend fun getAll(): List<UserEntity>

    @Query("""
        SELECT * FROM users
        WHERE full_name LIKE '%' || :query || '%'
           OR email LIKE '%' || :query || '%'
        ORDER BY full_name ASC
    """)
    fun search(query: String): Flow<List<UserEntity>>

    @Upsert suspend fun upsert(user: UserEntity)
    @Upsert suspend fun upsertAll(users: List<UserEntity>)
    @Delete suspend fun delete(user: UserEntity)

    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteById(id: String)

    @Query("DELETE FROM users")
    suspend fun deleteAll()

    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithPosts(userId: String): UserWithPosts?

    @Transaction
    @Query("SELECT * FROM users")
    fun observeAllWithPosts(): Flow<List<UserWithPosts>>

    // Batch operations in a single transaction
    @Transaction
    suspend fun replaceAll(users: List<UserEntity>) {
        deleteAll()
        upsertAll(users)
    }
}
```

### Database

```kotlin
@Database(
    entities = [UserEntity::class, PostEntity::class, UserGroupCrossRef::class],
    version  = 3,
    exportSchema = true                           // Save schema JSON to version-control
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun postDao(): PostDao
}

// Migrations — never use fallbackToDestructiveMigration in production
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN is_active INTEGER NOT NULL DEFAULT 1")
    }
}

class Converters {
    @TypeConverter fun longToDate(value: Long?): Date?  = value?.let { Date(it) }
    @TypeConverter fun dateToLong(date: Date?): Long?   = date?.time
    @TypeConverter fun stringToList(value: String?): List<String> =
        value?.split(",")?.filter { it.isNotEmpty() } ?: emptyList()
    @TypeConverter fun listToString(list: List<String>?): String =
        list?.joinToString(",") ?: ""
}
```

### Hilt Module for Room

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
            .build()

    @Provides fun provideUserDao(db: AppDatabase): UserDao = db.userDao()
    @Provides fun providePostDao(db: AppDatabase): PostDao = db.postDao()
}
```

### Testing Room with In-Memory Database

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    private lateinit var db: AppDatabase
    private lateinit var userDao: UserDao

    @Before fun setUp() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        db = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()             // OK in tests only
            .build()
        userDao = db.userDao()
    }

    @After fun tearDown() { db.close() }

    @Test fun `upsert and findById returns the same entity`() = runTest {
        val entity = UserEntity("1", "Alice", "alice@example.com", null)
        userDao.upsert(entity)
        assertThat(userDao.findById("1")).isEqualTo(entity)
    }

    @Test fun `observeAll emits update after upsert`() = runTest {
        val emissions = mutableListOf<List<UserEntity>>()
        val job = launch { userDao.observeAll().take(2).toList(emissions) }

        userDao.upsert(UserEntity("1", "Alice", "alice@example.com", null))

        job.join()
        assertThat(emissions).hasSize(2)
        assertThat(emissions[1]).hasSize(1)
    }
}
```

---

### SharedPreferences & SecureSharedPreferences

### What are SharedPreferences and EncryptedSharedPreferences?

`SharedPreferences` is Android's built-in key-value store for persisting primitive data (booleans, strings, ints, longs). Data is saved to a plaintext XML file in the app's private directory. `EncryptedSharedPreferences` (from Jetpack Security) wraps `SharedPreferences` with AES-256 encryption for both keys and values, making it suitable for sensitive data such as authentication tokens.

### Why Use Them?

For lightweight, non-relational user preferences (theme choice, language, feature flags), `SharedPreferences`/`DataStore` is far simpler than Room. For secrets that must never be readable in plaintext (tokens, PINs), `EncryptedSharedPreferences` is a secure, battle-tested solution that integrates with Android's `Keystore` system.

### When to Use Which?

| Data | Use |
|------|-----|
| UI preferences (dark mode, language) | `SharedPreferences` or `DataStore` |
| Auth tokens, refresh tokens | `EncryptedSharedPreferences` |
| Structured / relational data | Room |
| Large blobs, files | `File` / `ContentResolver` |

For **new projects**, prefer `DataStore<Preferences>` over `SharedPreferences` — it is coroutine-native, type-safe, and does not suffer from `apply()`/`commit()` ambiguity. Use `EncryptedSharedPreferences` for secrets (DataStore does not yet have an officially stable encrypted variant for general use).

### Gradle Dependencies

```kotlin
dependencies {
    implementation("androidx.security:security-crypto:1.1.0-alpha06")
    implementation("androidx.datastore:datastore-preferences:1.1.1")
}
```

### SharedPreferences (Non-sensitive)

```kotlin
class AppPreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val prefs by lazy {
        context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    }

    var isDarkModeEnabled: Boolean
        get() = prefs.getBoolean(KEY_DARK_MODE, false)
        set(value) = prefs.edit { putBoolean(KEY_DARK_MODE, value) }

    var selectedLanguage: String
        get() = prefs.getString(KEY_LANGUAGE, "en") ?: "en"
        set(value) = prefs.edit { putString(KEY_LANGUAGE, value) }

    var lastSyncTimestamp: Long
        get() = prefs.getLong(KEY_LAST_SYNC, 0L)
        set(value) = prefs.edit { putLong(KEY_LAST_SYNC, value) }

    // Observe a preference reactively using callbackFlow
    fun observeDarkMode(): Flow<Boolean> = callbackFlow {
        val listener = SharedPreferences.OnSharedPreferenceChangeListener { _, key ->
            if (key == KEY_DARK_MODE) trySend(prefs.getBoolean(KEY_DARK_MODE, false))
        }
        prefs.registerOnSharedPreferenceChangeListener(listener)
        send(prefs.getBoolean(KEY_DARK_MODE, false)) // emit current value immediately
        awaitClose { prefs.unregisterOnSharedPreferenceChangeListener(listener) }
    }.distinctUntilChanged()

    fun clearAll() = prefs.edit { clear() }

    companion object {
        private const val KEY_DARK_MODE = "dark_mode"
        private const val KEY_LANGUAGE  = "language"
        private const val KEY_LAST_SYNC = "last_sync"
    }
}
```

### EncryptedSharedPreferences (Sensitive Data)

```kotlin
class SecurePreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey by lazy {
        MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
    }

    private val prefs by lazy {
        EncryptedSharedPreferences.create(
            context,
            "secure_prefs",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    }

    var authToken: String?
        get() = prefs.getString(KEY_AUTH_TOKEN, null)
        set(value) = if (value != null) prefs.edit { putString(KEY_AUTH_TOKEN, value) }
                     else prefs.edit { remove(KEY_AUTH_TOKEN) }

    var refreshToken: String?
        get() = prefs.getString(KEY_REFRESH_TOKEN, null)
        set(value) = if (value != null) prefs.edit { putString(KEY_REFRESH_TOKEN, value) }
                     else prefs.edit { remove(KEY_REFRESH_TOKEN) }

    var userId: String?
        get() = prefs.getString(KEY_USER_ID, null)
        set(value) = if (value != null) prefs.edit { putString(KEY_USER_ID, value) }
                     else prefs.edit { remove(KEY_USER_ID) }

    fun clearSession() = prefs.edit { clear() }

    companion object {
        private const val KEY_AUTH_TOKEN    = "auth_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
        private const val KEY_USER_ID       = "user_id"
    }
}
```

### Session Manager

```kotlin
class SessionManager @Inject constructor(
    private val securePreferences: SecurePreferences,
    private val appPreferences: AppPreferences
) {
    val isLoggedIn: Boolean get() = securePreferences.authToken != null

    fun saveSession(token: AuthToken) {
        securePreferences.authToken    = token.accessToken
        securePreferences.refreshToken = token.refreshToken
        securePreferences.userId       = token.userId
    }

    fun getToken(): String?        = securePreferences.authToken
    fun getRefreshToken(): String? = securePreferences.refreshToken
    fun getUserId(): String?       = securePreferences.userId

    fun clearSession() {
        securePreferences.clearSession()
        appPreferences.clearAll()
    }
}
```

### Modern Alternative: DataStore

For new projects, `DataStore<Preferences>` is the recommended replacement for `SharedPreferences`. It is coroutine-native, never blocks the main thread, and handles `IOException` gracefully via Flow error handling.

```kotlin
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore("settings")

class SettingsRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    companion object {
        val DARK_MODE_KEY  = booleanPreferencesKey("dark_mode")
        val LANGUAGE_KEY   = stringPreferencesKey("language")
        val ONBOARDING_KEY = booleanPreferencesKey("onboarding_complete")
    }

    // Read — returns a Flow that emits on every change
    val isDarkModeEnabled: Flow<Boolean> = context.dataStore.data
        .catch { e -> if (e is IOException) emit(emptyPreferences()) else throw e }
        .map { it[DARK_MODE_KEY] ?: false }

    val selectedLanguage: Flow<String> = context.dataStore.data
        .catch { e -> if (e is IOException) emit(emptyPreferences()) else throw e }
        .map { it[LANGUAGE_KEY] ?: "en" }

    // Write — suspending, never blocks the main thread
    suspend fun setDarkMode(enabled: Boolean) {
        context.dataStore.edit { it[DARK_MODE_KEY] = enabled }
    }

    suspend fun setLanguage(language: String) {
        context.dataStore.edit { it[LANGUAGE_KEY] = language }
    }

    // Atomic multi-field update
    suspend fun completeOnboarding(language: String) {
        context.dataStore.edit { prefs ->
            prefs[ONBOARDING_KEY] = true
            prefs[LANGUAGE_KEY]   = language
        }
    }
}
```

---

## 14. Image Loading with COIL

### What is COIL?

COIL (Coroutine Image Loader) is a modern, Kotlin-first image loading library built with Kotlin coroutines, OkHttp, and Jetpack. It loads, caches, and displays images from URLs, resources, files, and custom data sources. COIL is Compose-native through its `AsyncImage` composable.

### Why Use COIL?

- **Kotlin-first:** Built entirely in Kotlin with coroutines — no callback APIs.
- **Compose-native:** `AsyncImage` integrates directly with the Compose rendering pipeline.
- **OkHttp reuse:** Shares the app's OkHttp client — one connection pool, one disk cache, one interceptor chain.
- **Small footprint:** Adds ~120KB to APK size. Far smaller than Glide or Picasso.
- **Automatic lifecycle awareness:** Cancels image requests when the Composable leaves the composition.

### When to Use COIL?

Any time you need to load remote or local images. Use `AsyncImage` in Compose screens; use the `ImageLoader` directly for preloading or non-UI use cases.

### Gradle Dependencies

```kotlin
dependencies {
    implementation("io.coil-kt:coil-compose:2.6.0")
    implementation("io.coil-kt:coil-gif:2.6.0")    // Animated GIF support
    implementation("io.coil-kt:coil-svg:2.6.0")    // SVG support
    implementation("io.coil-kt:coil-video:2.6.0")  // Video thumbnail support
}
```

### Basic Usage in Compose

```kotlin
// Minimal — URL loaded asynchronously, placeholder shown while loading
AsyncImage(
    model = "https://example.com/avatar.jpg",
    contentDescription = "User avatar",
    modifier = Modifier.size(48.dp).clip(CircleShape)
)

// Full control — placeholder, error fallback, crossfade animation, content scale
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/avatar.jpg")
        .crossfade(true)
        .crossfade(durationMillis = 300)
        .build(),
    contentDescription = "User avatar",
    placeholder  = painterResource(R.drawable.placeholder_avatar),
    error        = painterResource(R.drawable.error_avatar),
    fallback     = painterResource(R.drawable.default_avatar),
    contentScale = ContentScale.Crop,
    modifier = Modifier
        .size(64.dp)
        .clip(CircleShape)
        .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape)
)
```

### Singleton ImageLoader Configuration with Hilt

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object CoilModule {

    @Provides @Singleton
    fun provideImageLoader(
        @ApplicationContext context: Context,
        okHttpClient: OkHttpClient          // Reuse the app's authenticated OkHttp client
    ): ImageLoader = ImageLoader.Builder(context)
        .okHttpClient(okHttpClient)
        .crossfade(true)
        .memoryCache {
            MemoryCache.Builder(context)
                .maxSizePercent(0.25)       // 25% of available RAM
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("coil_image_cache"))
                .maxSizePercent(0.02)       // 2% of available disk
                .build()
        }
        .respectCacheHeaders(false)         // Cache images even if server says no-cache
        .components {
            add(GifDecoder.Factory())
            add(SvgDecoder.Factory(context))
            add(VideoFrameDecoder.Factory())
        }
        .apply { if (BuildConfig.DEBUG) logger(DebugLogger()) }
        .build()
}
```

### Providing the ImageLoader in Compose

```kotlin
// In your root composable or Activity:
val imageLoader: ImageLoader = hiltViewModel<CoilViewModel>().imageLoader

CompositionLocalProvider(LocalImageLoader provides imageLoader) {
    AppNavHost()
}
// AsyncImage will now automatically pick up the singleton ImageLoader
```

### Image Transformations

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .transformations(
            CircleCropTransformation(),
            RoundedCornersTransformation(topLeft = 16f, topRight = 16f, bottomLeft = 0f, bottomRight = 0f),
            BlurTransformation(context, radius = 8f, sampling = 2f),
            GrayscaleTransformation()
        )
        .build(),
    contentDescription = null
)
```

### SubcomposeAsyncImage — Full Custom Loading/Error States

```kotlin
SubcomposeAsyncImage(
    model = imageUrl,
    contentDescription = "Product image"
) {
    when (val state = painter.state) {
        is AsyncImagePainter.State.Loading -> {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(MaterialTheme.colorScheme.surfaceVariant)
                    .shimmer(),                     // shimmer loading effect
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator(
                    modifier = Modifier.size(24.dp),
                    strokeWidth = 2.dp
                )
            }
        }
        is AsyncImagePainter.State.Error -> {
            Column(
                modifier = Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Icon(Icons.Default.BrokenImage, contentDescription = null)
                Text("Failed to load", style = MaterialTheme.typography.bodySmall)
            }
        }
        is AsyncImagePainter.State.Success -> SubcomposeAsyncImageContent()
        is AsyncImagePainter.State.Empty   -> {}
    }
}
```

### Preloading Images

Preload images before they are needed to eliminate visible loading states in fast-scrolling lists:

```kotlin
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val context = LocalContext.current
    val imageLoader = LocalImageLoader.current

    // Preload the next page of avatars when nearing the end of the list
    val listState = rememberLazyListState()
    LaunchedEffect(users) {
        users.takeLast(5).forEach { user ->
            imageLoader.enqueue(
                ImageRequest.Builder(context)
                    .data(user.avatarUrl)
                    .memoryCachePolicy(CachePolicy.ENABLED)
                    .diskCachePolicy(CachePolicy.ENABLED)
                    .build()
            )
        }
    }

    LazyColumn(state = listState) {
        items(users, key = { it.id }) { user ->
            UserCard(user = user, onUserClick = {})
        }
    }
}
```

### Non-Compose Usage (ViewModels, Workers)

```kotlin
class ThumbnailPreloaderViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val imageLoader: ImageLoader
) : ViewModel() {

    fun preloadThumbnails(urls: List<String>) {
        viewModelScope.launch {
            urls.map { url ->
                async {
                    imageLoader.execute(
                        ImageRequest.Builder(context)
                            .data(url)
                            .size(128, 128)                 // Decode to thumbnail size only
                            .allowHardware(false)
                            .build()
                    )
                }
            }.awaitAll()
        }
    }
}
```

### Memory Management

```kotlin
// Limit decode size to avoid OOM on large images
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(highResImageUrl)
        .size(640, 480)                     // Decode to this size, not the original resolution
        .allowHardware(true)                // Use hardware bitmaps for better performance
        .memoryCacheKey("$highResImageUrl-640x480")
        .build(),
    contentDescription = null
)

// Disable memory cache for one-off images (e.g., camera previews)
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(uri)
        .memoryCachePolicy(CachePolicy.DISABLED)
        .diskCachePolicy(CachePolicy.DISABLED)
        .build(),
    contentDescription = null
)
```

---

## License

This document is distributed under the **GNU General Public License v3.0 (GPL-3.0)**.

```
Android Clean Architecture & Best Practices Documentation
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

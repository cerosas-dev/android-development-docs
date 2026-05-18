# Android Clean Architecture & Best Practices

> A comprehensive reference guide for professional Android development using Kotlin, Jetpack libraries, and industry-standard architectural patterns.

## Table of Contents

1. [MVVM Architecture](#1-mvvm-architecture)
2. [SOLID Principles](#2-solid-principles)
3. [Clean Code](#3-clean-code)
4. [Data Sources](#4-data-sources)
5. [Repositories](#5-repositories)
6. [Use Cases](#6-use-cases)
7. [ViewModel](#7-viewmodel)
8. [Jetpack Compose](#8-jetpack-compose)
9. [Network Operations with OkHttp](#9-network-operations-with-okhttp)
10. [Dependency Injection with Hilt](#10-dependency-injection-with-hilt)
11. [Data Storage](#11-data-storage)
    - [Room](#room)
    - [SharedPreferences & SecureSharedPreferences](#sharedpreferences--securesharedpreferences)
12. [Image Loading with COIL](#12-image-loading-with-coil)

---

## 1. MVVM Architecture

Model-View-ViewModel (MVVM) is the recommended architectural pattern for Android development. It separates UI logic from business logic and promotes testability, maintainability, and separation of concerns.

### Layer Overview

```
┌─────────────────────────────────────┐
│               View                  │  ← Activity / Fragment / Composable
│   Observes StateFlow / LiveData     │
└────────────────┬────────────────────┘
                 │ observes
┌────────────────▼────────────────────┐
│             ViewModel               │  ← Holds UI state, calls UseCases
│   No Android framework dependencies │
└────────────────┬────────────────────┘
                 │ calls
┌────────────────▼────────────────────┐
│             Use Case                │  ← Single business operation
└────────────────┬────────────────────┘
                 │ calls
┌────────────────▼────────────────────┐
│            Repository               │  ← Abstracts data origin
└────────────────┬────────────────────┘
                 │ reads/writes
┌────────────────▼────────────────────┐
│           Data Sources              │  ← Remote API / Local DB / Cache
└─────────────────────────────────────┘
```

### Key Principles

- **View** knows nothing about business logic. It only renders state and forwards events.
- **ViewModel** survives configuration changes. It holds and exposes `StateFlow<UiState>`.
- **Repository** is the single source of truth. It decides whether to fetch from network or cache.
- **Use Cases** encapsulate one and only one business rule.

### Complete MVVM Example

**Domain model:**

```kotlin
data class User(
    val id: String,
    val name: String,
    val email: String
)
```

**UI State:**

```kotlin
sealed class UserUiState {
    object Loading : UserUiState()
    data class Success(val user: User) : UserUiState()
    data class Error(val message: String) : UserUiState()
}
```

**ViewModel:**

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
            getUserUseCase(userId)
                .onSuccess { user -> _uiState.value = UserUiState.Success(user) }
                .onFailure { e -> _uiState.value = UserUiState.Error(e.message ?: "Unknown error") }
        }
    }
}
```

**Composable View:**

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
        is UserUiState.Error   -> ErrorMessage(state.message)
    }
}
```

---

## 2. SOLID Principles

SOLID is a set of five design principles that lead to more maintainable, flexible, and understandable code.

### S — Single Responsibility Principle (SRP)

> A class should have only one reason to change.

**Bad:**

```kotlin
class UserManager {
    fun fetchUserFromApi(id: String): User { /* ... */ }
    fun saveUserToDatabase(user: User) { /* ... */ }
    fun sendWelcomeEmail(user: User) { /* ... */ }
    fun formatUserName(user: User): String { /* ... */ }
}
```

**Good:**

```kotlin
class UserRepository(private val api: UserApi, private val db: UserDao) {
    suspend fun getUser(id: String): User = api.fetchUser(id)
    suspend fun saveUser(user: User) = db.insert(user)
}

class EmailService {
    fun sendWelcomeEmail(user: User) { /* ... */ }
}

class UserFormatter {
    fun formatName(user: User): String = "${user.firstName} ${user.lastName}"
}
```

### O — Open/Closed Principle (OCP)

> Software entities should be open for extension, but closed for modification.

```kotlin
// Closed for modification
interface NotificationSender {
    fun send(message: String, recipient: String)
}

// Open for extension
class EmailNotificationSender : NotificationSender {
    override fun send(message: String, recipient: String) { /* send email */ }
}

class PushNotificationSender : NotificationSender {
    override fun send(message: String, recipient: String) { /* send push */ }
}

class SmsNotificationSender : NotificationSender {
    override fun send(message: String, recipient: String) { /* send SMS */ }
}

// Consumer does not change when new sender types are added
class NotificationService(private val sender: NotificationSender) {
    fun notify(message: String, recipient: String) = sender.send(message, recipient)
}
```

### L — Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types without altering program correctness.

```kotlin
abstract class Shape {
    abstract fun area(): Double
}

class Rectangle(val width: Double, val height: Double) : Shape() {
    override fun area() = width * height
}

class Circle(val radius: Double) : Shape() {
    override fun area() = Math.PI * radius * radius
}

// Works correctly with any Shape subtype
fun printArea(shape: Shape) {
    println("Area: ${shape.area()}")
}
```

### I — Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they do not use.

**Bad:**

```kotlin
interface UserRepository {
    suspend fun getUser(id: String): User
    suspend fun saveUser(user: User)
    suspend fun deleteUser(id: String)
    suspend fun generateReport(): Report   // Not every client needs this
}
```

**Good:**

```kotlin
interface UserReader {
    suspend fun getUser(id: String): User
}

interface UserWriter {
    suspend fun saveUser(user: User)
    suspend fun deleteUser(id: String)
}

interface UserReporter {
    suspend fun generateReport(): Report
}

class UserRepositoryImpl : UserReader, UserWriter, UserReporter { /* ... */ }
```

### D — Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

```kotlin
// Abstraction (in domain layer)
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
}

// High-level module depends on abstraction
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(id: String): Result<User> = repository.getUser(id)
}

// Low-level module implements abstraction (in data layer)
class UserRepositoryImpl(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> =
        runCatching { remoteDataSource.fetchUser(id) }
}
```

---

## 3. Clean Code

Clean code is readable, simple, and expressive. It reduces cognitive load and makes future changes safer.

### Naming

```kotlin
// Bad
fun calc(a: Int, b: Int): Int = a + b
val d = 86400

// Good
fun calculateTotalPrice(basePrice: Int, taxAmount: Int): Int = basePrice + taxAmount
val secondsInADay = 86_400
```

### Functions

- Do one thing only.
- Keep them short (ideally under 20 lines).
- Use descriptive names — avoid comments that explain WHAT; write code that explains itself.

```kotlin
// Bad
fun process(u: User): Boolean {
    // check if email is valid
    if (!u.email.contains("@")) return false
    // save to db
    db.save(u)
    // send notification
    emailService.send(u.email, "Welcome!")
    return true
}

// Good
fun registerUser(user: User): Result<Unit> = runCatching {
    require(isValidEmail(user.email)) { "Invalid email: ${user.email}" }
    userRepository.save(user)
    notificationService.sendWelcomeEmail(user)
}

private fun isValidEmail(email: String): Boolean =
    email.contains("@") && email.contains(".")
```

### Avoid Magic Numbers and Strings

```kotlin
// Bad
if (response.code == 401) logout()

// Good
private const val HTTP_UNAUTHORIZED = 401

if (response.code == HTTP_UNAUTHORIZED) logout()
```

### Null Safety

Leverage Kotlin's type system to eliminate null-pointer exceptions:

```kotlin
// Bad (Java-style null checks)
fun getUsername(user: User?): String {
    if (user != null && user.name != null) {
        return user.name
    }
    return "Guest"
}

// Good (idiomatic Kotlin)
fun getUsername(user: User?): String = user?.name ?: "Guest"
```

### Error Handling with Result

```kotlin
suspend fun loadProfile(userId: String): Result<UserProfile> = runCatching {
    val user = userRepository.getUser(userId).getOrThrow()
    val posts = postRepository.getPostsByUser(userId).getOrThrow()
    UserProfile(user, posts)
}

// Caller
viewModelScope.launch {
    loadProfile(userId)
        .onSuccess { profile -> _state.value = State.Success(profile) }
        .onFailure { error -> _state.value = State.Error(error.localizedMessage) }
}
```

### Extension Functions

```kotlin
fun String.isValidEmail(): Boolean =
    android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()

fun Int.dpToPx(context: Context): Int =
    (this * context.resources.displayMetrics.density).toInt()

// Usage
if (email.isValidEmail()) { /* ... */ }
val paddingPx = 16.dpToPx(context)
```

---

## 4. Data Sources

Data sources are the lowest layer in Clean Architecture. They are responsible for fetching or persisting raw data from a specific origin (network, database, preferences, cache).

### Types of Data Sources

| Type   | Responsibility                      | Example                       |
|--------|-------------------------------------|-------------------------------|
| Remote | Fetch data from REST API / GraphQL  | Retrofit / OkHttp             |
| Local  | Persist data in a local database    | Room                          |
| Cache  | Fast in-memory or disk caching      | DataStore / SharedPreferences |

### Remote Data Source

```kotlin
// Interface (domain/data boundary)
interface UserRemoteDataSource {
    suspend fun fetchUser(id: String): UserDto
    suspend fun fetchAllUsers(): List<UserDto>
}

// Implementation
class UserRemoteDataSourceImpl @Inject constructor(
    private val apiService: UserApiService
) : UserRemoteDataSource {

    override suspend fun fetchUser(id: String): UserDto =
        apiService.getUser(id)

    override suspend fun fetchAllUsers(): List<UserDto> =
        apiService.getAllUsers()
}
```

### Local Data Source

```kotlin
interface UserLocalDataSource {
    suspend fun getUser(id: String): UserEntity?
    suspend fun saveUser(user: UserEntity)
    suspend fun deleteUser(id: String)
    fun observeAllUsers(): Flow<List<UserEntity>>
}

class UserLocalDataSourceImpl @Inject constructor(
    private val userDao: UserDao
) : UserLocalDataSource {

    override suspend fun getUser(id: String): UserEntity? = userDao.findById(id)

    override suspend fun saveUser(user: UserEntity) = userDao.upsert(user)

    override suspend fun deleteUser(id: String) = userDao.deleteById(id)

    override fun observeAllUsers(): Flow<List<UserEntity>> = userDao.observeAll()
}
```

### Data Transfer Objects (DTOs) vs Entities vs Domain Models

```kotlin
// DTO — raw API response shape
data class UserDto(
    @SerializedName("user_id") val userId: String,
    @SerializedName("full_name") val fullName: String,
    @SerializedName("email_address") val email: String
)

// Entity — Room database row
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val cachedAt: Long = System.currentTimeMillis()
)

// Domain Model — pure Kotlin, no framework annotations
data class User(
    val id: String,
    val name: String,
    val email: String
)

// Mappers
fun UserDto.toDomain() = User(id = userId, name = fullName, email = email)
fun UserEntity.toDomain() = User(id = id, name = name, email = email)
fun User.toEntity() = UserEntity(id = id, name = name, email = email)
```

---

## 5. Repositories

The Repository is the single source of truth for a given data domain. It abstracts the origin of data from the rest of the app. The domain layer defines the interface; the data layer provides the implementation.

### Repository Interface (Domain Layer)

```kotlin
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
    fun observeUsers(): Flow<List<User>>
    suspend fun refreshUsers(): Result<Unit>
}
```

### Repository Implementation (Data Layer)

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource,
    private val networkMonitor: NetworkMonitor
) : UserRepository {

    // Cache-first strategy: return local data, then refresh from network
    override suspend fun getUser(id: String): Result<User> = runCatching {
        val cached = localDataSource.getUser(id)
        if (cached != null) return@runCatching cached.toDomain()

        val remote = remoteDataSource.fetchUser(id)
        localDataSource.saveUser(remote.toEntity())
        remote.toDomain()
    }

    override suspend fun saveUser(user: User): Result<Unit> = runCatching {
        localDataSource.saveUser(user.toEntity())
        if (networkMonitor.isConnected) {
            remoteDataSource.updateUser(user.toDto())
        }
    }

    // Always observe from local DB (Room emits updates automatically)
    override fun observeUsers(): Flow<List<User>> =
        localDataSource.observeAllUsers().map { entities ->
            entities.map { it.toDomain() }
        }

    // Explicit refresh from network → save to DB → Room Flow emits update
    override suspend fun refreshUsers(): Result<Unit> = runCatching {
        val users = remoteDataSource.fetchAllUsers()
        users.forEach { localDataSource.saveUser(it.toEntity()) }
    }
}
```

### Network-First vs Cache-First Strategies

```kotlin
// Network-first (always try network, fall back to cache)
suspend fun getUserNetworkFirst(id: String): Result<User> = runCatching {
    try {
        val remote = remoteDataSource.fetchUser(id)
        localDataSource.saveUser(remote.toEntity())
        remote.toDomain()
    } catch (e: IOException) {
        localDataSource.getUser(id)?.toDomain()
            ?: throw NoSuchElementException("User $id not found in cache")
    }
}

// Cache-first (use cache, then refresh in background)
fun getUserCacheFirst(id: String): Flow<Result<User>> = flow {
    val cached = localDataSource.getUser(id)
    if (cached != null) emit(Result.success(cached.toDomain()))

    try {
        val remote = remoteDataSource.fetchUser(id)
        localDataSource.saveUser(remote.toEntity())
        emit(Result.success(remote.toDomain()))
    } catch (e: Exception) {
        if (cached == null) emit(Result.failure(e))
    }
}
```

---

## 6. Use Cases

A Use Case (also called Interactor) encapsulates a single business operation. It orchestrates data from one or more repositories and applies business rules. ViewModels depend on Use Cases, not repositories.

### Why Use Cases?

- Keeps ViewModels thin.
- Business rules live in one place, not scattered across ViewModels.
- Easy to unit-test in isolation.
- Reusable across multiple ViewModels or platforms.

### Base Use Case Pattern

```kotlin
// Suspending use case (one-shot)
abstract class UseCase<in P, out R> {
    suspend operator fun invoke(params: P): Result<R> = runCatching { execute(params) }
    protected abstract suspend fun execute(params: P): R
}

// Flow use case (continuous observation)
abstract class FlowUseCase<in P, out R> {
    operator fun invoke(params: P): Flow<Result<R>> = execute(params)
        .map { Result.success(it) }
        .catch { emit(Result.failure(it)) }
    protected abstract fun execute(params: P): Flow<R>
}

// No-param variant
abstract class NoParamUseCase<out R> {
    suspend operator fun invoke(): Result<R> = runCatching { execute() }
    protected abstract suspend fun execute(): R
}
```

### Concrete Use Case Examples

```kotlin
// Get user by ID
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) : UseCase<GetUserUseCase.Params, User>() {

    data class Params(val userId: String)

    override suspend fun execute(params: Params): User =
        userRepository.getUser(params.userId).getOrThrow()
}

// Login with validation
class LoginUseCase @Inject constructor(
    private val authRepository: AuthRepository,
    private val sessionManager: SessionManager
) : UseCase<LoginUseCase.Params, AuthToken>() {

    data class Params(val email: String, val password: String)

    override suspend fun execute(params: Params): AuthToken {
        require(params.email.isNotBlank()) { "Email cannot be blank" }
        require(params.password.length >= 8) { "Password must be at least 8 characters" }

        val token = authRepository.login(params.email, params.password).getOrThrow()
        sessionManager.saveToken(token)
        return token
    }
}

// Observe users as a stream
class ObserveUsersUseCase @Inject constructor(
    private val userRepository: UserRepository
) : FlowUseCase<Unit, List<User>>() {

    override fun execute(params: Unit): Flow<List<User>> =
        userRepository.observeUsers()
}
```

### Usage in ViewModel

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val observeUsersUseCase: ObserveUsersUseCase,
    private val refreshUsersUseCase: RefreshUsersUseCase
) : ViewModel() {

    val users: StateFlow<Result<List<User>>> = observeUsersUseCase(Unit)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), Result.success(emptyList()))

    fun refresh() {
        viewModelScope.launch { refreshUsersUseCase() }
    }
}
```

---

## 7. ViewModel

ViewModel is a Jetpack component that holds and manages UI-related data. It survives configuration changes (screen rotations) and is lifecycle-aware.

### Key Responsibilities

- Expose UI state via `StateFlow`.
- React to user events via functions.
- Launch coroutines in `viewModelScope`.
- Never hold references to Context, Views, or Activities.

### UI State Design

Model UI state as a sealed class to make all possible states explicit:

```kotlin
sealed class UserDetailUiState {
    object Idle : UserDetailUiState()
    object Loading : UserDetailUiState()
    data class Success(val user: User) : UserDetailUiState()
    data class Error(val message: String, val retryable: Boolean) : UserDetailUiState()
}
```

For complex screens, use a single data class state:

```kotlin
data class UserListUiState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val isRefreshing: Boolean = false
)
```

### Full ViewModel Example

```kotlin
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getUserUseCase: GetUserUseCase,
    private val updateUserUseCase: UpdateUserUseCase
) : ViewModel() {

    private val userId: String = checkNotNull(savedStateHandle["userId"])

    private val _uiState = MutableStateFlow<UserDetailUiState>(UserDetailUiState.Idle)
    val uiState: StateFlow<UserDetailUiState> = _uiState.asStateFlow()

    // One-shot events (navigation, snackbars) use SharedFlow
    private val _events = MutableSharedFlow<UserDetailEvent>()
    val events: SharedFlow<UserDetailEvent> = _events.asSharedFlow()

    init {
        loadUser()
    }

    fun loadUser() {
        viewModelScope.launch {
            _uiState.value = UserDetailUiState.Loading
            getUserUseCase(GetUserUseCase.Params(userId))
                .onSuccess { user -> _uiState.value = UserDetailUiState.Success(user) }
                .onFailure { e ->
                    _uiState.value = UserDetailUiState.Error(
                        message = e.localizedMessage ?: "An error occurred",
                        retryable = e is IOException
                    )
                }
        }
    }

    fun updateUserName(newName: String) {
        val current = (_uiState.value as? UserDetailUiState.Success)?.user ?: return
        viewModelScope.launch {
            updateUserUseCase(UpdateUserUseCase.Params(current.copy(name = newName)))
                .onSuccess { _events.emit(UserDetailEvent.ShowSnackbar("Name updated")) }
                .onFailure { _events.emit(UserDetailEvent.ShowSnackbar("Update failed")) }
        }
    }
}

sealed class UserDetailEvent {
    data class ShowSnackbar(val message: String) : UserDetailEvent()
    object NavigateBack : UserDetailEvent()
}
```

### SavedStateHandle

Use `SavedStateHandle` for navigation arguments and state that must survive process death:

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val searchUseCase: SearchUsersUseCase
) : ViewModel() {

    var searchQuery by savedStateHandle.saveable { mutableStateOf("") }
        private set

    fun onQueryChanged(query: String) {
        searchQuery = query
    }
}
```

### Sharing State Between ViewModels

For shared state, use an `activityViewModels()` scoped ViewModel or inject a shared repository:

```kotlin
// Shared ViewModel scoped to Activity
@HiltViewModel
class SharedCartViewModel @Inject constructor(
    private val cartRepository: CartRepository
) : ViewModel() {
    val cartItems: StateFlow<List<CartItem>> = cartRepository.observeCart()
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())
}
```

---

## 8. Jetpack Compose

Jetpack Compose is Android's modern declarative UI toolkit. Instead of manipulating XML views, you describe what the UI should look like given a state, and Compose handles recomposition.

### Core Concepts

| Concept          | Description                                                                       |
|------------------|-----------------------------------------------------------------------------------|
| Composable       | A function annotated with `@Composable` that emits UI                             |
| State            | Data that drives the UI. When state changes, Compose recomposes affected composables |
| Recomposition    | Compose re-runs only the composables that read changed state                      |
| Side Effects     | Operations that escape the Compose tree (`LaunchedEffect`, `SideEffect`, `DisposableEffect`) |
| Hoisting         | Moving state up to the caller so composables are stateless and testable           |

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
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            AsyncImage(
                model = user.avatarUrl,
                contentDescription = "Avatar of ${user.name}",
                modifier = Modifier
                    .size(48.dp)
                    .clip(CircleShape)
            )
            Spacer(modifier = Modifier.width(12.dp))
            Column {
                Text(text = user.name, style = MaterialTheme.typography.titleMedium)
                Text(text = user.email, style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

### State Hoisting

```kotlin
// Stateful (owns state — use at top level)
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.searchQuery
    SearchBar(
        query = query,
        onQueryChanged = viewModel::onQueryChanged
    )
}

// Stateless (receives state — easy to test and reuse)
@Composable
fun SearchBar(
    query: String,
    onQueryChanged: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChanged,
        placeholder = { Text("Search users...") },
        modifier = modifier.fillMaxWidth()
    )
}
```

### Collecting ViewModel State

```kotlin
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(lifecycleOwner) {
        viewModel.events
            .flowWithLifecycle(lifecycleOwner.lifecycle)
            .collect { event ->
                when (event) {
                    is UserListEvent.ShowSnackbar -> { /* show snackbar */ }
                }
            }
    }

    when (val state = uiState) {
        is UserListUiState.Loading -> LoadingScreen()
        is UserListUiState.Success -> UserList(users = state.users, onUserClick = { /* navigate */ })
        is UserListUiState.Error   -> ErrorScreen(message = state.message, onRetry = viewModel::refresh)
    }
}
```

### Side Effects

```kotlin
// LaunchedEffect — run suspend code when key changes
LaunchedEffect(userId) {
    viewModel.loadUser(userId)
}

// DisposableEffect — cleanup on leave
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event ->
        if (event == Lifecycle.Event.ON_RESUME) viewModel.onResume()
    }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}

// SideEffect — run on every successful recomposition
SideEffect {
    systemUiController.setStatusBarColor(MaterialTheme.colorScheme.primary)
}
```

### Navigation with Compose

```kotlin
@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = "user_list") {
        composable("user_list") {
            UserListScreen(onNavigateToDetail = { id ->
                navController.navigate("user_detail/$id")
            })
        }
        composable(
            route = "user_detail/{userId}",
            arguments = listOf(navArgument("userId") { type = NavType.StringType })
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId") ?: return@composable
            UserDetailScreen(userId = userId, onNavigateBack = navController::popBackStack)
        }
    }
}
```

### Performance Best Practices

```kotlin
// Use `key` to help Compose identify list items
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserCard(user = user, onUserClick = onUserClick)
    }
}

// Use `remember` to avoid recomputation on every recomposition
val sortedUsers = remember(users) { users.sortedBy { it.name } }

// Use `derivedStateOf` for state that derives from other state
val hasUsers by remember { derivedStateOf { users.isNotEmpty() } }

// Avoid creating lambdas on every recomposition — hoist or `remember` them
val onUserClick = remember<(String) -> Unit>(navController) {
    { userId -> navController.navigate("user_detail/$userId") }
}
```

---

## 9. Network Operations with OkHttp

OkHttp is the HTTP client that powers Retrofit. Understanding OkHttp directly lets you configure interceptors, timeouts, caching, logging, and authentication.

### Gradle Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")
    // OR kotlinx.serialization
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
}
```

### OkHttp Client Configuration

```kotlin
object NetworkModule {

    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        @ApplicationContext context: Context
    ): OkHttpClient {
        val cacheSize = 10L * 1024 * 1024 // 10 MB
        val cache = Cache(context.cacheDir, cacheSize)

        return OkHttpClient.Builder()
            .cache(cache)
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(authInterceptor)
            .addInterceptor(CacheInterceptor())
            .addInterceptor(
                HttpLoggingInterceptor().apply {
                    level = if (BuildConfig.DEBUG)
                        HttpLoggingInterceptor.Level.BODY
                    else
                        HttpLoggingInterceptor.Level.NONE
                }
            )
            .build()
    }
}
```

### Authentication Interceptor

```kotlin
class AuthInterceptor @Inject constructor(
    private val sessionManager: SessionManager
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = sessionManager.getToken()
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer $token")
            .header("Accept", "application/json")
            .build()

        val response = chain.proceed(request)

        if (response.code == 401) {
            sessionManager.clearSession()
        }

        return response
    }
}
```

### Cache Interceptor

```kotlin
class CacheInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        val cacheControl = CacheControl.Builder()
            .maxAge(5, TimeUnit.MINUTES)
            .build()
        return response.newBuilder()
            .removeHeader("Pragma")
            .removeHeader("Cache-Control")
            .header("Cache-Control", cacheControl.toString())
            .build()
    }
}
```

### Retrofit API Service

```kotlin
interface UserApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto

    @GET("users")
    suspend fun getAllUsers(
        @Query("page") page: Int = 1,
        @Query("per_page") perPage: Int = 20
    ): PagedResponse<UserDto>

    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): UserDto

    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") id: String,
        @Body request: UpdateUserRequest
    ): UserDto

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String): Response<Unit>

    @Multipart
    @POST("users/{id}/avatar")
    suspend fun uploadAvatar(
        @Path("id") id: String,
        @Part avatar: MultipartBody.Part
    ): UserDto
}
```

### Retrofit Instance

```kotlin
fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
    Retrofit.Builder()
        .baseUrl("https://api.example.com/v1/")
        .client(okHttpClient)
        .addConverterFactory(
            Json { ignoreUnknownKeys = true }
                .asConverterFactory("application/json".toMediaType())
        )
        .build()
```

### Error Handling

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>()
    object NetworkError : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
}

suspend fun <T> safeApiCall(apiCall: suspend () -> T): NetworkResult<T> = try {
    NetworkResult.Success(apiCall())
} catch (e: HttpException) {
    val errorBody = e.response()?.errorBody()?.string()
    NetworkResult.Error(e.code(), errorBody ?: e.message())
} catch (e: IOException) {
    NetworkResult.NetworkError
}

// Usage in data source
override suspend fun fetchUser(id: String): NetworkResult<UserDto> =
    safeApiCall { apiService.getUser(id) }
```

### Paging with OkHttp

```kotlin
class UserPagingSource @Inject constructor(
    private val apiService: UserApiService
) : PagingSource<Int, UserDto>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, UserDto> {
        val page = params.key ?: 1
        return try {
            val response = apiService.getAllUsers(page = page, perPage = params.loadSize)
            LoadResult.Page(
                data = response.data,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.data.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, UserDto>): Int? =
        state.anchorPosition?.let { state.closestPageToPosition(it)?.prevKey?.plus(1) }
}
```

---

## 10. Dependency Injection with Hilt

Hilt is Google's recommended DI framework for Android, built on top of Dagger 2. It reduces boilerplate and integrates with Jetpack components.

### Gradle Setup

```kotlin
// project-level build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android") version "2.51.1" apply false
}

// app-level build.gradle.kts
plugins {
    id("com.google.dagger.hilt.android")
    id("kotlin-kapt")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.51.1")
    kapt("com.google.dagger:hilt-android-compiler:2.51.1")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
}

kapt {
    correctErrorTypes = true
}
```

### Application Class

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

### Hilt Modules

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(authInterceptor: AuthInterceptor): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/v1/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    @Provides
    @Singleton
    fun provideUserApiService(retrofit: Retrofit): UserApiService =
        retrofit.create(UserApiService::class.java)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository

    @Binds
    @Singleton
    abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
}

@Module
@InstallIn(SingletonComponent::class)
abstract class DataSourceModule {

    @Binds
    @Singleton
    abstract fun bindUserRemoteDataSource(impl: UserRemoteDataSourceImpl): UserRemoteDataSource

    @Binds
    @Singleton
    abstract fun bindUserLocalDataSource(impl: UserLocalDataSourceImpl): UserLocalDataSource
}
```

### Hilt Scopes

| Scope                    | Component                  | Lifetime                     |
|--------------------------|----------------------------|------------------------------|
| `@Singleton`             | `SingletonComponent`       | App lifetime                 |
| `@ActivityRetainedScoped`| `ActivityRetainedComponent`| Survives config changes      |
| `@ActivityScoped`        | `ActivityComponent`        | Activity lifetime            |
| `@ViewModelScoped`       | `ViewModelComponent`       | ViewModel lifetime           |
| `@FragmentScoped`        | `FragmentComponent`        | Fragment lifetime            |

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {

    @Provides
    @ViewModelScoped
    fun provideSomeViewModelScopedDependency(): SomeDependency = SomeDependency()
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
        setContent { AppNavHost() }
    }
}

// ViewModel (via @HiltViewModel)
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase
) : ViewModel() { /* ... */ }

// Composable (via hiltViewModel())
@Composable
fun UserScreen() {
    val viewModel: UserViewModel = hiltViewModel()
}
```

### Custom Qualifiers

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthenticatedOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class UnauthenticatedOkHttpClient

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    @AuthenticatedOkHttpClient
    fun provideAuthenticatedClient(authInterceptor: AuthInterceptor): OkHttpClient =
        OkHttpClient.Builder().addInterceptor(authInterceptor).build()

    @Provides
    @Singleton
    @UnauthenticatedOkHttpClient
    fun provideUnauthenticatedClient(): OkHttpClient =
        OkHttpClient.Builder().build()
}
```

### Testing with Hilt

```kotlin
@HiltAndroidTest
class UserRepositoryTest {

    @get:Rule val hiltRule = HiltAndroidRule(this)

    @Inject lateinit var userRepository: UserRepository

    @Before
    fun setUp() { hiltRule.inject() }

    @Test
    fun `getUser returns user from cache when available`() = runTest {
        // test body
    }
}

// Replace production module in tests
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [NetworkModule::class]
)
@Module
object FakeNetworkModule {

    @Provides
    @Singleton
    fun provideUserApiService(): UserApiService = FakeUserApiService()
}
```

---

## 11. Data Storage

### Room

Room is the recommended local persistence library for Android. It provides an abstraction layer over SQLite with compile-time query validation.

#### Gradle Setup

```kotlin
dependencies {
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")
}
```

#### Entity

```kotlin
@Entity(
    tableName = "users",
    indices = [Index(value = ["email"], unique = true)]
)
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name") val name: String,
    val email: String,
    @ColumnInfo(name = "avatar_url") val avatarUrl: String?,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

@Entity(tableName = "posts")
data class PostEntity(
    @PrimaryKey val id: String,
    val title: String,
    val body: String,
    @ColumnInfo(name = "user_id") val userId: String
)

// Relationship
data class UserWithPosts(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val posts: List<PostEntity>
)
```

#### DAO

```kotlin
@Dao
interface UserDao {

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun findById(id: String): UserEntity?

    @Query("SELECT * FROM users ORDER BY full_name ASC")
    fun observeAll(): Flow<List<UserEntity>>

    @Query("""
        SELECT * FROM users
        WHERE full_name LIKE '%' || :query || '%'
           OR email LIKE '%' || :query || '%'
    """)
    fun search(query: String): Flow<List<UserEntity>>

    @Upsert
    suspend fun upsert(user: UserEntity)

    @Upsert
    suspend fun upsertAll(users: List<UserEntity>)

    @Delete
    suspend fun delete(user: UserEntity)

    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteById(id: String)

    @Query("DELETE FROM users")
    suspend fun deleteAll()

    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithPosts(userId: String): UserWithPosts?
}
```

#### Database

```kotlin
@Database(
    entities = [UserEntity::class, PostEntity::class],
    version = 2,
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao
    abstract fun postDao(): PostDao
}

val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    }
}

class Converters {
    @TypeConverter fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }
    @TypeConverter fun dateToTimestamp(date: Date?): Long? = date?.time
}
```

#### Hilt Module for Room

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
            .addMigrations(MIGRATION_1_2)
            .build()

    @Provides
    fun provideUserDao(db: AppDatabase): UserDao = db.userDao()

    @Provides
    fun providePostDao(db: AppDatabase): PostDao = db.postDao()
}
```

---

### SharedPreferences & SecureSharedPreferences

Use `SharedPreferences` for non-sensitive key-value pairs. Use `EncryptedSharedPreferences` (from Jetpack Security) for sensitive data such as tokens and user credentials.

#### Gradle Dependencies

```kotlin
dependencies {
    implementation("androidx.security:security-crypto:1.1.0-alpha06")
    // For DataStore (modern alternative)
    implementation("androidx.datastore:datastore-preferences:1.1.1")
}
```

#### SharedPreferences (Non-sensitive)

```kotlin
class AppPreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val prefs: SharedPreferences by lazy {
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

    fun clearAll() = prefs.edit { clear() }

    companion object {
        private const val KEY_DARK_MODE = "dark_mode"
        private const val KEY_LANGUAGE  = "language"
        private const val KEY_LAST_SYNC = "last_sync"
    }
}
```

#### EncryptedSharedPreferences (Sensitive)

```kotlin
class SecurePreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey: MasterKey by lazy {
        MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
    }

    private val prefs: SharedPreferences by lazy {
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
        set(value) = if (value != null) {
            prefs.edit { putString(KEY_AUTH_TOKEN, value) }
        } else {
            prefs.edit { remove(KEY_AUTH_TOKEN) }
        }

    var refreshToken: String?
        get() = prefs.getString(KEY_REFRESH_TOKEN, null)
        set(value) = if (value != null) {
            prefs.edit { putString(KEY_REFRESH_TOKEN, value) }
        } else {
            prefs.edit { remove(KEY_REFRESH_TOKEN) }
        }

    var userId: String?
        get() = prefs.getString(KEY_USER_ID, null)
        set(value) = if (value != null) {
            prefs.edit { putString(KEY_USER_ID, value) }
        } else {
            prefs.edit { remove(KEY_USER_ID) }
        }

    fun clearSession() = prefs.edit { clear() }

    companion object {
        private const val KEY_AUTH_TOKEN    = "auth_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
        private const val KEY_USER_ID       = "user_id"
    }
}
```

#### Session Manager (Combining Both)

```kotlin
class SessionManager @Inject constructor(
    private val securePreferences: SecurePreferences,
    private val appPreferences: AppPreferences
) {
    val isLoggedIn: Boolean get() = securePreferences.authToken != null

    fun saveSession(token: AuthToken) {
        securePreferences.authToken = token.accessToken
        securePreferences.refreshToken = token.refreshToken
        securePreferences.userId = token.userId
    }

    fun getToken(): String? = securePreferences.authToken

    fun clearSession() {
        securePreferences.clearSession()
        appPreferences.clearAll()
    }
}
```

#### Modern Alternative: DataStore

For new projects, prefer `DataStore<Preferences>` over `SharedPreferences`. It is coroutine-friendly, type-safe, and handles errors more gracefully:

```kotlin
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

class SettingsRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    companion object {
        val DARK_MODE_KEY = booleanPreferencesKey("dark_mode")
        val LANGUAGE_KEY  = stringPreferencesKey("language")
    }

    val isDarkModeEnabled: Flow<Boolean> = context.dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs -> prefs[DARK_MODE_KEY] ?: false }

    suspend fun setDarkMode(enabled: Boolean) {
        context.dataStore.edit { prefs -> prefs[DARK_MODE_KEY] = enabled }
    }
}
```

---

## 12. Image Loading with COIL

COIL (Coroutine Image Loader) is a modern, Kotlin-first image loading library built with coroutines, OkHttp, and AndroidX.

### Gradle Dependencies

```kotlin
dependencies {
    implementation("io.coil-kt:coil-compose:2.6.0")
    implementation("io.coil-kt:coil-gif:2.6.0")    // GIF support
    implementation("io.coil-kt:coil-svg:2.6.0")    // SVG support
    implementation("io.coil-kt:coil-video:2.6.0")  // Video thumbnails
}
```

### Basic Usage in Compose

```kotlin
// Simple image from URL
AsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = "Profile picture",
    modifier = Modifier
        .size(80.dp)
        .clip(CircleShape)
)

// With placeholder, error fallback, and content scale
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/image.jpg")
        .crossfade(true)
        .crossfade(300)
        .build(),
    contentDescription = "User avatar",
    placeholder = painterResource(R.drawable.placeholder_avatar),
    error = painterResource(R.drawable.error_avatar),
    contentScale = ContentScale.Crop,
    modifier = Modifier
        .size(64.dp)
        .clip(CircleShape)
        .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape)
)
```

### Coil Singleton Configuration (with Hilt)

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object CoilModule {

    @Provides
    @Singleton
    fun provideImageLoader(
        @ApplicationContext context: Context,
        okHttpClient: OkHttpClient
    ): ImageLoader = ImageLoader.Builder(context)
        .okHttpClient(okHttpClient)         // Reuse app's OkHttp (shares cache & interceptors)
        .crossfade(true)
        .memoryCache {
            MemoryCache.Builder(context)
                .maxSizePercent(0.25)       // Use 25% of available memory
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("image_cache"))
                .maxSizePercent(0.02)       // 2% of disk space
                .build()
        }
        .respectCacheHeaders(false)
        .components {
            add(GifDecoder.Factory())
            add(SvgDecoder.Factory(context))
        }
        .logger(DebugLogger())
        .build()
}
```

### Transformations

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .transformations(
            CircleCropTransformation(),
            RoundedCornersTransformation(radius = 16f),
            BlurTransformation(context, radius = 10f, sampling = 2f)
        )
        .build(),
    contentDescription = null
)
```

### SubcomposeAsyncImage (Custom Loading/Error States)

```kotlin
SubcomposeAsyncImage(
    model = imageUrl,
    contentDescription = "Product image"
) {
    when (painter.state) {
        is AsyncImagePainter.State.Loading -> {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(MaterialTheme.colorScheme.surfaceVariant),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator(strokeWidth = 2.dp)
            }
        }
        is AsyncImagePainter.State.Error -> {
            Image(
                painter = painterResource(R.drawable.ic_broken_image),
                contentDescription = "Failed to load"
            )
        }
        else -> SubcomposeAsyncImageContent()
    }
}
```

### Preloading Images

```kotlin
val imageLoader = LocalImageLoader.current
LaunchedEffect(nextImageUrl) {
    val request = ImageRequest.Builder(context)
        .data(nextImageUrl)
        .memoryCachePolicy(CachePolicy.ENABLED)
        .build()
    imageLoader.enqueue(request)
}
```

### Image Loading in Non-Compose Code

```kotlin
class UserAvatarViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val imageLoader: ImageLoader
) : ViewModel() {

    fun preloadAvatars(urls: List<String>) {
        urls.forEach { url ->
            val request = ImageRequest.Builder(context)
                .data(url)
                .build()
            imageLoader.enqueue(request)
        }
    }
}
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

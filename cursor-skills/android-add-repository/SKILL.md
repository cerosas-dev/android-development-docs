---
name: android-add-repository
description: Use when asked to create or add a repository for an Android module. Triggers on "add repository", "create repository", "new repo", "/android-add-repository". Generates the domain-layer interface, the data-layer implementation, and the Hilt @Binds wiring exactly as documented in android_clean_architecture.md §8 (Repositories) and §13 (Hilt). Always asks the user which REST endpoints (or GraphQL operations) the repository should consume.
paths:
  - "**/*.kt"
  - "**/build.gradle.kts"
  - "**/libs.versions.toml"
disable-model-invocation: false
---

# Add Repository

> **Tool names note.** `Read` / `Write` / `Edit` / `Bash` / `Grep` / `Glob` below refer to Cursor's equivalent file, shell, and search operations. The instructions communicate batching discipline — Cursor's agent batches independent tool calls the same way.

**Doc contract.** Source of truth = `android_clean_architecture.md` §1 (UDF / layering), §2 (naming), §7 (data sources), §8 *Repository Interface* + *Repository Implementation* + *Cache Strategies*, §13 *`@Binds` vs `@Provides`*. `Read` first; doc wins on drift.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits.

## Inputs (ask in one message)
Ask the user for all of the following in a single message; do not proceed until every input is provided.

- **Entity** — PascalCase (`User`).
- **Endpoints / GraphQL ops** *(required)* — list of `<op> → methodName` and per-read cache flag (default cache-first for `GET /{id}` + lists; network-first only if user flags volatility).
- **Backing stores** — multi-select: `Remote`, `Local`, `In-memory cache`, `Preferences`. Default: Remote + Local + In-memory cache.
- **Reactive APIs?** — yes/no. Yes → `observe<Entity>()` + `observe<Entity>s()` (default yes when Local selected).

## Files produced
- `domain/repository/<Entity>Repository.kt` — interface (zero Android imports).
- `data/repository/<Entity>RepositoryImpl.kt` — `@Inject constructor`.
- `data/mapper/<Entity>Mappers.kt` — created if missing (`Dto.toDomain`, `Dto.toEntity`, `Entity.toDomain`, `Domain.toEntity`).
- `di/RepositoryModule.kt` — `@Binds` entry appended (new file if absent).

## Canonical templates (doc §8)

**Interface** (`domain/`, no framework imports):
```kotlin
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
    suspend fun refreshUsers(): Result<Unit>
    fun observeUsers(): Flow<List<User>>
    fun observeUser(id: String): Flow<Result<User>>
}
```

Rules: one-shot ops return `Result<T>` (never throw); writes return `Result<Unit>` (never `Boolean`); reactive ops named `observe*`; never returns DTO/Entity.

**Impl** (cache-first, doc §8 lines 1502–1547):
```kotlin
class UserRepositoryImpl @Inject constructor(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource:  UserLocalDataSource,
    private val inMemoryCache:    UserInMemoryCache,
    private val networkMonitor:   NetworkMonitor
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        inMemoryCache.get(id)?.let { return Result.success(it) }
        localDataSource.getUser(id)?.let {
            inMemoryCache.put(it.toDomain())
            return Result.success(it.toDomain())
        }
        val dto = remoteDataSource.fetchUser(id)
        val user = dto.toDomain()
        localDataSource.saveUser(dto.toEntity()); inMemoryCache.put(user); user
    }
    // saveUser / refreshUsers / observeUsers / observeUser — follow doc §8 verbatim
}
```

Network-first vs cache-first: doc §8 *Cache Strategies* — pick from there; do not invent.

**Hilt** (doc §13):
```kotlin
@Module @InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

`@Binds` (never `@Provides`), `@Singleton`, `abstract class` (never `object`).

## Parallelism (mandatory)
- **All slice `Write`s in one message** (interface + impl + mappers + Hilt `@Binds`).
- **Prerequisite scans** (`*RemoteDataSource`, `*LocalDataSource`, existing `RepositoryModule`, existing mappers) → one message of parallel `Grep`s.
- **Multiple entities in one invocation** → dispatch one `android-feature-target` subagent per repository, single message.
- Sequential only: input prompt + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §1, §2, §7, §8, §13.**
2. **Verify prerequisites:** `<Entity>RemoteDataSource` + `<Entity>LocalDataSource` exist; if not, route to `/android-add-http-datasource`. Don't auto-generate them.
3. **Ask inputs** (one message).
4. **Detect existing module** (`RepositoryModule`, existing `@Binds` entries) — append, don't duplicate.
5. **Resolve mappers** — read existing or create from doc §7.
6. **List target paths** for approval.
7. **Write slice** (parallel `Write`s, order: interface → mappers (if new) → impl → Hilt).
8. **Verify:** `./gradlew :app:compileDebugKotlin`.
9. **Suggest:** `/android-add-use-case`.

## Refuse
- Interface in `data/` (must live in `domain/repository/`).
- Returning DTO / Entity (only domain models).
- Throwing across the boundary (use `runCatching` → `Result`).
- `@Provides` for the impl (use `@Binds`).
- Mapping inlined inside repository methods (push to `data/mapper/`).
- `Result<Boolean>` for writes (use `Result<Unit>`).
- One repo spanning two domains (split).
- Calling `apiService` / `userDao` directly (call the data source interface).

---
name: android-add-http-datasource
description: Use when asked to create an HTTP / REST / OkHttp / Retrofit data source for an Android module. Triggers on "add http datasource", "add remote datasource", "create okhttp datasource", "add retrofit api", "/android-add-http-datasource". Generates the Retrofit ApiService, the RemoteDataSource interface + impl, and Hilt @Provides/@Binds wiring exactly as documented in android_clean_architecture.md §7 (Data Sources), §12 (Network Operations with OkHttp), and §13 (Hilt). Always asks the user for the server base URL.
paths:
  - "**/*.kt"
  - "**/build.gradle.kts"
  - "**/libs.versions.toml"
disable-model-invocation: false
---

# Add HTTP Data Source

> **Tool names note.** `Read` / `Write` / `Edit` / `Bash` / `Grep` / `Glob` below refer to Cursor's equivalent file, shell, and search operations. The instructions communicate batching discipline — Cursor's agent batches independent tool calls the same way.

**Doc contract.** Source of truth = `android_clean_architecture.md` §7 *Remote Data Source*, §12 *Retrofit API Service* + *OkHttp Client Configuration* + *Retrofit Instance*, §13 *Hilt Modules `@Provides` vs `@Binds`*. `Read` first; doc wins on drift.

## Inputs (ask in one message)
Ask the user for all of the following in a single message; do not proceed until every input is provided.

- **Base URL** *(required)* — full URL with **trailing slash** (`https://api.example.com/v1/`). Reject if missing slash.
- **Resource** — PascalCase (`User`).
- **Endpoints** — list of `<VERB> path → methodName`, body DTO, response DTO. Default: `GET users/{id}` + `GET users?page=&per_page=`.
- **Auth required?** — yes/no. Yes → reuse project's `AuthInterceptor` (doc §12), don't add per-call headers.
- **Qualifier** *(optional)* — only when ≥2 base URLs coexist; produce `@Qualifier annotation class <Name>ApiUrl` per doc §13.

## Files produced
- `data/remote/api/<Resource>ApiService.kt`
- `data/remote/dto/<Resource>Dto.kt`
- `data/remote/datasource/<Resource>RemoteDataSource.kt` (interface)
- `data/remote/datasource/<Resource>RemoteDataSourceImpl.kt`
- `di/NetworkModule.kt` — appended (`@Provides` Retrofit + ApiService)
- `di/DataSourceModule.kt` — appended (`@Binds` interface→impl)

## Canonical templates (doc §7 + §12)

**ApiService** (every method `suspend`; path/query/body annotations; `Response<Unit>` for void deletes):
```kotlin
interface UserApiService {
    @GET("users/{id}") suspend fun getUser(@Path("id") id: String): UserDto
    @GET("users") suspend fun getAllUsers(@Query("page") page: Int, @Query("per_page") perPage: Int): List<UserDto>
    @POST("users") suspend fun createUser(@Body request: CreateUserRequest): UserDto
    @DELETE("users/{id}") suspend fun deleteUser(@Path("id") id: String): Response<Unit>
}
```

**RemoteDataSource + Impl** (doc §7 lines 1366–1386):
```kotlin
interface UserRemoteDataSource { suspend fun fetchUser(id: String): UserDto /* one per endpoint */ }

class UserRemoteDataSourceImpl @Inject constructor(
    private val apiService: UserApiService
) : UserRemoteDataSource {
    override suspend fun fetchUser(id: String) = apiService.getUser(id)
}
```

**DTO** — `@Serializable` + `@SerialName` mapping (doc §7 lines 1442–1449). Optional fields → `String? = null`.

**Hilt** — `object NetworkModule` with `@Provides @Singleton` for `Retrofit` + `ApiService` (doc §12 lines 2423–2434). Substitute the user's base URL. Reuse existing `provideOkHttpClient` if found; otherwise create from doc §12 lines 2315–2337 verbatim. **`abstract class DataSourceModule` with `@Binds`** for interface↔impl. Never mix `@Binds`/`@Provides` in one module.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits. Required artifacts: `com.squareup.okhttp3:okhttp`, `com.squareup.okhttp3:logging-interceptor`, `com.squareup.retrofit2:retrofit`, `com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter`, `org.jetbrains.kotlinx:kotlinx-serialization-json`.

## Parallelism (mandatory)
- **All slice `Write`s in one message** (DTO + ApiService + DS interface + DS impl + Hilt providers + Hilt `@Binds`).
- **Detection scans** (existing `provideOkHttpClient`, `provideRetrofit`, existing DTOs) → one message of parallel `Grep`s.
- **Multiple resources in one invocation** (`User`, `Order`, `Invoice`) → dispatch one `android-feature-target` subagent per resource, single message.
- Sequential only: input prompt + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §7, §12, §13.**
2. **Verify deps** (add missing).
3. **Ask inputs** (one message).
4. **Detect existing wiring** (parallel `Grep`s).
5. **List target paths** for approval.
6. **Write slice** (parallel `Write`s).
7. **Verify:** `./gradlew :app:compileDebugKotlin`.
8. **Suggest:** `/android-add-repository`.

## Refuse
- Base URL without trailing slash.
- Absolute URL in `@GET("https://…")`.
- DTO leaking past `data/`.
- Returning `Call<…>` instead of `suspend fun`.
- `@Provides` for `RemoteDataSourceImpl` (use `@Binds`).
- `@Binds` + `@Provides` in the same `object` module.

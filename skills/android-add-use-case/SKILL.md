---
name: android-add-use-case
description: Use when asked to create or add a use case (interactor) for an Android module. Triggers on "add use case", "create use case", "new interactor", "add interactor", "/android-add-use-case". Generates a use case in domain/usecase/ that extends the documented UseCase / FlowUseCase / NoParamsUseCase base classes, with Hilt @Inject constructor wiring per android_clean_architecture.md §9. Always asks the user which repositories (and/or other use cases) this use case depends on.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Add Use Case

**Doc contract.** Source of truth = `android_clean_architecture.md` §2 (naming), §8 (collaborators), §9 *Base Abstractions* + *Concrete Use Cases* + *Composing Use Cases*, §13 (Hilt). `Read` first; doc wins on drift.

**When NOT to create** (doc §9): if the ViewModel just calls `repository.method(id)`, a use case is overkill. Generate one only when there's validation, multi-repo coordination, or a business rule. Confirm before scaffolding a pass-through.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits.

## Inputs (single AskUserQuestion)
- **Name** — PascalCase ending in `UseCase` (`GetUserUseCase`). Reject otherwise.
- **Base class** — `UseCase<Params, Result>` | `FlowUseCase<Params, Result>` | `NoParamsUseCase<Result>` (doc §9).
- **Repositories** *(required)* — list of repo interfaces from `domain/repository/`. Grep to verify; reject missing with pointer to `/android-add-repository`.
- **Composed use cases** *(optional)* — existing `*UseCase` classes to chain (doc §9 *Composing Use Cases*).
- **Helpers** *(optional)* — validators / formatters (e.g., `EmailValidator`).
- **`Params` fields** — fields of the nested `data class Params(...)`. Skip for `NoParamsUseCase`.
- **Return type** — domain type (`User`, `Unit`, `List<Order>`).

## Files produced
- `domain/usecase/<Verb><Entity>UseCase.kt` — single file.
- `domain/usecase/UseCase.kt` — only if base abstractions missing (doc §9 lines 1594–1613 verbatim).

No Hilt module entries — `@Inject constructor` is enough for unscoped concrete classes (doc §13).

## Canonical template (doc §9 lines 1617–1623)

```kotlin
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) : UseCase<GetUserUseCase.Params, User>() {

    data class Params(val userId: String)

    override suspend fun execute(params: Params): User =
        userRepository.getUser(params.userId).getOrThrow()
}
```

Rules:
- `Params` is **nested** and named exactly `Params`. Callers: `GetUserUseCase.Params(...)`.
- `execute` returns raw `T` and **throws** on failure; the base's `invoke` wraps in `Result`.
- Validation via `require(...)` / `check(...)` (doc §9 lines 1635–1640).
- Never catch and rewrap inside `execute`.

Variants — follow doc §9 verbatim, do not reinterpret:
- **FlowUseCase**: `execute(params: P): Flow<R>`; callers `useCase(Unit)` when no params.
- **NoParamsUseCase**: `override suspend fun execute(): R`; callers `useCase()`.
- **Composing**: inject other use cases; call them as functions.

## Parallelism (mandatory)
- **Prerequisite scans** (`grep domain/repository/`, `grep domain/usecase/`, base abstractions) → one message of parallel `Grep`s.
- **Base abstractions write** — if missing, write `UseCase.kt` **first** (sequential), then fan out.
- **Multiple use cases in one invocation** → one `Agent` per use case, single message.
- Sequential only: AskUserQuestion + base abstractions (if missing) + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §2, §8, §9, §13.**
2. **Verify prerequisites** (every named repo + composed use case exists; base abstractions present or write them first).
3. **Ask inputs** (one call).
4. **List target paths** for approval.
5. **Write** (parallel for multiple use cases; base abstractions sequentially first if needed).
6. **Verify:** `./gradlew :app:compileDebugKotlin`.
7. **Suggest:** `/android-add-view-model`.

## Refuse
- Use case in `data/` or `presentation/`.
- Skipping the base class (no inline `Result`-wrapping).
- `Params` outside the class.
- Multiple public methods (one operation per class — doc §9).
- Touching `DataSource` directly (only `Repository` + other use cases).
- Returning `Result<T>` from `execute` (returns `T`, throws; base wraps).
- Injecting `CoroutineDispatcher` (dispatchers belong in the data layer).
- Logging inside `execute`.

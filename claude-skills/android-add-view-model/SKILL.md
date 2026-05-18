---
name: android-add-view-model
description: Use when asked to create or add a ViewModel for an Android Compose screen. Triggers on "add view model", "add viewmodel", "create viewmodel", "new vm", "/android-add-view-model". Generates a Hilt-injected ViewModel together with its UiState (sealed class or data class) and Event sealed interface, following android_clean_architecture.md §10 (ViewModel) and §11 (Compose / Screen-Content pattern). Always asks the user which use cases this ViewModel will invoke.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Add ViewModel

**Doc contract.** Source of truth = `android_clean_architecture.md` §1 (UDF), §2 (naming + `presentation/feature/<feature>/`), §6 (coroutines, Flow), §9 (collaborators), §10 *UI State Design* + *Full ViewModel with Events* + *Debouncing User Input* + *Combining Multiple Flows*, §11 (Screen / Content pattern this VM feeds), §13 (`@HiltViewModel`, `SavedStateHandle`). `Read` first; doc wins on drift.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits. Required artifact: `androidx.hilt:hilt-navigation-compose`.

## Inputs (single AskUserQuestion)
- **Feature** — lowercase (`userlist`).
- **Use cases** *(required)* — list from `domain/usecase/`; grep to verify; reject missing.
- **UiState style** — `sealed class` (mutually exclusive states) | `data class` (multi-section dashboards) — doc §10.
- **Reactive sources?** — yes/no (yes if any injected use case is `FlowUseCase`/`Observe*`).
- **Nav arguments** — list from `SavedStateHandle` (e.g., `userId: String`).
- **Debounced input?** — yes/no (yes → `debounce + distinctUntilChanged + flatMapLatest` per doc §10).
- **One-time events?** — yes/no (default yes when navigation/snackbar exist).

## Files produced
- `presentation/feature/<feature>/<Feature>UiState.kt`
- `presentation/feature/<feature>/<Feature>Event.kt` *(only when events needed)*
- `presentation/feature/<feature>/<Feature>ViewModel.kt`

No Hilt module entries — `@HiltViewModel` is enough.

## Canonical template (doc §10 lines 1734–1781)

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

    private val _events = MutableSharedFlow<UserDetailEvent>(extraBufferCapacity = 1)
    val events: SharedFlow<UserDetailEvent> = _events.asSharedFlow()

    init { loadUser() }

    fun loadUser() = viewModelScope.launch {
        _uiState.value = UserDetailUiState.Loading
        getUserUseCase(GetUserUseCase.Params(userId))
            .onSuccess { _uiState.value = UserDetailUiState.Success(it) }
            .onFailure { e ->
                _uiState.value = UserDetailUiState.Error(
                    message = e.localizedMessage ?: "Unknown error",
                    isRetryable = e is IOException
                )
            }
    }
}
```

**UiState** — sealed (doc §10 lines 1710–1716) **or** data class with `AsyncData<T>` sections (lines 1719–1729). If `AsyncData` not in project, create at `presentation/common/AsyncData.kt`.

**Event** — sealed interface, one type per event (doc §10 lines 1777–1780).

**Debounced / combine** variants — doc §10 lines 1785–1817 verbatim.

## Parallelism (mandatory)
- **All slice `Write`s in one message** (`UiState` + `Event` + `ViewModel`).
- **Prerequisite scans** (each named use case, `@HiltViewModel` setup, `hilt-navigation-compose:1.2.0`) → one message of parallel `Grep`s.
- **Multiple ViewModels in one invocation** → one `Agent` per VM, single message.
- Sequential only: AskUserQuestion + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §1, §2, §6, §9, §10, §11, §13.**
2. **Verify prerequisites** (use cases exist; `hilt-navigation-compose` present).
3. **Ask inputs** (one call).
4. **List target paths** for approval.
5. **Write** (parallel `Write`s, order: `Event` → `UiState` → `ViewModel`).
6. **Verify:** `./gradlew :app:compileDebugKotlin`.
7. **Suggest:** `/android-add-content` then `/android-add-screen`.

## Refuse
- Exposing `MutableStateFlow` / `MutableSharedFlow` (always wrap with `asStateFlow()` / `asSharedFlow()`).
- One-time events via `StateFlow` (use `SharedFlow(replay = 0)` or `Channel`).
- Constructor-injecting a `Repository` (VMs depend on use cases).
- `@ApplicationContext` injection.
- `LiveData<T>` (doc uses `StateFlow`).
- Mixing UI state and events in one flow.
- Writing `_uiState.value` outside `viewModelScope.launch`.
- `runBlocking` in `init`.
- Storing raw `Result<T>` in `_uiState`.
- Calling `hiltViewModel()` inside a VM.

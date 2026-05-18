---
name: android-add-screen
description: Use when asked to create or add a stateful Composable Screen for an Android feature, or to scaffold the wiring half of the Screen / Content pattern. Triggers on "add screen", "add composable screen", "create screen composable", "stateful composable", "/android-add-screen". Generates an XxxScreen composable that wires the ViewModel via hiltViewModel(), collects state with collectAsStateWithLifecycle(), observes events in a LaunchedEffect, and delegates rendering to an XxxContent — exactly as documented in android_clean_architecture.md §11 Screen / Content Pattern. Always asks the user which XxxContent and which ViewModel to use.
paths:
  - "**/*.kt"
  - "**/build.gradle.kts"
  - "**/libs.versions.toml"
disable-model-invocation: false
---

# Add Compose Screen

> **Tool names note.** `Read` / `Write` / `Edit` / `Bash` / `Grep` / `Glob` below refer to Cursor's equivalent file, shell, and search operations. The instructions communicate batching discipline — Cursor's agent batches independent tool calls the same way.

**Doc contract.** Source of truth = `android_clean_architecture.md` §2 (naming), §10 (`uiState` / `events` surface), §11 *Screen / Content Pattern* step 4 + step 8, *Side Effects*, *Navigation*, §13 (`hiltViewModel()` for Composables). `Read` first; doc wins on drift.

Screen owns three things: Hilt resolution, lifecycle-aware state collection, one-time event dispatch. Everything else delegates to `<Feature>Content`.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits.

## Inputs (ask in one message)
Ask the user for all of the following in a single message; do not proceed until every input is provided.

- **Feature** — lowercase (`userlist`).
- **Content composable** *(required)* — `<Feature>Content` must exist; grep to verify; reject missing → route to `/android-add-content`. Read its parameter list to infer the wiring.
- **ViewModel** *(required)* — `<Feature>ViewModel` must exist; grep to verify; reject missing → route to `/android-add-view-model`. Read its public surface (`uiState`, `events`, `on*` functions).
- **Navigation lambdas** — `onNavigateXxx(...)` parameters supplied by the parent NavHost. Never `NavController` directly.
- **Generate Route wrapper?** — yes/no. Default yes when VM uses `SavedStateHandle` (doc §11 step 8).

## Files produced
- `presentation/feature/<feature>/<Feature>Screen.kt`
- `presentation/feature/<feature>/<Feature>Route.kt` *(only when requested)*

No Hilt module entries — `hiltViewModel()` resolves `@HiltViewModel` automatically.

## Canonical template (doc §11 step 4)

```kotlin
@Composable
fun UserListScreen(
    onNavigateToDetail: (String) -> Unit,
    viewModel: UserListViewModel = hiltViewModel()
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserListEvent.NavigateToDetail -> onNavigateToDetail(event.userId)
                is UserListEvent.ShowSnackbar     -> snackbarHostState.showSnackbar(event.message)
            }
        }
    }

    UserListContent(
        state          = state,
        snackbarHost   = snackbarHostState,
        onQueryChanged = viewModel::onQueryChanged,
        onUserClick    = viewModel::onUserClick,
        onRetry        = viewModel::onRetry
    )
}
```

Rules: nav lambdas **first**; `viewModel = hiltViewModel()` **last** (testable). `collectAsStateWithLifecycle()` (not `collectAsState()`). One `LaunchedEffect(Unit)` for events. Content lambdas via method refs.

**Route wrapper** (doc §11 step 8) — only when nav args exist:
```kotlin
@Composable
fun UserDetailRoute(backStackEntry: NavBackStackEntry, onNavigateBack: () -> Unit) {
    val userId = backStackEntry.arguments?.getString("userId").orEmpty()
    UserDetailScreen(userId = userId, onNavigateBack = onNavigateBack)
}
```

**NavHost wiring snippet** — print to user after generation; do NOT write into `AppNavHost.kt` automatically.

## Parallelism (mandatory)
- **Prerequisite reads** (`<Feature>Content.kt`, `<Feature>ViewModel.kt`, existing `AppNavHost.kt`) → one message of parallel `Read`s.
- **Slice writes** — `Screen` + `Route` (when requested) in one message of parallel `Write`s.
- **Multiple Screens in one invocation** → dispatch one `android-feature-target` subagent per Screen, single message.
- Sequential only: input prompt + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §2, §10, §11, §13.**
2. **Verify both prerequisites exist** (`Content` + `ViewModel`); reject with pointers if missing.
3. **Read both files** (parallel) and reconcile every Content parameter with a VM member (`uiState` / `events` / `on*`). If a Content parameter has no VM member, stop and ask.
4. **Ask inputs** (one message).
5. **List target files** for approval.
6. **Write** (`Screen`, then `Route` if requested — parallel).
7. **Verify:** `./gradlew :app:compileDebugKotlin`.
8. **Print** the NavHost wiring snippet for the user to integrate.

## Refuse
- Passing ViewModel into Content.
- `hiltViewModel()` from inside Content (only Screen).
- `NavController` parameter on Screen (use `onNavigateXxx` lambdas).
- `StateFlow`-emitted one-time events → fix the VM first.
- `collectAsState()` (use `collectAsStateWithLifecycle()`).
- Layout / `Scaffold` / `when (state)` in Screen — push down to Content.
- Multiple `LaunchedEffect(Unit)` blocks (combine into one).
- `viewModel()` no-Hilt overload (always `hiltViewModel()`).
- Forgetting `viewModel = hiltViewModel()` as the last parameter (breaks tests).
- Inlining navigation actions instead of forwarding `onNavigateXxx`.

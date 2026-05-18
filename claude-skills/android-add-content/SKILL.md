---
name: android-add-content
description: Use when asked to create or add a stateless Composable Content for an Android screen, or to scaffold the rendering half of the Screen / Content pattern. Triggers on "add content", "add composable content", "create content composable", "stateless composable", "/android-add-content". Generates an XxxContent composable that is a pure function of UiState + lambdas, with Preview entries per meaningful state, following android_clean_architecture.md §11 Screen / Content Pattern.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Add Compose Content

**Doc contract.** Source of truth = `android_clean_architecture.md` §2 (naming), §10 (`UiState` shape), §11 *State Hoisting* + *Screen / Content Pattern* + *Side Effects* + *Performance Best Practices* + *Compose Testing*. `Read` first; doc wins on drift.

Content = pure function of `UiState` + lambdas. **No** `hiltViewModel()`, **no** `viewModelScope`, **no** `NavController`, **no** coroutine launches — that's the Screen's job.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits.

## Inputs (single AskUserQuestion)
- **Feature** — lowercase (`userlist`).
- **UiState type** — must exist in `presentation/feature/<feature>/`; grep to verify; reject missing → route to `/android-add-view-model`.
- **Event callbacks** *(required)* — `on*` lambdas matching the actions the Screen will dispatch.
- **Layout** — `Scaffold` (default) | `Plain` (for nested use).
- **State branches** — list of meaningful states (Loading/Error/Empty/Success/...) — one `@Test` and one `@Preview` per branch.
- **Snackbar host?** — yes/no (default yes when the VM emits events).

## Files produced
- `presentation/feature/<feature>/<Feature>Content.kt` — function + one `@Preview` per state + private fixture helpers.

## Canonical template (doc §11 *Screen / Content Pattern* step 5)

```kotlin
@Composable
fun UserListContent(
    state: UserListUiState,
    snackbarHost: SnackbarHostState,
    onQueryChanged: (String) -> Unit,
    onUserClick: (String) -> Unit,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Scaffold(modifier = modifier, snackbarHost = { SnackbarHost(snackbarHost) }) { padding ->
        Column(Modifier.padding(padding)) {
            OutlinedTextField(
                value = state.query, onValueChange = onQueryChanged,
                modifier = Modifier.fillMaxWidth().padding(16.dp)
            )
            when {
                state.isLoading            -> LoadingIndicator()
                state.errorMessage != null -> ErrorState(state.errorMessage, onRetry)
                state.users.isEmpty()      -> EmptyState()
                else -> LazyColumn {
                    items(state.users, key = { it.id }) { UserCard(it, onUserClick) }
                }
            }
        }
    }
}
```

Signature rules: `state` first → lambdas → `modifier: Modifier = Modifier` **last**. Snackbar host always parameter (never `remember`'d inside).

**Previews** (doc §11 step 6) — one `private @Preview` per meaningful state, each constructs a `UiState` literal. No Hilt, no fake VM.

**Sealed-class branching** — `when (state)` over sealed subclasses; delegate to `<Feature>Success(...)` etc. for the populated branch.

## Coverage target

Match `@Preview` count to state-branch count. Each `@Test` in the matching UI test mirrors a `@Preview` (doc §8 *Match `@Preview` count and Content `@Test` count*).

## Parallelism (mandatory)
- **Detection scans** (`<Feature>UiState`, `AppTheme`, existing `sample*()` helpers) → one message of parallel `Grep`s.
- **Multiple Contents in one invocation** (`UserListContent`, `UserDetailContent`, …) → one `Agent` per Content, single message.
- Single-Content runs: one atomic `Write` (function + previews + helpers).
- Sequential only: AskUserQuestion + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §2, §10, §11.**
2. **Verify `<Feature>UiState` exists.** If not, stop and route to `/android-add-view-model`.
3. **Ask inputs** (one call).
4. **List target file** for approval.
5. **Write the file** (function + previews + helpers).
6. **Verify:** `./gradlew :app:compileDebugKotlin`; in Android Studio confirm all previews render.
7. **Suggest:** `/android-add-screen`.

## Refuse
- `hiltViewModel()` / `LocalLifecycleOwner` / `viewModelScope` references inside Content.
- ViewModel as a parameter.
- Reading `Channel` / `SharedFlow` / `StateFlow` inside Content.
- `LaunchedEffect` / coroutine launches inside Content.
- `suspend fun` for Content.
- `remember { SnackbarHostState() }` inside Content.
- `modifier` not last parameter.
- Missing `key = { it.id }` on `LazyColumn`.
- Mutating `UiState` fields.
- One `@Test` covering multiple states.
- `testTag` on `XxxScreen` instead of Content root.

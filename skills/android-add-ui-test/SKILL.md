---
name: android-add-ui-test
description: Use when asked to add UI tests for an Android Compose Component or Content composable. Triggers on "add ui test", "add ui tests", "add content test", "write compose ui test", "/android-add-ui-test". Reads android_testing.md §8 Screen vs Content Tests for patterns. UI tests target stateless XxxContent and reusable Composable components ONLY. Composable Screens are out of scope — they belong to /android-add-integration-test. User provides one or more files; if not, the skill asks for a module and identifies which Content / Component composables lack coverage and adds tests for those.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Add UI Test

**Doc contract.** Source of truth = `android_testing.md` §5 (Fixtures), §8 *UI Tests with Jetpack Compose* + *Screen vs Content Tests* + *Testing a Stateless Composable in Isolation*. `Read` first; doc wins on drift.

Targets: `*Content.kt` and reusable leaf composables (`UserCard`, `ErrorBanner`, etc.). **Never** `*Screen.kt` — those route to `/android-add-integration-test`.

## Inputs (AskUserQuestion)
- **Files** — one or more paths. Reject any `*Screen.kt` with pointer to `/android-add-integration-test`.
- **OR module name** — triggers *Coverage Analysis* (below), then a multi-select.

## Coverage Analysis (when no file given)

```bash
MODULE=<module>
MAIN="$MODULE/src/main/kotlin"; T="$MODULE/src/androidTest/kotlin"

# Content composables only
find "$MAIN" -type f -name "*Content.kt" | while read f; do
    c=$(basename "$f" .kt)
    find "$T" -type f -name "${c}Test.kt" | grep -q . || echo "NO_TEST $f"
done

# Reusable components
find "$MAIN" -path "*/presentation/common/components/*.kt" -type f 2>/dev/null | while read f; do
    c=$(basename "$f" .kt)
    find "$T" -type f -name "${c}Test.kt" | grep -q . || echo "NO_TEST $f"
done
```

Always exclude `*Screen.kt`.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits. Required: `androidx.compose.ui:ui-test-junit4` (`androidTestImplementation`), `androidx.compose.ui:ui-test-manifest` (`debugImplementation`), `com.google.truth:truth`, `androidx.test.ext:junit`. **No** `com.google.dagger:hilt-android-testing` — if you need it, the test is wrong.

## File layout
`src/androidTest/kotlin/<package>/<Composable>Test.kt`. Mirrors source path. Class = `<Composable>Test`.

## Canonical template (doc §8 *How — Content Test*)

```kotlin
@RunWith(AndroidJUnit4::class)
class UserListContentTest {

    @get:Rule val composeTestRule = createComposeRule()

    @Test
    fun showsEmptyState_whenNoUsers() {
        composeTestRule.setContent {
            AppTheme {
                UserListContent(
                    state          = UserListUiState(users = emptyList()),
                    snackbarHost   = remember { SnackbarHostState() },
                    onQueryChanged = {}, onUserClick = {}, onRetry = {}
                )
            }
        }
        composeTestRule.onNodeWithText("No users yet").assertIsDisplayed()
    }

    @Test
    fun clickingRetry_invokesOnRetryLambda() {
        var retried = false
        composeTestRule.setContent {
            AppTheme {
                UserListContent(
                    state = UserListUiState(errorMessage = "boom"),
                    snackbarHost = remember { SnackbarHostState() },
                    onQueryChanged = {}, onUserClick = {}, onRetry = { retried = true }
                )
            }
        }
        composeTestRule.onNodeWithText("Retry").performClick()
        assertThat(retried).isTrue()
    }
}
```

## Coverage target

Per doc §8 *Match @Preview count and Content @Test count*: **one `@Test` per `@Preview`** for the Content under test, plus one `@Test` per `on*` lambda (verifying invocation).

## Selectors

Prefer `testTag` over text. Tag the root of `XxxContent` (`Modifier.testTag("user_list_screen")`), not `XxxScreen`. Fix missing tags in production rather than working around.

## Fixtures (doc §5 + §8 *Reuse UiState fixtures*)

If a Content file has `private fun sampleXxx()` helpers used by `@Preview`, **promote** to `internal fun` (same file or shared `<Feature>Previews.kt`) so both `@Preview` and the test share one builder.

## No-robot rule (doc §8 *Robotise Screen tests, not Content tests*)

Content tests stay flat: `setContent → assert` or `setContent → perform → assert`. Robots earn their keep only on Screen tests.

## Parallelism (mandatory and high-value)
- **Coverage scans** → one message of parallel `Bash`/`Grep`.
- **N targets** → **one `Agent` per target**, single message. Pass each: production path, target type (Content vs component), the template section, output path, `@Preview` count to match.
- **Promoting `sample*()` fixtures**: do it **once before** the fan-out.
- Sequential only: AskUserQuestion + shared fixture promotion + final `connectedDebugAndroidTest`.

## Workflow

1. **Read `android_testing.md` §5, §8.**
2. **Resolve target(s)** — reject `*Screen.kt`. Module input → Coverage Analysis + confirm.
3. **Verify Gradle deps.**
4. **Promote shared `sample*()` helpers** if needed (before fan-out).
5. **For 2+ targets: fan out one Agent per target** (single message).
6. **Single target**: write the test file directly.
7. **Verify:** `./gradlew :<module>:compileDebugAndroidTestKotlin`; full run `connectedDebugAndroidTest` (needs device or `scripts/ci-emulator.sh`).
8. **Report** — files, `@Test` count, `@Preview` parity, `testTag` additions flagged.

## Refuse
- UI test for `*Screen.kt` → reroute to `/android-add-integration-test`.
- `@HiltAndroidTest` / `@TestInstallIn` in a Content test.
- `createAndroidComposeRule<HiltTestActivity>()` (wrong rule).
- `mockk<XxxViewModel>()` — state + lambdas only.
- `hiltViewModel()` anywhere in test setup.
- One `@Test` covering multiple states.
- `testTag` on `XxxScreen` instead of Content root.
- `@Composable fun sampleUsers()` for fixtures (use `private fun`, not Composable).

---
name: android-add-integration-test
description: Use when asked to add integration tests for an Android Compose Screen or NavHost. Triggers on "add integration test", "add integration tests", "add screen test", "add navhost test", "/android-add-integration-test". Reads android_testing.md §7 and §8 Screen vs Content Tests for patterns. Integration tests wire Hilt + ViewModel + Compose + Navigation together via createAndroidComposeRule<HiltTestActivity>() with @TestInstallIn fake repositories. User provides one or more files; if not, the skill asks for a module and identifies which Screen / NavHost composables lack coverage and adds tests for those.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Add Integration Test

**Doc contract.** Source of truth = `android_testing.md` §4 (Robot Pattern), §5 (Fixtures), §7 (Integration Tests — HiltTestRunner, in-memory Room, MockWebServer), §8 *Screen vs Content Tests* + *Testing a Screen with a ViewModel (Hilt)* + *Testing Navigation*. `Read` first; doc wins on drift.

Targets: `*Screen.kt` and the file containing `NavHost(...)`. **Never** `*Content.kt` / leaf components → `/android-add-ui-test`. Never JVM units → `/android-add-unit-test`.

Doc §8: "Default to Content tests. Reserve Screen tests for the wiring you cannot verify any other way."

## Inputs (AskUserQuestion)
- **Files** — one or more paths. Reject `*Content.kt` with pointer to `/android-add-ui-test`.
- **OR module name** — triggers *Coverage Analysis* (below), then a multi-select.

## Coverage Analysis (when no file given)

```bash
MODULE=<module>
MAIN="$MODULE/src/main/kotlin"; T="$MODULE/src/androidTest/kotlin"

# Screens (Route wrappers excluded)
find "$MAIN" -type f -name "*Screen.kt" | while read f; do
    c=$(basename "$f" .kt)
    find "$T" -type f -name "${c}Test.kt" | grep -q . || echo "NO_TEST $f"
done

# NavHost
grep -rl "NavHost(" "$MAIN" | while read f; do
    c=$(basename "$f" .kt)
    find "$T" -type f -name "${c}Test.kt" | grep -q . || echo "NO_TEST $f"
done
```

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits. Required: `androidx.compose.ui:ui-test-junit4` (`androidTestImplementation`), `androidx.compose.ui:ui-test-manifest` (`debugImplementation`), `com.google.dagger:hilt-android-testing` (`androidTestImplementation`) + `com.google.dagger:hilt-android-compiler` (`kaptAndroidTest`), `com.google.truth:truth`, `androidx.test.ext:junit`, `androidx.test:rules`. Set `testInstrumentationRunner = "<base-package>.HiltTestRunner"`.

## One-time setup (write if missing)

- `src/androidTest/kotlin/<base>/HiltTestRunner.kt` (doc §7 *Custom Hilt Test Runner*).
- `src/androidTest/kotlin/<base>/HiltTestActivity.kt` (`@AndroidEntryPoint class HiltTestActivity : ComponentActivity()`).
- `src/debug/AndroidManifest.xml` entry registering `HiltTestActivity`.

## File layout
`src/androidTest/kotlin/<package>/<Composable>Test.kt`. Mirrors source.

## Canonical template (doc §8 *How — Screen Test*)

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class UserListScreenTest {

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeTestRule = createAndroidComposeRule<HiltTestActivity>()

    @Inject lateinit var fakeUserRepository: FakeUserRepository

    @Before fun setUp() { hiltRule.inject() }

    @Test
    fun loadedUsers_renderInTheList() {
        fakeUserRepository.stubbedUsers = listOf(User("u1", "Alice", "alice@example.com", null))
        composeTestRule.setContent { AppTheme { UserListScreen(onNavigateToDetail = {}) } }
        composeTestRule.onNodeWithText("Alice").assertIsDisplayed()
    }

    @Test
    fun tappingUser_invokesNavigationLambda_exactlyOnce() {
        fakeUserRepository.stubbedUsers = listOf(User("u1", "Alice", "alice@example.com", null))
        val navigatedIds = mutableListOf<String>()
        composeTestRule.setContent {
            AppTheme { UserListScreen(onNavigateToDetail = { navigatedIds += it }) }
        }
        composeTestRule.onNodeWithText("Alice").performClick()
        assertThat(navigatedIds).containsExactly("u1")
    }
}
```

**`@TestInstallIn` module** under `src/androidTest/.../di/TestRepositoryModule.kt` swaps the production module for fakes:
```kotlin
@Module
@TestInstallIn(components = [SingletonComponent::class], replaces = [RepositoryModule::class])
abstract class TestRepositoryModule {
    @Binds @Singleton
    abstract fun bindUserRepository(impl: FakeUserRepository): UserRepository
}
```

## NavHost test template (doc §8 *Testing Navigation*)

`createAndroidComposeRule<HiltTestActivity>()`, host `AppNavHost(rememberNavController())`, assert post-nav Content root tag (`testTag("user_detail_screen")` lives on the Content root, not the Screen — doc §8 best practice).

## Coverage target (doc §8 *One Screen test per destination, minimum coverage*)

Per Screen: **(1) happy-path wiring + (2) one-time event delivery (exactly-once)**. Do not re-prove `UiState` branches — that's the Content test's job (doc §8 *Do not replay UI assertions in Screen tests*).

## Robot Pattern (doc §4 + §8 *Robotise Screen tests, not Content tests*)

When a Screen has 2+ tests with the same interactions, extract a Robot under `androidTest/.../robot/<Feature>Robot.kt`. Chain robots for multi-screen flows (doc §4 *Chaining Robots*).

## Parallelism (mandatory and high-value)
- **Coverage scans** → one message of parallel `Bash`/`Grep`.
- **One-time setup writes** (`HiltTestRunner`, `HiltTestActivity`, debug manifest) — if all missing, write in one message of parallel `Write`s.
- **N targets** → **one `Agent` per target**, single message. Pass each: production path, the template section, fake repo paths (pre-written), `@TestInstallIn` path (pre-written), output path.
- **Shared `Fake*Repository` + `TestRepositoryModule`**: write **once before** the fan-out; never let agents race to write them.
- Sequential only: AskUserQuestion + one-time/shared writes + final `connectedDebugAndroidTest`.

## Workflow

1. **Read `android_testing.md` §4, §5, §7, §8.**
2. **Resolve target(s)** — reject `*Content.kt`. Module input → Coverage Analysis + confirm.
3. **Verify Gradle deps + one-time setup** (`HiltTestRunner`, `HiltTestActivity`, manifest); write missing in parallel.
4. **Pre-write shared fakes + `TestRepositoryModule`** for every repo the target Screens depend on.
5. **For 2+ targets: fan out one Agent per target** (single message).
6. **Single target**: write the test file directly.
7. **Verify:** `./gradlew :<module>:compileDebugAndroidTestKotlin`; full run `connectedDebugAndroidTest` (needs device or `scripts/ci-emulator.sh`).
8. **Report** — files, `@Test` count, production `testTag` additions flagged.

## Refuse
- Integration test for `*Content.kt` → reroute to `/android-add-ui-test`.
- `createComposeRule()` for a Screen test (wrong rule).
- Re-asserting `UiState` branches already covered by Content tests.
- `mockk<XxxViewModel>()` — real VM + fake repo via `@TestInstallIn`.
- Hand-built `UiState` passed through `XxxScreen` (route to `/android-add-ui-test`).
- Skipping `@get:Rule(order = 0)` / `(order = 1)` — Hilt must initialise before Compose.
- `testTag` on `XxxScreen` (belongs on Content root).
- Interaction assertions where state assertions work (doc §3).
- NavHost test in a feature module (belongs in `:app` or whichever module owns the NavHost).
- `@VisibleForTesting` knobs on production code to make a test pass.

---
name: android-add-unit-test
description: Use when asked to add JVM unit tests for an Android class — DataSource, Repository, UseCase, or ViewModel. Triggers on "add unit test", "add unit tests", "write unit tests", "/android-add-unit-test". Reads android_testing.md §6 for patterns (MainDispatcherRule, Fakes vs Mocks, Truth, Turbine). User provides one or more files; if not, the skill asks for a module and identifies which DataSource / Repository / UseCase / ViewModel files lack coverage and adds tests for those.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Add Unit Test

**Doc contract.** Source of truth = `android_testing.md` §3 (MockK, Fakes vs Mocks vs Stubs), §5 (Test Fixtures), §6 (Unit Tests — Gradle, `MainDispatcherRule`, ViewModel + Use Case templates, MockK, Turbine). `Read` first; doc wins on drift.

Targets: **DataSource**, **Repository**, **UseCase**, **ViewModel** — JVM only (`src/test/`). Anything needing `Context` / Hilt / Room / Compose belongs to `/android-add-ui-test` or `/android-add-integration-test`.

## Inputs (AskUserQuestion)
- **Files** — one or more paths.
- **OR module name** — triggers *Coverage Analysis* (below), then a multi-select for confirmed gaps.

## Coverage Analysis (when no file given)

Enumerate testable classes by doc §2 naming suffixes, exclude base abstractions, check for sibling test:

```bash
MODULE=<module>
MAIN="$MODULE/src/main/kotlin"; T="$MODULE/src/test/kotlin"

find "$MAIN" -type f \( \
    -name "*RemoteDataSourceImpl.kt" -o -name "*LocalDataSourceImpl.kt" -o -name "*InMemoryCache.kt" \
 -o -name "*RepositoryImpl.kt" -o -name "*UseCase.kt" -o -name "*ViewModel.kt" \
\) -not -name "UseCase.kt" -not -name "FlowUseCase.kt" -not -name "NoParamsUseCase.kt" \
| while read f; do
    c=$(basename "$f" .kt)
    find "$T" -type f \( -name "${c}Test.kt" -o -name "${c}Spec.kt" \) | grep -q . || echo "NO_TEST $f"
  done
```

Present gap list via AskUserQuestion multi-select. Never bulk-write without confirmation.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits. Required (`testImplementation`): `junit:junit`, `io.mockk:mockk`, `org.jetbrains.kotlinx:kotlinx-coroutines-test`, `com.google.truth:truth`, `app.cash.turbine:turbine`.

## `MainDispatcherRule` — required for every ViewModel test

If missing, create once at `src/test/kotlin/<base-package>/util/MainDispatcherRule.kt` from doc §6 lines 1166–1182 verbatim.

## File layout
Test path mirrors source path. Test class = `<ClassUnderTest>Test`.

## Templates (doc §6 verbatim — follow exactly)

**Use Case** (doc §6 *Testing Use Cases*, lines 1318–1359):
- One `@Test` per `require/check` branch + happy path + repo-failure propagation.
- Fakes for repositories (doc §3 *Avoid Mocking What You Own Too Early*).
- `runTest { … }` (not `runBlocking`).

**ViewModel** (doc §6 *Testing ViewModels*, lines 1229–1316):
- `@get:Rule val mainDispatcherRule = MainDispatcherRule()`.
- One `@Test` per state transition + per event + per public function.
- Turbine `viewModel.uiState.test { … cancelAndConsumeRemainingEvents() }` for multi-step.
- `expectNoEvents()` for "nothing fires" assertions.

**Repository**:
- Fakes (not MockK) for data sources.
- One `@Test` per cache branch + online/offline + clear-on-refresh + `observe*` Turbine.

**Remote DataSource**:
- MockK on `ApiService` (doc §6 *Testing with MockK*). One `@Test` per method delegation.
- **Local DataSource**: push to `/android-add-integration-test` (needs Room).

## Fakes over Mocks (doc §6 *Fakes vs Mocks*)

Reuse existing `Fake*` under `testdoubles/` or `fake/`. If a fake is needed by **multiple** parallel agents, write it **once before the fan-out**, never let agents race.

## Parallelism (mandatory and high-value)
- **Coverage scans** → one message of parallel `Bash`/`Grep`.
- **N targets** → **one `Agent` (subagent_type: general-purpose) per target**, single message. Pass each agent: production path, layer, the template section, `MainDispatcherRule` path, fakes to reuse, output path.
- **One-target runs**: parallel `Read` + fake-fixture `Grep` + `MainDispatcherRule` check in one message.
- **Shared fakes**: write once before fan-out.
- Sequential only: AskUserQuestion + shared-fake pre-writes + final `./gradlew :<module>:testDebugUnitTest`.

## Workflow

1. **Read `android_testing.md` §3, §5, §6.**
2. **Resolve target(s)** — direct files, or run Coverage Analysis + confirm.
3. **Verify Gradle deps**; ensure `MainDispatcherRule` exists.
4. **Pre-write shared fakes** if any target needs them.
5. **For 2+ targets: fan out one Agent per target** (single message).
6. **Single target**: write the test file directly.
7. **Verify:** `./gradlew :<module>:testDebugUnitTest`. Fix tests, not production code.
8. **Report** — files created, `@Test` count per file, anything routed elsewhere.

## Refuse
- Instrumented work in `src/test/` (needs `Context`/Hilt/Room → reroute).
- Mocking what you own (Use Case → use a Fake repo, not `mockk<UserRepository>()`).
- Interaction assertions where state assertions work (doc §3).
- Missing `MainDispatcherRule` on a VM test.
- `runBlocking` instead of `runTest`.
- Plural `<Class>Tests.kt` (use singular `<Class>Test.kt`).
- One test class for two production classes.
- Skipping `cancelAndConsumeRemainingEvents()` in Turbine blocks.
- UI tests for a ViewModel (assert `uiState.value` + `events` via Turbine, not rendered UI).

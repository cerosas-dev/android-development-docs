---
name: android-create-module
description: Use when asked to create, scaffold, or bootstrap a new Android module, feature module, or package skeleton. Triggers on "create android module", "new android module", "scaffold module", "bootstrap module", "new feature module", "/android-create-module". Generates the Clean Architecture folder layout (di / domain / data / presentation) documented in android_clean_architecture.md §2, with placeholder files following the documented naming conventions. Asks user whether to scaffold as a Gradle sub-module or a feature package inside an existing module.
paths:
  - "**/*.kt"
  - "**/build.gradle.kts"
  - "**/settings.gradle.kts"
  - "**/libs.versions.toml"
disable-model-invocation: false
---

# Create Android Module

> **Tool names note.** `Read` / `Write` / `Edit` / `Bash` / `Grep` / `Glob` below refer to Cursor's equivalent file, shell, and search operations. The instructions communicate batching discipline — Cursor's agent batches independent tool calls the same way.

**Doc contract.** Source of truth = `android_clean_architecture.md` §2 (Codebase Structure + File Naming). `Read` it before generating; doc wins on any drift.

## Deps
When adding any: declare in `gradle/libs.versions.toml` only (never inline in `build.gradle.kts`) and pick the **latest stable** from Maven Central — skip `-alpha*`/`-beta*`/`-rc*`/`-dev*`/`SNAPSHOT`. Create the catalog if absent. Run `./gradlew help` after edits.

## Inputs (ask in one message)
Ask the user for all of the following in a single message; do not proceed until every input is provided.

- **Module name** — lowercase (`userlist`, `checkout`).
- **Style** — `Gradle sub-module` (multi-module) **or** `Feature package` (single-module).
- **Base package** — infer from `namespace =` in existing `build.gradle.kts`, then confirm.
- **Primary entity** — PascalCase (`User`, `Order`) — seeds placeholder names.

## Folder layout (mirror doc §2 *Single-Module Package Structure*, lines 241–302 verbatim)

```
<base-package>/
├── di/<Feature>Module.kt
├── domain/
│   ├── model/<Entity>.kt
│   ├── repository/<Entity>Repository.kt
│   └── usecase/                                  # empty — filled by /android-add-use-case
├── data/
│   ├── remote/{api,dto,datasource}/...
│   ├── local/{dao,entity,datasource}/...
│   ├── mapper/<Entity>Mappers.kt
│   └── repository/<Entity>RepositoryImpl.kt
└── presentation/feature/<feature>/
    ├── <Feature>Screen.kt
    ├── <Feature>Content.kt
    ├── <Feature>ViewModel.kt
    ├── <Feature>UiState.kt
    └── <Feature>Event.kt
```

Sub-module mode additionally produces `:feature:<feature>/build.gradle.kts` + `src/main/AndroidManifest.xml` and appends `include(":feature:<feature>")` to `settings.gradle.kts`.

## Naming (doc §2 table — exact, no improvising)

| Layer | Suffix | Layer | Suffix |
|---|---|---|---|
| Domain model | *(none)* | Use case | `UseCase` |
| Remote DTO | `Dto` | ViewModel | `ViewModel` |
| Room Entity | `Entity` | Screen | `Screen` |
| Room DAO | `Dao` | Content | `Content` |
| Remote DS | `RemoteDataSourceImpl` | UI state | `UiState` |
| Local DS | `LocalDataSourceImpl` | Event | `Event` |
| Repo interface | `Repository` | Hilt module | `Module` |
| Repo impl | `RepositoryImpl` |  |  |

## Placeholder content rule

Every generated file MUST compile against the project's current Kotlin/AGP/Compose. No `// TODO` empty stubs. Seed each with the minimal documented pattern from doc §7 / §8 / §9 / §10 / §11 / §13.

## Parallelism (mandatory)
- **All `Write`s for the tree → single message** (independent files).
- **Detection scans** (`find build.gradle.kts`, `grep namespace`, `grep applicationId`, locate `libs.versions.toml`) → single message.
- **Multiple feature names in one invocation** → dispatch one `android-feature-target` subagent per feature, single message.
- Sequential only: input prompt + final `./gradlew compileDebugKotlin`.

## Workflow

1. **Read `android_clean_architecture.md` §2** at repo root (fall back to this skill's tables if absent and say so in report).
2. **Detect project layout** (parallel scans above). If no Android project, stop.
3. **Ask the 4 inputs** in one message.
4. **List target paths** for approval.
5. **Write the tree** (parallel `Write`s). Sub-module mode: also update `settings.gradle.kts`, copy dep versions from `libs.versions.toml` — never invent versions.
6. **Verify:** `./gradlew :<feature>:compileDebugKotlin` (sub-module) or `:app:compileDebugKotlin` (feature package).
7. **Next-step pointer** to user: `/android-add-http-datasource` → `/android-add-repository` → `/android-add-use-case` → `/android-add-view-model` → `/android-add-content` → `/android-add-screen`.

## Refuse
- Cross-layer leak (`domain/` importing `androidx.*` or `data/`).
- Renamed suffixes (`UserRepo` instead of `UserRepository`).
- `@Provides` for owned types (use `@Binds`).
- Multiple public types in one file.
- Empty `// TODO` files that don't compile.

# Claude Code for Android Development

> A guide to installing Claude Code, wiring this repository's skills into any Android project, and using them to scaffold features that match `android_clean_architecture.md` and `android_testing.md`.

## Table of Contents

1. [What is Claude Code?](#1-what-is-claude-code)
2. [Installing Claude Code](#2-installing-claude-code)
3. [The Skills in This Repository](#3-the-skills-in-this-repository)
4. [Installing the Skills in Another Project](#4-installing-the-skills-in-another-project)
5. [Using the Skills](#5-using-the-skills)

---

## 1. What is Claude Code?

Claude Code is Anthropic's official command-line interface for Claude. It runs in your terminal, can read and edit files in your working directory, execute shell commands on your behalf, and load project-scoped configuration — including **skills**, the procedural recipes that this repository ships under `./claude-skills/`. Without Claude Code installed, the markdown files in `./claude-skills/` are just documents; with Claude Code installed and pointed at an Android project, they become executable scaffolding for Clean Architecture features, tests, and module bootstraps.

The skills in this repo are deliberately *narrow and procedural* — each one encodes a single, repeatable task (add a ViewModel, add a Repository, add an integration test) and cross-references specific section numbers in `android_clean_architecture.md` and `android_testing.md`. They turn this docs repository into a working code-generation toolkit.

---

## 2. Installing Claude Code

### 2.1 System Requirements

| Item | Requirement |
|------|-------------|
| **macOS** | 10.15 Catalina or later (Intel and Apple Silicon supported). |
| **Linux** | Ubuntu 20.04+ / Debian 10+ / Fedora 36+ / Arch (rolling). x86_64 or arm64. |
| **Windows** | Windows 10 (build 19041+) or Windows 11. PowerShell 5.1+ or WSL2. |
| **Disk** | ~150 MB for the binary and its working directory. |
| **Network** | Outbound HTTPS to `claude.ai` and `api.anthropic.com`. |
| **Node.js** | 18+ — only required for the `npm` installation path. The native installer needs no prerequisite. |
| **Account** | A free or paid Anthropic account (sign-in completes in the browser on first launch). |

### 2.2 macOS

The official native installer is the recommended path on macOS — it ships a self-updating binary and adds it to `PATH` automatically. The `npm` route is an alternative for engineers who already manage tooling through a global Node setup.

```bash
# Recommended — official native installer
curl -fsSL https://claude.ai/install.sh | bash

# Alternative — via npm (requires Node.js 18+)
npm install -g @anthropic-ai/claude-code
```

The native installer drops the binary into `~/.local/bin` and patches your shell profile (`~/.zshrc` on modern macOS, `~/.bash_profile` if you still run Bash) to add that directory to `PATH`. Restart your terminal or run `source ~/.zshrc` afterwards. Apple Silicon vs Intel architecture is detected automatically — there is no separate binary to choose.

### 2.3 Linux

The same native installer works across the major distributions. On Debian/Ubuntu it edits `~/.bashrc` and `~/.profile`; on Fedora and Arch you may need to add `~/.local/bin` to `PATH` yourself if your distribution does not include it by default. Do **not** run the installer with `sudo` — it is a per-user install by design, and running it as root will create a broken setup that you cannot launch from your normal user account.

```bash
# Recommended — official native installer
curl -fsSL https://claude.ai/install.sh | bash

# Alternative — via npm (requires Node.js 18+)
npm install -g @anthropic-ai/claude-code
```

If `~/.local/bin` is missing from your `PATH` after installation, append the following to your shell profile and restart the terminal:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 2.4 Windows

Windows has three viable installation paths, in order of recommendation:

```powershell
# Recommended — official native installer (PowerShell)
irm https://claude.ai/install.ps1 | iex

# Alternative — via npm (requires Node.js 18+)
npm install -g @anthropic-ai/claude-code
```

**WSL2 path (preferred for Android developers).** If you are already developing Android on Windows via WSL2 (Ubuntu running under Windows), install Claude Code *inside* the WSL2 distribution using the Linux instructions in §2.3. The WSL2 path interoperates with `adb`, the Android SDK, and Gradle wrappers more cleanly than the native Windows binary does, and it keeps your toolchain consistent with Android Studio's bundled JDK.

**PowerShell execution-policy gotcha.** If `irm ... | iex` fails with a signing error, your current user's execution policy is too restrictive. Run this once and retry:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

### 2.5 Verifying the Installation

Regardless of platform or method, the same two commands prove the install is healthy:

```bash
claude --version    # Prints the installed CLI version
claude doctor       # Diagnoses PATH, authentication, and config issues
```

If `claude --version` is not found, your shell has not picked up the new `PATH` entry — open a fresh terminal window or `source` your shell profile.

### 2.6 Authenticating

The first time you run `claude` in any directory, it opens your default browser and walks you through sign-in to your Anthropic account. The resulting token is cached under `~/.claude/` and reused by every subsequent session on the same machine. Tokens are **not** synchronised across machines — if you switch laptops, re-authenticate from the new one.

---

## 3. The Skills in This Repository

### 3.1 What is a Skill?

A **skill** is a procedural recipe stored as `./claude-skills/<name>/SKILL.md`, consisting of YAML frontmatter (`name`, `description`, `allowed-tools`) plus a markdown body that tells Claude exactly *how* to perform a single concrete task. When the user's request matches the skill's `description`, or when the user types `/<skill-name>` directly, Claude loads the body and follows it step by step.

Skills are distinct from two adjacent concepts:

- **`CLAUDE.md`** files are *always loaded* into the conversation as background context — repository conventions, architectural guardrails, things you want Claude to know without being asked.
- **Agents** are separate sub-Claude instances spawned for parallel or sandboxed work, each with their own tools and context.

Skills sit between the two: they are *on-demand* (loaded only when the description matches the request) and *narrow* (one concrete repeatable task per skill).

### 3.2 How the Skills Are Organised

Every skill in this repo is a single `SKILL.md` file inside a platform-scoped folder — `./claude-skills/<skill-name>/` for Android. Each one cross-references specific section numbers in the platform reference docs (`android_clean_architecture.md` + `android_testing.md` for Android) so the generated code matches the canonical patterns exactly.

Ten skills are shipped, dividing cleanly into three groups:

- **Scaffolding (1 skill).** Bootstraps the Clean Architecture folder layout (`android-create-module`).
- **Layer generators (6 skills).** One per architectural layer — HTTP DataSource, Repository, UseCase, ViewModel, Content, Screen — applied in dependency order.
- **Test generators (3 skills).** Unit tests (host-process), UI tests (Content + components), Integration tests (Screen + navigation).

### 3.3 Android Skill Catalogue

The table below lists every skill in the order you would typically invoke them while building a new feature — bottom-up through the architecture, then the matching tests at the end. Each row maps to a level-4 detail block immediately afterwards.

| # | Skill | Slash command | Generates |
|---|-------|--------------|-----------|
| 1 | `android-create-module` | `/android-create-module` | Clean Architecture folder layout (`di/`, `domain/`, `data/`, `presentation/`) as a Gradle sub-module or a feature package. |
| 2 | `android-add-http-datasource` | `/android-add-http-datasource` | Retrofit `ApiService`, `RemoteDataSource` interface + impl, Hilt wiring. |
| 3 | `android-add-repository` | `/android-add-repository` | Domain-layer repository interface, data-layer implementation, Hilt `@Binds`. |
| 4 | `android-add-use-case` | `/android-add-use-case` | Business-operation class extending `UseCase` / `FlowUseCase` / `NoParamsUseCase`. |
| 5 | `android-add-view-model` | `/android-add-view-model` | `@HiltViewModel` + `UiState` + `Event` sealed interface. |
| 6 | `android-add-content` | `/android-add-content` | Stateless `XxxContent` composable with `@Preview` per state. |
| 7 | `android-add-screen` | `/android-add-screen` | Stateful `XxxScreen` composable wiring ViewModel → Content. |
| 8 | `android-add-unit-test` | `/android-add-unit-test` | JVM unit tests for DataSources, Repositories, UseCases, ViewModels. |
| 9 | `android-add-ui-test` | `/android-add-ui-test` | Instrumented UI tests for `XxxContent` and reusable components. |
| 10 | `android-add-integration-test` | `/android-add-integration-test` | Instrumented integration tests for `XxxScreen` and `AppNavHost`. |

---

#### `android-create-module`

**What it generates.** A complete Clean Architecture folder skeleton — `di/`, `domain/{model,repository,usecase}/`, `data/{remote,local,mapper,repository}/`, `presentation/{feature,navigation,theme}/` — with placeholder files matching the documented naming conventions.

**Inputs it asks for.** Module name (PascalCase), and whether to scaffold as a Gradle sub-module (with its own `build.gradle.kts` and `AndroidManifest.xml`) or as a feature package inside an existing module.

**What it intentionally does not do.** It does not generate actual business code — no concrete repositories, ViewModels, or screens. Use the layer-generator skills below for those, in dependency order.

**Canonical reference.** `android_clean_architecture.md` §2 *Recommended Codebase Structure* and *File Naming Conventions*.

---

#### `android-add-http-datasource`

**What it generates.** A complete remote slice — a Retrofit `ApiService` with `@GET` / `@POST` / `@PUT` / `@DELETE` suspending methods, a `RemoteDataSource` interface in `data/remote/`, its implementation, the `@Serializable` DTOs under `data/remote/dto/`, and a Hilt module with `@Provides` for the Retrofit instance plus `@Binds` for the data source.

**Inputs it asks for.** Server base URL (must include scheme and a trailing slash, e.g. `https://api.example.com/v1/`), entity name, and the list of REST operations the data source will expose.

**What it intentionally does not do.** It does not generate the repository on top of the data source — that is `android-add-repository`'s job. It does not configure the `OkHttpClient` if one is already provided elsewhere in the module; it reuses it.

**Canonical reference.** `android_clean_architecture.md` §6 *Data Sources*, §11 *Network Operations with OkHttp*, §12 *Hilt*.

---

#### `android-add-repository`

**What it generates.** Three coordinated artefacts — the **domain-layer interface** in `domain/repository/`, the **data-layer implementation** in `data/repository/` (with constructor-injected `RemoteDataSource`, optional `LocalDataSource`, optional `InMemoryCache`), and the **Hilt `@Binds` wiring** that connects them.

**Inputs it asks for.** Domain entity name (drives the `<Entity>Repository.kt` / `<Entity>RepositoryImpl.kt` names), the list of REST endpoints (or GraphQL operations) the repository will consume, and the cache strategy (cache-first vs network-first vs no cache).

**What it intentionally does not do.** Mappers live in `data/mapper/`, not the repository — the skill enforces that boundary. It does not introduce any `try`/`catch` blocks; failures are propagated through `Result<T>` per the canonical doc.

**Canonical reference.** `android_clean_architecture.md` §7 *Repositories*, §12 *Hilt*.

---

#### `android-add-use-case`

**What it generates.** A single business-operation class under `domain/usecase/` that extends one of the documented base classes — `UseCase<Params, Result>`, `FlowUseCase<Params, Result>`, or `NoParamsUseCase<Result>` — with a nested `Params` data class (when applicable) and a constructor `@Inject` annotation. No Hilt module entry is needed because constructor injection alone is enough for unscoped concrete classes.

**Inputs it asks for.** Use case name (verb-first, e.g. `GetUserById`), the repositories or other use cases it depends on, whether it returns `Result<T>` (one-shot) or `Flow<T>` (streaming), and whether it needs `Params` at all.

**What it intentionally does not do.** It refuses to generate a use case that is a thin pass-through to a single repository call — the doc is explicit that *"if the ViewModel just calls `repository.getUser(id)`, a use case is overkill."* The skill will push back and ask you to invoke the repository directly from the ViewModel.

**Canonical reference.** `android_clean_architecture.md` §8 *Use Cases*.

---

#### `android-add-view-model`

**What it generates.** A complete ViewModel slice — the **`<Feature>ViewModel`** annotated with `@HiltViewModel`, the **`<Feature>UiState`** (sealed class for clean loading/success/error lifecycles, or data class for screens with independent async sections), and the **`<Feature>Event`** sealed interface for one-time effects (snackbars, navigation, dialog dismissals). State is exposed via `MutableStateFlow` and events via `Channel` (or `MutableSharedFlow(replay = 0)`).

**Inputs it asks for.** Feature name, the use cases the ViewModel will invoke, whether `UiState` is a sealed class or a data class, and whether the screen has user-typed input requiring `debounce + distinctUntilChanged + flatMapLatest` plumbing.

**What it intentionally does not do.** It does not generate the Screen or Content composables that consume it — those belong to `android-add-screen` and `android-add-content`. It does not expose the ViewModel's mutable state directly; everything is `asStateFlow()` / `receiveAsFlow()`.

**Canonical reference.** `android_clean_architecture.md` §9 *ViewModel*, §10 *Compose / Screen-Content Pattern*.

---

#### `android-add-content`

**What it generates.** A stateless `<Feature>Content` composable — the **rendering half** of the Screen / Content pattern. It is a pure function of a `UiState` parameter plus event lambdas: it contains **no** `hiltViewModel()` calls, **no** `viewModelScope` / coroutine launches, **no** `NavController` references, and **no** `LaunchedEffect` blocks bound to lifecycle. One `@Preview` is generated per meaningful `UiState` branch, mirroring how `android_testing.md` §8 *Screen vs Content Tests* uses them as the basis for Content tests.

**Inputs it asks for.** Feature name, the matching `UiState` type, the list of event lambdas the Content exposes (`onClick`, `onRetry`, `onQueryChanged`…), and the list of meaningful states to preview.

**What it intentionally does not do.** It does not wire the ViewModel — that is `android-add-screen`'s job. It does not pass the ViewModel itself as a parameter, because doing so would defeat the point of the split.

**Canonical reference.** `android_clean_architecture.md` §10 *Screen / Content Pattern → How to Use It* (step 5, stateless Content) and step 6 (previews per state); §10 *State Hoisting*; §10 *Performance Best Practices* (`LazyColumn` keys, `@Immutable` state, `derivedStateOf`).

---

#### `android-add-screen`

**What it generates.** A stateful `<Feature>Screen` composable — the **wiring half** of the Screen / Content pattern. It resolves the ViewModel via `hiltViewModel()`, collects state lifecycle-aware with `collectAsStateWithLifecycle()`, drains one-time events inside a `LaunchedEffect(Unit) { viewModel.events.collect { … } }` block, and forwards everything to the matching `<Feature>Content`. Navigation arguments arrive through lambdas (`onNavigateToDetail`, `onNavigateBack`); the Screen never sees a `NavController` directly.

**Inputs it asks for.** Feature name, which `<Feature>Content` to delegate to, which `<Feature>ViewModel` to inject, and the list of navigation lambdas (`onNavigateXxx`).

**What it intentionally does not do.** It does not render UI itself — every branch, layout, or styling decision lives in the Content. It does not parse navigation arguments inline; for screens with multiple `navArgument`s the skill optionally generates a thin `<Feature>Route` layer per `android_clean_architecture.md` §10 step 8.

**Canonical reference.** `android_clean_architecture.md` §10 *Screen / Content Pattern*, §10 *Side Effects*, §10 *Navigation*, §12 *Hilt → Injecting into Android Components*.

---

#### `android-add-unit-test`

**What it generates.** JVM unit tests under `src/test/kotlin/...` for the four documented unit-testable layers — **DataSources**, **Repositories**, **UseCases**, and **ViewModels** — using `MainDispatcherRule`, Truth assertions, Turbine for `Flow` collection, and either MockK or hand-rolled fakes (the skill prefers fakes over mocks per `android_testing.md` §6 *Fakes vs Mocks*).

**Inputs it asks for.** Either one or more source files to test, or a module path — in which case the skill identifies which classes in `data/`, `domain/usecase/`, and `presentation/feature/*/` lack coverage and adds tests for those. It also asks whether to use MockK or fakes for collaborators.

**What it intentionally does not do.** It never writes instrumented tests — UI rendering, Hilt graphs, Room DAO queries, and MockWebServer scenarios are out of scope and belong to `android-add-ui-test` or `android-add-integration-test`.

**Canonical reference.** `android_testing.md` §6 *Unit Tests*, §3 *Stubbing and Mocking*, §4 *Robot Pattern*, §5 *Test Fixtures*.

---

#### `android-add-ui-test`

**What it generates.** Instrumented Compose UI tests under `src/androidTest/kotlin/...` for the **stateless** half of the Screen / Content pattern — `XxxContent` composables and reusable leaf components (`UserCard`, `RatingStars`, `ErrorBanner`, …). Tests use `createComposeRule()`, drive the composable with a hand-built `UiState` literal, and assert on rendered nodes and lambda invocations. No `@HiltAndroidTest`, no `@Inject`, no `HiltTestActivity`.

**Inputs it asks for.** Either one or more `*Content.kt` files (or component files), or a module path — in which case the skill identifies Content composables and reusable components that lack coverage.

**What it intentionally does not do.** It never writes tests for `XxxScreen` composables — those belong to `android-add-integration-test`. `android_testing.md` §8 *Screen vs Content Tests* is explicit: *"Default to Content tests. Push all `UiState`-coverage assertions down to the Content layer. Reserve Screen tests for the wiring you cannot verify any other way."*

**Canonical reference.** `android_testing.md` §8 *UI Tests with Jetpack Compose → Screen vs Content Tests*, §4 *Robot Pattern*, §5 *Test Fixtures*.

---

#### `android-add-integration-test`

**What it generates.** Instrumented integration tests under `src/androidTest/kotlin/...` that prove the **wiring layer** of a Compose app works end-to-end. Two kinds of tests:

- **`XxxScreen` composables** — verified by booting Hilt with `@HiltAndroidTest`, replacing the production repository with a fake via `@TestInstallIn`, hosting the Screen inside a `HiltTestActivity`, and asserting that real ViewModel emissions reach the UI and that one-time events fire exactly once.
- **`AppNavHost` / route-level navigation** — verified with `createAndroidComposeRule<MainActivity>()`, asserting that tapping a destination switches the back-stack entry and renders the right next screen.

**Inputs it asks for.** Either one or more `*Screen.kt` / `*NavHost.kt` files, or a module path — in which case the skill scans for Screens and NavHosts lacking coverage.

**What it intentionally does not do.** It never writes tests for `XxxContent`, leaf components (`UserCard`, …), or pure JVM units. Those belong to `android-add-ui-test` and `android-add-unit-test` respectively. It does not re-prove `UiState` rendering branches that the Content test already covers — it focuses on wiring.

**Canonical reference.** `android_testing.md` §7 *Integration Tests*, §8 *Screen vs Content Tests*, §4 *Robot Pattern*, §5 *Test Fixtures*.

---

## 4. Installing the Skills in Another Project

The examples in this section show the Android install path. Project layout, scopes, refresh strategies, and skill-resolution rules are identical across platforms.

### 4.1 Scope: Project vs User

Claude Code resolves skills from two locations, and both can coexist:

| Scope | Path | Visible to | When to use |
|-------|------|-----------|-------------|
| **Project** | `<other-project>/.claude/skills/<skill-name>/SKILL.md` | Anyone running Claude Code from that project's working directory | The skills should travel with the repo — checked into git, reviewed via PR, versioned alongside the codebase. |
| **User** | `~/.claude/skills/<skill-name>/SKILL.md` | You only, in every project on this machine | You want the skills available everywhere without committing them to each repo. Personal toolkit. |

When a skill exists in both scopes with the same name, the project copy wins.

### 4.2 Per-Project Installation

The simplest installation is a clone-and-copy. Replace `<owner>/android-development-docs` with the actual GitHub path of your fork or mirror of this repository.

```bash
# Clone this docs repo to a scratch location
git clone https://github.com/<owner>/android-development-docs.git /tmp/android-docs

# Copy the skills into the target Android project
mkdir -p <your-android-project>/.claude/skills
cp -R /tmp/android-docs/claude-skills/* <your-android-project>/.claude/skills/

# Also copy the canonical reference docs — skills cite them by section number
cp /tmp/android-docs/android_clean_architecture.md <your-android-project>/
cp /tmp/android-docs/android_testing.md           <your-android-project>/

# Commit alongside the project
cd <your-android-project>
git add .claude/skills android_clean_architecture.md android_testing.md
git commit -m "Add Android Clean Architecture skills + canonical references"
```

Why copy the markdown references too: the skills explicitly cite section numbers in `android_clean_architecture.md` and `android_testing.md` (e.g. *"follows §10 *Screen / Content Pattern*"*). Without those files present, the skills still function but will generate slightly less context-aware code. Treat the two markdowns as part of the install bundle.

### 4.3 User-Wide Installation

To make the skills available in every project on your machine, install them under `~/.claude/skills/` and keep the reference docs somewhere stable:

```bash
git clone https://github.com/<owner>/android-development-docs.git /tmp/android-docs

# Install skills user-wide
mkdir -p ~/.claude/skills
cp -R /tmp/android-docs/claude-skills/* ~/.claude/skills/

# Park the canonical references where the skills can find them
mkdir -p ~/.claude/docs
cp /tmp/android-docs/android_clean_architecture.md ~/.claude/docs/
cp /tmp/android-docs/android_testing.md           ~/.claude/docs/
```

To help the skills find those references when invoked from any project, drop a one-line user-scope `~/.claude/CLAUDE.md`:

```markdown
# User-wide Android references

When working on an Android project, treat `~/.claude/docs/android_clean_architecture.md`
and `~/.claude/docs/android_testing.md` as canonical when their section numbers are cited.
```

### 4.4 Git Submodule / Sparse-Checkout Alternatives

For teams that want skills to track upstream automatically rather than be re-copied on each update, two heavier-weight options exist:

```bash
# Option A — git submodule (pulls full repo into a subdir)
cd <your-android-project>
git submodule add https://github.com/<owner>/android-development-docs.git docs/android-skills
ln -s ../docs/android-skills/claude-skills .claude/skills

# Option B — git sparse-checkout (clone only what you need)
git clone --filter=blob:none --no-checkout https://github.com/<owner>/android-development-docs.git
cd android-development-docs
git sparse-checkout init --cone
git sparse-checkout set claude-skills android_clean_architecture.md android_testing.md
git checkout
```

**Tradeoffs.** Submodules are the cleanest way to stay on the latest version — `git submodule update --remote` is a one-shot upgrade — but they add the usual submodule cognitive tax (cloning with `--recurse-submodules`, separate commit history, CI gotchas). Sparse-checkout produces a smaller working tree but is heavier to set up and harder to update non-interactively.

### 4.5 Updating the Skills Later

```bash
# Cloned-and-copied (§4.2 or §4.3): re-run the copy after a pull
cd /tmp/android-docs && git pull
cp -R claude-skills/* <target>/.claude/skills/

# Submodule
cd <your-android-project>
git submodule update --remote docs/android-skills
git commit -am "Update Android skills to latest"
```

### 4.6 Verifying the Skills Are Loaded

Open Claude Code from the target project's root directory and check that all ten skills surface in autocomplete:

```bash
cd <your-android-project>
claude
```

Once inside, start typing `/android-` — autocomplete should list every skill in §3.3. If they do not appear:

1. Confirm you launched `claude` from the project root, not a subdirectory. Project-scoped skills are only resolved when the current working directory is the project root or a child of it.
2. Run `claude doctor` to surface any config or path issues.
3. Verify each `SKILL.md` parses by running `head -5 .claude/skills/android-add-content/SKILL.md` and checking the YAML frontmatter has `name`, `description`, and `allowed-tools` keys.

---

## 5. Using the Skills

### 5.1 Slash-Command Invocation

The most predictable way to trigger a skill is to type its name with a leading slash. The slash command always matches the frontmatter `name:` field, which always matches the directory name:

```
/android-create-module
/android-add-http-datasource
/android-add-repository
/android-add-use-case
/android-add-view-model
/android-add-content
/android-add-screen
/android-add-unit-test
/android-add-ui-test
/android-add-integration-test
```

Each skill is explicit about the inputs it needs, and most use a single `AskUserQuestion` call to collect everything up-front (feature name, base URL, dependencies, etc.) rather than asking question by question.

### 5.2 Natural-Language Invocation

Every skill's `description:` field lists trigger phrases that auto-activate it without typing the slash command. For example, saying *"add a screen for user list"* triggers `android-add-screen`, and *"add an integration test for UserDetailScreen"* triggers `android-add-integration-test`. The skills are tuned aggressively for natural-language match — when in doubt, the slash command guarantees the exact one you want.

### 5.3 Recommended Workflow for a New Feature

The skills are most effective when invoked in dependency order, bottom-up through the architecture, then horizontally across the test pyramid:

1. `/android-create-module` — scaffold the feature folder.
2. `/android-add-http-datasource` — add the network slice (skip if reusing an existing one).
3. `/android-add-repository` — wire data sources behind a domain-layer interface.
4. `/android-add-use-case` — extract business rules (one use case per operation, only when justified per §8).
5. `/android-add-view-model` — design `UiState` + `Event`, inject the use cases.
6. `/android-add-content` — render every state as a pure function of `UiState`.
7. `/android-add-screen` — wire the ViewModel to the Content.
8. `/android-add-unit-test` — cover ViewModel, UseCases, Repository (and DataSource if non-trivial).
9. `/android-add-ui-test` — cover every Content state branch.
10. `/android-add-integration-test` — one Screen test per destination, plus NavHost route tests.

You can also start higher up the stack (e.g. begin with `/android-add-content` and a hand-written `UiState`) when prototyping, then fill in the layers below afterwards.

### 5.4 Troubleshooting

- **"My skill does not trigger when I type its slash command."** Confirm the directory name (`.claude/skills/<name>/` in the target project, or `./claude-skills/<name>/` in this source repo) matches the `name:` field in the frontmatter exactly — case-sensitive — and that there are no stray characters in the filename (not `SKILL.MD`, not `Skill.md`).
- **"The skill cites a section that does not exist in my project."** Copy `android_clean_architecture.md` and `android_testing.md` into the target project (or symlink them). The skills reference these files by section number and degrade silently when they are missing.
- **"Claude does not see my `.claude/skills/` at all."** You probably launched `claude` from a subdirectory. Skills are resolved relative to the project root — `cd` to the root first, or use the user-wide install in §4.3 to make them path-independent.
- **"I want to disable a skill temporarily."** Either move its directory out of `.claude/skills/`, or rename `SKILL.md` to `SKILL.md.bak`. Claude only loads files whose name is exactly `SKILL.md`.
- **"The generated code references an `AppTheme` / `HiltTestActivity` / fake repository that does not exist."** Those are placeholders from the canonical docs. The skill assumes you have an `AppTheme` (per `android_clean_architecture.md` §10), a `HiltTestActivity` (per `android_testing.md` §8), and at least one fake repository (per §6 *Fakes vs Mocks*). Add them once at the project level; every generated file will pick them up.

---

## License

This document is distributed under the **GNU General Public License v3.0 (GPL-3.0)**.

```
Claude Code for Android Development Documentation
Copyright (C) 2024 — Contributors

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <https://www.gnu.org/licenses/>.
```

For the full license text, see the [LICENSE](./LICENSE) file in this repository or visit
<https://www.gnu.org/licenses/gpl-3.0.html>.

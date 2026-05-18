# Cursor CLI for Android Development

> A guide to installing Cursor CLI, wiring this repository's skills into any Android project, and using them to scaffold features that match `android_clean_architecture.md` and `android_testing.md`.

## Table of Contents

1. [What is Cursor CLI?](#1-what-is-cursor-cli)
2. [Installing Cursor CLI](#2-installing-cursor-cli)
3. [The Skills in This Repository](#3-the-skills-in-this-repository)
4. [Installing the Skills in Another Project](#4-installing-the-skills-in-another-project)
5. [Using the Skills](#5-using-the-skills)

---

## 1. What is Cursor CLI?

Cursor CLI is Cursor's terminal-native coding agent. The binary is called `cursor-agent`. It runs in your shell, can read and edit files in your working directory, execute shell commands on your behalf, and load project-scoped configuration — including **skills**, the procedural recipes that this repository ships under `./cursor-skills/`. Without Cursor CLI installed, the markdown files in `./cursor-skills/` are just documents; with Cursor CLI installed and pointed at an Android project, they become executable scaffolding for Clean Architecture features, tests, and module bootstraps.

The skills in this repo are deliberately *narrow and procedural* — each one encodes a single, repeatable task (add a ViewModel, add a Repository, add an integration test) and cross-references specific section numbers in `android_clean_architecture.md` and `android_testing.md`. They turn this docs repository into a working code-generation toolkit, exactly the same way the Claude-Code-flavoured `./claude-skills/` directory does — Cursor CLI and Claude Code are two engines that read different on-disk formats but invoke the same procedural intent.

Skills are a first-class concept in Cursor 2.4 and later. They live under `.cursor/skills/<name>/SKILL.md` (project scope) or `~/.cursor/skills/` (user scope), and they activate either by **slash command** (`/<skill-name>`) or by **natural-language description match** — same UX as Claude Code, different on-disk format.

---

## 2. Installing Cursor CLI

### 2.1 System Requirements

| Item | Requirement |
|------|-------------|
| **macOS** | 12 Monterey or later (Intel and Apple Silicon supported). |
| **Linux** | Ubuntu 20.04+ / Debian 11+ / Fedora 36+ / Arch (rolling). x86_64 or arm64. |
| **Windows** | Windows 10 (build 19041+) or Windows 11 via WSL2. The native Windows shell is not the supported path for `cursor-agent`. |
| **Disk** | ~200 MB for the binary, models cache, and working directory. |
| **Network** | Outbound HTTPS to `cursor.com` and `api.cursor.com`. |
| **Node.js** | Not required for the official installer. Some optional MCP servers may require it. |
| **Account** | A Cursor account (Free, Pro, or Business). Sign-in completes in the browser on first launch. |

### 2.2 macOS

```bash
curl https://cursor.com/install -fsS | bash
```

The installer drops `cursor-agent` into `~/.local/bin` and patches your shell profile (`~/.zshrc`) to add that directory to `PATH`. Restart your terminal or run `source ~/.zshrc` afterwards. Apple Silicon vs Intel architecture is detected automatically.

### 2.3 Linux

The same installer works across the major distributions:

```bash
curl https://cursor.com/install -fsS | bash
```

On Debian/Ubuntu it edits `~/.bashrc`; on Fedora and Arch you may need to add `~/.local/bin` to `PATH` yourself if your distribution does not include it by default. Do **not** run the installer with `sudo` — it is a per-user install by design.

If `~/.local/bin` is missing from your `PATH` after installation, append the following to your shell profile and restart the terminal:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 2.4 Windows

The supported path is **WSL2** with an Ubuntu distribution. Install Cursor CLI inside the WSL2 distribution using the Linux instructions above. The WSL2 path interoperates with `adb`, the Android SDK, and Gradle wrappers more cleanly than any native Windows shell, and it keeps your toolchain consistent with Android Studio's bundled JDK.

```bash
# Inside your WSL2 Ubuntu shell:
curl https://cursor.com/install -fsS | bash
```

### 2.5 Verifying the Installation

```bash
cursor-agent --version    # Prints the installed CLI version
cursor-agent --help       # Surfaces the full flag catalogue and built-in slash commands
```

If `cursor-agent --version` is not found, your shell has not picked up the new `PATH` entry — open a fresh terminal window or `source` your shell profile.

### 2.6 Authenticating

The first time you run `cursor-agent` in any directory, it opens your default browser and walks you through sign-in to your Cursor account. The resulting token is cached under `~/.cursor/` and reused by every subsequent session on the same machine. Tokens are **not** synchronised across machines — if you switch laptops, re-authenticate from the new one.

For headless use (CI, scripts), authenticate once interactively on the machine, then invoke `cursor-agent -p "…" --force` non-interactively from the same user account.

---

## 3. The Skills in This Repository

### 3.1 What is a Skill?

A **skill** is a procedural recipe stored as `.cursor/skills/<name>/SKILL.md`, consisting of YAML frontmatter (`name`, `description`, optional `paths`, `disable-model-invocation`) plus a markdown body that tells the agent exactly *how* to perform a single concrete task. When the user's request matches the skill's `description`, or when the user types `/<skill-name>` directly, the agent loads the body and follows it step by step.

Skills are distinct from two adjacent concepts:

- **Rules** (`.cursor/rules/*.mdc`) are *always-on* context — repository conventions or architectural guardrails attached by glob to any matching file. They are the Cursor equivalent of `CLAUDE.md`.
- **Subagents** (`.cursor/agents/<name>.md`) are separate agent instances spawned by a parent skill or by you, each with their own tools and context. This repo ships two reusable subagents under `cursor-skills/.agents/` for per-target fan-out.

Skills sit between rules and subagents: they are *on-demand* (loaded only when the description matches the request or the slash command fires) and *narrow* (one concrete repeatable task per skill).

### 3.2 How the Skills Are Organised

Every skill in this repo is a single `SKILL.md` file inside `./cursor-skills/<skill-name>/` — no auxiliary `references/`, `scripts/`, or `assets/` directories, because these skills are pure procedure. Each one cross-references specific section numbers in `android_clean_architecture.md` and `android_testing.md` so the generated code matches the canonical patterns exactly.

The ten skills divide cleanly into three groups:

- **Scaffolding (1 skill).** `android-create-module` bootstraps the Clean Architecture folder layout.
- **Layer generators (6 skills).** One per architectural layer — HTTP DataSource, Repository, UseCase, ViewModel, Content, Screen — applied in dependency order.
- **Test generators (3 skills).** Unit tests (JVM), UI tests (instrumented, Content + components), Integration tests (instrumented, Screen + NavHost).

Two **subagents** live alongside the skills under `./cursor-skills/.agents/`:

- **`android-feature-target`** — handles one feature inside a multi-feature scaffolding / layer-generator invocation (e.g. *"scaffold modules for userlist, checkout, and orders"*). Dispatched by the scaffolding and layer-generator skills.
- **`android-test-target`** — handles one production file inside a multi-target test-generation invocation (e.g. *"add UI tests for every Content in `:feature:userlist`"*). Dispatched by the test-generation skills.

### 3.3 Skill Catalogue

The table below lists every skill in the order you would typically invoke them while building a new feature — bottom-up through the architecture, then the matching tests at the end.

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

The catalogue is intentionally identical to the Claude Code version (`claude-code.md §3.3`) — same names, same procedural intent, same canonical doc references. The on-disk format and install path differ; the behaviour does not. For per-skill detail blocks (inputs, what each skill intentionally does *not* do, canonical doc references), refer to `claude-code.md §3.3` immediately after the same table — those descriptions apply to both ports of the skills, and the Cursor `SKILL.md` files preserve every section heading from their Claude twins.

### 3.4 Differences from the Claude Code Format

The Cursor SKILL.md files in `./cursor-skills/` are near-mechanical rewrites of the Claude skills in `./claude-skills/`. Three surgical differences:

| Surface | Claude Code | Cursor CLI |
|---------|-------------|-----------|
| **Frontmatter — tool gating** | `allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion` | *Not supported.* Cursor has no per-skill tool gating. Tool restrictions are described in the body (the `Refuse` list and the `Tool names note` at the top of each skill) so the agent self-enforces. |
| **Frontmatter — auto-attach** | *(none)* | `paths:` glob list (`**/*.kt`, `**/build.gradle.kts`, …) — keeps the skill visible when relevant files are open. |
| **Structured input prompt** | Implicit `AskUserQuestion` tool call | Plain-language instruction *"Ask the user for all of the following in a single message; do not proceed until every input is provided"* — Cursor has no structured input tool, so the agent asks inline. |
| **Multi-target fan-out** | `Agent` tool spawning one general-purpose agent per target | `cursor-skills/.agents/android-feature-target` (scaffolding / layer-generator skills) or `cursor-skills/.agents/android-test-target` (test-generation skills) — purpose-built subagents that the parent skill dispatches in parallel. |

Everything else — the doc contract, the canonical Kotlin templates, the `Refuse` lists, the `Parallelism (mandatory)` discipline — is byte-identical between the two ports.

---

## 4. Installing the Skills in Another Project

### 4.1 Scope: Project vs User

Cursor CLI resolves skills from two locations, and both can coexist:

| Scope | Path | Visible to | When to use |
|-------|------|-----------|-------------|
| **Project** | `<other-project>/.cursor/skills/<skill-name>/SKILL.md` | Anyone running Cursor CLI from that project's working directory | The skills should travel with the repo — checked into git, reviewed via PR, versioned alongside the codebase. |
| **User** | `~/.cursor/skills/<skill-name>/SKILL.md` | You only, in every project on this machine | You want the skills available everywhere without committing them to each repo. Personal toolkit. |

When a skill exists in both scopes with the same name, the project copy wins.

Subagents follow the same rule: `<other-project>/.cursor/agents/<name>.md` (project) or `~/.cursor/agents/<name>.md` (user).

### 4.2 Per-Project Installation

The simplest installation is a clone-and-copy. Replace `<owner>/android-development-docs` with the actual GitHub path of your fork or mirror of this repository.

```bash
# Clone this docs repo to a scratch location
git clone https://github.com/<owner>/android-development-docs.git /tmp/android-docs

# Copy the skills into the target Android project
mkdir -p <your-android-project>/.cursor/skills
cp -R /tmp/android-docs/cursor-skills/android-* <your-android-project>/.cursor/skills/

# Copy the subagents alongside the skills
mkdir -p <your-android-project>/.cursor/agents
cp /tmp/android-docs/cursor-skills/.agents/*.md <your-android-project>/.cursor/agents/

# Also copy the canonical reference docs — skills cite them by section number
cp /tmp/android-docs/android_clean_architecture.md <your-android-project>/
cp /tmp/android-docs/android_testing.md           <your-android-project>/

# Commit alongside the project
cd <your-android-project>
git add .cursor/skills .cursor/agents android_clean_architecture.md android_testing.md
git commit -m "Add Android Clean Architecture Cursor skills + canonical references"
```

Why copy the markdown references too: the skills explicitly cite section numbers in `android_clean_architecture.md` and `android_testing.md` (e.g. *"follows §11 *Screen / Content Pattern*"*). Without those files present, the skills still function but will generate slightly less context-aware code. Treat the two markdowns as part of the install bundle.

### 4.3 User-Wide Installation

To make the skills available in every project on your machine:

```bash
git clone https://github.com/<owner>/android-development-docs.git /tmp/android-docs

# Install skills user-wide
mkdir -p ~/.cursor/skills ~/.cursor/agents
cp -R /tmp/android-docs/cursor-skills/android-* ~/.cursor/skills/
cp /tmp/android-docs/cursor-skills/.agents/*.md ~/.cursor/agents/

# Park the canonical references where the skills can find them
mkdir -p ~/.cursor/docs
cp /tmp/android-docs/android_clean_architecture.md ~/.cursor/docs/
cp /tmp/android-docs/android_testing.md           ~/.cursor/docs/
```

To help the skills find those references when invoked from any project, drop a one-line user-scope `~/.cursor/rules/android-references.mdc`:

```markdown
---
description: Pointers to user-wide Android canonical reference docs
alwaysApply: false
---

When working on an Android project, treat `~/.cursor/docs/android_clean_architecture.md`
and `~/.cursor/docs/android_testing.md` as canonical when their section numbers are cited.
```

### 4.4 Git Submodule / Sparse-Checkout Alternatives

For teams that want skills to track upstream automatically rather than be re-copied on each update:

```bash
# Option A — git submodule (pulls full repo into a subdir)
cd <your-android-project>
git submodule add https://github.com/<owner>/android-development-docs.git docs/android-skills
ln -s ../docs/android-skills/cursor-skills .cursor/skills

# Option B — git sparse-checkout (clone only what you need)
git clone --filter=blob:none --no-checkout https://github.com/<owner>/android-development-docs.git
cd android-development-docs
git sparse-checkout init --cone
git sparse-checkout set cursor-skills android_clean_architecture.md android_testing.md
git checkout
```

**Tradeoffs.** Submodules are the cleanest way to stay on the latest version — `git submodule update --remote` is a one-shot upgrade. Sparse-checkout produces a smaller working tree but is heavier to set up and harder to update non-interactively.

### 4.5 Updating the Skills Later

```bash
# Cloned-and-copied (§4.2 or §4.3): re-run the copy after a pull
cd /tmp/android-docs && git pull
cp -R cursor-skills/android-* <target>/.cursor/skills/
cp cursor-skills/.agents/*.md <target>/.cursor/agents/

# Submodule
cd <your-android-project>
git submodule update --remote docs/android-skills
git commit -am "Update Android Cursor skills to latest"
```

### 4.6 Verifying the Skills Are Loaded

From the target project's root directory, launch `cursor-agent` and check that all ten skills surface via slash-command completion:

```bash
cd <your-android-project>
cursor-agent
```

Once inside, start typing `/android-` — completion should list every skill in §3.3. If they do not appear:

1. Confirm you launched `cursor-agent` from the project root, not a subdirectory. Project-scoped skills are only resolved when the current working directory is the project root or a child of it.
2. Verify each `SKILL.md` parses by running `head -10 .cursor/skills/android-add-content/SKILL.md` and checking the YAML frontmatter has `name` and `description` keys.
3. Confirm the subagents are in place: `ls .cursor/agents/android-*-target.md` should list two files.

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

Each skill is explicit about the inputs it needs and asks for everything in a single message up-front (feature name, base URL, dependencies, etc.) rather than question by question.

### 5.2 Natural-Language Invocation

Every skill's `description:` field lists trigger phrases that auto-activate it without typing the slash command. For example, saying *"add a screen for user list"* triggers `android-add-screen`, and *"add an integration test for UserDetailScreen"* triggers `android-add-integration-test`. The skills are tuned aggressively for natural-language match — when in doubt, the slash command guarantees the exact one you want.

The `disable-model-invocation: false` line in each skill's frontmatter is what keeps natural-language activation on. Setting it to `true` would force callers to use the slash command — useful for skills you only want to fire when explicitly requested, but not the default here.

### 5.3 Headless / Scripted Invocation

For CI, scripts, or batch generation, `cursor-agent -p` runs a single prompt non-interactively:

```bash
cursor-agent -p "/android-create-module — feature is userlist, style feature package, base package com.example.app, primary entity User" \
             --force \
             --output-format json
```

Useful flags:

- `-p "…"` — the prompt to run (skill invocation, natural-language request, or both).
- `--force` (a.k.a. `--yolo`) — skip approval prompts for file edits and shell commands. Pair with `--sandbox` if you want a more conservative variant.
- `--output-format text|json|stream-json` — `stream-json` is useful for CI pipelines that consume tool-call events as they happen.
- `--model auto|<id>` — pin a model for reproducibility.
- `--worktree` — run inside an isolated git worktree (recommended for parallel batch runs over the same repo).
- `--resume <session-id>` — pick up a previous session.

### 5.4 Recommended Workflow for a New Feature

The skills are most effective when invoked in dependency order, bottom-up through the architecture, then horizontally across the test pyramid:

1. `/android-create-module` — scaffold the feature folder.
2. `/android-add-http-datasource` — add the network slice (skip if reusing an existing one).
3. `/android-add-repository` — wire data sources behind a domain-layer interface.
4. `/android-add-use-case` — extract business rules (one use case per operation, only when justified per §9).
5. `/android-add-view-model` — design `UiState` + `Event`, inject the use cases.
6. `/android-add-content` — render every state as a pure function of `UiState`.
7. `/android-add-screen` — wire the ViewModel to the Content.
8. `/android-add-unit-test` — cover ViewModel, UseCases, Repository (and DataSource if non-trivial).
9. `/android-add-ui-test` — cover every Content state branch.
10. `/android-add-integration-test` — one Screen test per destination, plus NavHost route tests.

You can also start higher up the stack (e.g. begin with `/android-add-content` and a hand-written `UiState`) when prototyping, then fill in the layers below afterwards.

### 5.5 Troubleshooting

- **"My skill does not trigger when I type its slash command."** Confirm the directory name (`.cursor/skills/<name>/`) matches the `name:` field in the frontmatter exactly — case-sensitive — and that there are no stray characters in the filename (not `SKILL.MD`, not `Skill.md`). Cursor only loads files whose name is exactly `SKILL.md`.
- **"The skill cites a section that does not exist in my project."** Copy `android_clean_architecture.md` and `android_testing.md` into the target project (or symlink them). The skills reference these files by section number and degrade silently when they are missing.
- **"Cursor does not see my `.cursor/skills/` at all."** You probably launched `cursor-agent` from a subdirectory. Skills are resolved relative to the project root — `cd` to the root first, or use the user-wide install in §4.3 to make them path-independent.
- **"I want to disable a skill temporarily."** Either move its directory out of `.cursor/skills/`, or set `disable-model-invocation: true` in its frontmatter (the slash command still works, but description-based auto-activation is suppressed).
- **"A skill dispatched `android-feature-target` / `android-test-target` and the agent could not find it."** Confirm `.cursor/agents/android-feature-target.md` and `.cursor/agents/android-test-target.md` exist in the same project (or user-wide). Skills that fan out depend on these subagents being installed alongside them.
- **"The generated code references an `AppTheme` / `HiltTestActivity` / fake repository that does not exist."** Those are placeholders from the canonical docs. The skill assumes you have an `AppTheme` (per `android_clean_architecture.md` §11), a `HiltTestActivity` (per `android_testing.md` §8), and at least one fake repository (per §6 *Fakes vs Mocks*). Add them once at the project level; every generated file will pick them up.

---

## License

This document is distributed under the **GNU General Public License v3.0 (GPL-3.0)**.

```
Cursor CLI for Android Development Documentation
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

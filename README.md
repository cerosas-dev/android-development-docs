# Android Development Docs

### A curated reference set for professional Android development with Kotlin, Jetpack Compose, and Clean Architecture — plus companion guides for using Claude Code *and* Cursor CLI to scaffold features that match these standards.

## Welcome

Hey there, and thanks for dropping by. Whether you're a first-week Android engineer wondering where ViewModels are supposed to live, a seasoned dev hunting for a refresher on Hilt scopes, or a tech lead onboarding a brand-new team — this repo is for you. Pour yourself a coffee, pick a doc, and dive in. There's no required reading order, no quiz at the end, and no judgment if you CTRL-F your way straight to the Compose section.

This repository collects the docs that an Android engineer at any level can use as a single source of truth: the architecture and patterns to follow, the testing strategy to apply at each layer, and the AI-assisted workflow that ties the two together. Each document stands on its own and is written to be read end-to-end or jumped into via its table of contents.

## Why standards, guidelines, and patterns?

Software is a team sport. The moment two engineers touch the same codebase, every decision either *compounds* — through shared vocabulary, predictable structure, and reviewable diffs — or it *fragments*, leaving everyone to relearn the rules every Monday morning. Standards are how we stop relearning and start shipping.

A good standard does three things at once:

- **It frees up brain cells.** When the shape of a Repository, a Use Case, or a Compose Screen is already decided, you get to spend your thinking on the feature instead of the file layout. Less yak-shaving, more building.
- **It makes onboarding humane.** A new teammate can read one doc, recognise themselves in the codebase by Friday, and contribute meaningfully the week after — instead of pattern-matching across dozens of subtly different examples and quietly losing confidence.
- **It makes code review about the work, not the wallpaper.** When everyone agrees on naming, packaging, and testing conventions, reviews focus on real things — logic, edge cases, user impact — instead of bikeshedding tabs vs spaces for the thousandth time.

Patterns aren't rules to be obeyed; they're **agreements between teammates** so we can move faster, together. They're also a kindness to your future self: the you who returns to a module six months from now should find a friendly, familiar layout — not a treasure map written by a stranger. The docs below are those agreements, written down so they're easier to keep, easier to teach, and easier to evolve when something better comes along.

## Contents

This repo currently covers two mobile platforms and two AI coding-agent engines. The engine guides below apply to both platforms — each one explains how to install the agent and wire the platform-specific skills into any project.

| Document | Description | Sections |
|----------|-------------|----------|
| [`android_clean_architecture.md`](./android_clean_architecture.md) | Reference guide for professional Android development using Kotlin, Jetpack libraries, and industry-standard architectural patterns — MVVM, SOLID, Coroutines, Data Sources, Repositories, Use Cases, ViewModel, Compose (incl. the Screen / Content pattern), OkHttp, Hilt, Room, SharedPreferences, and COIL. | 15 |
| [`android_testing.md`](./android_testing.md) | Reference for testing Android applications — the Testing Pyramid, TDD, stubbing & mocking, the Robot pattern, test fixtures, unit tests, integration tests, UI tests with Jetpack Compose Test (Screen vs Content), and screenshot tests with the Jetpack Screenshot Testing plugin. | 9 |
| [`claude-code.md`](./claude-code.md) | Guide to installing Claude Code on macOS / Linux / Windows, wiring this repository's scaffolding skills into any iOS or Android project, and using them to generate code that adheres to the platform reference docs below. | 5 |
| [`cursor-cli.md`](./cursor-cli.md) | Guide to installing Cursor CLI (`cursor-agent`) on macOS / Linux / WSL2, wiring this repository's Cursor-flavoured skills (under `./cursor-skills/`) into any Android project, and using them to generate code that adheres to the platform reference docs below. | 5 |

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

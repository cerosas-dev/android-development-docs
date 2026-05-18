---
name: android-feature-target
description: Process one feature within a multi-feature Android scaffolding, layer-generator, or ViewModel invocation. The parent skill (android-create-module, android-add-http-datasource, android-add-repository, android-add-use-case, android-add-view-model, android-add-content, android-add-screen) dispatches one of these per feature when the user requests several in a single invocation. Operates on a single feature in isolation; never recurses.
model: auto
readonly: false
is_background: false
---

# Android Feature Target

You are processing a single feature inside a multi-feature Android scaffolding / layer-generator invocation. The parent skill has already:

- Read the canonical doc sections from `android_clean_architecture.md`.
- Verified prerequisites (existing modules, data sources, repositories, use cases, ViewModels, Content composables — as relevant to the parent skill).
- Resolved tool dependencies (Gradle catalog, base abstractions like `UseCase.kt` or `AsyncData.kt`).
- Collected user inputs.

## Your input contract

The parent skill hands you, in its dispatch prompt, exactly:

1. **Feature name** (lowercase, e.g. `userlist`).
2. **Base package** of the consuming module.
3. **Per-skill payload** — whichever subset applies to the parent skill:
   - For `android-create-module`: style (sub-module vs feature package), primary entity (PascalCase).
   - For `android-add-http-datasource`: base URL (with trailing slash), resource, endpoint list, auth flag.
   - For `android-add-repository`: entity, endpoint list, backing stores, reactive flag.
   - For `android-add-use-case`: use case name, base class, repositories, params fields, return type.
   - For `android-add-view-model`: use cases, UiState style, reactive sources, nav args, debounce flag, events flag.
   - For `android-add-content`: UiState type path, event callbacks, layout, state branches, snackbar flag.
   - For `android-add-screen`: Content path, ViewModel path, navigation lambdas, route-wrapper flag.
4. **Pre-written shared paths** (if any) — files the parent already wrote that you must reuse by import, not re-create.

## What you do

1. Execute the parent skill's `Workflow` for **this one feature only**, starting from the `Write` step (steps before that — reads, prerequisite verification, input collection — are parent-skill responsibility and are already complete).
2. Honour the parent skill's `Refuse` list verbatim.
3. Batch independent file writes in a single tool-call group.
4. Produce a short structured report: feature name, files written (paths), compile status, anything skipped with reason.

## You must not

- Recurse into another `android-feature-target` dispatch.
- Touch any file outside `presentation/feature/<feature>/`, `data/...`, `domain/...`, `di/...` belonging to *your* feature.
- Modify shared files (`settings.gradle.kts`, `libs.versions.toml`, shared Hilt modules) — the parent skill owns those and writes them once before fan-out.
- Re-read doc sections already pre-loaded by the parent.
- Ask the user any new questions (the parent collected every input up-front).
- Run the final Gradle verify — the parent skill runs one verify after collecting all subagent reports.

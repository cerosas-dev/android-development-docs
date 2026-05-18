# Android Testing Guide

> A comprehensive reference for testing Android applications using industry-standard tools and practices — covering the testing pyramid, TDD, unit tests, integration tests, UI tests with Jetpack Compose Test, and screenshot tests with the Jetpack Screenshot Testing plugin.

## Table of Contents

1. [The Testing Pyramid](#1-the-testing-pyramid)
2. [Test-Driven Development (TDD)](#2-test-driven-development-tdd)
3. [Unit Tests](#3-unit-tests)
4. [Integration Tests](#4-integration-tests)
5. [UI Tests with Jetpack Compose](#5-ui-tests-with-jetpack-compose)
6. [Screenshot Tests](#6-screenshot-tests)

---

## 1. The Testing Pyramid

### What is the Testing Pyramid?

The Testing Pyramid is a framework for thinking about how many tests to write at each level of abstraction. Coined by Mike Cohn and later adapted for mobile by Google, it arranges tests into three tiers — Unit, Integration, and UI — ordered from fastest/cheapest at the bottom to slowest/most expensive at the top.

```
              ╔══════════════════╗
              ║    UI Tests      ║  ~10% — slow, brittle, expensive
              ║  (End-to-End)    ║
          ╔═══╩══════════════════╩═══╗
          ║   Integration Tests     ║  ~20% — moderate speed and cost
          ║  (Components together)  ║
      ╔═══╩═════════════════════════╩═══╗
      ║          Unit Tests             ║  ~70% — fast, cheap, precise
      ║   (Single class / function)     ║
      ╚═════════════════════════════════╝
```

Each tier has a different purpose, speed, and cost profile. Misbalancing the pyramid — writing too many UI tests and too few unit tests — leads to a slow, unreliable test suite that developers stop trusting.

### Why Follow the Pyramid?

The pyramid reflects an economic reality: **unit tests are orders of magnitude cheaper to write, run, and maintain than UI tests.** A unit test runs in milliseconds on the JVM with no emulator required. A UI test takes seconds (or minutes) and requires an emulator or physical device. A suite of 500 unit tests can run in under 30 seconds; 500 UI tests might take an hour.

The pyramid also reflects **failure locality**. When a unit test fails, you know exactly which class broke. When a UI test fails, it could be caused by any layer in the stack — network, database, ViewModel, or the UI itself.

### When to Write Which Test?

| Test Level | When to write it | Tools |
|-----------|-----------------|-------|
| Unit | For every non-trivial function, class, or business rule | JUnit 4, MockK, Coroutines Test |
| Integration | When two or more components must work together correctly | Hilt Testing, Room in-memory, MockWebServer |
| UI | For critical user journeys and screen-level behaviour | Jetpack Compose Test |
| Screenshot | For UI component visual regressions | Jetpack Screenshot Testing |

> **Note:** The pyramid is a guide, not a law. A pure UI app with no business logic may need proportionally more UI tests. A pure library with no UI may need zero. Apply the rationale, not the percentages mechanically.

### Android-Specific Pyramid

In Android, the tiers map to two test source sets:

| Source set | Runs on | Speed | Examples |
|-----------|---------|-------|---------|
| `src/test/` | JVM only — no Android framework | Milliseconds | Unit tests, use case tests, ViewModel tests |
| `src/androidTest/` | Emulator or real device | Seconds to minutes | UI tests, integration tests with Room/DB |
| `src/screenshotTest/` | JVM (rendered via Compose tooling) | Seconds | Screenshot / visual regression tests |

Always prefer `src/test/` (JVM) over `src/androidTest/` (device) when possible. The fewer tests that require a device, the faster your CI pipeline.

---


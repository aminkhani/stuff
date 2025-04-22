```table-of-contents
title: Table of Contents
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
## 🐞 What Is a Flaky Test?

- **Non‑deterministic behavior** – The test’s outcome isn’t solely determined by the code it’s verifying.
- **Occurs unpredictably** – Runs green in one CI build, red in the next, even though nothing has changed.
- **Hard to reproduce** – Might pass locally but fail on CI, or only break under heavy load.
---
## ⚠️ Why Flaky Tests Are Problematic

1. **Noise in CI**
    - Frequent false alarms make teams ignore failures.
2. **Wasted time**
    - Developers spend precious minutes rerunning builds to “see if it was just a flake.”
3. **Hidden bugs**
    - Real defects can be buried under the noise of flaky failures.
---
## 🔍 Common Causes

|Category|Examples|
|---|---|
|**Timing & Concurrency**|Race conditions, timeouts, reliance on `sleep()`|
|**External Dependencies**|Network calls, databases, file I/O, third‑party APIs|
|**Order Dependency**|Tests that pass only when run in a certain sequence|
|**Shared State**|Global variables, caches, file-system artifacts|
|**Randomness**|Tests using random inputs without seeding the RNG|
|**Resource Limits**|Hitting port collisions, insufficient memory/CPU on CI|

---
## 🛠️ How to Detect Flakiness

1. **Repeated Runs**
```bash
# pytest: run tests 10 times
pytest --maxfail=1 --disable-warnings --count=10
```
2. **CI Analytics**
    - Many CI systems (Jenkins, GitLab, GitHub Actions) can track failure history per test.
3. **Flake‑Detection Tools**
    - e.g. [pytest‑flaky](https://pypi.org/project/pytest%E2%80%91flaky/) to automatically mark and retry flakey tests.
---
## 🧹 Strategies to Fix & Prevent

### 1. Isolate Tests
- Use fixtures to set up/tear down state cleanly.
- Avoid shared files or ports; use temporary directories or random ports.
### 2. Mock External Calls
- Replace real HTTP/database calls with mocks or in‑memory fakes.
### 3. Control Timing
- Don’t rely on arbitrary `sleep()`; instead, poll with timeouts or use virtual clocks.
- Seed random generators:
```bash
import random
random.seed(42)
```
### 4. Enforce Order Independence
- Design each test so it can run in any order.
### 5. Retry Wisely
- Use `pytest-flaky` or built‑in retries, but **only** as a band‑aid while you fix root causes.
### 6. Resource Management
- Clean up sockets, files, threads.
- Use unique names/IDs per test run.
---
### ✅ Key Takeaways

- **Flaky tests** undermine trust—treat them as high‑priority bugs.
- **Identify causes**: timing, external dependencies, shared state, order, randomness.
- **Mitigate** by isolating tests, mocking, controlling timing, and cleaning up resources.
- **Monitor** your suite regularly (CI analytics, repeat runs) to catch new flakiness early.

By rooting out and preventing flaky tests, you’ll keep your pipeline green, your team focused, and your product quality high.

---
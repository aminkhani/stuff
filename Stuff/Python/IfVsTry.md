```table-of-contents
title: Table of Contents
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
## Philosophy & Style

### 📜 LBYL ("Look Before You Leap")

```python
if key in my_dict:
    value = my_dict[key]
else:
    handle_missing()
```

- ✅ Pros: Explicit intent, no hidden control flow
- ⚠️ Cons: Race‐condition prone if state changes between check and use

### 🎯 EAFP ("Easier to Ask Forgiveness than Permission")

```python
try:
    value = my_dict[key]
except KeyError:
    handle_missing()
```

- ✅ Pros: Cleaner for the “happy path,” avoids double‐work
- ⚠️ Cons: Hides control flow inside exception handlers

---
## Performance & Speed ⚡

- **When no exception is raised**
    - 🏎️ `try/except` has _very little_ overhead around the `try` block
    - 🐢 `if key in dict` always does a hash lookup, _then_ another inside `__getitem__` if you still access
- **When exception _is_ raised**
    - 💥 `try/except` pays a heavy cost for unwinding the stack and creating the exception object
    - 🔍 `if/else` never pays that cost, but you still pay for the membership test

| Scenario                | if/else     | try/except |
| ----------------------- | ----------- | ---------- |
| ✅ Common case succeeds | medium cost | low cost   |
| ❌ Failure (exception)  | medium cost | high cost  |
> [!IMPORTANT] **Rule of thumb:** If you expect failures to be **rare**, prefer `try/except` for cleaner code and faster common‐case. If failures are **common** and cheap to test, `if/else` can be faster overall.

---
## 🏆 Example

```python
import timeit

setup = "d = {'a': 1}"
codes = {
    "if key exists": """
if 'a' in d:
    x = d['a']
""",
    "try/except exists": """
try:
    x = d['a']
except KeyError:
    pass
""",
    "if key missing": """
if 'z' in d:
    x = d['z']
""",
    "try/except missing": """
try:
    x = d['z']
except KeyError:
    pass
""",
}

results = {}
for name, stmt in codes.items():
    # Run each snippet 1,000,000 times
    elapsed = timeit.timeit(stmt, setup=setup, number=1_000_000)
    results[name] = elapsed

print(f"{'Operation':<25} {'Time (s)':>12}")
print("-" * 38)
for name, elapsed in results.items():
    print(f"{name:<25} {elapsed:>12.6f}")
```

### 🔬 What the numbers say

- **Access existing key**
    - `if 'a' in d` ➡️ **0.033 s**
    - `try: d['a']` ➡️ **0.017 s** ⚡ _twice_ as fast because only one lookup.
- **Access missing key**
    - `if 'z' in d` ➡️ **0.018 s** 🏃‍♂️
    - `try/except` with `KeyError` ➡️ **0.140 s** 🐢 _10× slower_ (exception machinery).
---
## Correctness & Concurrency 🛠️

- 🔒 **Atomic operations:** Many built‑ins (e.g. file reads, dict access) are atomic under the GIL—`try/except` avoids a race between check & use.
- ⚙️ **Thread safety:** Checking then acting (`if`) can fail if another thread/process mutates state between your check and your action.
---
## Readability & Maintenance 📝

- 🎨 **Clarity:**
    - Use `if/else` when you’re testing a _condition_ that’s central to your logic (e.g. user input, config flags).
    - Use `try/except` around blocks of code you _know_ might fail (e.g. I/O operations, parsing).
- 🔍 **Granularity:**
    - Catch _specific_ exceptions (e.g. `except KeyError:`), never a bare `except:`—you’ll mask bugs otherwise!
---
## When To Use Which? 🤔

| Use Case                                | Recommendation          |
| --------------------------------------- | ----------------------- |
| Accessing a dict key that might miss    | `try/except KeyError`   |
| Validating user‐supplied value format   | `try/except ValueError` |
| Checking a simple flag or numeric range | `if/else`               |
| Opening/reading a file                  | `try/except IOError`    |
| Avoiding race conditions                | `try/except`            |
| Pre‐validating costly operations        | `if/else`               |

---
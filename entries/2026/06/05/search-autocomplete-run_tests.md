# File: search-autocomplete/run_tests.py

**Date:** 2026-06-05
**Time:** 14:01

# `search-autocomplete/run_tests.py`

## Purpose

This is a convenience test runner script — a one-click way to execute the test suite for the search-autocomplete module. It exists so a developer can run `python3 run_tests.py` from the `search-autocomplete/` directory instead of remembering the full pytest invocation. Its only responsibility is to delegate to pytest with the correct test file and exit with pytest's return code.

## Key Components

There are no classes, functions, or constants. The entire module is a top-level script body:

- **Line 3** — The docstring doubles as a usage hint, documenting the equivalent manual command: `python3 -m pytest test_search_autocomplete.py -v`.
- **Line 6** — The single operational line: constructs the test file path by string-replacing `run_tests.py` with `test_search_autocomplete.py` in `__file__`, passes it to `pytest.main()` with `-v` for verbose output, and wraps the result in `sys.exit()` to propagate the exit code.

## Patterns

**Path derivation via string replacement.** Rather than hardcoding an absolute or relative path, it uses `__file__.replace("run_tests.py", "test_search_autocomplete.py")` to locate the test file relative to itself. This is a lightweight sibling-file resolution pattern — it works because both files sit in the same directory and the script's own filename is stable.

**Programmatic pytest invocation.** Calling `pytest.main([...])` runs pytest in-process rather than spawning a subprocess. This is the standard pattern for wrapper scripts — it avoids shell escaping issues and gives direct access to the integer exit code.

**Convention across the repo.** The `url-shortener/` directory has an identical `run_tests.py`. This is a repeated pattern in the repo, not a one-off.

## Dependencies

**Imports:**
- `sys` — for `sys.exit()` to set the process exit code.
- `pytest` — the test framework, invoked programmatically.

**Implicit dependency:**
- `test_search_autocomplete.py` — the actual test module. This file is useless without it.

**Imported by:** Nothing. This is a leaf entry-point script.

## Flow

1. Python loads the script.
2. `__file__` resolves to the script's path (e.g., `/Users/ben/git/sdi-implementations/search-autocomplete/run_tests.py`).
3. `.replace()` swaps the filename portion, producing the path to `test_search_autocomplete.py`.
4. `pytest.main()` discovers and runs all tests in that file with verbose output. It returns an integer exit code (0 = all passed, 1 = failures, 2 = interrupted, etc.).
5. `sys.exit()` terminates the process with that code.

## Invariants

- The script **must** be named `run_tests.py` — the `.replace()` logic depends on it. Renaming the file without updating the replacement string would cause pytest to receive the original filename as the test target, which would fail or run the wrong file.
- `test_search_autocomplete.py` **must** exist in the same directory. If missing, pytest will error with a "file not found" collection error.

## Error Handling

There is none in this file. If pytest can't find or collect the test file, pytest itself will print the error and return a non-zero exit code, which `sys.exit()` faithfully propagates. No exceptions are caught or suppressed.

## Topics to Explore

- [file] `search-autocomplete/test_search_autocomplete.py` — The actual test suite this script runs; understanding the tests is the reason this file exists
- [file] `search-autocomplete/search_autocomplete.py` — The implementation under test; the system being validated
- [file] `url-shortener/run_tests.py` — Identical pattern in another module; confirms this is a repo convention, not a one-off
- [file] `search-autocomplete/plan.md` — Design decisions and scope for the search-autocomplete system
- [general] `pytest-exit-codes` — Understanding what pytest's integer return codes mean (0–5) and how CI systems consume them

## Beliefs

- `run-tests-relies-on-own-filename` — The path derivation uses `__file__.replace("run_tests.py", ...)`, so renaming this script without updating the replacement string breaks test discovery
- `run-tests-is-repo-convention` — At least two modules (`search-autocomplete`, `url-shortener`) use identical `run_tests.py` wrapper scripts, establishing a repo-level pattern
- `run-tests-propagates-exit-code` — `sys.exit(pytest.main(...))` ensures the process exit code matches pytest's result, making this script CI-compatible
- `run-tests-has-no-error-handling` — The script delegates all error reporting to pytest; it catches nothing and adds no fallback behavior


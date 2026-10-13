# 05 — Testing with pytest

Files: `tests/test_app.py`, `pytest.ini`, `requirements-dev.txt`.

## Why automated tests matter in CI/CD

Automated tests are the **quality gate**. If any test fails, the pipeline stops
before an image is pushed or deployed. Without tests, "continuous deployment"
just means "continuously deploying bugs".

## The test pyramid

```
        /\        End-to-end / UI tests   (few, slow, brittle)
       /  \
      /----\      Integration tests       (some)
     /      \
    /--------\    Unit tests              (many, fast, cheap)   ← this project
```

This project has **unit tests** (fast, no network) plus a **smoke test**
(a tiny end-to-end check after deployment).

## `pytest.ini`

```ini
[pytest]
pythonpath = .
testpaths = tests
```
* `pythonpath = .` — adds the project root to Python's import path so
  `from app.app import app` works.
* `testpaths = tests` — only look for tests in `tests/`.

## Test file walkthrough

```python
from app.app import app

def client():
    app.config["TESTING"] = True
    return app.test_client()
```
* Flask's **test client** simulates HTTP requests **in-process** — no server,
  no ports, no network. It is fast and deterministic.
* `TESTING = True` makes Flask propagate exceptions to the test instead of
  hiding them behind a generic 500 page.

| Test | What it asserts | Why it matters |
|------|-----------------|----------------|
| `test_health` | status 200 and JSON exactly `{"status": "ok"}` | Probes depend on this contract |
| `test_version` | status 200 and `"version"` key present | Deploy verification depends on it |
| `test_index` | status 200 and body contains `Deployed by Jenkins` | The dashboard renders |
| `test_unknown_route_returns_404` | `/does-not-exist` → 404 | Negative test: errors behave correctly |

### pytest conventions
* Files named `test_*.py`, functions named `test_*` are auto-discovered.
* Plain `assert` statements are used (pytest rewrites them to show detailed diffs).
* Fixtures (not used here) would be the idiomatic way to share the `client()` helper.

## How tests run in this pipeline: *inside Docker*

The `Dockerfile` has a `test` stage that ends with `RUN pytest -q`.

```
docker build --target test -t cicd-demo:test .
```

Why this is clever:
* **Build fails if tests fail** — a failing `RUN` command aborts the docker build,
  returns non-zero, and Jenkins marks the stage as failed.
* **Clean environment** — no dependency on what is installed on Jenkins.
* **Same environment as production** — same Python version and packages.
* The test image is **throwaway**; Jenkins deletes it in `post { always { ... docker rmi ... } }`.

`-q` means *quiet* (compact output).

## Typical pytest output

```
....                                                       [100%]
4 passed in 0.12s
```

A failure shows the failing assertion with values, which is shown in the Jenkins console log.

## Exit codes — the foundation of automation

Everything in CI keys off **exit codes**: `0` = success, non-zero = failure.
`pytest` exits non-zero on test failure → `RUN` fails → `docker build` fails →
`sh` step fails → Jenkins stage and build fail → `post { failure { ... } }` runs.

## Coverage of this suite (and gaps)

Covered: all 3 routes + a 404. Not covered: concurrency, `APP_VERSION` env
override, failure modes. Good next steps:

```python
def test_version_uses_env(monkeypatch):
    monkeypatch.setenv("APP_VERSION", "1.2.3")   # note: APP_VERSION is read at import time,
    ...                                          # so you'd need to reload the module or refactor.
```

> **Observation:** `APP_VERSION` is read once at import. To unit-test env
> overrides you would refactor to read it inside the function or use `importlib.reload`.

Other tools to add later: `pytest-cov` (coverage), `flake8`/`ruff` (lint),
`black` (format), `bandit` (security lint), `trivy` (image vulnerability scan).

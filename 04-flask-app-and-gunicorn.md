# 04 — The Flask Application and gunicorn

File: `app/app.py` (46 lines). Dependencies: `requirements.txt`
(`Flask==3.0.3`, `gunicorn==22.0.0`).

## What the app is

A **microservice**: a small, single-purpose web service. It exposes three HTTP
endpoints and has no database.

| Endpoint | Method | Returns | Purpose |
|----------|--------|---------|---------|
| `/` | GET | HTML page | Visual dashboard showing version + pod name |
| `/health` | GET | `{"status":"ok"}` with HTTP 200 | Used by Kubernetes probes and the smoke test |
| `/version` | GET | JSON: version, pod, UTC time | Proves *which* version and *which pod* answered |

## Code walkthrough

```python
import os
import socket
from datetime import datetime, timezone
from flask import Flask, jsonify
```
* `os` — read environment variables.
* `socket` — `socket.gethostname()` returns the machine name. **Inside a
  Kubernetes pod the hostname equals the pod name** (e.g. `cicd-demo-7d9c...-abcde`),
  which lets you see load balancing in action.
* `datetime, timezone` — timestamps in UTC (best practice for servers).
* `Flask, jsonify` — the web framework and a helper that builds JSON responses.

```python
app = Flask(__name__)
APP_VERSION = os.getenv("APP_VERSION", "dev")
```
* `app` is the **WSGI application object**. gunicorn imports it as `app.app:app`
  (module `app/app.py`, variable `app`).
* `APP_VERSION` comes from an **environment variable**, default `"dev"`.
  The Dockerfile sets `ENV APP_VERSION=${APP_VERSION}` from a build argument,
  and Jenkins passes `--build-arg APP_VERSION=${IMAGE_TAG}`. So the running app
  literally reports the build number + Git SHA that created it.
  This follows the **12-factor app** rule: *configuration lives in the environment*.

```python
@app.route("/")
def index():
    pod = socket.gethostname()
    return f"""<!doctype html> ... {APP_VERSION} ... {pod} ..."""
```
* `@app.route` is a **decorator** that maps a URL path to a function.
* The function returns an HTML string (Flask wraps it in a 200 response).
* In the f-string, CSS braces are doubled (`{{ }}`) so Python doesn't treat them as placeholders.

```python
@app.route("/health")
def health():
    return jsonify(status="ok"), 200
```
Returns JSON and an explicit status code. A health endpoint should be **cheap,
fast and dependency-light**. (In a real app it might also check the database.)

```python
@app.route("/version")
def version():
    return jsonify(version=APP_VERSION, pod=socket.gethostname(),
                   time=datetime.now(timezone.utc).isoformat())
```

```python
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
Runs Flask's **development server** only when you do `python app/app.py`.
It is **not** used in the container (gunicorn is). `host="0.0.0.0"` means
"listen on all network interfaces" (required inside containers; `127.0.0.1`
would be unreachable from outside the container).

## Why `app/__init__.py` exists (and is empty)

It marks `app/` as a Python **package**, so `from app.app import app` works in
tests and `app.app:app` works for gunicorn.

## REST, HTTP and JSON basics

* **HTTP status codes**: `200` OK, `404` Not Found, `500` Server Error. Kubernetes
  probes treat 200–399 as success.
* **JSON** is the standard machine-readable response format.
* **Idempotent GET**: calling `/health` repeatedly changes nothing.

## WSGI and why gunicorn

**WSGI** (Web Server Gateway Interface) is the Python standard that connects a
web server to a Python web app.

```
Client ──HTTP──► gunicorn (WSGI server) ──calls──► Flask app object
```

Flask's built-in server is single-purpose for development. **gunicorn** is a
production WSGI server that offers multiple worker processes, process
supervision and better stability.

The container start command:

```
gunicorn -b 0.0.0.0:5000 -w 2 app.app:app
```

| Part | Meaning |
|------|---------|
| `-b 0.0.0.0:5000` | Bind to all interfaces on port 5000 |
| `-w 2` | **2 worker processes** (so each pod can handle 2 requests concurrently) |
| `app.app:app` | `module_path:variable` → file `app/app.py`, object `app` |

Rule of thumb for workers: `(2 × CPU cores) + 1`, but here the pod is limited to
`250m` CPU, so 2 is sensible.

## Dependency pinning

`Flask==3.0.3` (exact version) rather than `Flask>=3` ensures **reproducible
builds**: the same Dockerfile produces the same dependencies today and next
month. The trade-off: you must update versions deliberately (use Dependabot/Renovate).

`requirements-dev.txt` starts with `-r requirements.txt` (include runtime deps)
and adds `pytest==8.3.2`. Splitting runtime vs dev dependencies keeps test
tooling **out of the production image**.

## Running locally without Docker (optional)

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
pytest -q
python app/app.py        # dev server → http://localhost:5000
# or the production-style server:
gunicorn -b 0.0.0.0:5000 -w 2 app.app:app
```

## Ideas to extend the app (practice)

* Add `/ready` separate from `/health` (readiness vs liveness semantics).
* Add structured JSON logging.
* Add Prometheus metrics at `/metrics`.
* Read a `GREETING` env var from a Kubernetes **ConfigMap**.

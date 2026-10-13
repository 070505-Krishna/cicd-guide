# 15 — Scripts, Makefile and Shell Concepts

## Why wrap commands in scripts?

Typing 10 commands in the right order is error-prone. Scripts and a Makefile
make the workflow **repeatable, documented and one-command**.

## Bash safety header

All scripts start with `#!/usr/bin/env bash` (the **shebang**: which interpreter to use) and:

```bash
set -euo pipefail
```
| Flag | Effect |
|------|--------|
| `-e` | Exit immediately if any command fails |
| `-u` | Treat unset variables as errors |
| `-o pipefail` | A pipeline fails if **any** stage fails (not just the last) |

`teardown.sh` uses `set -uo pipefail` **without `-e`** on purpose: cleanup should
keep going even if one step (e.g., nothing to delete) fails.

`cd "$(dirname "$0")/.."` — change to the repo root regardless of where the script
was launched (`$0` = script path; `dirname` strips the filename; `..` goes up from `scripts/`).

## `scripts/local-test.sh` — test + build + probe, no Jenkins/K8s

```bash
docker build --target test    -t cicd-demo:test  .   # run unit tests
docker build --target runtime -t cicd-demo:local .   # production image
docker rm -f cicd-demo-smoke >/dev/null 2>&1 || true # remove leftovers
docker run -d --name cicd-demo-smoke -p 5000:5000 cicd-demo:local >/dev/null
trap 'docker rm -f cicd-demo-smoke >/dev/null 2>&1 || true' EXIT
for i in $(seq 1 15); do curl -fs localhost:5000/health >/dev/null && break; sleep 1; done
echo ">> /health  -> $(curl -fs localhost:5000/health)"
echo ">> /version -> $(curl -fs localhost:5000/version)"
echo "PASS"
```
Concepts:
* **`>/dev/null 2>&1`** — discard stdout and stderr (`2>&1` = send stderr to where stdout goes).
* **`|| true`** — ignore failure (the container may not exist).
* **`trap '...' EXIT`** — run cleanup when the script ends, **even on error**.
* **Readiness loop** — poll up to 15 times, 1 second apart, `break` as soon as `/health` answers. Containers need a moment to boot; never assume instant readiness.
* **`curl -fs`** — fail on HTTP errors, silent.
* `$(...)` — command substitution.
* Because of `set -e`, if `/health` never comes up, the later `curl` fails and the script exits non-zero (no "PASS").

It mirrors what the pipeline does, so you can validate before pushing.

## `scripts/setup.sh` — one-command environment

Step by step:

1. **Pre-flight checks**
   ```bash
   for tool in docker kind kubectl; do
     command -v "$tool" >/dev/null || { echo "ERROR: '$tool' is not installed"; exit 1; }
   done
   docker info >/dev/null 2>&1 || { echo "ERROR: Docker daemon is not running"; exit 1; }
   ```
   `command -v` checks if a program exists. Failing early with a clear message is good UX.
2. **Idempotent cluster creation**
   ```bash
   if kind get clusters | grep -qx "$CLUSTER"; then echo "already exists"
   else kind create cluster --name "$CLUSTER" --config kind/kind-config.yaml; fi
   ```
   `grep -qx` = quiet, whole-line match. Safe to rerun.
3. **Write kubeconfig** with the internal address → `./kubeconfig`.
4. **Start Jenkins**:
   * if `jenkins/.env` exists → use both compose files (JCasC, fully automatic), `AUTO=1`
   * else → plain `docker compose up` (manual wizard), `AUTO=0`
5. `docker network connect kind jenkins 2>/dev/null || true` — attach Jenkins to the kind network (ignore "already connected").
6. **Print instructions**. In manual mode it waits for and prints the initial admin password:
   ```bash
   until docker exec jenkins test -f /var/jenkins_home/secrets/initialAdminPassword 2>/dev/null; do sleep 2; done
   ```
   (`until cmd; do …; done` loops until the command succeeds.)

## `scripts/teardown.sh`

```bash
(cd jenkins && docker compose -f docker-compose.yml down -v 2>/dev/null)
kind delete cluster --name cicd 2>/dev/null
rm -f kubeconfig k8s/deployment.rendered.yaml
```
* Parentheses `( … )` run in a **subshell**, so the `cd` doesn't affect the rest of the script.
* Removes Jenkins + its volume, the cluster, and generated files.

## `Makefile`

`make` runs named **targets**. Each recipe line must start with a **TAB**.

```make
IMAGE ?= cicd-demo:local     # ?= means "set only if not already set" → override: make build IMAGE=foo
CLUSTER ?= cicd
.PHONY: help test build run ...   # these names are commands, not files
```

| Target | Does |
|--------|------|
| `help` | Lists targets |
| `test` | `docker build --target test -t cicd-demo:test .` |
| `build` | Build runtime image `$(IMAGE)` |
| `run` | Depends on `build`, then `docker run --rm -p 5000:5000 $(IMAGE)` |
| `local-test` | `./scripts/local-test.sh` |
| `cluster` / `jenkins` | `./scripts/setup.sh` (both names, one recipe) |
| `deploy-local` | Build → `kind load` → apply manifests with `sed` → rollout status → **no Jenkins, no registry** |
| `status` | `kubectl -n cicd-demo get pods,svc` |
| `logs` | `kubectl -n cicd-demo logs -l app=cicd-demo --tail=50` |
| `clean` | `./scripts/teardown.sh` |

Concepts:
* **`.PHONY`** — prevents confusion if a file named `test` exists.
* **Prerequisites** — `run: build` means "build runs first".
* **`$(VAR)`** — Make variable; **`$$`** would be a literal `$` for the shell.
* **`@echo`** — the `@` hides the command itself, printing only output.
* In `deploy-local`: `sed "s|DOCKER_IMAGE_PLACEHOLDER|$(IMAGE)|" k8s/deployment.yaml | kubectl apply -f -` — pipes the rendered manifest straight into `kubectl` (`-f -` reads stdin).
* On Windows use PowerShell commands from the README or run Make inside WSL2.

## Windows vs Linux paths in the README

| Task | PowerShell | Bash |
|------|-----------|------|
| Probe endpoint | `Invoke-RestMethod http://localhost:5000/health` | `curl -fs localhost:5000/health` |
| Remove file | `Remove-Item kubeconfig -ErrorAction SilentlyContinue` | `rm -f kubeconfig` |
| Loop | `1..4 \| ForEach-Object { ... }` | `for i in $(seq 4); do ...; done` |

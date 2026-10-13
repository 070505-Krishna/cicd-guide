# 09 — Jenkins Fundamentals

## What Jenkins is

Jenkins is an open-source **automation server** written in Java. It runs jobs
(builds, tests, deployments) automatically in response to triggers. It is the
orchestrator in this project.

## Architecture

```
┌────────────────────┐        ┌──────────────┐
│ Controller (master)│───────►│ Agent / node │   (runs the actual build steps)
│ - web UI :8080     │        └──────────────┘
│ - job config       │        ┌──────────────┐
│ - scheduling       │───────►│ Agent        │
│ - plugin system    │        └──────────────┘
└────────────────────┘
        ▲ port 50000 is for inbound agents
```

* **Controller**: UI, configuration, scheduling.
* **Agent**: machine/container that executes work. **Executor** = one build slot on a node.
* In this demo there are **no separate agents**: the controller runs builds itself
  (`agent any`, `numExecutors: 2` in JCasC). Port `50000` is exposed for future agents but isn't needed.

Best practice in production: keep **0 executors on the controller** and run
builds on ephemeral agents (containers/pods).

## Jobs and job types

| Type | Description |
|------|-------------|
| Freestyle | Click-configured steps (legacy style) |
| **Pipeline** | Defined by a script (**Jenkinsfile**) — used here |
| Multibranch Pipeline | Auto-creates a pipeline per Git branch/PR |

This project's job `cicd-demo` is a **Pipeline job from SCM**: Jenkins clones
the repo and reads `Jenkinsfile` from it (`scriptPath('Jenkinsfile')`).

## Declarative vs Scripted Pipeline

| | Declarative (used here) | Scripted |
|---|---|---|
| Syntax | Structured: `pipeline { agent … stages { … } post { … } }` | Free-form Groovy |
| Learning curve | Easier, opinionated | Flexible, harder |
| Validation | Syntax checked up-front | Runtime |

Inside declarative pipelines, a `script { ... }` block lets you drop into
Groovy for logic (used for computing `IMAGE_TAG`).

## Plugins

Jenkins' functionality comes from plugins. The project installs these in
`jenkins/Dockerfile`:

| Plugin | Why it's needed |
|--------|-----------------|
| `git` | Checkout from Git |
| `workflow-aggregator` | The bundle that enables Pipeline (Jenkinsfile) |
| `credentials-binding` | `withCredentials { }` |
| `plain-credentials` | Secret file / secret text credential types |
| `pipeline-stage-view` | Visual stage grid in the UI |
| `timestamper` | Powers `timestamps()` option in logs |
| `configuration-as-code` | JCasC — configure Jenkins from YAML |
| `job-dsl` | `pipelineJob('cicd-demo') { ... }` seeding in JCasC |

Installed with `jenkins-plugin-cli --plugins ...` at image build time, so the image is self-contained and reproducible.

## Credentials

Jenkins has an encrypted **credentials store**. Pipelines refer to them by **ID**,
never by value:

| Credential ID | Type | Holds |
|---------------|------|-------|
| `dockerhub-creds` | Username with password | Docker Hub username + PAT |
| `kubeconfig` | Secret file | Kubernetes connection file |

`withCredentials([...])` temporarily exposes them as environment variables for the
enclosed steps only, and Jenkins **masks** secret values in console output (shows `****`).

Key point: IDs must match **exactly** (`dockerhub-creds`, `kubeconfig`), or you'll get
"credentials not found".

## Triggers

* **`pollSCM`** — Jenkins polls Git on a schedule (used).
* **Webhook** — Git host notifies Jenkins.
* **`cron`** — time-based.
* **Manual** — "Build Now" in UI, or the REST API.
* **Upstream** — after another job.

The README's PowerShell "Option B" shows the REST API: authenticate with Basic
auth, fetch a **crumb** from `/crumbIssuer/api/json` (a CSRF protection token),
add it as a header, then `POST /job/cicd-demo/build`. Jenkins requires crumbs on
POST requests to stop cross-site request forgery.

## Security model in the demo

JCasC config:
```yaml
securityRealm:  local: { allowsSignup: false, users: [admin] }
authorizationStrategy: loggedInUsersCanDoAnything: { allowAnonymousRead: false }
```
* Local user database; **no self-signup**.
* Any logged-in user is effectively admin (fine for a demo; use role-based
  strategy / matrix authorization in real life).
* `runSetupWizard=false` skips the first-run wizard.

## Build numbers, workspace, environment variables

* `BUILD_NUMBER` — auto-incrementing integer per job (used in the tag).
* **Workspace** — the directory where Jenkins checks out the repo for the build.
* Built-in env vars: `BUILD_NUMBER`, `JOB_NAME`, `WORKSPACE`, etc.
* `disableConcurrentBuilds()` guarantees one run at a time.

## Where state lives

`/var/jenkins_home` holds jobs, builds, plugins, secrets. It is mounted as the
Docker **named volume** `jenkins_home` so it survives container restarts.
`docker compose down -v` deletes it.

## Jenkins vs alternatives

| Tool | Hosted by | Pipeline file | Notes |
|------|-----------|---------------|-------|
| Jenkins | You | `Jenkinsfile` | Most flexible; you operate it |
| GitHub Actions | GitHub | `.github/workflows/*.yml` | Zero infrastructure for GitHub repos |
| GitLab CI | GitLab | `.gitlab-ci.yml` | Integrated with GitLab |
| CircleCI | SaaS | `.circleci/config.yml` | Fast, SaaS |

Concepts transfer directly: triggers → jobs → steps → secrets → artifacts.

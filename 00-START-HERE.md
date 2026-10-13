# CI/CD Demo (Jenkins + Docker + Kubernetes) — Complete Study Guide

This guide explains **every file and every concept** in the repository
`cicd-demo-jenkins` (a fork of `likhith07-en/cicd-demo-jenkins`).

The project is small, but it touches almost every core DevOps idea:
version control, automated testing, containers, registries, pipelines-as-code,
Kubernetes deployments, health checks, rollbacks and secrets handling.

## How to read this guide

| # | File | What you learn |
|---|------|----------------|
| 01 | `01-project-overview-and-architecture.md` | The big picture, file map, and the end-to-end flow |
| 02 | `02-cicd-concepts.md` | CI vs CD vs CD, pipelines, stages, artifacts, deployment strategies |
| 03 | `03-git-and-github.md` | Git basics, branches, forks, polling vs webhooks, `.gitignore` |
| 04 | `04-flask-app-and-gunicorn.md` | `app/app.py` line by line, REST endpoints, WSGI, gunicorn |
| 05 | `05-testing-with-pytest.md` | `tests/test_app.py`, `pytest.ini`, test client, why tests gate builds |
| 06 | `06-docker-fundamentals.md` | Images, containers, layers, networking, volumes, the Docker socket |
| 07 | `07-dockerfile-multistage-deep-dive.md` | The project's multi-stage `Dockerfile` and `.dockerignore` |
| 08 | `08-registries-and-image-tagging.md` | Docker Hub, tags, `latest`, immutable tags, PATs |
| 09 | `09-jenkins-fundamentals.md` | Controller, agents, jobs, plugins, credentials, triggers |
| 10 | `10-jenkinsfile-walkthrough.md` | Every stage and directive of the `Jenkinsfile` |
| 11 | `11-jenkins-container-setup-and-jcasc.md` | `jenkins/` folder: Dockerfile, Compose, JCasC, `.env` |
| 12 | `12-kubernetes-fundamentals.md` | Cluster architecture, Pods, Deployments, Services, namespaces |
| 13 | `13-k8s-manifests-walkthrough.md` | `k8s/*.yaml` line by line, probes, resources, rolling updates, HPA, Ingress |
| 14 | `14-kind-and-networking.md` | kind, port mapping, kubeconfig, the `kind` Docker network |
| 15 | `15-scripts-makefile-and-shell.md` | `scripts/*.sh`, `Makefile`, bash safety flags |
| 16 | `16-security-and-best-practices.md` | Secrets, non-root, least privilege, what to harden in production |
| 17 | `17-troubleshooting-and-debugging.md` | Every symptom in `docs/TROUBLESHOOTING.md` explained + commands |
| 18 | `18-glossary-qa-and-exercises.md` | Glossary, interview questions, hands-on exercises, next steps |

## Suggested learning path

1. Read **01** and **02** to get the mental model.
2. Read **04 → 05 → 06 → 07 → 08** (the "build" half: app, tests, container, registry).
3. Read **09 → 10 → 11** (the automation engine: Jenkins).
4. Read **12 → 13 → 14** (the "run" half: Kubernetes).
5. Finish with **15 → 18** for operations, security and practice.

## A note on accuracy

All explanations are based on the actual repository contents. Where the
project has a limitation or a subtle behaviour (for example, when the automatic
rollback does and does not fire), it is called out in a **"Observation"** box so
you understand real-world trade-offs rather than only the happy path.

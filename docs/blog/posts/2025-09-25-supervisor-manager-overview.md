---
title: "Supervisor Manager: One-Command Python App Deployment"
date: 2025-09-25
authors:
  - isra
categories:
  - Tooling
tags:
  - python
  - devops
  - supervisor
  - deployment
  - tooling
---

# Supervisor Manager: One-Command Python App Deployment

A Python tool that wraps [Supervisor](http://supervisord.org/) with templates for Flask, Streamlit, Django, FastAPI, and background services — so deploying a new app is a single command instead of hand-written config.

<!-- more -->

## Context

I run a few Python apps on a homelab server — mixed Flask APIs, Streamlit dashboards, Telegram bots, background workers. Supervisor is the right tool to keep them alive and restart on crash, but hand-rolling `*.conf` files, log paths, env vars, and start scripts for each app is tedious and error-prone.

Supervisor Manager generates that config from a small spec, so I can deploy a new app with one command and have it managed, logged, and restart-on-failure from day one.

---

## What It Does

- Single-command deploy for Python apps (`deploy <name> <path> <type> <port>`)
- Built-in config templates for Flask, Streamlit, Django, FastAPI, background services, Telegram bots
- Lifecycle ops: start / stop / restart / status
- Auto-generates shell start scripts and supervisor config
- Log rotation configured per app
- Interactive CLI for browsing / managing running apps
- Batch deploy multiple apps from a single manifest

---

## Architecture

Three main components:

| Component | Role |
|---|---|
| `SupervisorManager` | Central controller — orchestrates generate / deploy / lifecycle ops |
| `AppConfig` | Dataclass holding per-app settings (name, port, path, env, type) |
| `SupervisorTemplates` | Framework-specific config blueprints (Flask, Streamlit, FastAPI, …) |

The separation means adding a new framework = adding a new template class. The manager and config layer don't change.

---

## Supported App Types

| Type | Example |
|---|---|
| Flask | REST APIs, internal tools |
| FastAPI | Async APIs, ML inference endpoints |
| Django | Full web apps |
| Streamlit | Internal dashboards |
| Background service | Long-running workers, cron-adjacent jobs |
| Telegram bot | Long-polling bot processes |

---

## Quick Start

### CLI

```bash
# Deploy a Flask app on port 5000
supervisor-manager deploy my-api /home/user/apps/my-api flask 5000

# Deploy a Streamlit dashboard
supervisor-manager deploy dashboard /home/user/apps/dashboard streamlit 8501

# Lifecycle
supervisor-manager start my-api
supervisor-manager stop my-api
supervisor-manager restart my-api
supervisor-manager status
```

### Python API

```python
from supervisor_manager import SupervisorManager, AppConfig

mgr = SupervisorManager()

mgr.deploy(AppConfig(
    name="my-api",
    path="/home/user/apps/my-api",
    app_type="flask",
    port=5000,
    env={"FLASK_ENV": "production"},
))

mgr.start("my-api")
```

---

## What Gets Generated

When you deploy an app, Supervisor Manager creates:

```
/home/user/apps/my-api/
├── start.sh               # generated launcher
├── logs/
│   ├── my-api.out.log
│   └── my-api.err.log
└── (your app files)

/etc/supervisor/conf.d/
└── my-api.conf            # supervisor config
```

The generated `start.sh` activates the virtualenv, exports env vars, and runs the app with the right entry point for its type (e.g., `gunicorn` for Flask, `streamlit run` for Streamlit).

The `my-api.conf` sets:

- `autostart=true`, `autorestart=true`
- `stdout_logfile` / `stderr_logfile` pointing to the app's `logs/` dir
- `stdout_logfile_maxbytes=10MB`, `stdout_logfile_backups=5` (rotation)
- `environment=` block from the `AppConfig.env` dict

---

## Why This Over Hand-Rolled Configs

- **Zero config errors** — you can't mistype `autorestart=Treu` if you never write the file.
- **Consistency** — every app gets the same log path shape, rotation, and restart policy, so operational commands work the same for every app.
- **Framework-aware defaults** — Streamlit apps get `--server.port` wired correctly, Flask apps get a sensible Gunicorn worker count.
- **Batch deploy** — spin up 10 apps from one manifest when moving to a new server.
- **Env var hygiene** — secrets go into the supervisor config, not committed shell scripts.

---

## When Not to Use This

- **If you're on Kubernetes** — use Deployments + ConfigMaps instead. Supervisor Manager is for single-host (laptop / VPS / homelab) deployments.
- **If you need complex orchestration** — dependent startup order, health-check-gated rollouts, etc. Supervisor has basic ordering via `priority` but isn't a full init system.
- **If your apps are ephemeral** — for short-lived scripts, use cron or a task queue.

---

## Key Takeaways

- Supervisor is great; writing its config by hand is not — automate the boilerplate.
- A 6-line `AppConfig` beats a 40-line hand-edited `.conf` file every time.
- Keep the template layer separate from the orchestration layer so new frameworks drop in cleanly.
- For homelab / single-host Python workloads, Supervisor + a thin wrapper hits a sweet spot between "just run it" and "full Kubernetes".

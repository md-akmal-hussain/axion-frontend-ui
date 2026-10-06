# Axion UI

Frontend for the **Axion Intelligence Platform**: an agentic AI predictive-insights dashboard for fleet summary, device diagnostics, asset hierarchy, system topology, alarms and events, and correlation analysis.

Built with **Vite + TypeScript** and deployed automatically to an **Ubuntu VM on Azure**, served by **Nginx**, through an **Azure DevOps** CI/CD pipeline.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Getting Started (Local)](#getting-started-local)
- [CI/CD Pipeline](#cicd-pipeline)
- [Azure DevOps Setup](#azure-devops-setup)
- [VM Prerequisites](#vm-prerequisites)
- [Deployment Flow](#deployment-flow)
- [Rollback](#rollback)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
Developer ──git push──► Azure DevOps Repo
                              │ trigger (develop / main)
                              ▼
        ┌──────────────────────────────────────────┐
        │ Stage 1: Build   (Windows agent)         │
        │ Node 20 → npm install → npm run build    │
        │ dist/ ──► Pipeline Artifact              │
        └──────────────────────────────────────────┘
                              │
                              ▼
        ┌──────────────────────────────────────────┐
        │ Stage 2: Deploy  (Windows agent)         │
        │ Download artifact                        │
        │ SFTP  ──► VM:/tmp/axion-ui               │
        │ SSH   ──► install/verify Nginx, publish  │
        └──────────────────────────────────────────┘
                              │
                              ▼
        Ubuntu VM: /var/www/html ──► Nginx ──► Browser
```

The build output is a set of **static files**. Nginx only serves them; all dashboard data is fetched by the browser from the backend API at runtime.

## Tech Stack

| Layer      | Technology                              |
|------------|-----------------------------------------|
| Frontend   | Vite, TypeScript                        |
| CI/CD      | Azure DevOps YAML Pipelines             |
| Build host | Self-hosted Windows agent (`Yaml_Pool`) |
| Transport  | SSH / SFTP (service connection)         |
| Web server | Nginx on Ubuntu                         |
| Cloud      | Microsoft Azure (VM)                    |

## Getting Started (Local)

**Requirements:** Node.js 20.x and npm.

```bash
# install dependencies
npm install

# start dev server
npm run dev

# type-check and production build (output in dist/)
npm run build
```

## CI/CD Pipeline

Defined in [`azure-pipelines.yml`](./azure-pipelines.yml).

| Item           | Value                              |
|----------------|------------------------------------|
| Trigger        | Push to `develop` or `main`        |
| Agent pool     | `Yaml_Pool` (self-hosted, Windows) |
| Node version   | `20.x`                             |
| Artifact name  | `axion-ui-frontend`                |
| SSH connection | `ssh-svc`                          |
| Nginx web root | `/var/www/html`                    |

### Stage 1: Build Frontend

1. Install Node.js 20.
2. `npm install` and `npm run build` (`tsc -b && vite build`).
3. Publish `dist/` as the pipeline artifact `axion-ui-frontend`.

### Stage 2: Deploy Frontend

1. Download the artifact.
2. Copy files to the VM at `/tmp/axion-ui` using `CopyFilesOverSSH`.
3. Run deploy commands on the VM using `SSH@0`:
   - Install Nginx if it is missing and enable it.
   - Replace the contents of `/var/www/html` with the new build.
   - Set ownership to `www-data` and permissions to `755`.
   - Validate config with `nginx -t`, then reload Nginx.
   - Remove temporary files.

> **Why a separate artifact?** The exact build that was produced is the one that gets deployed, and any previous artifact can be re-deployed without rebuilding.

## Azure DevOps Setup

1. **Agent pool:** register a self-hosted agent in a pool named `Yaml_Pool`.
2. **SSH service connection:** go to *Project Settings → Service connections → New → SSH* and create a connection named `ssh-svc` with the VM's host, port `22`, username, and password or private key.
3. **Pipeline:** create a pipeline from the repo's `azure-pipelines.yml`.

> Credentials live only in the service connection. They are never stored in the repository.

## VM Prerequisites

- Ubuntu VM reachable over SSH (port 22) from the build agent.
- The SSH user must have **passwordless `sudo`**.
- Azure **NSG inbound rules** allowing HTTP `80` (and HTTPS `443` if used).
- Internet access from the VM to install Nginx via `apt`, on the first deploy only.

### SPA routing (recommended)

If the app uses client-side routes, add this to the Nginx `location /` block so a page refresh does not return 404:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Then run:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## Deployment Flow

```
git push → pipeline starts
        → Build: npm install + build → artifact published
        → Deploy: artifact → VM:/tmp/axion-ui → /var/www/html
        → nginx -t → reload → live
```

## Rollback

The pipeline does not keep an on-VM backup. To roll back, open a previous successful pipeline run and re-run its **Deploy** stage, which re-deploys that run's artifact.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `NPM Install` finishes in about 1 second and `tsc is not recognized` appears in the build | On Windows agents `npm` is `npm.cmd`; without `call`, the batch script stops after it | Prefix every npm command with `call` and check `errorlevel` (already done in the pipeline) |
| `sudo: nginx: command not found` | Nginx is not installed on the VM | The pipeline installs it automatically if missing |
| Site does not open in the browser | Port 80 is blocked | Open inbound `80` in the VM's NSG |
| Refreshing a route returns 404 | SPA fallback is missing | Add the `try_files` rule shown above |
| Dashboard shows `0` assets and an empty map | UI is deployed but cannot reach the backend API | Check the API base URL used at build time (Vite env variables are baked in during build), CORS, and the browser's Network tab |
| `apt` lock errors on a new VM | `unattended-upgrades` holds the lock | The pipeline waits for the lock with `DPkg::Lock::Timeout=300` |

---

**Maintainer:** DevOps Team

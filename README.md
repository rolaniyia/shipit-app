# ShipIt App

A minimal Python web app that demonstrates a complete **containerised CI/CD pipeline**: build with Docker, publish to GitHub Container Registry (GHCR), and deploy to Azure Container Apps automatically on every push to `main`.

The app itself is deliberately tiny — it serves a single HTML page over HTTP — because the real point of this repository is the automation: code committed to GitHub ships itself to the cloud with zero manual steps.

---

## Table of Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Run locally with Python](#run-locally-with-python)
  - [Run locally with Docker](#run-locally-with-docker)
- [How the pipeline works](#how-the-pipeline-works)
- [Deployment target](#deployment-target)
- [Configuration](#configuration)
  - [Environment variables](#environment-variables)
  - [GitHub Actions secrets](#github-actions-secrets)
- [Contributing](#contributing)
- [License](#license)

---

## What it does

The application listens on a configurable TCP port (default `8080`) and responds to every `GET` request with a small HTML page:

> **Shipped automatically by GitHub Actions!**
> Built with Docker, shipped to Azure, automated with GitHub Actions.

It is built entirely on Python's standard library — no framework, no third-party runtime dependencies.

## Architecture

```text
┌──────────────┐   git push   ┌─────────────────┐   build & push   ┌─────────────────────────┐
│  Developer   │ ───────────▶ │  GitHub Actions │ ───────────────▶ │  GHCR (ghcr.io/...)     │
└──────────────┘              │  (deploy.yml)   │                  └────────────┬────────────┘
                              └────────┬────────┘                               │ docker pull
                                       │ az containerapp update                  │
                                       ▼                                        ▼
                              ┌──────────────────────────────────────────────────────┐
                              │  Azure Container Apps  (shipit-app / shipit-rg)      │
                              └──────────────────────────────────────────────────────┘
```

1. A developer pushes to the `main` branch.
2. [.github/workflows/deploy.yml](.github/workflows/deploy.yml) builds the Docker image from [Dockerfile](Dockerfile).
3. The image is pushed to the repository's GitHub Container Registry (GHCR).
4. The workflow authenticates to Azure and points the existing Container App at the new image with `az containerapp update`.
5. Azure Container Apps pulls the new image and rolls it out.

## Tech stack

| Layer        | Technology                                   |
| ------------ | -------------------------------------------- |
| App          | Python 3.12 (stdlib `http.server`)           |
| Container    | Docker, `python:3.12-slim` base image        |
| CI/CD        | GitHub Actions                               |
| Registry     | GitHub Container Registry (GHCR)             |
| Hosting      | Azure Container Apps                         |
| Auth         | Azure service principal (OIDC-free login)    |

## Repository structure

```text
shipit-app/
├── .github/
│   └── workflows/
│       └── deploy.yml   # CI/CD pipeline: build → push → deploy
├── app.py               # Python HTTP server (the whole app)
├── Dockerfile           # Container definition
└── README.md            # This file
```

## Getting started

### Prerequisites

- **Python 3.12+** (to run the app natively)
- **Docker** (to build and run the container)
- **Azure CLI** and an **Azure Container Apps** environment (only if you want to deploy yourself)

### Run locally with Python

```bash
# Start the server on the default port 8080
python3 app.py

# Or choose a different port
PORT=3000 python3 app.py
```

Verify it's up:

```bash
curl http://localhost:8080
```

### Run locally with Docker

```bash
# Build the image
docker build -t shipit-app .

# Run it, mapping port 8080 on your machine to the container
docker run --rm -p 8080:8080 shipit-app
```

Verify it's up:

```bash
curl http://localhost:8080
```

## How the pipeline works

The single workflow in [.github/workflows/deploy.yml](.github/workflows/deploy.yml) runs on:

- **`push`** to the `main` branch, and
- **manual runs** via `workflow_dispatch`.

It performs three steps:

1. **Build & push the image**
   Logs in to GHCR and builds the image, tagging it with the short commit SHA:

   ```text
   ghcr.io/<owner>/shipit-app:<short-sha>
   ```

2. **Log in to Azure**
   Authenticates using a service principal stored in the `AZURE_CREDENTIALS` secret.

3. **Deploy to Azure Container Apps**
   Updates the `shipit-app` Container App in the `shipit-rg` resource group to use the freshly built image:

   ```bash
   az containerapp update \
     --name shipit-app \
     --resource-group shipit-rg \
     --image ghcr.io/<owner>/shipit-app:<short-sha>
   ```

The workflow declares `packages: write` permission so it can publish images to GHCR.

## Deployment target

The app is deployed to an Azure Container App named **`shipit-app`** inside the resource group **`shipit-rg`**. The target Container App must already exist — this workflow updates an existing app rather than provisioning one from scratch.

## Configuration

### Environment variables

| Variable | Default | Purpose                                       |
| -------- | ------- | --------------------------------------------- |
| `PORT`   | `8080`  | TCP port the HTTP server binds to             |

### GitHub Actions secrets

| Secret               | Purpose                                                              |
| -------------------- | -------------------------------------------------------------------- |
| `AZURE_CREDENTIALS`  | JSON credentials for an Azure service principal (used by `azure/login`) |
| `GITHUB_TOKEN`       | Provided automatically; used to authenticate to GHCR                 |

To create `AZURE_CREDENTIALS`, register a service principal with `Contributor` access on the `shipit-rg` resource group and store its output JSON as a repository secret.

## Contributing

Contributions are welcome. Please open an issue to discuss larger changes before submitting a pull request, and keep commits focused and well-described.

## License

No license is currently specified. All rights are reserved by the repository owner unless stated otherwise.

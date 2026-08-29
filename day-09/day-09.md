# Day 09 — Trivy Image Vulnerability Scanning

This document covers adding **Trivy** to the CI pipeline — scanning both the backend and frontend Docker images for known vulnerabilities *before* they're pushed to GitHub Container Registry.

## Overview

Up to Day 08, the pipeline built and pushed images without ever checking what's actually inside them. A base image (`python:3.12-slim`, `postgres:16`, etc.) or a pinned dependency can carry known CVEs without anyone noticing — until Day 09.

**What changed:**
- Each build job (`build-backend`, `build-frontend`) now builds the image **locally first** (not pushed yet), scans it with Trivy, and only pushes to GHCR if the scan passes.
- Trivy is configured to fail the job (`exit-code: 1`) on any **CRITICAL** or **HIGH** severity vulnerability — a vulnerable image never reaches the registry.
- This is a classic **SAST-adjacent security gate**: image scanning happens statically, before the image is ever deployed or run against real traffic.

---

## `.github/workflows/ci.yml`

```yaml
name: CI - DB, Backend, Frontend with Trivy Scanning

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  BACKEND_IMAGE: ${{ github.repository }}/trackforge-backend
  FRONTEND_IMAGE: ${{ github.repository }}/trackforge-frontend

jobs:
  db-check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: trackforge
          POSTGRES_PASSWORD: trackforge
          POSTGRES_DB: trackforge
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U trackforge -d trackforge"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Verify Postgres is reachable
        run: |
          sudo apt-get install -y postgresql-client
          pg_isready -h localhost -p 5432 -U trackforge -d trackforge

  build-backend:
    needs: db-check
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build backend image (local, for scanning)
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: false
          load: true
          tags: ${{ env.REGISTRY }}/${{ env.BACKEND_IMAGE }}:${{ github.sha }}

      - name: Scan backend image with Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.BACKEND_IMAGE }}:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push backend image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.BACKEND_IMAGE }}:latest
            ${{ env.REGISTRY }}/${{ env.BACKEND_IMAGE }}:${{ github.sha }}

  build-frontend:
    needs: build-backend
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build frontend image (local, for scanning)
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          file: ./frontend/Dockerfile
          push: false
          load: true
          tags: ${{ env.REGISTRY }}/${{ env.FRONTEND_IMAGE }}:${{ github.sha }}

      - name: Scan frontend image with Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.FRONTEND_IMAGE }}:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push frontend image
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          file: ./frontend/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.FRONTEND_IMAGE }}:latest
            ${{ env.REGISTRY }}/${{ env.FRONTEND_IMAGE }}:${{ github.sha }}
```

---

## Key concept: build → scan → push

Each build job now has three distinct stages instead of one:

1. **Build locally** (`push: false, load: true`) — the image is built and loaded into the runner's local Docker daemon, but not sent anywhere yet.
2. **Scan** — Trivy inspects the local image's OS packages and language dependencies against its vulnerability database, looking for known CVEs.
3. **Push** — only runs if the scan step didn't fail. The image is rebuilt (Docker's layer cache makes this near-instant) and pushed to GHCR.

If Trivy finds a CRITICAL or HIGH severity vulnerability, `exit-code: 1` fails the step immediately — the push step never runs, and the job (and everything downstream via `needs:`) is marked failed.

---

## Commands — CI (Day 09)

### 1. Edit the workflow file
```bash
nano .github/workflows/ci.yml
```

### 2. Commit and push to trigger the pipeline
```bash
git add .
git commit -m "Day 9: add Trivy vulnerability scanning before image push"
git push origin main
```

### 3. Watch the run
Go to the repo's **Actions** tab — each build job now shows a "Scan ... image with Trivy" step with a full vulnerability table in the logs (package, installed version, fixed version, severity).

### 4. Run Trivy locally against an image (optional, for debugging before pushing)
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image trackforge-backend:latest
```

---

## What's next

The pipeline now checks DB connectivity, lints, tests build order, and scans for vulnerabilities before anything ships. Future days will build on this:
- Automated tests (pytest) between `db-check` and the build jobs
- Move toward CD — auto-deploy once both images pass scanning and are pushed
- Tune Trivy's severity threshold / add a `.trivyignore` for accepted risks

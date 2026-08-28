# Day 08 — Backend CI Job Depends on DB Check

This document covers extending the sequential CI chain from Day 07 by adding a **`db-check`** job that `build-backend` now depends on — so the pipeline is: `db-check` → `build-backend` → `build-frontend`.

## Overview

Day 07 made `build-frontend` wait on `build-backend`. Day 08 adds one more link at the front of the chain: before the backend image is even built, CI spins up a **temporary Postgres service container** and verifies it's reachable. Only once that succeeds does `build-backend` run.

**Why this matters:**
- Mirrors the real dependency in `docker-compose.yml`, where `backend` has `depends_on: db: condition: service_healthy` — the CI pipeline now enforces the same ordering as production.
- Catches a broken Postgres image, bad connection config, or credentials mismatch *before* wasting time building and pushing a backend image that would fail to connect anyway.
- `db-check` doesn't touch the real `trackforge-db` — GitHub Actions spins up an **ephemeral** Postgres container scoped to that job only, using the same credentials as local dev for a realistic check.

---

## `.github/workflows/ci.yml`

```yaml
name: CI - DB then Backend then Frontend

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

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push backend image
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

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push frontend image
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

## Key concept: GitHub Actions `services:`

```yaml
db-check:
  services:
    postgres:
      image: postgres:16
      ...
      ports:
        - 5432:5432
```

- `services:` on a job spins up a **sidecar container** alongside the job's runner, available for the duration of that job only.
- It's destroyed automatically when the job finishes — nothing persists, nothing to clean up.
- The runner can reach it on `localhost:5432` because GitHub Actions runs job steps directly on the runner VM, with service containers port-mapped to it (this is different from how containers reach each other by *name* inside `docker-compose.yml` — there's no shared user-defined network here).
- The `options:` health check (`pg_isready`) ensures the step doesn't proceed until Postgres has actually finished starting up, not just been scheduled.

---

## The full dependency chain (Day 07 → Day 08)

```
db-check  →  build-backend  →  build-frontend
```

- If `db-check` fails, both `build-backend` and `build-frontend` are skipped.
- If `db-check` passes but `build-backend` fails, `build-frontend` is skipped.
- Only a fully green chain results in both images being pushed to GHCR.

---

## Commands — CI (Day 08)

### 1. Edit the workflow file
```bash
nano .github/workflows/ci.yml
```

### 2. Commit and push to trigger the pipeline
```bash
git add .github/workflows/ci.yml
git commit -m "Day 8: add db-check job that build-backend depends on"
git push origin main
```

### 3. Watch the run
Go to the repo's **Actions** tab — `db-check` runs first (spinning up its own Postgres), then `build-backend`, then `build-frontend`, each waiting on the one before it.

---

## Suggested Project Structure

```
day-08/
└── .github/
    └── workflows/
        └── ci.yml
```

---

## What's next

The pipeline now checks DB connectivity, lints, builds, and orders everything correctly. Future days will build on this:
- Add automated tests as a job between `db-check` and `build-backend`, running against the ephemeral Postgres
- Add image vulnerability scanning (Trivy) before push
- Move toward CD — auto-deploy once both images are successfully pushed

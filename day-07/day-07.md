# Day 07 — Sequential CI Jobs (Frontend depends on Backend)

This document covers restructuring the CI workflow so the **frontend build job only runs after the backend build job succeeds**, instead of both running independently/in parallel.

## Overview

Up to Day 6, the pipeline had `lint` → `build-and-push` (one job building both images). Day 07 splits the build step into two separate jobs — `build-backend` and `build-frontend` — with an explicit dependency between them using GitHub Actions' `needs:` keyword.

**Why this matters:**
- By default, GitHub Actions jobs run **in parallel** unless told otherwise.
- Making `build-frontend` depend on `build-backend` enforces a strict order: backend builds and pushes first, and frontend only starts once that succeeds.
- If `build-backend` fails, `build-frontend` is **skipped entirely** — no wasted build time on a frontend image when the backend is already broken.
- This mirrors a real-world scenario where the frontend's build or tests might need a healthy/published backend image to run against.

---

## `.github/workflows/ci.yml`

```yaml
name: CI - Backend then Frontend

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
  build-backend:
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

## Key concept: `needs:`

```yaml
build-frontend:
  needs: build-backend
```

- Without `needs:`, both jobs would start at the same time (parallel), since they don't share state by default.
- With `needs: build-backend`, GitHub Actions builds a dependency graph: `build-frontend` waits in a queued state until `build-backend` finishes.
- If `build-backend` **fails or is cancelled**, `build-frontend` is automatically **skipped** (shown greyed out in the Actions UI) — it does not run at all.
- `needs:` can also take a list (`needs: [job1, job2]`) if a job depends on multiple prior jobs.

---

## Commands — CI (Day 07)

### 1. Edit the workflow file
```bash
nano .github/workflows/ci.yml
```

### 2. Commit and push to trigger the pipeline
```bash
git add .github/workflows/ci.yml
git commit -m "Day 7: make frontend CI job depend on backend job"
git push origin main
```

### 3. Watch the run
Go to the repo's **Actions** tab on GitHub — `build-backend` runs first; `build-frontend` stays queued until it completes, then runs.

### 4. Trigger manually without a push (optional)
Since the workflow has `workflow_dispatch:`, it can also be triggered manually from the Actions tab → "Run workflow".

---

## Suggested Project Structure

```
day-07/
└── .github/
    └── workflows/
        └── ci.yml
```

---

## What's next

The pipeline now enforces a build order. Future days will build on this:
- Add automated tests as a job between lint and build
- Add image vulnerability scanning (e.g. Trivy) before push
- Move toward CD — auto-deploy once both images are successfully pushed

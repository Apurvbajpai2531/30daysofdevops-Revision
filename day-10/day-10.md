# Day 10 — Automated Tests (pytest) in CI

This document covers adding a **`test`** job to the CI pipeline — running the backend's pytest suite against a live Postgres instance, right after `db-check` and before any image gets built.

## Overview

The pipeline could already prove Postgres was reachable (Day 08) and that images were vulnerability-free (Day 09) — but it never proved the **application code actually works**. Day 10 closes that gap.

**What changed:**
- New `test` job, positioned between `db-check` and `build-backend`.
- Spins up its own Postgres service (same pattern as `db-check`), installs backend dependencies, and runs `pytest` against real endpoints.
- `build-backend` now depends on `test` passing — a failing test blocks every build and push downstream, the same way a failing lint or scan already did.

**New pipeline chain:**
```
db-check → test → build-backend (build→scan→push) → build-frontend (build→scan→push)
```

---

## `.github/workflows/ci.yml` (relevant new job)

```yaml
  test:
    needs: db-check
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
    env:
      DATABASE_URL: postgresql://trackforge:trackforge@localhost:5432/trackforge
      SECRET_KEY: test-secret-key
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install backend dependencies
        run: |
          cd backend
          pip install -r requirements.txt
          pip install pytest httpx

      - name: Run pytest suite
        run: |
          cd backend
          pytest -v
```

`build-backend` gained `needs: test` in place of `needs: db-check`.

---

## Sample test file — `backend/tests/test_smoke.py`

```python
"""
Basic smoke tests for the TrackForge backend API.
"""
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)


def test_app_starts():
    """Sanity check — the app object loads without error."""
    assert app is not None


def test_register_and_login_flow():
    """Register a user, then log in with the same credentials."""
    register_payload = {
        "username": "citestuser",
        "email": "citestuser@example.com",
        "password": "TestPassword123",
    }
    register_resp = client.post("/api/auth/register", json=register_payload)
    assert register_resp.status_code in (200, 201, 400)  # 400 if already exists from a prior run

    login_resp = client.post(
        "/api/auth/login",
        json={"email": "citestuser@example.com", "password": "TestPassword123"},
    )
    assert login_resp.status_code == 200
    assert "access_token" in login_resp.json() or "token" in login_resp.json()
```

> Adjust import paths and payload shapes to match your actual `app/main.py`, router paths, and schema fields if they differ.

---

## Key concept: why a separate `test` job instead of adding steps to `build-backend`

- **Isolation** — the test job gets its own fresh Postgres service, independent of whatever `db-check` was using. No shared state between jobs.
- **Speed** — `db-check`, `test`, and later stages can be reasoned about (and eventually parallelized where it makes sense) as distinct units of work with clear pass/fail signals.
- **Clarity in the Actions UI** — a failing test shows up as a red `test` job specifically, not buried inside a `build-backend` job that's also trying to build Docker images.

---

## Commands — CI (Day 10)

### 1. Add the test file locally
```bash
mkdir -p backend/tests
nano backend/tests/test_smoke.py
```

### 2. Run tests locally before pushing (requires local Postgres running)
```bash
cd backend
pip install pytest httpx
pytest -v
```
### 4. Commit and push to trigger the pipeline
```bash
git add backend/tests/ .github/workflows/ci.yml
git commit -m "Day 10: add pytest test job to CI pipeline"
git push origin main
```

### 5. Watch the run
Go to the repo's **Actions** tab — `db-check` runs, then `test` (with its own Postgres), then `build-backend` only starts if both passed.
---

## What's next

The pipeline now verifies DB connectivity, application behavior, and image safety before anything ships. Future days will build on this:
- Test coverage reporting (`pytest-cov`) with an enforced minimum threshold
- Move toward CD — auto-deploy once the full chain passes

# Day 01 — Dockerizing Backend & Frontend

This document covers the multi-stage `Dockerfile`s used to containerize the **backend** and **frontend** services for Day 01.

## Overview

Both services use a **multi-stage build**:

1. **Stage 1 (`builder`)** — installs Python dependencies into an isolated user directory (`--user`), using the full `python:3.12` image (has build tools available).
2. **Stage 2 (`runtime`)** — copies only the installed packages from the builder stage into a lightweight `python:3.12-slim` image, keeping the final image small.

This pattern avoids shipping compilers/build dependencies in the final image, reducing size and attack surface.

---

## Backend — `backend/Dockerfile`

```dockerfile
# ---------- Stage 1: builder ----------
FROM python:3.12 AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim

WORKDIR /app

COPY --from=builder /root/.local /root/.local

ENV PATH=/root/.local/bin:$PATH

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

**What it does:**
- Installs dependencies from `requirements.txt` in the builder stage.
- Copies compiled/installed packages (`/root/.local`) into the slim runtime image.
- Adds `/root/.local/bin` to `PATH` so installed console scripts are usable.
- Copies application source code.
- Exposes port `8000`.
- Runs the app with `python app.py`.

### Build & Run
```bash
cd backend
docker build -t day01-backend .
docker run -p 8000:8000 day01-backend
```

---

## Frontend — `frontend/Dockerfile`

```dockerfile
# ---------- Stage 1: builder ----------
FROM python:3.12 AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ---------- Stage 2: runtime ----------
FROM python:3.12-slim

WORKDIR /app

COPY --from=builder /root/.local /root/.local

ENV PATH=/root/.local/bin:$PATH

COPY . .

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**What it does:**
- Same multi-stage pattern as the backend.
- Runs the app with **Uvicorn**, serving a FastAPI/ASGI app located at `app/main.py` (the `app` object inside it).
- Binds to `0.0.0.0:8000` so it's reachable from outside the container.

> Note: despite the name "frontend," this Dockerfile is actually running a Python ASGI service (FastAPI/Uvicorn), not a JS-based UI (React/Vue/etc). If your frontend is actually a Node-based app, it needs its own `Dockerfile` (e.g. `node:20-alpine` build → static file server or `npm run start`). Keeping this file as-is only makes sense if this "frontend" is itself a Python API/service layer.

### Build & Run
```bash
cd frontend
docker build -t day01-frontend .
docker run -p 8001:8000 day01-frontend
```
*(mapped to host port `8001` so it doesn't collide with the backend's `8000`)*

---

## Fixes Applied From the Draft

The original snippets had a couple of small issues, cleaned up above:

| Issue | Fix |
|---|---|
| `RUN echo "Stage 1 is succeed"` / `RUN echo "Stage 2 is succeed"` debug lines | Removed — these were just build-log markers, not needed in production Dockerfiles |
| `RUN` instruction placed **after** `CMD` | Removed — `CMD` should be the last instruction; anything after it still executes at build time but is confusing and non-idiomatic |
| Both stages had identical labels/structure copy-pasted | Kept structure but separated into two files, one per service |

---

## Suggested Project Structure

```
day-01/
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
├── frontend/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       └── main.py
└── README.md
```


